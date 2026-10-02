## 背景

对 `recrowdship` 项目的 ALNS 算法做寻优能力审计,目标是提高跳出局部最优的能力和寻优深度。项目为 crowdsourced transshipment CVRP,核心在 `alns/alns.py` 主循环 + `alns/operators/{destroy,repair}.py` 算子池。

## 诊断:七个瓶颈

| 瓶颈 | 具体表现 | 影响 |
|------|---------|------|
| 接受准则单一 | 仅 SA,温度到 `min_temp=0.01` 后 $\exp(-\Delta/T)\approx 0$ | 后期退化为贪心,丧失跳出能力 |
| 重启策略单一 | 停滞只从 best 重启 + reheat | 反复在同一邻域打转 |
| 破坏规模上限低 | `max_removal_fraction` 默认 0.2 = `removal_fraction` | 自适应增长被禁用 |
| 无接受率反馈 | 破坏规模/温度只看停滞迭代数 | 搜索状态感知粗糙 |
| 修复贪心偏置 | 6 个 repair 均为 regret/greedy 变体 | 修复方向单一 |
| 无独立局部搜索 | 无定期 2-opt/or-opt | 缺深度开采 |
| 无 I/D 周期 | 全程一个节奏 | 缺开采/探索交替 |

## 五项改进

### A. LAHC 混合接受准则

Late Acceptance Hill Climbing 维护长度 $L$ 的历史成本环,接受条件:

$$\text{accept} \iff f(s_{\text{cand}}) \le \max(\text{history}) \;\lor\; f(s_{\text{cand}}) < f(s_{\text{cur}})$$

- `--acceptance lahc`:全程 LAHC
- `--acceptance hybrid`:前半程 SA,后半程切换 LAHC,兼顾前期发散与后期跳出

### B. 接受率自适应破坏规模

滑动窗口(默认 100)跟踪接受率 $\rho$:

$$q = \begin{cases} \lfloor 0.7\,q_{\text{base}} \rfloor & \rho > 0.3 \text{ (开发)} \\ \lfloor 1.5\,q_{\text{base}} \rfloor & \rho < 0.1 \text{ (探索)} \\ q_{\text{base}} & \text{otherwise} \end{cases}$$

参数:`--adaptive-removal --accept-window 100`

### C. 分级停滞响应

将单一重启扩展为三级(阈值 $\theta$ = `stagnation_reheat`):

| 级别 | 触发 | 动作 |
|------|------|------|
| 软 | 停滞 $\ge \theta$ | 小 reheat($0.2\,T_0$),留在 current |
| 中 | 停滞 $\ge 2\theta$ | 回 best,中 reheat($0.5\,T_0$) |
| 硬 | 停滞 $\ge 3\theta$ | 回 best,大 reheat($0.8\,T_0$)+ 重置算子权重 |

参数:`--tiered-stagnation --stagnation-reheat 25`

### D. 周期性局部搜索

对 best 的 truck routes 做 2-opt(段反转)+ or-opt(段重定位),每次移动通过四重可行性门控:

1. `compute_route_schedule`(时间窗)
2. `crowd_routes_feasible`(crowd 同步)
3. `truck_load`(容量)
4. `_truck_replenishment_load_feasible`(replenishment 负载轨迹)

first-improvement 策略,`alns.py` 对结果加 `check_solution` 防御层。参数:`--local-search-interval 25 --local-search-on-best`

### E. 噪声修复算子

在 regret 值上叠加均匀噪声:

$$r' = r + U(-\eta,\eta)\cdot\max(1, |c_0|)$$

使插入顺序非确定性贪心,多样化修复轨迹。已注册进 `REPAIR_OPS`,默认 $\eta=0.15$。

## 改动文件

- `alns/operators/repair.py`:`fast_regret_insertion` 加 `regret_noise` 参数;新增 `local_search_2opt`、`local_search_or_opt`、`intensify_local_search`、`noisy_regret_repair`
- `alns/alns.py`:`run` 加 7 个参数,主循环集成 A/B/C/D;`REPAIR_OPS` 注册噪声修复
- `main.py`:暴露 `--acceptance`、`--lahc-list-length`、`--adaptive-removal`、`--accept-window`、`--tiered-stagnation`、`--local-search-interval`、`--local-search-on-best`

## 关键修复

初版局部搜索漏检 replenishment 模式的 `_truck_replenishment_load_feasible`,2-opt 重排节点后负载轨迹可能违反,而 `solution_objective` 仍算出更低成本,导致 best 变不可行。已补检查 + `check_solution` 防御层。

## 验证结果

`validation_30_replenishment.json`,200 迭代,seed 1,同参数(`--stagnation-reheat 10 --max-removal-fraction 0.4`):

| 配置 | 目标 F | 运行时间 | 可行 |
|------|--------|---------|------|
| 基线(新功能关) | 594.811 | 62.8s | 是 |
| 增强(全开) | 523.275 | 65.6s | 是 |

改进 12.0%,开销约 +4.5%。delivery 模式(`instances/test_delivery_50.json`,200 迭代)亦验证可行:393.513,8 direct + 42 crowd。

## 使用命令

```bash
# 生成实例
python generate_instance.py --customers 50 --trucks 4 --crowd 15 \
  --crowd-mode delivery --seed 42 --output instances/test_delivery_50.json

# 增强求解
python main.py --instance instances/test_delivery_50.json --iterations 500 --seed 1 \
  --acceptance hybrid --adaptive-removal --tiered-stagnation \
  --stagnation-reheat 25 --max-removal-fraction 0.4 \
  --local-search-interval 25 --local-search-on-best
```

## 设计原则

- 所有增强默认关闭,不破坏现有实验复现性
- 每项可独立消融(如只开 `--acceptance lahc` 验证 LAHC 贡献)
- 建议正式实验用相同 seed 跑各组合,归因各组件贡献