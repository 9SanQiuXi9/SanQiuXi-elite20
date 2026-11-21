# 三秋羲_C5A_深度分析

> 对象：`neolaf2/neoskills` @ `7c4dada`（2026-11-21）
> 方法：静态阅读（clone 后读源码与文档）+ 结构统计（`find` / `grep` / `wc -l`）。未在该仓库上执行测试套件，涉及运行行为的判断均标注为「读代码推断」。

## 0. 事实底稿

| 项目 | 数值 / 结论 | 依据 |
|---|---|---|
| 语言与版本 | Python ≥ 3.13 | `pyproject.toml:6` |
| 包版本 | 0.4.2 | `pyproject.toml:3`、`src/neoskills/__init__.py:3` |
| 源码规模 | 7,531 行（`src/` 下 Python） | `find src -name "*.py" \| xargs wc -l` |
| 测试 | 172 个 `def test_` | `grep -r "def test_" tests \| wc -l` |
| CI | lint → test → build 三段 | `.github/workflows/ci.yml` |
| 分发 | PyPI（hatchling 构建，`neoskills` 入口） | `pyproject.toml:47-49` |
| 许可证 | MIT | `LICENSE`、`pyproject.toml:5` |
| 自带技能 | 2 个（bank-status、skill-dedup） | `skills/` |
| 维护型子 Agent | 4 个 | `agents/` |

## 1. 架构分析

### 1.1 五层结构

```
┌──────────────────────────────────────────────┐
│ 分发层   PyPI wheel / Claude Code 插件 / MCP   │
├──────────────────────────────────────────────┤
│ 接口层   CLI (Click)          /ns 命令 + Agent │
├──────────────────────────────────────────────┤
│ 领域层   Cellar  Tap  Linker  OntologyEngine  │
├──────────────────────────────────────────────┤
│ 模型层   SkillNode  OntologyEdge  SkillGraph  │
├──────────────────────────────────────────────┤
│ 存储层   文件系统（tap 目录 + SKILL.md + yaml）│
└──────────────────────────────────────────────┘
```

- **存储层没有数据库。** `~/.neoskills/` 就是全部状态：`config.yaml` 存目标与默认 tap，`taps/<name>/` 是 git clone 下来的技能仓库，`cache/` 放备份和临时文件（`core/cellar.py:33-48`）。作者在 `TASKS.md` 里把这条原则写成「file-system-as-database」。
- **领域层是两套并行的抽象。** `core` 负责「技能放在哪、怎么链接」（Cellar/TapManager/Linker），`ontology` 负责「技能之间是什么关系」（Loader → SkillGraph → Engine）。两者通过 `SkillNode.path` 和 `source_location` 关联，而不是继承关系。这个边界切得比较干净：删掉 ontology 层，包管理器仍然可用；这正是 v0.4 能对 v0.3 保持向后兼容的原因。
- **接口层是「延迟注册」。** `cli/main.py` 先建 Click group，再在 `_register_later_commands()` 里逐个 `importlib.import_module` 并 `add_command`（`cli/main.py:22-62`）。好处是启动快、某个命令模块坏了不会拖垮整个 CLI；代价见 3.4。
- **分发层有三种形态**：命令行工具、被 Agent 调用的工具、Claude Code 内的 MCP 插件（`src/neoskills/runtime/claude/plugin.py`）。`/ns` 命令（`commands/ns.md`）再把 scan/import/deploy/dedup 四个动作路由给子 Agent，其余直接走 Bash 调 CLI。

### 1.2 一次 `ontology load` 的数据流

1. `OntologyLoader.load()` 先遍历 `default_source_trees()`：`~/.neoskills/taps/*/skills/*` 与 `~/.claude/skills/*`（`ontology/loader.py:44-64`）；
2. 对每个含 `SKILL.md` 的目录，解析 frontmatter 和可选的 `ontology.yaml`（`loader.py:121-164`）；
3. `_build_node()` 把两份元数据合并成一个 `SkillNode`，并根据「有没有 ontology.yaml / 有没有边 / 有没有生命周期」推断增强等级 L0–L3（`loader.py:185-198`）；
4. 边从 `ontology.yaml` 的 `edges:` 块展开为 `OntologyEdge`，再补一条 `belongs_to` 领域边（`loader.py:304-406`）；
5. `SkillGraph.add_node/add_edge` 同时维护六组倒排索引和正反向邻接表（`ontology/graph.py:42-52`），使按领域/类型/状态/标签/命名空间/增强等级的分面查询是 O(1)；
6. `OntologyEngine` 在图上提供 discover/deps/rdeps/validate/export 等高层操作，CLI 只做参数解析和输出。

