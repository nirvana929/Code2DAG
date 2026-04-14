# Code2DAG

<p align="center">
    <img src="./assets/logo.svg" alt="Code2DAG" width="30%">
</p>

<p align="center">
    <a href="https://github.com" rel="nofollow">
        <img src="https://img.shields.io/badge/ASAP-2026-blue.svg">
    </a>
    <a href="https://github.com" rel="nofollow">
        <img src="https://img.shields.io/badge/并行程序-DAG构建-orange.svg">
    </a>
    <a href="https://github.com" rel="nofollow">
        <img src="https://img.shields.io/badge/平台-Linux%20%7C%20POSIX-brightgreen.svg">
    </a>
    <a href="https://github.com" rel="nofollow">
        <img src="https://img.shields.io/badge/Python-%E2%89%A53.8-yellow.svg">
    </a>
</p>

<p align="center">
    <a href="README.md">English</a> | <a href="README_zh.md">中文</a>
</p>

**Code2DAG** 是一个面向调度的自动化工具链，用于从并行 C 程序中构建 DAG 任务模型。工具以源文件及其 GCC RTL 中间表示为输入，自动提取线程间执行依赖、构建并最小化程序 DAG、对节点执行时间进行剖析，并为各调度算法生成可直接在实时 Linux 上运行的优先级标注可执行文件。

在实时 Linux 平台上的实验表明，Code2DAG 引导的调度相较 Linux 完全公平调度器（CFS）平均可将程序完工时间（makespan）降低 **13.73%**，最优情形下降幅高达 **30.87%**。

> 本文档面向工具使用者，涵盖安装、运行与输出解读，术语与同名论文保持一致。

---

## 目录

