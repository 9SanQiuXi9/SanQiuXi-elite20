# 三秋羲_C5A_仓库分析

> 分析对象：`neolaf2/neoskills`（fork：`9SanQiuXi9/neoskills`）
> 分析基准：`main` 分支 `7c4dada`（2026-11-21 clone）
> 许可证：MIT ｜ 作者：Richard Tong ｜ 当前版本：0.4.2（`pyproject.toml`）

## 一、一句话概括

neoskills 是一个「Homebrew 式」的 AI Agent 技能管理器：把分散在 Claude Code、OpenCode、OpenClaw 里的技能定义集中到 git 仓库（tap）中，用符号链接部署到各 Agent，并用一层基于 `ontology.yaml` 的属性图描述技能之间的依赖、组合和生命周期。

它解决的不是「写技能」，而是「技能多了以后怎么管」：同一份技能被复制到多个 Agent 目录后产生版本漂移，靠手工同步无法维护，neoskills 用「单一事实来源 + 符号链接 + 图谱元数据」把这件事系统化。

## 二、仓库的目录结构

```
neoskills/
├── .claude-plugin/plugin.json   # 作为 Claude Code 插件分发时的清单
├── .github/workflows/           # CI：lint / test / build 三个 job
├── agents/                      # 4 个 Claude Code 子 Agent 定义
├── commands/ns.md               # /ns 斜杠命令，把 CLI 与 Agent 路由起来
├── docs/ontology-design.md      # 本体层设计文档
├── skills/                      # 仓库自带的两个技能
│   ├── bank-status/SKILL.md
│   └── skill-dedup/SKILL.md + scripts/dedup_scan.py
├── src/neoskills/               # Python 包（约 7,500 行）
│   ├── cli/                     # Click 命令层，main.py 负责注册
│   ├── core/                    # Cellar、tap、linker、config、frontmatter
│   ├── ontology/                # 属性图：models/graph/loader/engine 等
│   ├── adapters/                # claude-code / opencode / openclaw 适配器
│   ├── runtime/                 # Claude Code MCP 插件
│   └── meta/                    # 技能增强
├── tests/                       # 172 个测试（unit + integration）
├── README.md / CLAUDE.md        # 用户文档 / 项目工作指南
├── TASKS.md                     # 版本状态与设计决策日志
└── neoskills_v0_*.md            # 早期设计稿，保留作历史
```

入口在 `src/neoskills/cli/main.py`：一个 Click group，用「延迟导入」逐个注册子命令；`pyproject.toml` 把 `neoskills` 命令映射到 `neoskills.cli.main:cli`。

## 三、仓库里的技能

仓库把「技能」分成了两类：给 Agent 用的用户技能（`skills/`），以及给开发/维护流程用的子 Agent（`agents/`）。

### 1. bank-status（`skills/bank-status/SKILL.md`）

- **做什么**：回答「我的技能库里现在有什么」。它指导 Agent 读取技能库目录、统计技能数量与标签、检查已嵌入（符号链接）到 Agent 的技能，并检查当前配置了哪些目标。
- **输入**：用户的自然语言提问（技能库状态、清单、可用技能）。
- **输出**：一段摘要，包含技能库总数、各目标的技能数、已嵌入技能、最近导入的技能。
- **适用场景**：接手一台新机器、排查「为什么 Agent 看不到某个技能」、做技能库盘点。

### 2. skill-dedup（`skills/skill-dedup/SKILL.md` + `scripts/dedup_scan.py`）

- **做什么**：跨技能库、各 Agent 目录、插件目录扫描重复技能，并把重复分成三类：内容完全相同（SHA-256 一致）、同名但内容分叉、名称相似但 ID 不同。
- **输入**：技能库路径、要扫描的目标（默认 claude/opencode）、可选 GitHub 仓库、相似度阈值（默认 0.80）。
- **输出**：重复分组报告（可 `--json`），或在 `--resolve exact/diverged/all` 下直接把目标目录的副本替换为指向技能库的符号链接，并在改动前支持 `--dry-run`。
- **适用场景**：批量导入技能之后、多个 Agent 各自装过同一技能之后、定期做技能库卫生。

