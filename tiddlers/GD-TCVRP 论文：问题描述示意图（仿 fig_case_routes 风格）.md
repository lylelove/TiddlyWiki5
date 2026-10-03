## 任务
在 `grading_driven_truck_drone_vrp.tex` 第 2.1 节「问题描述」新增一幅示意图，视觉风格仿照 `fig:case_routes`（实验部分的固定算例路径结构图）。

## 决策
原问题描述小节已有一幅手写 TikZ 图 `fig:gd-tdvrp-problem`（3 个卡车客户 + 2 个无人机客户、无等级信息、无图例）。用户选择用 **Python/matplotlib 生成**，替换该 TikZ 图。

## 产物
- `algorithm/fig_problem_schematic.py`：独立绘图脚本，只依赖 matplotlib，不依赖实验求解器。
- `algorithm/output/fig_gdtvrp_problem.pdf` / `.png`（300 DPI）。
- `algorithm/README.md` 的「论文产物的复算脚本」表格已登记该脚本。

## 图的结构（双栏）
- **左图 (a) 路径结构示意**：配送中心 0；卡车道路弧（实线，#4e79a7）；无人机发射--客户--回收架次（虚线，#2a9d8f），发射/回收节点加圆圈标记；客户节点颜色=质量等级（沿用 `fig_case_routes` 的 grade_colors），形状=运输方式（圆点=卡车、三角=无人机）；高等级节点旁加窄时间窗括线，低等级加宽时间窗括线。
- **右图 (b) 分级进入模型的四条通道**：表格化列出等级 → 价值权重、衰减乘数、无人机资格概率（条形）、时间窗上限，并箭头指向「共同决定路径变量、目标权重与可行性；高等级 ≠ 必然用无人机」。

配色与等级参数直接对齐 `gccr_experiment.py` 的 GRADE_WEIGHT / GRADE_DECAY_MULTIPLIER / eligibility_probability / GRADE_WINDOW，保持全篇一致。

## LaTeX 集成
- `\label` 由 `fig:gd-tdvrp-problem` 改名为 `fig:gdtvrp-problem`；确认旧 label 在 .tex 源文件中无其他引用（仅存在于陈旧的 .aux 构建产物中，可安全忽略）。
- 在图前补一段引出文字，解释左右两图含义。
- `python algorithm/_tex_check.py` 通过：73 labels / 73 refs、无未引用 label、花括号与数学模式平衡、环境平衡。

## 环境备忘
本机 PowerShell 工具曾因工作区根目录缺少当前用户权限而无法启动（`SetNamedSecurityInfoW failed (Win32 5)`），用 `diagnose-windows-sandbox-acl` 技能脚本补齐权限后恢复。权限备份在 `.acl-recovery/`。
