## 背景

论文 `D:\github\craft\different_class`（卡车—无人机协同路径优化，中文 ctexart + XeLaTeX，数学建模竞赛稿）。用户要求：在 `main.tex` 第 2 节「问题描述」增加一张 **tex 画的图**，学术风格、黑白两色、图上不要过多说明性文字。

## 交付内容

- `sections/section2.tex`：在 §2.1「系统构成与运作流程」开头（`\subsection{系统构成与运作流程}` 之后、`\subsubsection{网络结构}` 之前）插入 `figure[htbp]`，label 为 `fig:problem`，并在该段末尾加了「三者的耦合关系见图~\ref{fig:problem}。」
- 两张子图并排（`minipage` 0.45 / 0.52 `\textwidth`），全 TikZ 纯代码，无外部图片文件（无需 `\includegraphics`，不受 `\graphicspath` 影响）：
  - **(a) 集货网络与两类协同模式**：集货中心 `0`（黑方块）、卡车路线（实线带箭头）、无人机航迹（虚线带箭头）。`i` = 随车模式采摘点（无人机在卡车路线上起飞/回收），`j` = 直飞模式采摘点（自集货中心往返）。2 条图例。
  - **(b) 抵达品质与阶跃售价**：横轴 $q^{\mathrm{arr}}$、纵轴「单位售价」；4 级阶跃售价曲线 + 虚线「连续腐损」光滑参照曲线；阈值 $\theta_3<\theta_2<\theta_1$；实心● $q_i^{\mathrm{T}}$（卡车）与空心○ $q_i^{\mathrm{D}}$（无人机）跨 $\theta_2$ 两侧；双向箭头 $w_i\Pi_{2}$ = 挽回的价格落差。2 条图例。

## 关键决策与约定

- **黑白**：只用 `black`/`white`、实线/虚线/点线、实心/空心圆区分语义，不引入任何颜色。
- **图上文字极少**：子图内只有坐标轴名、阈值符号、点位符号、2 行图例；所有解释性内容放进 `\caption`（题注不受「图上」约束，承担全部细节）。
- **字号**：`font=\scriptsize`，整图 `minipage` 包裹。
- **宏包位置**：`tikz` + `\usetikzlibrary{arrows.meta}` 同时加进 `main.tex` 与 `section2.tex` 的**独立编译前言**（该节有 `\ifdefined\maintex\else` 双前言结构，两处都要加，否则单文件编译失败）。

## 编号影响（需知悉）

插入后 §2 该图成为 **图 1**，§5 原有 9 张图整体后移一位（`fig:cont` 1→2 … `fig:showcase-dash` 9→10）。全文交叉引用由 LaTeX 自动解析，无需手改；`\ref` 全部正确。

> 备选方案：用 `\setcounter{figure}{0}` 重新播种可让 §5 编号不变，但会产生重复的「图 1」且 `\ref` 指向错乱，**不推荐**。

## 环境坑（重要）

- **`latexmk` 在本机全局损坏**：MiKTeX 自 2026-09-23 起因 `latexmk.log` 日志权限问题报 `MiKTeX encountered an internal error`，连 `latexmk --version` 都失败（与应用目录无关，是机器级问题）。
  → **改用 `xelatex -aux-directory=build -interaction=nonstopmode main.tex`**，连跑 3 遍收敛。该仓库 `latexmkrc` 设定 `$aux_dir='build'`、`$out_dir='.'`，所以 `-aux-directory=build` 能复现同样的产物布局。
- **`Select-Object -First N` 会早关管道**，把 xelatex 打断成只跑到第 4 页的假象；日志筛选要用 `*> file` 先落盘再筛。
- `build/` 整体被 `.gitignore` 忽略，仅 `main.pdf` 入库。

## 验证结论

- 81 页（基线 80 页，+1 页为插图与题注）。
- Overfull/Underfull 警告集合与 **HEAD 原始基线逐条完全一致**（14 条，`Compare-Object` 零差异）——新图未引入任何新警告。
- `sections/section2.tex` 单文件编译：**零** Overfull/Underfull 警告。
- 保住了文件原有 CRLF 行尾与无 BOM（编辑工具 `edit` 已验证不改行尾）。

## 可复用经验