这个数据流的关键取舍是：**每次 `load` 都全量扫描文件系统**。作者在 `TASKS.md` 的后续计划里已经承认需要图缓存（按 mtime 失效）。在设计规模（100–500 个技能）下全量扫描可以接受，但这是明确的性能上限。

### 1.3 三个值得借鉴的设计决策

**（1）`ontology.yaml` 旁挂而不是塞进 SKILL.md。**
SKILL.md 保持人类可读、可被不同 Agent 直接消费；本体元数据独立演进，而且被排除在技能内容校验和之外（`TASKS.md` 设计决策表）。这让「给老技能补元数据」不会改变技能本身的哈希，避免去重和同步逻辑误判为内容变更。

**（2）渐进增强 L0–L3。**
裸 SKILL.md 是 L0，加 type/domain/tags 是 L1，加显式边是 L2，加生命周期/版本/能力清单是 L3（`ontology/models.py:82-88`）。好处是迁移成本为零：现有技能不改也能被加载，`infer_domain_from_skill_id` 还会给出兜底领域（`loader.py:220-228`）。这是一种「先能用，再逐步变好」的元数据策略。

**（3）符号链接即部署。**
`~/.claude/skills/<name>` 指向 tap 里的技能目录，而不是复制一份。多个 Agent 共享同一份内容，修改立即生效，卸载只是删链接。把「部署」降级为「建链接」，是整个工具体验最轻的一环。

## 2. 技能清单（输入 / 输出 / 适用场景）

### 2.1 用户技能（`skills/`）

| 技能 | 输入 | 输出 | 适用场景 | 实现形态 |
|---|---|---|---|---|
| **bank-status** | 「技能库现在有什么」这类问题；读取技能库目录、状态文件、目标配置 | 技能总数、各目标技能数、已嵌入技能、最近导入 | 新机器接手、排查 Agent 看不到技能、例行盘点 | 纯提示词（SKILL.md） |
| **skill-dedup** | 技能库路径、目标列表（默认 claude/opencode）、可选 GitHub 仓库、相似度阈值（默认 0.80） | 重复分组报告（文本/JSON）；或执行 `--resolve exact/diverged/all` 后的符号链接替换结果 | 批量导入后清理、多 Agent 重复安装后治理、定期维护 | 提示词 + Python 脚本（`scripts/dedup_scan.py`） |

两者都是「元技能」：bank-status 读元数据，skill-dedup 改元数据归属。仓库没有提供任何面向业务的任务技能（写文档、做表、查资料），说明它把自己定位为**技能基础设施**，而不是技能内容库。

### 2.2 维护型子 Agent（`agents/`）

| Agent | 输入 | 输出 | 场景 |
|---|---|---|---|
| **skill-scanner** | 目标定义文件、各生态的发现路径 | 按目标分组的技能清单（ID、描述、frontmatter、目录/文件型） | 摸清一台机器上到底装了什么 |
| **skill-importer** | 来源（目标/git/网页）+ 技能 ID | 技能库条目、`provenance.yaml`、registry 更新 | 把散落技能收编进库 |
| **skill-deployer** | 技能或技能包 ID、目标 ID | 符号链接、必要时备份/恢复 | 部署与回滚 |
| **skill-dedup** | 同 skill-dedup 技能 | 重复分组 + 处理建议 | 自主执行去重流程 |

这四者的共同点是**只操作文件系统和配置，不调用模型做内容生成**（skill-dedup 的 `model: inherit` 只用于决策展示）。它们更像「有自然语言接口的运维脚本」，这也解释了为什么 `commands/ns.md` 把 scan/import/deploy/dedup 路由给 Agent，而把 validate/embed/config 留给 CLI——前四个需要多步判断，后几个是确定性操作。

## 3. 改进建议

按「影响面 × 修复成本」排序，每条都给出可定位的证据。

### 3.1 P0：删掉硬编码的个人绝对路径

`src/neoskills/cli/schedule_cmd.py:132`：

```python
@click.option(
    "--workspace", type=click.Path(path_type=Path), default=Path("/Users/rich/.openclaw/workspace")
)
```

这是作者本机的 macOS 路径，却被写成所有用户的默认值。在 Windows 上它会被解释为当前盘符下的 `\Users\rich\...`，在 Linux 上会直接创建一个 `/Users/rich/` 目录。`pyproject.toml` 声明 `Operating System :: OS Independent`，这一行与该声明直接冲突。