- [背景](#背景)
- [核心特性](#核心特性)
- [流水线概述](#流水线概述)
- [术语说明](#术语说明)
- [环境要求](#环境要求)
- [安装](#安装)
- [快速开始](#快速开始)
- [命令行参考](#命令行参考)
- [输出结构](#输出结构)
- [支持的调度算法](#支持的调度算法)
- [示例：`zhang1`](#示例zhang1)
- [实验评估](#实验评估)
- [适用范围](#适用范围)
- [仓库结构](#仓库结构)
- [引用](#引用)

---

## 背景

现代并行程序中，线程间常存在复杂且非显式的执行依赖关系。这种依赖结构可以用*有向无环图*（DAG）来表示，从而使基于 DAG 的调度方法得以通过关键路径感知来缩短程序完工时间。然而，从真实并行程序中正确、紧凑地构建 DAG 并非易事：线程生命周期事件、同步操作以及隐式数据依赖均需精确捕获。

现有依赖提取方法或仅针对特定依赖类型（如控制流或数据流），或依赖特定运行实例的动态观测，导致所得图结构与具体运行实例绑定，而非程序级表示。

**Code2DAG** 填补了这一空白。它对源代码与 GCC RTL 进行联合静态分析，提取三类线程间依赖——顺序函数调用链、线程生命周期关系（`pthread_create`/`pthread_join`）以及同步约束（`sem_post`/`sem_wait`、`pthread_mutex_lock`/`pthread_mutex_unlock`）——并将其统一融合为单一 DAG。在此基础上，工具将完整流程自动化，从原始源代码直接生成调度就绪的可执行文件，无需人工标注或手动建模。

---

## 核心特性

- **自动化 DAG 构建**：从并行 C 源代码与 GCC RTL 出发，无需人工标注
- **三类依赖捕获**：顺序依赖（函数调用链）、线程生命周期依赖（`pthread_create`/`pthread_join`）、同步依赖（`sem_post`/`sem_wait`、`pthread_mutex_lock`/`pthread_mutex_unlock`）
- **同步感知节点合并**：在保证所有依赖约束正确性的前提下显著缩减图规模
- **自动执行时间剖析**：在程序段边界插桩并执行，产生附有执行时间代价的 DAG 节点
- **五种调度算法内置**：FIFO、LPF、HEFT、T-level 及 Zhao（2020）
- **优先级标注代码生成**：输出可在实时 Linux `SCHED_FIFO` 策略下直接运行的优先级标注 C 程序
- **交互式 GUI 工作台**：支持 DAG 可视化、流水线控制与结果检查

---

## 流水线概述

Code2DAG 以两层流水线运行：

```
┌──────────────────────────────── DAG 构建层 ─────────────────────────────────┐
│                                                                              │
│  源码 / RTL  ──►  依赖提取   ──►  DAG 构建   ──►  节点合并  ──►  最小化 DAG │
│  分析                                                                        │
│  (阶段 a)        (阶段 b)        (阶段 c)        (阶段 d)                    │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────── DAG 调度层 ─────────────────────────────────┐
│                                                                              │
│  最小化 DAG  ──►  运行时度量  ──►  优先级分配  ──►  可执行 DAG              │
│                  (阶段 e)         (阶段 f)                                   │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

**阶段 a — 源码/RTL 分析：** 利用状态机驱动的正则表达式匹配，解析 C 源文件及对应的 GCC RTL `.expand` 转储，提取函数头、符号引用、源码位置及变量标识符。

**阶段 b — 依赖提取：** 识别三类线程间关系：
- *顺序依赖*：线程内直接函数调用链
- *线程生命周期依赖*：`pthread_create` / `pthread_join` 对应关系
- *同步依赖*：`sem_post` / `sem_wait` 及 `pthread_mutex_lock` / `pthread_mutex_unlock` 配对

**阶段 c — DAG 构建：** 以 DAG 构建中间表示为基础，构建*原始 DAG*，其中每个节点代表一个程序操作，每条有向边编码一条依赖约束。

**阶段 d — 节点合并：** 执行两阶段同步感知合并算法。第一阶段合并普通节点与互斥保护区域，第二阶段通过上行/下行方向遍历将同步控制节点（create、join、post、wait）与相邻中间节点合并，得到*最小化 DAG*。

**阶段 e — 运行时度量：** 在段边界处插桩程序并执行，记录各节点平均执行时间，产生*注释 DAG*。

**阶段 f — 优先级分配：** 对注释 DAG 应用所选调度算法计算各节点优先级，并在每个节点入口插入 `sp.sched_priority = prior_n` 和 `pthread_setschedparam(...)` 调用，输出*可执行 DAG*（优先级标注 C 源文件）。

---

## 术语说明

以下术语在本工具及同名论文中保持一致：

| 术语 | 定义 |
|:---|:---|
| **原始 DAG** | 直接由源码与 GCC RTL 分析构建的 DAG |
| **最小化 DAG** | 经同步感知节点合并后的 DAG |
| **注释 DAG** | 附有运行时执行代价的最小化 DAG |
| **可执行 DAG** | 由优先级分配后的注释 DAG 生成的插桩 C 程序 |

术语与发布输出文件的对应关系：

| 文件 | 对应术语 |
|:---|:---|
| `results/<case>/graphs/original_dag.png` | 原始 DAG（可视化） |
| `results/<case>/graphs/block_dag.png` | 最小化 DAG（合并图视图） |
| `results/<case>/algorithms/<algo>/source/<algo>.c` | 可执行 DAG（插桩源码） |

---

## 环境要求

| 依赖项 | 版本 | 用途 |
|:---|:---|:---|
| Python | ≥ 3.8 | 流水线运行时 |
| GCC | 现代版本均可 | RTL 转储生成与插桩程序编译 |
| Graphviz（`dot`） | 现代版本均可 | DAG 渲染为 PNG/SVG |

**平台说明：**
- DAG 构建阶段在具备 `python3`、`gcc`、`dot` 的 Unix-like 系统上可移植运行。
- 计时、插桩及生成的调度代码依赖 `pthread`、GCC 专有转储标志及实时调度设施，面向 Linux/POSIX 平台。
- Windows 不是端到端流水线的主要目标平台。

环境快速检验：

```bash
python3 --version
gcc --version
dot -V
```

Python 包依赖（见 `requirements.txt`）：

```
networkx>=2.6
pillow>=9.0.0
pydot>=1.4.2
matplotlib>=3.5
pandas>=1.5
seaborn>=0.12
flask>=2.2
```

---

## 安装

```bash
git clone <仓库地址>
cd <仓库父目录>
pip install -r Code2DAG/requirements.txt
```

确保 GCC 和 Graphviz 已加入 `PATH`。Python 流水线本身无需编译。

**目录说明：** 以包模块方式运行时，请在仓库父目录下执行：

```bash
python3 -m Code2DAG.pipeline.cli <子命令> [选项]
```

若需从任意工作目录运行，请将仓库父目录添加至 `PYTHONPATH`：

```bash
PYTHONPATH=/绝对路径/<仓库父目录> python3 -m Code2DAG.pipeline.cli list
```

---

## 快速开始

推荐以 `zhang1` 作为入门示例——该程序包含一条强关键链和若干填充线程，可直观体现关键路径感知调度的收益。

在仓库父目录下执行：

```bash
python3 -m Code2DAG.pipeline.cli run_all \
  --source Code2DAG/source_files/zhang1/zhang1.c \
  --level level2 \
  --rule effective_line_merge
```

运行成功后，最终结果发布于 `results/zhang1/`。

启动交互式 GUI 工作台：

```bash
python3 Code2DAG/run_gui.py
```

---

## 命令行参考

流水线通过 `python3 -m Code2DAG.pipeline.cli` 控制，主子命令为 `run_all`（端到端执行全部阶段）。

```
python3 -m Code2DAG.pipeline.cli <子命令> [选项]
```

| 子命令 | 说明 |
|:---|:---|
| `run_all` | 执行完整流水线：collect → blocks → timing → schedule → instrument |
| `collect` | 阶段 a–c：解析源码/RTL，构建原始 DAG |
| `blocks` | 阶段 d：执行节点合并，产生最小化 DAG |
| `timing` | 阶段 e：插桩并剖析节点执行时间 |
| `schedule` | 阶段 f：使用所选算法计算调度优先级 |
| `instrument` | 生成最终插桩 C 源文件（可执行 DAG） |
| `list` | 列出可用用例及其流水线状态 |

**`run_all` 主要选项：**

| 选项 | 说明 |
|:---|:---|
| `--source <路径>` | 输入 C 源文件路径 |
| `--level <level1\|level2>` | 分段层级；`level2` 启用信号量感知合并 |
| `--rule <规则>` | 节点合并规则；默认为 `effective_line_merge` |
| `--algo <名称>` | 所用调度算法（默认：全部五种） |

---

## 输出结构

每个处理用例的最终结果发布于 `results/`，中间产物保留于 `intermediate_results/`：

```
results/<case_name>/
├── graphs/
│   ├── original_dag.png           # 原始 DAG 可视化
│   ├── block_dag.png              # 最小化 DAG 可视化
│   └── original_dag.round.dot    # 原始 DAG 的 DOT 源文件
└── algorithms/
    ├── FIFO/
    │   ├── source/FIFO.c          # 可执行 DAG（插桩源码）
    │   └── meta/summary.json      # 产物元数据
    ├── LPF/
    ├── heft/
    ├── t_level/
    └── zhao2020/
```

`results/` 是面向用户的主入口。`intermediate_results/` 暴露流水线内部各阶段产物，供调试与检查使用。

---

## 支持的调度算法

Code2DAG 为五种调度算法生成插桩输出：

| 算法 | 策略类型 | 说明 |
|:---|:---|:---|
| **FIFO** | 基线 | 先进先出；不具备 DAG 感知能力 |
| **LPF** | 关键路径 | 最长路径优先；优先调度剩余最长路径上的节点 |
| **HEFT** | 排名 | 异构最早完成时间；基于上行排名的优先级分配 |
| **T-level** | 静态 | 基于顶层（top-level）的静态优先级分配 |
| **Zhao2020** | 规则 | 来自 Zhao 等（2020）的基于规则优先级策略 |

`results/<case>/algorithms/` 下每个算法目录包含由相同输入生成的插桩程序，可在相同基准下直接对比不同调度策略的效果。

---

## 示例：`zhang1`

`zhang1` 是一个包含强关键链（`c0 → c1 → c2 → c3 → c4`）和多条填充线程的并行程序，专为突出关键路径感知调度收益而设计。

**输入：** [source_files/zhang1/zhang1.c](source_files/zhang1/zhang1.c)

**发布产物：**

| 产物 | 路径 |
|:---|:---|
| 原始 DAG | [results/zhang1/graphs/original_dag.png](results/zhang1/graphs/original_dag.png) |
| 最小化 DAG | [results/zhang1/graphs/block_dag.png](results/zhang1/graphs/block_dag.png) |
| DOT 源文件 | [results/zhang1/graphs/original_dag.round.dot](results/zhang1/graphs/original_dag.round.dot) |
| LPF 可执行 DAG | [results/zhang1/algorithms/LPF/source/LPF.c](results/zhang1/algorithms/LPF/source/LPF.c) |
| HEFT 可执行 DAG | [results/zhang1/algorithms/heft/source/heft.c](results/zhang1/algorithms/heft/source/heft.c) |
| Zhao2020 可执行 DAG | [results/zhang1/algorithms/zhao2020/source/zhao2020.c](results/zhang1/algorithms/zhao2020/source/zhao2020.c) |

**建议阅读顺序：**
1. 从输入程序 [zhang1.c](source_files/zhang1/zhang1.c) 出发。
2. 打开 [original_dag.png](results/zhang1/graphs/original_dag.png)，检查原始 DAG。
3. 打开 [block_dag.png](results/zhang1/graphs/block_dag.png)，检查最小化（合并后）DAG。
4. 对比 `results/zhang1/algorithms/` 下各调度策略生成的插桩程序。
5. 查阅各 `meta/summary.json` 了解产物映射关系与元数据。

---

## 实验评估

实验在运行 Ubuntu 20.04.6 LTS 的 VMware 虚拟机上进行，搭载第 13 代 Intel Core i5-13400F 处理器（约 2.5 GHz），使用实时 Linux 内核（5.15.96-rt61）。每项实验重复 100 次，取平均执行时间。

### 实验配置

使用六种具有代表性 DAG 结构（DAG 1–6）在两种 CPU 亲和性配置下评估：
- **2 核**配置：CPU 亲和性 `0–1`
- **4 核**配置：CPU 亲和性 `0–1–2–3`

与三类基线对比：Linux CFS、`SCHED_FIFO`（非 DAG 感知优先级策略）以及 Code2DAG 生成的四种 DAG 引导策略（FIFO、LPF、HEFT、Zhao）。

### 主要结果

<table align="center">
<thead>
<tr>
<th align="center">配置</th>
<th align="center">指标</th>
<th align="center">结果</th>
</tr>
</thead>
<tbody>
<tr>
<td align="center">全部 DAG（2 核 &amp; 4 核）</td>
<td align="center">相较 CFS 的平均 makespan 降幅</td>
<td align="center"><strong>13.73%</strong></td>
</tr>
<tr>
<td align="center">最优情形（DAG 3，4 核）</td>
<td align="center">相较 CFS 的 makespan 降幅</td>
<td align="center"><strong>30.87%</strong></td>
</tr>
<tr>
<td align="center">插桩开销</td>
<td align="center">单节点优先级插入代价</td>
<td align="center"><strong>约 0.2 µs</strong></td>
</tr>
</tbody>
</table>

DAG 引导策略在大多数 DAG 结构和两种核心配置下均稳定优于 CFS。收益在具有清晰关键路径结构的 DAG（DAG 3、4、5、6）上最为突出；对路径长度接近均等的结构，DAG 感知的差异化程度有限，收益相对较小。

---

## 适用范围

Code2DAG 适用于通过标准 POSIX 接口表达并发与同步的并行 C 程序：

- 线程生命周期：`pthread_create`、`pthread_join`
- 信号量同步：`sem_post`、`sem_wait`
- 互斥锁：`pthread_mutex_lock`、`pthread_mutex_unlock`

使用上述原语的程序可由当前流水线完整分析并建模为 DAG。其他并发模型（如 OpenMP、C++ `std::thread`）目前不在支持范围内。

---

## 仓库结构

```
Code2DAG/
├── pipeline/               # 命令行入口与各阶段流水线实现
│   ├── cli.py              # 主命令行接口
│   ├── runner.py           # 流水线编排
│   ├── collector.py        # DAG 收集与构建
│   ├── timing.py           # 剖析与代价标注
│   ├── schedule.py         # 调度算法分发
│   ├── instrument.py       # 优先级插桩与代码生成
│   ├── algo/               # 调度算法实现（LPF、HEFT、T-level、Zhao、FIFO）
│   └── rules/              # 节点合并规则实现
├── generation/             # 源码与 RTL 分析、DAG 构建
├── core/                   # 共享数据模型与解析工具
├── level1/                 # 第一层分段与插桩
├── level2/                 # 第二层分段（信号量感知合并）
├── ui/                     # GUI 工作台
├── visualization/          # 交互式 DAG 查看器
├── assets/                 # Logo 与视觉资源
├── source_files/           # 输入 C 程序（测试用例）
├── results/                # 最终发布产物（DAG 视图、插桩代码）
├── intermediate_results/   # 内部流水线中间产物
├── tools/                  # 辅助工具（运行时对比、基准测试）
├── requirements.txt
└── README.md
```

---

## 引用

如在研究中使用 Code2DAG，请引用同名论文：

```bibtex
@inproceedings{code2dag2026,
  title     = {Code2DAG: An Automatic DAG Construction Tool for Scheduling Parallel Programs},
  booktitle = {Proceedings of the International Symposium on Advanced Parallel Processing (ASAP)},
  year      = {2026},
}
```

---

## 致谢

1. [TACLeBench](https://github.com/tacle/tacle-bench) — 对比评估所用 WCET 基准测试集
2. [NetworkX](https://networkx.org/) — DAG 构建与调度底层图分析库
3. [Graphviz](https://graphviz.org/) — DAG 渲染