- 该仓库 `sections/*.tex` 是「独立可编译草稿」，每节自带 `\ifdefined\maintex\else` 前言；**加宏包必须同时改 `main.tex` 和该节前言两处**，否则单文件编译会挂。
- 编译器坏了先看 `$env:LOCALAPPDATA\MiKTeX\miktex\log\*.log`，别怀疑自己的 `.tex`。

## 追加：§2.2 / §2.3 / §2.4 各新增一张同类图（2026-10-01）

用户要求「在第二节里再添加几个类似的说明图片」，并确认**每小节一张、共 3 张**，风格与 `fig:problem` 完全一致；明确**只改 .tex、不编译**。

现在 §2 共 4 张图，均为纯 TikZ、黑白、图内文字极少、细节全放 `\caption`：

| 位置 | label | 面板 (a) | 面板 (b) |
| --- | --- | --- | --- |
| §2.1 | `fig:problem` | 集货网络与两类协同模式 | 抵达品质与阶跃售价 |
| §2.2 | `fig:kinetics` | 两种运输方式的货舱温度剖面（$T^{\mathrm{amb}}/T^{\mathrm{T}}/T^{\mathrm{D}}$ 三条点线导引 + 卡车实线降温 + 无人机虚线升温） | 暴露量的分段累积与装卸阶跃（折线斜率 = 速率常数，竖直阶跃 = $\Delta_{\mathrm{M}}$） |
| §2.3 | `fig:grade` | 价值曲线的导数结构（阶跃售价 vs 虚线连续腐损参照，标 $\Pi_1..\Pi_3$） | 等级下降损失沿暴露量的累积（横轴 $\tau_i$，实心●卡车 / 空心○无人机的实现位置，两位置间台阶之和 = `eq:criterion` 左端第一项求和范围） |
| §2.4 | `fig:critical` | 边际保鲜价值的阈值集中（脉冲只出现在 $\theta_a$，高度 $w_i\Pi_a$；区间内恒零故无曲线段） | 按降级时间裕量 $\sigma_i$ 的三类批次划分（无望 / 临界 / 充裕，并标 T/D 推荐方式） |

### 本轮新增的关键约定

- **样式名避开 TikZ 内置键**：`step`、`mark`、`pos`、`ref` 都是 TikZ 自带键，用作自定义 `.style` 名有被误解析的风险，已分别改名为 `val`、`cont`、`jump`、`marker`、`bnd`。写新图时不要用这些名字。
- **不新增任何宏包/库**：三张图只用 `fig:problem` 已验证可编译的构造（`->`、`dashed`、`densely dotted`、`fill`/`fill=white` 圆、`.. controls ..`、`<->`、`\foreach`、`rectangle`、`circle`）。因此 `main.tex` 与 `section2.tex` 前言的 `\usepackage{tikz}` / `\usetikzlibrary{arrows.meta}` 无需改动。
- **符号约定全文统一**：**实心● = 卡车，空心○ = 无人机**（沿 `fig:problem` 的用法）。
- **编号一致性**：$\theta_1>\theta_2>\theta_3$（$\theta_1$ 在品质轴右端 = 最高等级门槛），价格落差按跨越顺序取 $\Pi_1/\Pi_2/\Pi_3$，各面板高度比例一致。
- **站位声明**：隐含 $\tau^{\mathrm{D}}<\tau^{\mathrm{T}}$ 的图已在 caption 里注明「若 $\Delta\tau_i<0$ 则两个分界位置互换」，与正文「$\Delta\tau_i$ 符号不确定」一致。

### 本轮未做的验证（遵用户指令）

按用户要求**未执行任何编译**，只做静态核验：括号 928/928 平衡、`figure` 4/4、`tikzpicture` 8/8、`minipage` 8/8、`equation` 18/18、全部 `\label`/`\ref` 本地可解析、所有自定义 tikz 样式都在同一 `tikzpicture` 内定义过、CRLF 与无 BOM 保持。`main.pdf` 相对新图已过期，下次编译时需重跑（`xelatex -aux-directory=build -interaction=nonstopmode main.tex` 三遍；`latexmk` 在本机仍不可用）。

### 编号影响（累计）

插入 3 张图后，§2 的 4 张图占据图 1–4，§5 原有 9 张图整体后移**三位**（`fig:cont` 1→4 … ）。所有 `\ref` 由 LaTeX 自动重算，无需手改。