**建议**：默认值改为 `None`，在函数体内用 `Path.home() / ".openclaw" / "workspace"` 解析；README 中补一句默认路径说明。
**验证方式**：在临时 `HOME` 下运行 `neoskills schedule daily --help`，确认默认值不再包含 `rich`。

### 3.2 P1：文档与 v0.4 目录模型已经脱节

`Cellar` 在 v0.4 只创建 `taps/`、`cache/`、`config.yaml`（`core/cellar.py:33-48`），但多处文档仍在描述 v0.2 的 `LTM/bank/` 结构：

- `agents/skill-dedup.md:50` 指示从 `~/.neoskills/LTM/bank/skills/skill-dedup/canonical/scripts/dedup_scan.py` 运行脚本；
- `agents/skill-importer.md:46-54` 把技能库描述为 `LTM/bank/skills/<id>/canonical|variants|metadata.yaml`；
- `agents/skill-deployer.md:43-47` 的链接目标是 `LTM/bank/skills/<name>/variants/claude-code`；
- `skills/skill-dedup/SKILL.md:87-93` 的扫描位置第一条仍是 `~/.neoskills/LTM/bank/skills/`；
- `skills/bank-status/SKILL.md:12-15` 让 Agent 读 `registry.yaml`、`state.yaml`、`LTM/mappings/targets/`，这些文件在当前初始化流程里不会生成。

更实际的问题是 `skills/skill-dedup/scripts/dedup_scan.py:28-34` 仍然尝试导入 `neoskills.core.workspace`、`neoskills.mappings.resolver`、`neoskills.bank.store` 三个已不存在的模块，失败后静默把 `NEOSKILLS_AVAILABLE` 置为 `False`；同时 `find_bank_skills()` 直接拼 `bank_root / "LTM" / "bank" / "skills"`（`dedup_scan.py:126-128`）。结果是在 v0.4 工作区里，「技能库」这一侧永远扫不到东西，去重只剩 Agent 目录之间的比较。

**建议**：以 `Cellar` 为唯一事实来源，重写四个 Agent/skill 文档里的路径；`dedup_scan.py` 改为遍历 `taps/*/skills/*`，删除死导入，找不到工作区时给出明确错误而不是静默降级。修复后应补一个「v0.4 工作区 + 两个重复技能」的集成测试。

### 3.3 P1：版本号在五个地方漂移

| 位置 | 值 |
|---|---|
| `pyproject.toml:3` | 0.4.2 |
| `src/neoskills/__init__.py:3` | 0.4.2 |
| `CLAUDE.md:7`、`CLAUDE.md:100-101` | 0.4.1 |
| `.claude-plugin/plugin.json:4` | 0.1.0 |
| `core/cellar.py:11`（默认配置） | 0.4.0 |

`TASKS.md` 记录过一次「版本字符串修复」，但只修了包内的两处。插件清单仍是 0.1.0，会直接影响 Claude Code 的插件更新判断；`CLAUDE.md` 的发布流程还在要求人工同步两个文件，等于把漂移当成流程。

**建议**：以 `importlib.metadata.version("neoskills")` 为唯一来源，`__init__.py` 从包元数据读取；`Cellar` 默认配置写入 `__version__`；插件清单在构建时由脚本注入版本。这样发布流程从「记得改 N 处」变成「只改 pyproject」。

### 3.4 P2：CLI 静默吞掉导入错误

`cli/main.py:52-59`：

```python
try:
    mod = importlib.import_module(module_path)
    cmd = getattr(mod, cmd_name, None)
    if cmd and cmd.name not in cli.commands:
        cli.add_command(cmd)
except (ImportError, AttributeError):
    pass
```

这带来两个后果：一是全新安装后如果某个依赖缺失（例如 `gitpython` 没装上），对应命令会直接从 `--help` 里消失，用户看到的是「命令不存在」而不是「缺少依赖」；二是开发时拼错导入路径也不会有任何提示。

**建议**：把 `pass` 换成带命令名的 `logger.debug`，并在 `neoskills doctor` 里增加「命令注册完整性」检查；对 `install`、`list`、`tap` 这类核心命令，导入失败应直接抛出而不是跳过。

### 3.5 P2：同一件事有两套 frontmatter 解析器