### 3. 四个维护型子 Agent（`agents/`）

- **skill-scanner**：扫描各 Agent 生态，产出技能清单（ID、描述、是否有 frontmatter、目录型还是文件型）。
- **skill-importer**：把技能从目标目录、git 仓库或网页导入技能库，并记录来源（provenance）。
- **skill-deployer**：把技能或技能包部署到目标 Agent，负责符号链接的建立、备份与恢复。
- **skill-dedup**：把上面的去重流程包装成可自主执行的 Agent 工作流。

`commands/ns.md` 是它们的调度入口：`/ns scan`、`/ns import`、`/ns deploy`、`/ns dedup` 走 Agent，其余命令直接走 CLI。这种「同一个能力，既能命令行调用，也能让 Agent 自主调用」的双入口设计，是这个仓库比较有意思的地方。

## 四、我觉得最有意思的技能：skill-dedup

原因有三点。

**第一，它解决的是技能生态扩张后的真问题。** 技能库小的时候，重复不是问题；一旦同一个技能同时存在于 tap、`~/.claude/skills/` 和某个插件缓存里，修改就会分叉。skill-dedup 把「重复」拆成 exact / diverged / name-similar 三类，而不是笼统地说「有重复」，这直接决定了处理策略：完全一致可以安全地换成符号链接，分叉要人工比较，名称相似可能只是巧合。分类是这类工具的核心价值。

**第二，它是「操作技能的技能」。** 仓库里另一个技能 bank-status 只是读状态，而 skill-dedup 会改文件系统（替换副本为符号链接），所以它必须自带 `--dry-run`、备份目录和「先报告、后执行」的流程。这种 meta 层的技能，比普通的任务技能更考验边界设计。

**第三，它和我自己做过的技能评测工作相关。** 我在 C4 阶段做过提交物评测器，评测器关心「这份提交是否完整」；skill-dedup 关心「这份技能是否重复」。两者都在做「技能资产的质量治理」，只是维度不同。读它的分类逻辑和阈值设计，对我理解技能库的长期维护很有帮助。

## 五、我是怎么读懂它的

不是只看 README，而是按「结构 → 入口 → 数据模型 → 行为」四步走：

1. `git clone --depth 1` 后先看顶层目录和 `README.md`，建立「它是什么」的假设；
2. 读 `CLAUDE.md` 和 `TASKS.md`，拿到作者自己写的架构说明、已知问题和设计决策；
3. 从 `cli/main.py` 的注册表反推命令面，再顺着 `core/`（Cellar、tap、linker）和 `ontology/loader.py`（文件系统 → 图）读数据流；
4. 用 `find`、`grep`、`wc -l` 核对文档里的说法：例如 README 说 172 个测试，`grep -r "def test_" tests | wc -l` 的结果正好是 172；README 说 7,500 行左右，`src/` 实际 7,531 行。

## 六、读的时候留下的疑问

分析时也发现了几处「文档和代码对不上」的地方，这里先记录，详细证据和改进建议放在深度分析里：

- `agents/` 下的多个 Agent 和 `skills/skill-dedup/SKILL.md` 仍在描述 `~/.neoskills/LTM/bank/` 这套旧目录，而 v0.4 的 `Cellar` 实际只有 `taps/`、`cache/`、`config.yaml`；
- `dedup_scan.py` 仍然 `import neoskills.bank.store`，这个模块在当前包里已不存在；
- 版本号在 `pyproject.toml`、`CLAUDE.md`、`plugin.json`、`Cellar` 默认配置里不一致。

这些不是「挑错」，而是读仓库时最值得记下来的信号：一个项目的文档漂移，往往说明它的演进速度快于维护能力。

## 七、结论

neoskills 的定位很清楚：不做「技能市场」，而做「技能的本机包管理器 + 关系图」。它的设计重心在 `core`（tap/link/config）和 `ontology`（图模型），技能本身反而很少。读懂它的关键不是看那两个技能，而是理解「文件系统即数据库、符号链接即部署、YAML 旁挂即元数据」这三条贯穿全仓库的假设。
