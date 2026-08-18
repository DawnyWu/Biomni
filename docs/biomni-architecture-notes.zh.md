# Biomni 架构笔记与领域迁移评估（中文）

> 本文是一份问答式研究笔记，基于对本仓库（Biomni）源码的实际通读，以及对 [ToolUniverse](https://github.com/mims-harvard/ToolUniverse) 仓库的实际 clone 与统计。
> 所有数字均为读代码后实测所得，非引用宣传材料。文中 `file:line` 引用对应写作时的仓库状态。
>
> 记录日期：2026-08
> 对应 Biomni 版本：`main` @ `400c1f3`
>
> 📎 姊妹文档：
> - [科学 Agent 生态三者对比：Biomni / ToolUniverse / SCP](./scientific-agent-ecosystem-comparison.zh.md)
>   —— 三方对比，含分层模型、SCP 是 MCP SDK fork 的代码级证据、选型决策树。
> - [OPTIMADE 笔记：材料数据的跨库统一查询标准](./optimade-materials-data-notes.zh.md)
>   —— 材料方向的数据基础设施，含实测验证，**并修正了本文「材料可跳过数据湖」的说法**。

## 目录

- [第一部分：Biomni 是什么，特殊在哪里](#第一部分biomni-是什么特殊在哪里)
  - [Q1.1 框架是什么？](#q11-框架是什么)
  - [Q1.2 真正特殊的地方](#q12-真正特殊的地方)
- [第二部分：四个核心设计问题](#第二部分四个核心设计问题)
  - [Q2.1 「把环境当成 action space」是什么意思？举个例子](#q21-把环境当成-action-space是什么意思举个例子)
  - [Q2.1 补充：这和普通 coding agent 有什么区别？](#q21-补充这和普通-coding-agent-有什么区别)
  - [Q2.2 数据湖是什么？为什么要有这个东西？](#q22-数据湖是什么为什么要有这个东西)
  - [Q2.3 Python 解释器持久化有什么用？好处是什么？](#q23-python-解释器持久化有什么用好处是什么)
  - [Q2.4 「opencode 没有持久 REPL」具体是什么意思？举个例子](#q24-opencode-没有持久-repl具体是什么意思举个例子)
- [第三部分：和 ToolUniverse 的区别](#第三部分和-tooluniverse-的区别)
- [第四部分：能否改成材料化学方向](#第四部分能否改成材料化学方向)
- [第五部分：用 opencode 做这样一个 agent 容易么](#第五部分用-opencode-做这样一个-agent-容易么)
- [附录：关键代码位置速查](#附录关键代码位置速查)

---

## 第一部分：Biomni 是什么，特殊在哪里

Stanford 的通用生物医学研究 agent（`snap-stanford/Biomni`，[bioRxiv 论文](https://www.biorxiv.org/content/10.1101/2025.05.30.656746v1)）。

### Q1.1 框架是什么？

**LangGraph + LangChain Core，但框架不是重点。**

依赖极薄。`pyproject.toml` 里只有三个：`pydantic`、`langchain`、`python-dotenv`；conda 环境锁 `langgraph==0.3.18`。核心 agent 是 `biomni/agent/a1.py` 单文件 3001 行，用 `StateGraph` 搭三个节点：

```python
# biomni/agent/a1.py:1606
workflow = StateGraph(AgentState)
workflow.add_node("generate", generate)
workflow.add_node("execute", execute)
if self_critic:
    workflow.add_node("self_critic", execute_self_critic)
```

状态定义简单到有点夸张，只有消息列表和一个路由标记：

```python
# biomni/agent/a1.py:51
class AgentState(TypedDict):
    messages: list[BaseMessage]
    next_step: str | None
```

图的拓扑：

```
START → generate → (next_step=="execute") → execute → generate
                 → (next_step=="generate") → generate      （纯思考轮）
                 → (next_step=="end")      → END / self_critic
```

其他参数：停止序列 `</execute>` 和 `</solution>`，递归上限 500 步，observation 截断 1 万字符，默认超时 600 秒。

**结论：这个 loop 本身没有秘密，照抄一两天就够。难的是下面这些。**

### Q1.2 真正特殊的地方

#### (a) 把「环境」当成 action space（详见 [Q2.1](#q21-把环境当成-action-space是什么意思举个例子)）

喂给模型的资源分四类，实测规模：

| 资源类型 | 数量 | 位置 |
| --- | --- | --- |
| 工具函数 | **224 个**，22 个领域模块（约 3.25 万行实现 + 7 千行 schema） | `biomni/tool/` |
| 数据湖 | **76 个文件**，约 11 GB，首次运行自动从 S3 下载 | `env_desc.py: data_lake_dict` |
| 软件目录 | **113 条** Python/R/CLI 包描述 | `env_desc.py: library_content_dict` |
| Know-how 文档 | 2 篇 markdown（实验最佳实践，全文进 system prompt） | `biomni/know_how/` |

工具分布看得出重心：`database` 40、`pharmacology` 25、`genomics` 19、`molecular_biology` 18、`microbiology` 12、`physiology` 11。

#### (b) 用代码执行代替 function-calling（CodeAct 模式）

模型不发 tool call，输出 XML 标签：

```
# biomni/agent/a1.py:1124
1) Interact with a programming environment and receive the corresponding output within <observe></observe>.
   Your code should be enclosed using "<execute>" tag ...
   - For Python code (default): <execute> print("Hello World!") </execute>
   - For R code:                <execute> #!R\nlibrary(ggplot2) </execute>
   - For Bash:                  <execute> #!BASH\nls -la </execute>
```

调用工具的方式是让模型自己写 `from biomni.tool.genomics import xxx`。仓库里另留了 `biomni/agent/react.py`（用传统 `llm.bind_tools()`），说明两种范式是对比过之后选的。

#### (c) 资源检索是 prompt-based，不是 embedding-based

`ToolRetriever` 把全部 224 工具、76 数据集、113 库编号丢给 LLM，让它回下标：

```
# biomni/model/retriever.py:54
TOOLS: [list of indices]
DATA_LAKE: [list of indices]
LIBRARIES: [list of indices]
```

提示词明确要求「宁多勿少」，还写了领域启发式（数据库工具尽量全带、文献工具永远带、湿实验问题必带分子生物学工具）。默认开启（`config.py: use_tool_retriever=True`）。

#### (d) 若干「科研软件工程」细节

- **计划即 checklist**：system prompt 强制模型维护 `1. [ ]` / `[✓]` / `[✗]` 待办清单，每步后重新打印（`a1.py:1102-1119`）。
- **Commercial mode**：`env_desc.py`（76 数据集，学术）与 `env_desc_cm.py`（41 个，商用许可过滤）两套 action space，按 license 切换。know-how 也带 license 元数据做过滤。一般 agent 项目不会做这件事。
- **self-critic 是可选的 test-time scaling**：默认关闭，开启后 `<solution>` 之后 LLM 自我批评再回炉，`test_time_scale_round` 控轮数。
- **Biomni-R0**：在 Qwen-32B 上用 agent 交互数据做过 RL，即模型层也针对这个 loop 训练过。
- **Biomni-Eval1**：433 instances / 10 类生物推理任务的 benchmark。

**一句话总结：护城河是那 4 万行领域 action space 加一个要装 10 小时、占 30 GB 的 conda 环境，不是 agent 架构。**

---

## 第二部分：四个核心设计问题

### Q2.1 「把环境当成 action space」是什么意思？举个例子

#### 概念对比

**「工具即 action space」**（绝大多数 agent 框架的做法）：agent 能做的事 = 你注册的工具列表。动作空间是**可枚举的、有限的**。工具没覆盖到的需求就是死路。

**「环境即 action space」**（Biomni 的做法）：agent 能做的事 = 在一个装满了工具、数据、软件、知识的**计算环境里写任意代码**。动作空间是这四类资源的**组合**，是组合爆炸式的，不可枚举。

关键在于「写任意代码」这一步——它不属于任何一个工具，但它把所有资源粘起来。

#### 具体例子

拿 README 里的真实任务：

> "Plan a CRISPR screen to identify genes that regulate T cell exhaustion, generate 32 genes that maximize the perturbation effect."
> （设计一个 CRISPR 筛选，找出调控 T 细胞耗竭的基因，给出扰动效应最大的 32 个基因）

**如果是「工具即 action space」的 agent：**

它需要一个叫 `design_crispr_screen_for_t_cell_exhaustion()` 的工具。这个工具不存在。于是结局只有两种：要么摆烂做个网页搜索然后编 32 个基因名，要么直接说做不到。你可以去写这个工具——但下次用户问「B 细胞」「巨噬细胞」「不同的表型」，你又得写一个。工具粒度和任务粒度永远对不上。

**Biomni 的实际路径（四类资源交织）：**

1. **know-how**：检索到仓库里真实存在的 `biomni/know_how/sgRNA_design_guide.md`，全文进 context。模型于是知道 sgRNA 设计的实践规范（GC 含量、脱靶、每基因几条 guide）。
2. **数据湖**：从 76 个文件里挑出相关的读进内存。这个任务会用到——
   - `DepMap_CRISPRGeneEffect.csv`：全基因组 CRISPR 敲除效应
   - `DepMap_CRISPRGeneDependency.csv`：基因依赖概率
   - `DepMap_OmicsExpressionProteinCodingGenesTPMLogp1.csv`：细胞系表达谱
   - `affinity_capture-ms.parquet`：蛋白互作网络
3. **软件目录**：知道环境里装了 `scanpy`、`scikit-learn`、`pandas`，直接 `import`，不需要任何人给它包一个「跑 PCA 的工具」。
4. **工具**：调 `query_uniprot`、文献检索工具确认候选基因的功能注释和已有文献支持。
5. **⭐ 任意代码（关键一步）**：把上面所有东西 join 起来——按耗竭相关表达谱筛细胞系、算基因效应的效应量、用互作网络做通路富集、按 know-how 里的规则加约束、排序取前 32。

   **这一步不是任何一个工具。** 它是模型现场写的几十行 pandas + sklearn。没有人预先设计过「效应量 × 通路富集 × 表达过滤」这个特定组合。
6. **know-how 附带资源**：`biomni/know_how/resource/addgene_grna_sequences.csv` 里取出这 32 个基因对应的真实 sgRNA 序列。

第 5 步就是全部分歧所在。**工具是动作的原子，代码是动作的语法**——有了语法，有限的原子能生成无限的句子。这也是为什么 Biomni 需要持久 REPL（见 [Q2.3](#q23-python-解释器持久化有什么用好处是什么)）：第 2 步加载的 `DataFrame` 必须活到第 5 步。

#### 在代码里怎么体现

system prompt 把四类资源并列铺开，路径、描述一应俱全：

```
# biomni/agent/a1.py:1215
Environment Resources:

- Function Dictionary:
{tool_desc}

- Biological data lake
You can access a biological data lake at the following path: {data_lake_path}.
{data_lake_content}          ← 76 个文件，每个一行描述

- Software Library:
{library_content_formatted}  ← 113 个包，每个一行描述

- Note on using R packages and Bash scripts: ...
```

注意「data lake 在这个路径下」这句话——它没有给任何读数据的工具，只是**告诉模型数据在哪、是什么**，剩下的让模型自己写代码去读。这就是「环境」而非「工具」的思路：提供**可供性（affordance）**，不提供**接口**。

---

### Q2.1 补充：这和普通 coding agent 有什么区别？

一个很自然的质疑：「环境即 action space」听起来和 coding agent 干的事没什么两样。

**这个质疑基本上是对的，必须先承认。** Cursor / Claude Code / opencode 早就在做同一件事：

| Biomni 的四类资源 | coding agent 的对应物 |
| --- | --- |
| `<execute>` 写任意代码 | `bash` / `run_terminal_cmd` 工具 |
| 224 个工具函数 | 仓库里已有的函数，`grep` 出来 `import` |
| 113 条软件目录 | `pip list` / `requirements.txt` |
| 76 个数据湖文件 | 仓库里的文件，`ls` + `read_file` |
| know-how 文档 | `AGENTS.md` / `CLAUDE.md` / skills |

机制层面**没有任何新东西**：代码是万能胶水，把手头一切资源粘起来。`a1.py` 本质上就是一个为科学场景特化的 coding agent，它的 `<execute>` 加持久 REPL 等于「bash 工具 + Jupyter kernel」。

所以「环境即 action space」这个说法，**其实是把 coding agent 一直隐含的前提显式命名了而已**。没人给 coding agent 起这个名字，因为在代码领域这三个前提天然成立——而在科学领域它们全都不成立。这才是真正的区别所在。

#### 前提一：代码仓库自带描述，科学环境不自带

coding agent 靠**探索**发现自己有什么能力：`ls`、`grep`、读 `package.json`、看函数签名。这套办法之所以有效，是因为代码仓库**自我描述**——文件叫 `auth.py` 就是管认证的，函数叫 `parseConfig` 就是解析配置的，类型签名直接告诉你怎么调。

科学环境完全不是这样。`ls data_lake/` 给你的是：

```
affinity_capture-ms.parquet
DepMap_CRISPRGeneEffect.csv
czi_census_datasets_v4.parquet
```

`affinity_capture-ms` 是什么？你得知道 "affinity capture + mass spectrometry" 是一类检测**蛋白-蛋白相互作用**的实验方法，才能想到「哦这里面是 PPI 数据，我做通路分析可以用」。光看文件名，甚至加载进来看列名，都推不出这个结论——**它需要领域知识才能解读**。

`pip list` 同理：`viennarna`、`plannotate`、`pyscf`、`macs2` 这些名字，模型不会自发想到什么时候该用。

所以 Biomni 真正的产物不是「让模型写代码」（那是白送的），而是**那 76 行和 113 行一句话描述**——把一个不透明的文件堆变成一份可浏览的菜单。

说穿了：**Biomni ≈ 一个 coding agent + 一份为机器写的、带检索索引的科学环境 README。** 这个说法不如「环境即 action space」好听，但更准确。

#### 前提二：代码领域探索几乎免费，科学领域探索很贵

coding agent 的核心策略是「先探索，再动手」。这个策略成立的前提是探索**便宜且无副作用**：`grep -r "foo"` 几毫秒、不改任何东西、猜错了再 grep 一次就行。

科学领域的探索成本是另一个量级：

| 动作 | 代价 |
| --- | --- |
| 加载一个 3 GB parquet 看看相不相关 | 几分钟 |
| 跑一次 DFT 确认泛函选对了 | 几小时 + 机时预算 |
| 训一个模型看看特征有没有用 | 几十分钟 + GPU |
| 湿实验的一步 | 消耗试剂，**不可撤销** |

「先浏览一圈再决定」在这里行不通。**所以菜单必须预先给到，因为模型付不起逐个打开看的成本。** 这就为「把全部资源目录预载进 system prompt + 用检索控制预算」这个设计提供了非装饰性的理由——它不是审美选择，是成本结构逼出来的。

#### 前提三（最关键）：代码有免费的正确性裁判，科学没有

这条是三者里最深的，也最容易被忽略。

**coding agent 有一个近乎免费的 ground truth oracle**：测试跑过或跑不过、类型检查器报不报错、程序崩不崩、linter 满不满意。循环是**闭合**的。这正是 coding agent 效果这么好的根本原因——它能拿到廉价、可靠、自动的验证信号，然后对着信号迭代。

**科学 agent 没有这个东西。**

如果模型写了一段看起来完全合理的 pandas，但算基因必需性的时候用错了列、或者没处理批次效应，**什么都不会报错**。代码正常跑完，吐出一个数，那个数是错的。没有 `pytest` 能告诉你「这个统计检验选得对不对」「这个对照设置合不合理」。

一旦意识到这一点，Biomni 里很多看起来多余的设计立刻讲得通了 —— **它们全都是在人工补上那个缺失的裁判**：

| 设计 | 补的是什么 |
| --- | --- |
| **224 个 curated 工具** | 一个经过审核的 `query_uniprot()` 自带正确性保证，模型现场写的 API 调用没有 |
| **know-how 文档全文进 prompt** | 前置正确方法论，而不是等错误发生后再纠正（因为不会有人纠正） |
| **self-critic**（`a1.py:1579`） | 没有外部裁判，只好让模型自己当裁判 |
| **Eval1**（433 instances / 10 任务） | oracle 得手工造，不像测试套件那样白送 |
| **反编造的提示词护栏** | 见下 |

最后那条有个很直白的证据。`a1.py:1154` 的协议生成指令：

```
... but ONLY include information found in these resources.
Do not make up specifications, catalog numbers, or equipment details.
Prioritize accuracy over completeness.
```

**为什么要在提示词里专门写「不要编造货号」？** 因为编造出来的货号不会让任何东西崩掉。代码里编造一个不存在的函数名，`ImportError` 立刻打脸；协议里编造一个不存在的 Thermo 货号，一路畅通无阻，直到有人拿着它去下单。**没有编译器的领域，只能靠提示词当编译器。**

#### 所以真正的区别

> coding agent 也是「环境即 action space」——只不过它的环境（代码仓库）**自带描述、探索免费、且有测试当裁判**，所以这三件事从来不需要被专门命名。
>
> Biomni 做的事，是在这三个前提**全都不成立**的领域里，把它们人工补上：描述靠人写目录（76 + 113 行），探索成本靠预载 + 检索规避，正确性靠 curated 工具 + know-how + benchmark + 提示词护栏顶上。

这也解释了为什么 [第五部分](#第五部分用-opencode-做这样一个-agent-容易么) 的结论是「用 opencode 做计算材料 copilot 很容易」——因为计算材料任务里，你确实能把它压回一个代码仓库的形态（脚本、版本控制、可重跑），三个前提部分恢复了。反过来，越靠近湿实验、越依赖不可验证的领域判断，coding agent 那套就越不够用，需要补的东西越多。

**给自建 agent 的实践含义**：如果你要做材料化学方向，最值钱的投入顺序不是「造一个 agent 循环」（照抄即可），而是：

1. **给你的数据和软件写一份机器可读的目录**（对应前提一）——这是最被低估、也最不可替代的工作
2. **把不可验证的领域判断固化成 know-how 文档或 curated 工具**（对应前提三）——例如「XRD 定相该用什么容差」「什么时候必须做 Rietveld 精修」，这些是模型自己想不出、也不会被任何报错纠正的东西
3. **想清楚你的 oracle 是什么**（对应前提三）——能不能构造一批有标准答案的题？没有 oracle 的 agent 无法迭代改进，只能凭感觉

---

### Q2.2 数据湖是什么？为什么要有这个东西？

#### 是什么

物理上：`{path}/biomni_data/data_lake/` 下的 76 个文件，约 11 GB，首次创建 agent 时自动从 S3 下载。

```python
# biomni/agent/a1.py:163
check_and_download_s3_files(
    s3_bucket_url="https://biomni-release.s3.amazonaws.com",
    local_data_lake_path=data_lake_dir,
    expected_files=expected_data_lake_files,
    folder="data_lake",
)
```

逻辑上：每个文件在 `env_desc.py` 里配一行人类可读描述，这些描述会拼进 system prompt：

```python
# biomni/env_desc.py:2
data_lake_dict = {
    "affinity_capture-ms.parquet": "Protein-protein interactions detected via affinity capture and mass spectrometry.",
    "BindingDB_All_202409.tsv": "Measured binding affinities between proteins and small molecules for drug discovery.",
    "DepMap_CRISPRGeneEffect.csv": "Genome-wide CRISPR gene effect estimates for cancer cell lines, including all DepMap models.",
    "czi_census_datasets_v4.parquet": "Datasets from the Chan Zuckerberg Initiative's Cell Census.",
    ...
}
```

内容大致是：DepMap（癌症细胞系 CRISPR 筛选）、BindingDB（结合亲和力）、蛋白互作网络、单细胞数据集索引、药物重定位库、药物相互作用、GWAS 汇总统计等等——都是预先 ETL 成 parquet/CSV/TSV 的**统一格式副本**。

跳过下载的方式：`A1(path='./data', expected_data_lake_files=[])`。

#### 为什么要有

**理由一：解决「发现」问题，不只是存储问题。**

这是最容易被忽略但最重要的动机。LLM 不知道「你手上有 DepMap」。哪怕数据就在硬盘上，模型也不会主动想到用。把 76 行描述放进 system prompt，模型规划时的视野里才有这些数据。数据湖的一半价值在那 76 行**描述**，不在那 11 GB **文件**。

**理由二：生物数据的接入成本极高。**

生物医学数据散落在几十个数据库里，每个都有自己的 API、认证、限流、字段命名、ID 体系（Ensembl vs Entrez vs HGNC vs UniProt……），有的还要注册账号或签数据使用协议。让 agent 每次现场学一套 API 是不现实的——失败率高、token 消耗大、错误难恢复。预先 ETL 成统一格式落到本地，`pd.read_parquet()` 一行搞定。

**理由三：跨数据集 join 只有本地才做得到。**

`Q2.1` 的例子里要把「CRISPR 效应量」和「表达谱」和「蛋白互作」三个来源 join 起来。REST API 做不了 join，只能一条条查再自己拼；本地 parquet 用 pandas 是自然操作。**联合推理需要联合存储。**

**理由四：延迟与可复现性。**

本地 parquet 秒级读；API 查询几十秒、可能超时、可能限流、结果还会随时间变化。论文实验必须可复现，README 明确写了「This release was frozen as of April 15 2025」——冻结版本正是靠数据湖实现的。

#### 代价

- 11 GB 下载门槛，很多用户第一次上手就卡在这。
- **License 问题**：这直接催生了 commercial mode——`env_desc.py` 76 个数据集（学术），`env_desc_cm.py` 只剩 41 个（商用许可）。把非商用数据集注释掉了。
- **数据会过期**：冻结的代价就是不新鲜。
- 只对「数据密集 + API 混乱」的领域划算。

#### 对材料化学的启示（重要）

材料领域的数据库 API 化程度远好于生物：Materials Project 有官方 `mp-api`，还有 **OPTIMADE** 这种跨数据库统一查询标准（实测 28 家 provider / 45 个子库 / 2726 万条结构，MP、OQMD、AFLOW、NOMAD、COD 都实现了）。

**所以做材料版 agent 可以基本跳过数据湖这一步，走活 API。** 这恰好省掉了整个迁移工作里最重最贵的一环（数据托管 + 描述编写 + license 分层）。这也正是 ToolUniverse 选择的路线（见第三部分）。

> ⚠️ **这个判断需要打折。** 实测发现 OPTIMADE 只标准化了**结构与成分**，没有标准化**物性**——Materials Project 经 OPTIMADE 暴露的 27 个字段里没有形成能、也没有带隙。准确的说法是：可以跳过**结构数据**的托管（确实是最重的一环），但性质层仍需逐个对接各家原生 API，批量筛选时你终究会想建一份小型性质缓存。**是数量级减负，不是归零。**
> 详见 [OPTIMADE 笔记](./optimade-materials-data-notes.zh.md#️-重要限制标准化的是结构不是性质)。

---

### Q2.3 Python 解释器持久化有什么用？好处是什么？

#### 机制

一个模块级 dict，反复 `exec` 进去：

```python
# biomni/tool/support_tools.py:6-7
# Create a persistent namespace that will be shared across all executions
_persistent_namespace = {}

def run_python_repl(command: str) -> str:
    """Executes the provided Python command in a persistent environment and returns the output.
    Variables defined in one execution will be available in subsequent executions.
    """
    ...
    exec(command, _persistent_namespace)
```

不是子进程，不是每次新解释器——就是同一个进程、同一个命名空间，行为等价于一个 Jupyter kernel。

#### 好处

**1. 增量分析：昂贵的加载只做一次**

单细胞的 AnnData 动辄几个 GB，加载要几分钟。持久 REPL 下：

```python
# 第 3 步
adata = sc.read_h5ad('...')     # 花 5 分钟

# 第 7 步
sc.pp.neighbors(adata)          # 直接用

# 第 15 步
adata.obs.groupby('leiden')...  # 还在
```

无状态执行的话，每一步都得重新读盘。对生物数据这不是「慢一点」，是**根本跑不动**。

**2. 错误恢复极便宜（我认为这是最大的好处）**

长任务里失败是常态——列名拼错、维度不匹配、某个包的 API 变了。持久 REPL 下，第 12 步崩了，第 1-11 步的中间结果全都还在内存里，模型改一行重跑第 12 步就行。

无状态执行下，第 12 步崩了意味着要么整个脚本从头跑（贵，而且可能再崩在别处），要么模型得把前 11 步的状态重建出来（更容易出错）。

**对 20+ 步的任务，这一条直接决定成功率。**

**3. 上下文压缩：大对象留在内存，只有摘要进 prompt**

这是 CodeAct 相对 function-calling 的结构性优势。10 GB 的表达矩阵留在解释器里，进 context 的只有 `print(adata.shape)` 的那一行 `(50000, 20000)`。

对比 function-calling：工具返回值必须序列化成文本经过模型。返回一个大 DataFrame 就得截断，截断就丢信息，或者要设计一堆分页参数。持久 REPL 下这个问题不存在——数据从来不需要离开进程。

system prompt 里那条奇怪的规定就是为这个服务的：

```
# biomni/agent/a1.py:1135
When calling the existing python functions in the function dictionary,
YOU MUST SAVE THE OUTPUT and PRINT OUT the result.
For example, result = understand_scRNA(XXX) print(result)
```

「存下来」是为了后续步骤能用（好处 1、2），「打印出来」是为了模型能看见（好处 3）。一句话把两件事都要求了。

**4. 工具组合不需要框架支持**

工具 A 的输出直接是变量，喂给工具 B 就是 `b(a_result)`。不需要框架实现 state passing、变量引用、中间结果存储那一套机制。用 Python 自己的变量绑定就完事了。

**5. 符合科研人员的心智模型**

科学家本来就在 Jupyter 里工作。导出的执行日志天然是一份 research log（system prompt 也明确要求「print out the steps and results ... like a research log」），可以直接进补充材料。

#### 代价（同样重要）

**1. 超时控制被迫降级为 threading**

不能用 multiprocessing，因为子进程结束命名空间就没了。代码里明确写了这个取舍：

```python
# biomni/utils.py:183
def run_with_timeout(func, args=None, kwargs=None, timeout=600):
    """Run a function with a timeout using threading instead of multiprocessing.
    This allows variables to persist in the global namespace between function calls.
    """
```

后果：Python 里没法可靠地杀线程。超时后线程还在跑，只是不等它了——句柄泄漏、CPU 占用、后续步骤被拖慢都有可能。这是个真实的工程债。

**2. 状态污染，且难 debug**

第 3 步定义的 `df` 被第 9 步覆盖，第 14 步用到时行为诡异。模型没有「命名空间检查器」，只能靠 `print` 摸索。变量名冲突在长任务里是实际发生的问题。

**3. 不可重放**

同样的代码在不同 namespace 状态下结果不同。想复现一次运行，光有代码不够，还得有完整的执行顺序。这和数据湖追求的可复现性其实是矛盾的。

**4. 安全**

`exec` 跑在主进程，全权限。README 的警告不是客套：

> Currently, Biomni executes LLM-generated code with full system privileges. If you want to use it in production, please use in isolated/sandboxed environments.

想沙箱化就得整体上容器/gVisor，不能靠进程隔离——因为进程隔离恰好是被持久性排除掉的那个手段。

#### 小结

持久 REPL 是「**长时程数据密集科研任务**」这个场景下的正确选择：好处（增量分析、廉价错误恢复、上下文压缩）全都随任务步数和数据规模放大，代价（超时不干净、状态污染、沙箱困难）则相对固定。

反过来，如果任务是「查三个 API 然后总结」，持久性毫无价值，标准 function-calling 更简单更安全。**这是个场景决定的选择，不是普适的优劣。**

---

### Q2.4 「opencode 没有持久 REPL」具体是什么意思？举个例子

这是 [Q2.3](#q23-python-解释器持久化有什么用好处是什么) 的延伸，也是 [第五部分](#第五部分用-opencode-做这样一个-agent-容易么) 里那句「和 Biomni 最实质的差别」的展开。

#### 先说清一个容易混淆的层次问题

「持久」有两个完全不同的级别，混淆这两者会导致错误判断：

| 级别 | 持久的是什么 | 谁有 |
| --- | --- | --- |
| **① shell 会话持久** | 当前目录、环境变量、alias、shell 历史 | 传统终端、PTY、tmux |
| **② 解释器进程持久** | **内存里的 Python 对象**（DataFrame、模型权重、已加载的数据集） | Jupyter kernel、**Biomni** |

**关键在于：即使拿到了 ①，也拿不到 ②。**

因为 `python script.py` 这个进程一退出，进程里的所有对象就随之消失，和 shell 会话是否存活毫无关系。要拿到 ② 只有两条路：常驻一个 Python 解释器进程把代码送进去（Biomni 的做法：`exec(code, _persistent_namespace)`），或者连一个常驻的 Jupyter kernel。

opencode 的现状是**连 ① 都没有**。它的 `bash` 工具（`packages/opencode/src/tool/bash.ts`）每次调用都通过 `ChildProcessSpawner` 新起一个子进程，所以 `cd`、`export`、`source venv/bin/activate` 都不跨调用生效（社区有 issue [sst/opencode#23449](https://github.com/anomalyco/opencode/issues/23449) 在推 PTY 方案，[#6488](https://github.com/sst/opencode/issues/6488) 有详细调查）。Claude Code 略好一点——官方文档说「working directory persists between commands; shell state (everything else) does not」，即有 ① 的一部分。

但**这个区别对我们要讨论的事情不重要**，因为无论 ① 有没有，② 都没有。所以下面的对照例子对 opencode 和 Claude Code 同等适用。

#### 对照例子：一个材料筛选任务

任务（用你关心的材料方向举例）：

> 从 Materials Project 拉所有含锂的氧化物，算 Magpie 描述符，训一个形成能预测模型，看哪些特征重要，挑 20 个候选。

##### Biomni 版本（持久 REPL）

**轮 1** —— 拉数据，这一步很贵：

```python
<execute>
from mp_api.client import MPRester
with MPRester(API_KEY) as m:
    docs = m.materials.summary.search(
        elements=["Li", "O"],
        fields=["material_id", "formula_pretty", "formation_energy_per_atom", "band_gap"],
    )
print(len(docs))
</execute>
```

`<observation>` → `3184`　（耗时约 4 分钟）

**轮 2** —— 看看数据长什么样：

```python
<execute>
import pandas as pd
df = pd.DataFrame([d.model_dump() for d in docs])
print(df.shape)
print(df.describe())
</execute>
```

注意：`docs` 直接用，**没有重新下载**。

**轮 3** —— 算描述符：

```python
<execute>
from matminer.featurizers.composition import ElementProperty
ep = ElementProperty.from_preset("magpie")
X = ep.featurize_dataframe(df, col_id="composition")
</execute>
```

`<observation>` → `KeyError: 'composition'`　**← 崩了**

**轮 4** —— 修一行：

```python
<execute>
from pymatgen.core import Composition
df["composition"] = df["formula_pretty"].apply(Composition)
X = ep.featurize_dataframe(df, col_id="composition")
print(X.shape)
</execute>
```

**这是全场最关键的一步**：`df` 还在内存里，`ep` 也还在，模型只需要补一列然后重跑失败的那一行。**那 4 分钟的下载没有重来。**

**轮 5-6** —— 训模型、看特征重要性、画图，全程直接用 `X`、`df`、训好的 `model` 对象。用户随口一句「刚才那个 X，把 top 20 特征重要性画出来」，模型一行 `plot` 就完事。

##### opencode / Claude Code 版本（无持久解释器）

**轮 1** —— 必须写成脚本，而且**必须显式想到存盘**：

```bash
cat > 01_fetch.py <<'EOF'
from mp_api.client import MPRester
import pandas as pd
with MPRester(API_KEY) as m:
    docs = m.materials.summary.search(elements=["Li","O"], fields=[...])
df = pd.DataFrame([d.model_dump() for d in docs])
df.to_parquet("cache/raw.parquet")      # ← 这一行是生死线
print(df.shape)
EOF
python 01_fetch.py
```

如果模型**忘了写 `to_parquet`**（这在实践中经常发生，因为当下那一步并不需要它），那 4 分钟就白花了，下一步得从头再下载。

**轮 2** —— 新进程，一切从磁盘恢复：

```bash
cat > 02_featurize.py <<'EOF'
import pandas as pd
from matminer.featurizers.composition import ElementProperty
df = pd.read_parquet("cache/raw.parquet")     # ← 重新读
ep = ElementProperty.from_preset("magpie")
X = ep.featurize_dataframe(df, col_id="composition")
X.to_parquet("cache/features.parquet")        # ← 又一条生死线
EOF
python 02_featurize.py
```

`KeyError: 'composition'` —— 同样崩了。改脚本重跑。因为轮 1 存了 parquet，只重跑 featurize，还行。

**轮 6** —— 用户说「刚才那个 X，画个特征重要性」：

```bash
cat > 04_plot.py <<'EOF'
import pandas as pd, joblib
X = pd.read_parquet("cache/features.parquet")   # 恢复
model = joblib.load("cache/model.pkl")          # 恢复（前提是轮 5 存了）
...
EOF
python 04_plot.py
```

**每一次「再看一下」都要付一次反序列化的代价，而且前提是上一步恰好存了你现在需要的东西。**

#### 差别到底在哪：三点

**1. 不是「能不能做」，而是「谁来管状态」**

opencode 完全能完成这个任务。真正的区别是：**Biomni 把状态隐式地留在内存里，opencode 要求你显式地把状态序列化到磁盘。**

这个转换带来的成本是**认知负担前移**——模型在写第 1 步的时候，就必须预见到「第 4 步会需要这个中间结果」，从而决定存不存、存成什么格式。预见错了就得重跑。持久 REPL 下这个决策根本不存在，因为默认全都留着。

**2. 有些东西根本序列化不了（或者代价极高）**

这是最硬的限制。列一下科研场景里常见的、跨不过进程边界的状态：

| 状态类型 | 能落盘吗 |
| --- | --- |
| DataFrame / ndarray | ✔ parquet / npy，快 |
| sklearn 模型 | ✔ joblib |
| pymatgen `Structure` 列表 | △ 能 `as_dict()`，但笨重且慢 |
| **已加载到 GPU 的模型权重** | ✘ **每次重新加载，几分钟 + 几个 GB** |
| 打开的数据库连接 / API session（含鉴权态） | ✘ 必须重连重新鉴权 |
| backed 模式的 `AnnData` / h5ad handle | ✘ 句柄不可序列化 |
| matplotlib figure 的交互状态 | ✘ |
| 随机数状态、CUDA 上下文 | ✘ |

**GPU 模型权重那一行对材料方向特别致命。** 如果你用 CHGNet / M3GNet / MACE 这类机器学习势做结构松弛，加载模型到显存要几十秒到几分钟。持久 REPL 下加载一次用一整个 session；无状态模式下**每次调用都要重新加载**。做 100 个结构的批量松弛，这个差别就是几分钟 vs 几小时。

**3. opencode 版本反而更可复现——这是它的优势**

必须公平地说：那几个 `01_fetch.py` / `02_featurize.py` / `03_train.py` 加起来**就是一条完整、可重跑、可进补充材料的 pipeline**。

Biomni 那 6 轮对话的 log 你没法直接重跑——因为存在顺序依赖和隐式状态，同一段代码在不同的 namespace 状态下结果可能不同（这正是 [Q2.3 代价](#q23-python-解释器持久化有什么用好处是什么) 里「不可重放」那一条，而且它和数据湖追求的可复现性其实是矛盾的）。

**所以这是一个真实的取舍，不是 opencode 的缺陷**：

| | 持久 REPL（Biomni） | 落盘脚本（opencode） |
| --- | --- | --- |
| 探索速度 | 快，随口就能追问 | 慢，每次要恢复状态 |
| 错误恢复 | 便宜，改一行重跑一步 | 看运气，取决于上一步存了没 |
| 昂贵对象（GPU 权重等） | 加载一次 | 每次重载 |
| 可复现性 | 差，需要完整执行顺序 | **好，脚本自带即是 pipeline** |
| 沙箱化 | 难（不能靠进程隔离） | **易，天然进程隔离** |

**探索期想要前者，交付期想要后者。** 成熟的做法是探索用 Biomni 式的持久环境，定稿后让 agent 把过程整理成独立脚本。

#### 想在 opencode 里补上持久 REPL，怎么办

这个缺口是可以填的，而且填法很直接：**写一个 MCP server，内部维护一个常驻的 Python 解释器或 Jupyter kernel，对外暴露一个 `run_python(code)` 工具。**

- 用 `jupyter_client` 连一个常驻 kernel（工程上最稳，能拿到富输出、能中断、生态成熟），或者照 Biomni 那样简单粗暴地 `exec` 进一个模块级 dict（`biomni/tool/support_tools.py:6-7`）。
- 挂进 `opencode.json` 的 `mcp` 段即可。
- 这样就把 Biomni 的核心机制搬到了 opencode 上，而且是可复用资产——同一个 server 在 Claude Code、Cursor 里都能用（延续[第五部分](#建议把工具写成-mcp-server别赌框架)「别赌框架，押注 MCP」的思路）。
- 记得把 Biomni 踩过的坑一并处理：超时（不能用 multiprocessing，否则命名空间就没了，见 `biomni/utils.py:183` 的注释）、输出截断、以及**沙箱**（`exec` 全权限跑在你的机器上，要上容器）。

> ⚠️ 这个方案我只做了架构判断，**没有实现和验证**。社区已有若干 Jupyter MCP server 项目，选型前建议先看看现成的。

---

## 第三部分：和 ToolUniverse 的区别

对比对象：[mims-harvard/ToolUniverse](https://github.com/mims-harvard/ToolUniverse)，Harvard Medical School Zitnik Lab，论文 [Democratizing AI scientists using ToolUniverse](https://arxiv.org/abs/2509.23426)（Gao et al., 2025）。下面数字是我 clone 仓库后实测统计的。

### 一句话概括

**Biomni 是一个 agent（有大脑，有循环）；ToolUniverse 是一个 tool 层（只有手，没有大脑，大脑你自己接）。**

这不是程度差别，是范畴差别。ToolUniverse 的依赖里**没有 langgraph、没有 langchain、没有任何 StateGraph**——我 grep 过，零命中。它的交付物是一个 MCP server（`src/tooluniverse/smcp.py`）；agent loop 由你选的客户端提供（Claude Code、Gemini CLI、smolagents，论文里演示的是 Gemini CLI）。

论文自己的类比很准确：**「像 HTTP 标准化了客户端-服务器通信一样，ToolUniverse 定义了 AI 模型如何发起工具请求」**。它想做的是协议层，不是应用。

### 规模对比（实测）

| 维度 | Biomni | ToolUniverse |
| --- | --- | --- |
| **agent 循环** | 自带（LangGraph，3 节点） | **无**，自己接（Claude Code / Gemini CLI / smolagents） |
| 工具数 | 224（22 个模块） | 2687 个 JSON spec / 633 个文件（官方口径「1000+ 工具」，差值来自 API key 目录等非工具条目） |
| Python 文件数 | ~50 | **3379** |
| 代码行数 | 4.9 万 | **37.8 万** |
| 工具定义方式 | Python 函数 + 手写 Python schema（双文件，必须改 `read_module2api()` 的硬编码列表） | 声明式 **JSON spec** + `type` 字段指向 handler 类 |
| 最大工具类别 | `database` 40 | `BaseRESTTool` **263**（纯 JSON，零 Python 代码）、`FDADrugLabel` 152、`OpenTarget` 72 |
| 部署 | conda，**>10 小时，30 GB** | `pip install tooluniverse`，`python:3.12-slim` Docker |
| 本地数据 | **11 GB 数据湖**，76 文件 | **无**，全部走活 API |
| 代码执行 | 核心机制（持久 REPL + R + Bash） | 只是**一个工具**（`python_executor_tool.py`，无状态） |
| Skills | 2 篇 know-how 文档 | **153 个 skills** + 完整 Claude Code plugin |
| 模型训练 | Biomni-R0（Qwen-32B RL 微调） | 明确「无需训练或微调」（另有 TxAgent 是训练过的，属另一篇工作） |
| Benchmark | Biomni-Eval1（433 instances / 10 任务） | 高胆固醇血症 case study |

### 架构分歧一：工具发现的时机（最重要的差别）

**Biomni：循环开始前一次性检索，之后固定。**

```python
# biomni/agent/a1.py:1769
if self.use_tool_retriever:
    selected_resources_names = self._prepare_resources_for_retrieval(prompt)
    self.update_system_prompt_with_selected_resources(selected_resources_names)

inputs = {"messages": [HumanMessage(content=prompt)], "next_step": None}
for s in self.app.stream(inputs, ...):   # ← 图从这里才开始跑
```

检索发生在 `go()` 里、进图**之前**。整个任务期间工具集不变。

**ToolUniverse：把发现本身做成工具，agent 随时可调。**

`src/tooluniverse/data/finder_tools.json` 里有四个：

| 工具名 | 实现 | 机制 |
| --- | --- | --- |
| `Tool_RAG` | `ToolFinderEmbedding` | 向量检索（faiss-cpu） |
| `Tool_Finder` | `ToolFinderEmbedding` | 同上，功能更多 |
| `Tool_Finder_LLM` | `ToolFinderLLM` | LLM in-context 选择 |
| `Tool_Finder_Keyword` | `ToolFinderKeyword` | 关键词 + 停用词 + 词干还原 |

再配上 **Compact Mode**——把 1000+ 工具压缩成 4-5 个发现工具暴露给模型，官方说省约 99% context window。

**后果对比：**

- Biomni 一次任务能用哪些工具是第 0 轮就定死的。猜错了没有补救机会。好处是模型规划时**一次看到全部相关资源**（工具 + 数据 + 软件 + know-how 并列），视野完整，适合「先做全局计划再执行」。
- ToolUniverse 可以中途发现「我还需要个专利检索工具」然后现找。好处是能应对计划外分支，且 context 占用极小。代价是模型 turn-0 不知道自己有什么能力，规划偏保守。

这两条路各有道理，本质是**静态全局视野 vs 动态按需发现**的取舍。

### 架构分歧二：工具怎么定义

**Biomni** 是双文件模式，两边都要写 Python：

```
biomni/tool/genomics.py                   ← 实现
biomni/tool/tool_description/genomics.py  ← 手写 schema（description list）
biomni/utils.py:846                       ← 还得把模块名加进硬编码的 fields 列表
```

**ToolUniverse** 是声明式的。一个工具 spec 长这样：

```json
{
  "name": "ADMETAI_predict_physicochemical_properties",
  "description": "Predicts physicochemical properties ... for a given list of molecules in SMILES format.",
  "parameter": {
    "type": "object",
    "properties": { "smiles": { "oneOf": [...], "description": "SMILES string(s) ..." } },
    "required": ["smiles"]
  },
  "type": "ADMETAITool",
  "required_packages": ["admet_ai"],
  "test_examples": [{ "smiles": ["CC(=O)OC1=CC=CC=C1C(=O)O"] }],
  "return_schema": ...
}
```

`type` 指向一个 handler 类。关键在于**263 个工具的 type 是通用的 `BaseRESTTool`**——意思是这 263 个工具**完全没有专属 Python 代码，纯 JSON 声明就跑起来了**。还有 `graphql_tool.py`、`xml_tools.json`（19 个）走同样的路子。

对包装 REST API 这件事，ToolUniverse 的边际成本比 Biomni 低一个数量级。spec 里还带 `test_examples` 和 `return_schema`，是可自动测试的。

### ToolUniverse 独有的机制

**1. 工具自我生成与优化（Tool Discoverer / Optimizer）**

`src/tooluniverse/data/tool_discovery_agents.json` 里是一整套多 agent 流水线：

- `ToolDiscover`（ComposeTool）：从一句话描述生成符合规范的新工具，**代码和 spec 同时生成**
- `UnifiedToolGenerator`、`XMLToolOptimizer`：迭代优化
- `TestResultsAnalyzer`、`CodeQualityAnalyzer`、`PackageAnalyzer`：自动测试与质量评估
- `dynamic_package_discovery`：现场搜 PyPI 找合适的包

这是一个「**会自己长大的工具库**」。Biomni 侧最接近的是 `add_tool()` 用 LLM 从函数源码生成 schema——只有生成 schema，没有生成实现、没有测试、没有迭代优化。差距很大。

**2. AgenticTool：agent 作为工具嵌套**

50 个工具的 type 是 `AgenticTool`，本身就是 LLM 调用。例如 `drug_discovery_agents.json` 里的 `DiseaseAnalyzerAgent`、`ADMETAnalyzerAgent`、`ClinicalTrialDesignAgent`。也就是「工具」这个抽象被推广到包含了「子 agent」。

**3. Tool Composer**

`compose_tool.py`：把多个工具组装成一个复合工作流工具，之后可以当单个工具调用。

**4. 153 个 Skills + Claude Code plugin**

`skills/` 下 153 个目录，`tooluniverse-admet-prediction`、`tooluniverse-antibody-engineering` 之类的领域工作流，还有 `devtu-create-tool`、`devtu-self-evolve` 这类开发元技能。

`plugin/` 下是一个完整的 Claude Code 插件：`agents/researcher.md`、8 个 slash commands（`/research`、`/literature-sweep`、`/cross-validate`、`/verify-references`……）、`hooks/hooks.json`、`settings.json`（预设 `find_tools` 等只读工具自动批准）。还有 `mcpb/` 做 MCP bundle 打包。

**形态上它明确是「给现有 coding agent 装备科学能力」，而不是「另做一个 agent」。**

**5. human-in-the-loop 与 safety 是一等公民类别**

论文把「human feedback」和「safety tools」列为独立工具类别。Biomni 没有对应设计。

**6. 高度依赖外部 API 授权**

有 39 个 `secret` 类型条目 + 45 条 `api_keys_catalog.json`。这是活 API 路线的必然代价：要配一堆 key。

### Biomni 独有的机制

1. **自带 agent 循环 + 持久 REPL + 多语言执行**（Python/R/Bash）。ToolUniverse 的 `python_executor_tool.py` 是把代码执行**当成一个工具**（方向反了），且无跨调用状态。[Q2.3](#q23-python-解释器持久化有什么用好处是什么) 讲的那些好处 ToolUniverse 拿不到。
2. **数据湖**。ToolUniverse 完全没有——它假设一切都能通过 API 拿到。
3. **Commercial mode**：按 license 分层的 action space。
4. **Biomni-R0**：针对自己这个 loop 做过 RL 的 32B 模型。ToolUniverse 走纯 in-context 路线（「no additional training or finetuning」）。
5. **Eval1 benchmark**：433 instances 的系统评测。
6. **计划 checklist 机制**：强制维护待办清单。

### 哲学差异

| | Biomni | ToolUniverse |
| --- | --- | --- |
| 定位口号 | "A **General-Purpose Biomedical AI Agent**" | "**Democratizing** AI scientists" |
| 交付物 | 一个完整系统 | 一个协议 + 一个工具目录 |
| 整合方式 | **垂直整合**（模型、loop、工具、数据、环境全包） | **水平平台**（只做工具层，模型和 loop 你选） |
| 类比 | 一台配置好的工作站 | 一套标准接口的外设总线 + 巨大外设库 |
| 优势 | 端到端体验一致、长任务能力强、可复现 | 上手成本低、工具规模大、可组合、能自我扩张 |
| 弱点 | 部署重（10h/30GB）、工具规模小一个量级、扩张慢 | 无状态、无长时程分析能力、依赖一堆 API key、能力上限取决于你接的客户端 |

有意思的是两边都在向对方靠：Biomni 加了 `add_mcp()` 和 `create_mcp_server()`（开始拥抱协议），ToolUniverse 加了 skills、`AgenticTool` 和 plugin（开始有 agent 味道）。

### 实际选型建议

**如果任务是「查询 + 检索 + 预测」类**（找靶点、查文献、算 ADMET、验专利）：**ToolUniverse**。工具覆盖广一个量级，`pip install` 就能用，接 Claude Code 或 Gemini CLI 直接干活。

**如果任务是「数据密集的多步分析」**（scRNA-seq 全流程、跨数据集联合建模、需要 20+ 步且中间态很贵）：**Biomni**。持久 REPL + 数据湖是为这个场景造的，ToolUniverse 结构上给不了。

**如果要发论文做一个新领域的 agent**：想清楚贡献点是什么。「curated environment + benchmark」→ 学 Biomni；「工具生态 + 协议」→ 学 ToolUniverse 或者直接给它贡献工具。

**最实用的组合：两个一起用。** Biomni 有 `add_mcp(config_path)`，ToolUniverse 交付物就是 MCP server（Dockerfile 里跑的就是 stdio transport 的 MCP server）。所以架构上可以让 Biomni 的 loop 直接消费 ToolUniverse 的 2000+ 工具:

- Biomni 提供：持久 REPL、数据湖、计划机制、R/Bash 执行
- ToolUniverse 提供：工具目录、Tool Finder、活 API 接入

> ⚠️ 这个组合我只做了架构可行性判断（接口对得上），**没有实际跑通验证**。真做的时候要注意：Biomni 的 prompt-based retriever 面对 2000+ 工具会爆 context，得改成调用 ToolUniverse 的 `Tool_Finder`，而不是把工具列表全铺进 prompt。

---

## 第四部分：能否改成材料化学方向

**能，而且比预想的顺。** 按层评估本仓库：

### 几乎零改动可复用的「底盘」

`a1.py` 的图循环、代码执行原语、`ToolRetriever`、`llm.py`（支持 OpenAI/Azure/Anthropic/Ollama/Gemini/Bedrock/Groq/Custom 八种 provider）、`config.py`、Gradio UI、PDF 导出，以及全套运行时扩展 API：

```python
add_tool(func)         # 自动用 LLM 从源码生成 schema，注册 + 注入 REPL
add_data(dict)         # 扩数据湖目录
add_software(dict)     # 扩软件目录
add_mcp(config.yaml)   # 挂 MCP server，异步工具自动包成同步函数
create_mcp_server()    # 反向：把 Biomni 工具暴露成 MCP
```

### 必须重写的「领域内容」

224 个工具里约 74% 是深度生物医学的（光 `database.py` 就 4975 行）；76 个数据湖文件全部作废；conda 环境里的 Bioconductor（DESeq2、clusterProfiler、WGCNA）、序列比对（BLAST、samtools、bowtie2、bwa）、群体遗传 CLI（PLINK2、GCTA、IQ-TREE、HOMER）全部无用；`biomni/task/` 和 `biomni/eval/` 的 benchmark 也是生物的。

### 一个意外的好消息

**分子化学的地基已经在了。** 现有环境已装 `rdkit`、`openbabel`、`openmm`、`pymol`、`biotite`、`descriptastorus`、`pytdc`，以及 AutoDock/Vina 对接工具链。

分子/有机化学方向几乎可以直接起飞；真正空白的是**无机固态材料**——没有 pymatgen、ASE、matminer、pyscf、phonopy、LAMMPS，也没有 VASP / Quantum ESPRESSO 接口。

### 路径 A：不 fork，纯运行时改造（做原型）

```python
from biomni.agent import A1

# 跳过 11GB 生物数据湖下载
agent = A1(path='./data', llm='claude-sonnet-4-5', expected_data_lake_files=[])

agent.add_software({
    'pymatgen': '[Python Package] 晶体结构分析、相图、电子结构后处理',
    'ase':      '[Python Package] 原子模拟环境，DFT/MD calculator 统一接口',
    'matminer': '[Python Package] 材料描述符与特征工程',
})
agent.add_tool(query_materials_project)      # 你自己写的函数，schema 自动生成
agent.add_data({'my_xrd_library.parquet': '实验 XRD 图谱与相归属'})
```

自定义资源会进 system prompt 的 "PRIORITY CUSTOM RESOURCES" 区块，优先级高于默认资源。今天就能跑。

**要注意有三处领域字符串硬编码，需覆盖：**

| 位置 | 内容 |
| --- | --- |
| `biomni/agent/a1.py:1099` | `You are a helpful biomedical assistant assigned with the task of problem-solving.` |
| `biomni/agent/a1.py:1226` | `You can access a biological data lake at the following path: ...` |
| `biomni/model/retriever.py:30` | `You are an expert biomedical research assistant. ...`（且下面的启发式规则 2/3/4 是生物专用的） |

### 路径 B：真 fork，做一个「Matmni」（要发论文的话）

按侵入性排序：

1. 新建 `biomni/tool/{crystallography,dft,electrochemistry,spectroscopy,synthesis,materials_db}.py` 及镜像的 `tool_description/*.py`，然后**把模块名加进 `biomni/utils.py:846` 那个硬编码的 `fields` 列表**——这是全仓库唯一一处真正的摩擦点，加领域必须改这行。
2. 重写 `env_desc.py` 两个字典。数据源换成 Materials Project（`mp-api`）、OQMD、AFLOW、NOMAD、COD、JARVIS；软件换成 pymatgen / ASE / matminer / pyiron / phonopy / pycalphad / OVITO，以及 CHGNet、M3GNet、MACE 这类机器学习势。
3. 改 system prompt 和 retriever prompt 的领域措辞与启发式（见上表三处）。
4. 新写 conda env yml——**会比生物版轻很多**，那 10 小时安装时间主要是 R 包和 CLI 工具贡献的。
5. 补 know-how 文档：Rietveld 精修、XRD 定相、TGA/DSC 解读、循环伏安、扣电装配等。markdown + 元数据 + license 的格式很适合材料实验知识。

**关键判断：材料领域有 OPTIMADE 这种跨数据库统一 API（实测 2726 万条结构可跨库统一查询），加上 MP / NOMAD 都有成熟在线 API，所以不需要托管 11 GB 级别的本地结构数据湖。** 这省掉了迁移工作里最重的一环（见 [Q2.2](#q22-数据湖是什么为什么要有这个东西) 结尾）。

但要注意分层：**OPTIMADE 管「结构发现」，不管「性质获取」**——MP 经 OPTIMADE 不提供形成能和带隙，那些得回退到 `mp-api`。所以性质层仍需适配，批量筛选时大概仍要一份小型本地缓存。完整分析见 [OPTIMADE 笔记](./optimade-materials-data-notes.zh.md)。

### 相关先行工作

ChemCrow（化学工具 agent）、Coscientist（自动化实验）、El Agente（量子化学），模型侧有微软 MatterGen / MatterSim。Biomni 的差异化在「curated environment + retrieval + 大规模 benchmark」这套方法论；搬到材料上，贡献点也应落在 action space 的构建，不是 loop。

---

## 第五部分：用 opencode 做这样一个 agent 容易么

**结论：做「计算材料科研 copilot」非常容易；做「Biomni 那种 general-purpose 科学 agent」不合适。**

opencode（`sst/opencode`）的四个扩展点：

| 扩展点 | 触发方式 | 位置 |
| --- | --- | --- |
| Agents | Tab 键切换 / `@name` 提及 | `.opencode/agents/*.md` |
| Skills | 模型按描述自动发现 | `.opencode/skills/*/SKILL.md` |
| Plugins | 生命周期事件（工具调用、文件编辑） | `.opencode/plugins/*.ts` |
| MCP | prompt 驱动的外部工具 | `opencode.json` → `mcp` |

### opencode 白送给你的

TUI、会话管理、多 provider、权限系统、LSP，以及最关键的 `bash` 工具——**这本身就是 CodeAct**。搭一个材料 agent 原型只需要：一个 markdown agent 文件 + 几篇 skills 当 know-how + 一个暴露 pymatgen/ASE 的 MCP server + 一份 `opencode.json`。这是配置工作量，不是软件项目工作量。

### 会失掉的三样东西

1. **没有持久 Python REPL**。opencode 的 `bash` 工具每次调用都新起子进程，跨轮变量不存活（连 `cd`、`export` 都不跨调用生效）。长分析得写成 `.py` 落盘再跑——某种意义上更可复现，但 [Q2.3](#q23-python-解释器持久化有什么用好处是什么) 里那些好处（尤其是廉价错误恢复）都拿不到。这是和 Biomni 最实质的差别。
   **→ 详细对照例子、以及怎么用一个 MCP server 把这个缺口补上，见 [Q2.4](#q24-opencode-没有持久-repl具体是什么意思举个例子)。**
2. **没有资源检索层**。opencode 靠 skills 描述匹配 + 你的 prompt 管理上下文。真有 200+ 工具会爆 context，得自己写插件实现检索。
3. **没有 environment / 数据湖概念**。conda 环境和数据自己管，也没有 commercial mode 那种 license 过滤。

另外 opencode 本质是**编码** agent，默认提示词和 UX 都假设你在代码仓库里。算材料（DFT、MD、pymatgen 后处理）非常贴合；湿实验协议生成、实验室自动化这类场景比较别扭。

### 建议：把工具写成 MCP server，别赌框架

Biomni 有 `add_mcp()`，opencode 原生支持 `mcp` 配置，ToolUniverse 本身就是个 MCP server——**同一个 MCP server 三边都能用**。所以：

1. 先把材料领域工具（Materials Project 查询、结构生成、VASP 输入生成、XRD 模拟、描述符计算……）实现成一个独立 MCP server。这部分是真正的知识资产，与框架无关。
2. 用 opencode 挂上它做日常科研，验证工具好不好用、缺什么。
3. 若目标是发表 general-purpose 材料 agent，再 fork Biomni，把验证过的工具按双文件规范沉淀进去，配 benchmark。

这样框架选择不再是一次性押注，先攒的是工具和 know-how，哪边都不浪费。

---

## 附录：关键代码位置速查

### Biomni（本仓库）

| 关注点 | 位置 |
| --- | --- |
| Agent 主体（3001 行） | `biomni/agent/a1.py` |
| LangGraph 图定义 | `biomni/agent/a1.py:1606` |
| AgentState | `biomni/agent/a1.py:51` |
| System prompt 主体 | `biomni/agent/a1.py:1099-1258` |
| `<execute>` / `<solution>` 说明 | `biomni/agent/a1.py:1121-1142` |
| 计划 checklist 机制 | `biomni/agent/a1.py:1102-1119` |
| 标签解析（含 `<think>`） | `biomni/agent/a1.py:1419-1443` |
| execute 节点（Python/R/Bash 分派） | `biomni/agent/a1.py:1471-1517` |
| self-critic | `biomni/agent/a1.py:1579-1602` |
| `go()`（含一次性检索） | `biomni/agent/a1.py:1759` |
| `add_tool` | `biomni/agent/a1.py:225` |
| `add_mcp` | `biomni/agent/a1.py:350` |
| `add_data` / `add_software` | `biomni/agent/a1.py:676` / `:777` |
| `create_mcp_server` | `biomni/agent/a1.py:1995` |
| 数据湖 S3 下载 | `biomni/agent/a1.py:163` |
| 持久 REPL | `biomni/tool/support_tools.py:6-7` |
| threading 超时（及原因注释） | `biomni/utils.py:183` |
| **`read_module2api()` 硬编码领域列表** | `biomni/utils.py:846` ← 加新领域必改 |
| 自定义函数转 schema | `biomni/utils.py:251` |
| 资源目录（学术 / 商用） | `biomni/env_desc.py` / `biomni/env_desc_cm.py` |
| Prompt-based 检索 | `biomni/model/retriever.py:14` |
| 工具注册表 | `biomni/tool/tool_registry.py:24` |
| Know-how 加载器 | `biomni/know_how/loader.py` |
| 替代 agent（function-calling 版） | `biomni/agent/react.py` |
| 环境安装脚本 | `biomni_env/setup.sh`、`bio_env.yml`、`fixed_env.yml`（742 行） |

### ToolUniverse（对比参考）

| 关注点 | 位置 |
| --- | --- |
| 主入口 / Tool Caller | `src/tooluniverse/execute_function.py` |
| MCP server | `src/tooluniverse/smcp.py` |
| Tool Finder（三种） | `tool_finder_embedding.py` / `tool_finder_llm.py` / `tool_finder_keyword.py` |
| Finder 工具 spec | `src/tooluniverse/data/finder_tools.json` |
| 工具自我生成 | `src/tooluniverse/data/tool_discovery_agents.json`、`compose_tool.py` |
| 通用 REST handler（263 个工具共用） | `src/tooluniverse/base_rest_tool.py` |
| 代码执行（作为工具） | `src/tooluniverse/python_executor_tool.py` |
| 全部工具 spec | `src/tooluniverse/data/*.json`（633 个文件） |
| Skills（153 个） | `skills/` |
| Claude Code plugin | `plugin/`（agents / commands / hooks / settings.json） |

---

## 核心结论汇总

1. **Biomni 的价值不在 agent 架构**（LangGraph 三节点，一两天能抄完），在于那 4 万行 curated 领域 action space 和 30 GB 的环境。
2. **「环境即 action space」的要点是让模型写任意代码去组合工具、数据、软件、知识**——工具是原子，代码是语法，有语法才有无限的表达力。
3. **但这个机制和 coding agent 并无二致**，Biomni 本质就是一个科学特化的 coding agent。真正的区别在三个前提：代码仓库自带描述而科学环境不自带（所以要人写目录）；代码探索免费而科学探索很贵（所以要预载 + 检索）；**代码有测试当正确性裁判而科学没有**（所以要 curated 工具 + know-how + benchmark + 反编造提示词护栏）。第三条最深——`a1.py:1154` 专门写「不要编造货号」，正是因为编造的货号不会让任何东西崩掉。
4. **数据湖一半的价值在那 76 行描述**（解决「发现」问题），不在 11 GB 文件；但它只对「数据分散 + API 混乱」的领域划算。材料领域有 [OPTIMADE](./optimade-materials-data-notes.zh.md) 让上游统一了 API，**结构数据这一层可以跳过；但性质层没有标准化（MP 经 OPTIMADE 不给形成能和带隙），仍需适配 + 小型缓存**。
5. **持久 REPL 最大的好处是廉价的错误恢复**，其次是上下文压缩；代价是超时控制被迫降级和沙箱困难。场景决定取舍，不是普适优劣。
6. **「shell 会话持久」和「解释器进程持久」是两个级别，拿到前者也拿不到后者**（`python x.py` 一退出，对象就随进程消失）。Claude Code 只有第一级的一部分（cwd），opencode 连第一级都没有（每次命令新起子进程）。后果是每一步都要显式落盘，而昂贵对象（GPU 上的 ML 势权重、数据库连接、h5ad 句柄）根本跨不过进程边界。反过来，落盘脚本天然可复现——**探索期要持久，交付期要落盘**。这个缺口可以用一个内部常驻 kernel 的 MCP server 补上。
7. **Biomni 与 ToolUniverse 是范畴不同的东西**：一个是垂直整合的 agent，一个是水平的工具协议层。ToolUniverse 工具规模大一个量级且部署极轻；Biomni 有状态、有数据、有长时程分析能力。架构上可以组合（Biomni loop + ToolUniverse MCP），但需要把 retriever 换成 `Tool_Finder`。
8. **迁移到材料化学：底盘全部可复用，领域内容全部要重写，唯一的代码摩擦点是 `utils.py:846` 的硬编码列表。** 分子化学有现成基础（rdkit / openmm / vina），无机固态是空白。
9. **不要赌框架，把领域工具写成 MCP server。** Biomni、opencode、ToolUniverse 三边都吃 MCP。