`core/frontmatter.py:8-43` 已经有一个按行边界找闭合 `---` 的解析器，并处理了 YAML 值里出现 `---` 的情况；但 `ontology/loader.py:121-145` 又写了一套，用 `content.split("---", 2)` 直接切分。后者在前置元数据里包含 `---` 的块标量时会把 YAML 截断，解析失败后静默退化为空元数据（`loader.py:137-141`）。对同一个仓库的技能做「CLI 视角」和「本体视角」的读取，可能出现两种结果。

**建议**：`loader._parse_skill_md` 改为调用 `core.frontmatter.parse_frontmatter`，只保留「name 缺失时回退到目录名」这一层包装；补一个含 `---` 块标量的测试用例。

### 3.6 P2：对外承诺与实现边界需要显式标注

- README 开篇把 OpenClaw 与 Claude Code、OpenCode 并列（`README.md:10`），但 `adapters/openclaw/adapter.py:1` 自述是「stub for v0.1」；
- README 的 Agent 目标表只列了两个（`README.md:243-248`），与开篇不一致；
- `ontology validate` 会把 `belongs_to -> domain` 报成断边（`CLAUDE.md:95`、`TASKS.md` 均已承认）；
- `ontology compose` 只生成 `CompositionSpec`，没有执行器（`TASKS.md` 后续计划第 4 条）。

这些都不是隐藏 bug，而是「路线图写在了功能列表里」。对使用者而言，最危险的是把它们当成已完成能力去搭工作流。

**建议**：在 README 的功能表里加「状态」列（stable / beta / stub），`validate` 对领域边单独归类为 warning，`compose` 命令输出里明确打印「仅生成规格，未提供运行时」。

## 4. 从仓库学到的技能设计知识

**（1）技能的价值一半在触发条件，一半在正文。**
`skill-dedup/SKILL.md` 的 `description` 写了四种触发场景（提到重复、想清理、问冗余、导入时可能已存在），并明确「也适用于导入可能已存在的技能时」。这比正文更能决定技能会不会被 Agent 正确调用。写技能时，`description` 应该写成「何时用」而不是「是什么」。

**（2）元数据要能渐进补齐，否则没人补。**
L0–L3 的设计承认「大多数技能一开始只有 SKILL.md」。如果一上来就要求完整的 capability/edges/lifecycle，现有技能永远迁不过来。先保证裸技能可用，再用 `enrich` 批量补 L1，是一条可执行的路。

**（3）旁挂文件是控制耦合的好工具。**
`ontology.yaml` 不进入 SKILL.md、不进入内容校验和，使「元数据演进」和「技能内容」解耦。做多 Agent 兼容时，把平台相关的映射放在旁挂文件里，比在正文里写条件分支更稳。

**（4）「文件系统即数据库」适合中小规模，但要提前设计失效策略。**
全量扫描让实现极简，也带来明确上限。作者自己已经计划按 mtime 做图缓存。技能库如果超过几百个，应该先加缓存再谈更多查询能力。

**（5）文档漂移是设计债，不是排版问题。**
这个仓库的代码质量不低（172 个测试、CI 三阶段、ruff 干净），但 Agent 文档仍停留在旧目录结构。它提醒我：当目录模型变更时，**所有引用该模型的提示词、脚本、示例都算代码**，需要在同一个 PR 里一起改，否则下一个使用者会按文档操作然后失败。

**（6）对「已完成」的定义要写进仓库。**
`TASKS.md` 把「compose 没有执行器」「validate 误报领域边」明确列为后续工作，这比假装功能完整更可信。我以后写技能或工具，也应该在 README 里保留一节「已知边界」，让评审者能区分「设计取舍」和「没做完」。

## 5. 验证方式与局限

**已验证（可复核）**：

- 目录结构、文件存在性：`find` / `ls`；
- 测试数量 172：`grep -r "def test_" tests | wc -l`；
- 源码行数 7,531：`wc -l`；
- 版本号漂移、硬编码路径、死导入、静默异常：逐文件阅读并记录行号；
- `main` 分支 HEAD 为 `7c4dada`：`git log --oneline -1`。

**未验证（不应被当作结论）**：

- 未安装 Python 3.13、未执行 `uv run pytest`，因此「172 个测试全部通过」来自 README/CI 配置，不是我本机跑出来的；
- 未实际运行 `neoskills` CLI，所有行为判断来自源码阅读；
- 未验证 OpenClaw 适配器在真实 Agent 中的表现。

写作时我把「读到的」和「跑过的」分开标注，避免把静态推断包装成运行结果。
