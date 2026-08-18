# 科学 Agent 生态三者对比：Biomni / ToolUniverse / SCP

> 对比对象：
>
> | 项目 | 出处 | 论文 |
> | --- | --- | --- |
> | **Biomni** | Stanford（snap-stanford） | [bioRxiv 2025.05.30.656746](https://www.biorxiv.org/content/10.1101/2025.05.30.656746v1) |
> | **ToolUniverse** | Harvard Medical School, Zitnik Lab（mims-harvard） | [arXiv 2509.23426](https://arxiv.org/abs/2509.23426) |
> | **SCP**（Science Context Protocol） | 上海人工智能实验室（InternScience） | [arXiv 2512.24189](https://arxiv.org/abs/2512.24189) |
>
> 本文所有结构性结论均来自**实际 clone 仓库后读代码与统计**，不是引用宣传材料。凡是无法从代码验证的（例如 SCP 托管在远端的工具数量），文中会明确标注「未验证」。
>
> 记录日期：2026-08。姊妹文档：
> - [Biomni 架构笔记](./biomni-architecture-notes.zh.md) —— Biomni 内部机制的详细拆解。
> - [OPTIMADE 笔记](./optimade-materials-data-notes.zh.md) —— 材料方向的数据基础设施与实测验证。

## 目录

- [一分钟结论](#一分钟结论)
- [第一步：先把「层」分清楚](#第一步先把层分清楚)
- [逐个拆解](#逐个拆解)
  - [Biomni = 一个 agent](#biomni--一个-agent)
  - [ToolUniverse = 一个工具库](#tooluniverse--一个工具库)
  - [SCP = 一个协议 + 一个托管平台](#scp--一个协议--一个托管平台)
- [总对比表](#总对比表)
- [七个关键差异](#七个关键差异)
- [重点：ToolUniverse 和 SCP 到底差在哪](#重点tooluniverse-和-scp-到底差在哪)
- [对材料化学方向的意义](#对材料化学方向的意义)
- [选型决策](#选型决策)
- [附录：验证方法与关键代码位置](#附录验证方法与关键代码位置)

---

## 一分钟结论

**这三个东西不在同一层，不是竞品关系。**把它们当竞品比较是理解这个领域最常见的错误。

| | 是什么 | 一句话 |
| --- | --- | --- |
| **Biomni** | 一个 **agent** | 有大脑（LangGraph 循环）、有手（224 工具）、有记忆（持久 REPL）、有资料室（11 GB 数据湖）。**开箱能干活，但只干生物医学。** |
| **ToolUniverse** | 一个 **工具库** | 2687 个工具规格 + 三种检索器，`pip install` 就全在你本地。**没有大脑，大脑你自己接**（Claude Code / Gemini CLI）。 |
| **SCP** | 一个 **协议 + 托管平台** | fork 了 MCP SDK，加上中心 Hub 和**真实的湿实验设备控制**。仓库里**没有工具**，2200+ 工具在上海 AI Lab 的服务器上，要 API Key。 |

最粗暴的类比：

- **Biomni** 像一台预装好系统和软件的**工作站** —— 插电就用，但只能做一类事。
- **ToolUniverse** 像一个巨大的**驱动程序库 + 外设仓库** —— 你得自己有主机。
- **SCP** 像一套**工业总线标准 + 一个云端调度中心** —— 它管的是「怎么把全世界的仪器接起来并且不出乱子」。

---

## 第一步：先把「层」分清楚

一个科学 agent 系统从上到下有六层。三个项目各自占哪几层，决定了它们的一切差异：

```
┌─────────────────────────────────────────────────────────────────┐
│  ① 模型层      GPT / Claude / Gemini / Qwen                      │
├─────────────────────────────────────────────────────────────────┤
│  ② Agent 循环   规划、执行、反思、终止判定                        │
├─────────────────────────────────────────────────────────────────┤
│  ③ 协议层      工具怎么描述、怎么调用、怎么发现                    │
├─────────────────────────────────────────────────────────────────┤
│  ④ 工具层      具体的 API 封装、模型推理、脚本                     │
├─────────────────────────────────────────────────────────────────┤
│  ⑤ 资源层      数据集、软件环境、知识文档                          │
├─────────────────────────────────────────────────────────────────┤
│  ⑥ 物理层      移液工作站、合成机器人、表征仪器                    │
└─────────────────────────────────────────────────────────────────┘
```

覆盖情况（●=核心投入，○=有但不是重点，—=不做）：

| 层 | Biomni | ToolUniverse | SCP |
| --- | :---: | :---: | :---: |
| ① 模型层 | ● 自训 Biomni-R0（Qwen-32B RL） | — 明确「无需训练」 | — |
| ② Agent 循环 | ● LangGraph 三节点 | — **完全不做** | ○ Hub 有编排，但非 agent 循环 |
| ③ 协议层 | — 只有内部约定 | ● 自定义 spec + MCP 传输 | ● **核心**，fork MCP 造新协议 |
| ④ 工具层 | ● 224 个 | ● **2687 个规格，核心** | ○ 2200+ 但**不在仓库里** |
| ⑤ 资源层 | ● **11 GB 数据湖 + conda 环境** | — 走活 API，无本地资源 | ○ 由各 Server 自己管 |
| ⑥ 物理层 | — | ○ 论文提及，代码里没有 | ● **真做了，MQTT 设备控制** |

**看这张表就够了**：

- Biomni 在 ①②④⑤ 重投入，③⑥ 空白 → **垂直整合的单体应用**
- ToolUniverse 在 ③④ 重投入，其余空白 → **横向的工具中间件**
- SCP 在 ③⑥ 重投入，④ 外置为服务 → **基础设施与标准**

三者**几乎不重叠**。真正重叠的只有 ToolUniverse 和 SCP 在协议层（③）的一小块，这也是本文后面要重点拆的地方。

---

## 逐个拆解

### Biomni = 一个 agent

详见 [Biomni 架构笔记](./biomni-architecture-notes.zh.md)，这里只列对比需要的要点。

- **框架**：LangGraph，3 节点（generate / execute / self_critic），核心是 `biomni/agent/a1.py` 单文件 3001 行。
- **执行方式**：CodeAct —— 模型输出 `<execute>` 标签里的 Python/R/Bash，跑在**持久解释器**里（`_persistent_namespace` 模块级 dict 反复 `exec`），变量跨轮存活。
- **action space 是「环境」而非「工具」**：224 工具 + 76 个数据湖文件 + 113 条软件目录 + know-how 文档，四类资源全部铺进 system prompt，让模型写任意代码去组合。
- **规模**：4.9 万行代码，11 GB 数据湖，conda 环境装 10 小时 / 占 30 GB。
- **领域**：约 100% 生物医学。
- **许可**：Apache-2.0（但集成的部分工具/数据有更严格的商用限制，故有 commercial mode）。

**独有能力**：持久状态的长时程数据分析、跨数据集本地 join、license 分层的 action space、自训推理模型、433 instances 的 benchmark（Eval1）。

**结构性短板**：只有生物医学；部署极重；工具规模比另两家小一个量级；扩张慢（加一个领域必须改 `biomni/utils.py:846` 的硬编码列表）。

---

### ToolUniverse = 一个工具库

- **没有 agent 循环。** 我 grep 过：依赖里没有 langgraph、没有 langchain，`StateGraph` 零命中。交付物是一个 MCP server（`src/tooluniverse/smcp.py`），大脑由你选的客户端提供（论文演示用 Gemini CLI）。
- **规模**：3379 个 Python 文件，**37.8 万行**，`src/tooluniverse/data/` 下 633 个 JSON 文件共 **2687 条工具规格**（官方口径「1000+ 工具」，差值来自 API key 目录等非工具条目）。
- **工具定义是声明式的**：JSON spec + `type` 字段指向 handler 类。**263 个工具共用通用的 `BaseRESTTool`，也就是纯 JSON 声明、零专属 Python 代码**。spec 里还自带 `test_examples` 和 `return_schema`，可自动测试。
- **三种工具检索器，且发现本身是工具**：`Tool_Finder`（faiss 向量）、`Tool_Finder_LLM`、`Tool_Finder_Keyword`，agent 在推理中随时可调。加 Compact Mode 把 1000+ 工具压成 4-5 个发现工具（官方说省约 99% context）。
- **工具库会自己长大**：`ToolDiscover` / `XMLToolOptimizer` / `TestResultsAnalyzer` / `CodeQualityAnalyzer` 一整套多 agent 流水线，能从一句话描述**同时生成代码和 spec**，自动测试、迭代优化，还能现场搜 PyPI 找包（`dynamic_package_discovery`）。
- **50 个工具本身是 LLM 调用**（`AgenticTool`，如 `DiseaseAnalyzerAgent`），即 agent 被抽象成了工具。
- **形态是「给现有 coding agent 装备科学能力」**：153 个 skills + 完整 Claude Code plugin（`plugin/` 下有 agents / 8 个 slash commands / hooks / settings.json）+ `mcpb/` bundle 打包。
- **部署极轻**：`pip install tooluniverse`，Docker 是 `python:3.12-slim`。**没有数据湖，全部走活 API**（代价是 39 个 secret 条目 + 45 条 API key 目录，要配一堆 key）。
- **领域**：基本全是生物医学 / 药物发现。
- **许可**：Apache-2.0。

**独有能力**：工具规模最大、工具自我生成与优化、部署最轻、可自托管（工具规格全在本地）。

**结构性短板**：无状态（`python_executor_tool.py` 把代码执行当成一个工具，且无跨调用状态，拿不到持久 REPL 的任何好处）；无长时程分析能力；能力上限取决于你接的客户端；领域仍局限于生物医学。

---

### SCP = 一个协议 + 一个托管平台

这个最容易被误解，因为它的仓库和它的产品是两回事。

#### 发现一：SCP 的参考实现是 MCP Python SDK 的 fork

我把 SCP 的 `src/scp/` 和官方 `mcp==1.9.4` 逐文件比对：

```
SCP 共 88 个 py 文件
  ├── 69 个与 MCP SDK 1.9.4 路径完全相同   ← 沿用（mcp → scp 重命名）
  └── 19 个是 SCP 新增
        ├── hub/   4 个文件, 1014 行
        └── lab/  15 个文件, 2157 行
```

也就是说，**13569 行里 SCP 自己写的是 3171 行，约 23%**；其余 77% 是 vendored 的 MCP SDK。

辅证：

- `src/scp/types.py:17` 注释写明「These bindings were generated from https://github.com/modelcontextprotocol/specification」
- `pyproject.toml:111` 有 `[tool.uv.sources] mcp = { workspace = true }`
- 依赖列表（anyio / httpx / httpx-sse / starlette / sse-starlette / uvicorn / pydantic）与 MCP SDK 完全一致
- **`LICENSE` 文件署名仍然是 `Copyright (c) 2024 Anthropic, PBC`** —— 直接沿用了 MCP SDK 的 MIT 许可文件

这不是贬义 —— MIT 许可下 fork 完全合规，而且「在 MCP 上做科学领域扩展」正是他们论文明说的主张（论文原话是 SCP「built on model–tool interaction protocols ... and extends them in three directions」）。但**认清这一点很重要**：SCP 不是从零设计的新协议，它是 MCP 加了三样东西。

#### 发现二：新增的 `lab/` 才是 SCP 真正的贡献

`lab/` 2157 行是三者中唯一真正触及物理世界的代码：

| 文件 | 内容 |
| --- | --- |
| `lab/cloud/mqtt.py`（24 KB） | `MQTTCloud` 类：`send_device_control()`、`wait_for_status_update()`、连接/断线/重订阅处理、异步回调线程 |
| `lab/lab_operator/base.py` | `@scp_register("action_name")` 装饰器，把**设备动作**注册成 SCP 工具；`dispatch_device_actions(device_name, device_action, params)`；docstring 里提到 "device twin"（设备数字孪生） |
| `lab/lab_operator/types.py` | `DeviceStatus` 枚举；`ActionResult` 带 `messageStatus`（1=最终结果，2=**中间结果**，-1=错误） |
| `lab/cloud/cloud_devices.py`、`cloud_consumer.py` | 设备控制指令下发与消息消费 |

**为什么必须新增这些？因为 MCP 的请求-响应模型装不下物理实验。** 一次合成可能跑几小时，中途要报进度、可能失败、需要暂停恢复。所以 SCP 补的正是这些：MQTT 发布订阅（而非 HTTP 请求响应）、Redis 存异步结果、`messageStatus=2` 的中间态推送、设备状态枚举。这是真实工程需求驱动的设计，不是概念包装。

#### 发现三：`hub/` 是注册中心 + 异步网关

`hub/hub_server.py`（Flask）的路由：

```
POST /register_server     注册一个 SCP Server
GET  /servers             列出已注册 Server
GET  /tools               列出全部工具
POST /tools/call_tool     统一调用入口
POST /set_result/         回写异步结果（Redis）
GET  /get_server_result/<request_id>   拉取异步结果
```

外加 `hub/permission/permission_server.py`（权限）、`hub/aliyun_oss.py`（阿里云 OSS 存产物）。

注意：**论文和中文手册描述的 Hub 智能编排能力**（解析高层意图 → 分解任务 → 对 top-k 方案按依赖/延迟/风险/成本排序 → AI 治理模块做冲突检测和资源预测）**在这份开源参考实现里我没有找到对应代码**。开源的 Hub 是一个注册表 + 异步调用网关，编排智能应该在托管的 SCP Hub 服务里。中文手册对此其实也有暗示：「展望未来，SCP 广场将全面引入 SCP Hub 的智能编排能力」。

#### 发现四：工具不在仓库里

**`find . -name "*.json"` 的结果是 0。** 仓库里一个工具规格都没有。

2200+ 工具在上海 AI Lab 的托管服务上。207 个 skills 里，**190 个需要 `SCP-HUB-API-KEY`**，指向的地址长这样：

```
https://scp.intern-ai.org.cn/api/v1/mcp/2/     （156 处引用）
https://scp.intern-ai.org.cn/api/v1/mcp/12/    （121 处）
https://scp.intern-ai.org.cn/api/v1/mcp/14/    （105 处）
...
```

skills 用的是 Anthropic Skills 格式（`SKILL.md` + YAML frontmatter: name / description / license / metadata），内容是**连远程 MCP server 的客户端配方**：

```python
self.transport = streamablehttp_client(
    url=self.server_url,
    headers={"SCP-HUB-API-KEY": self.api_key}
)
```

所以 SCP 的开源仓库 = **SDK（MCP fork）+ 协议规范 + 207 个调用远端服务的配方**。工具本身是需要注册账号的托管服务。

> ⚠️ **未验证声明**：2200+ 这个数字我无法从代码核实。另外论文（2025 年 11 月）写的是 **1,600+**，README 写的是 **2,200+**，存在口径差异，应理解为平台在持续增长。学科分布数据同样来自 README，未经独立验证。

#### 发现五：这是三者中唯一真正多学科的

README 给出的工具学科分布：

| 学科 | 占比 |
| --- | --- |
| 生物及相关技术 | 45.9% |
| **物理** | **21.1%** |
| **化学** | **11.6%** |
| **力学与材料科学** | **8.7%** |
| 数学 | 8.0% |
| 信息科学与计算 | 4.6% |

非生物部分合计约 **54%**。对比 Biomni（约 100% 生物医学）和 ToolUniverse（基本全是生物医学 / 药物发现），**SCP 是唯一把物理、化学、材料当作一等公民的**。这对材料化学方向意义重大（见后文）。

skills 按 8 大领域组织，其中药物发现 71 个、基因组学 41 个 —— skills 层面仍然偏生物，但工具层的学科面显著更宽。（实测 `skills/` 下有 207 个目录、每个都含 `SKILL.md`；README 写的是 206，差 1 个，无关紧要。）

- **许可**：MIT。

**独有能力**：真实的干湿闭环（MQTT 设备控制、设备孪生、中间态推送）、实验全生命周期与溯源、细粒度权限与多机构协作、真多学科覆盖。

**结构性短板**：核心资产（工具）不开源、需要 API Key、绑定托管平台；论文承诺的智能编排在开源实现中缺失；自有代码只有 3171 行，大部分价值在闭源服务里；无 agent 循环、无执行环境。

---

## 总对比表

| 维度 | Biomni | ToolUniverse | SCP |
| --- | --- | --- | --- |
| **本质** | agent（应用） | 工具库（中间件） | 协议 + 托管平台（基础设施） |
| **出处** | Stanford | Harvard HMS | 上海 AI Lab |
| **agent 循环** | ● LangGraph 三节点 | ✗ 自己接 | ✗ 自己接 |
| **协议立场** | 不做协议，内部约定 | 自定义 JSON spec，MCP 做传输 | **fork MCP 造 SCP** |
| **仓库代码量** | 4.9 万行 | **37.8 万行** | 1.36 万行（**自有仅 3171 行**，77% 是 MCP SDK） |
| **仓库内工具数** | 224 | **2687 条规格 / 633 文件** | **0**（托管在远端） |
| **宣称工具数** | 224 | 1000+ | 2200+（论文 1600+，未验证） |
| **工具定义方式** | Python 函数 + 手写 Python schema | 声明式 JSON（263 个零代码） | MCP 工具 schema + `@scp_register` 设备动作 |
| **工具发现** | 循环前一次性 LLM 选下标 | **三种检索器，且是可调工具** + Compact Mode | Hub `GET /tools` 注册表 |
| **代码执行** | ● **持久 REPL** + R + Bash | ○ 作为一个工具，无状态 | ✗ 不提供 |
| **本地数据** | ● **11 GB 数据湖 / 76 文件** | ✗ 全走活 API | ✗ 各 Server 自管 |
| **物理设备** | ✗ | ✗ | ● **MQTT + 设备孪生 + 中间态推送** |
| **实验生命周期** | ✗ | ✗ | ● 注册→规划→执行→监控→归档 |
| **权限/审计** | ○ 仅 license 分层（commercial mode） | ✗ | ● **细粒度认证授权 + 溯源** |
| **Skills** | 2 篇 know-how | 153 个 + Claude Code plugin | **207 个**（Anthropic Skills 格式） |
| **自训模型** | ● Biomni-R0（Qwen-32B RL） | ✗ 明确「无需训练」 | ✗ |
| **Benchmark** | ● Eval1（433 instances / 10 任务） | 单个 case study（高胆固醇血症） | 多个 use case |
| **学科覆盖** | 生物医学 ~100% | 生物医学为主 | **非生物约 54%**（物理 21% / 化学 12% / 材料 9%） |
| **部署成本** | conda，**10 小时 / 30 GB** | `pip install`，slim Docker | `pip install` + **需 API Key 才有工具** |
| **能否完全自托管** | ✔（数据湖需下载） | ✔（工具规格全在本地） | ✘ 工具在托管平台 |
| **许可** | Apache-2.0 | Apache-2.0 | MIT（LICENSE 仍署 Anthropic） |

---

## 七个关键差异

### 差异一：你实际拿到的是什么（最实际的一条）

| | 拿到的东西 | 断网还能用吗 |
| --- | --- | --- |
| Biomni | 一个能跑的 agent + 11 GB 数据 | **能**（数据在本地，除了工具里的 API 调用） |
| ToolUniverse | 2687 条工具规格 + 检索器 | 部分能（规格在本地，但工具本身多是 REST，要联网+key） |
| SCP | 一个 SDK + 207 个调用远端的配方 | **不能**（工具全在远端，且要 Key） |

这条差异决定了很多现实问题：能不能离线复现、能不能在内网/保密环境用、供应商锁定风险、以及**能不能读源码搞懂一个工具到底做了什么**。做严肃科研时这些不是小事。

### 差异二：对协议的三种立场

- **Biomni：不做协议。** 工具就是 Python 函数，agent 自己 import。后来加了 `add_mcp()` 和 `create_mcp_server()` 双向兼容 MCP，属于后补的兼容层，不是设计核心。
- **ToolUniverse：自定义 spec + 借 MCP 做传输。** 工具规格是自己的 JSON schema（带 `type` / `required_packages` / `test_examples` / `return_schema`），本地直接 Python 调用，远程走 MCP。**协议是手段。**
- **SCP：协议就是产品。** 直接 fork MCP SDK，因为 MCP 装不下科学场景（长时运行、中间态、设备、实验上下文、多机构权限）。**协议是目的。**

论文里两家都用了 HTTP 类比，但含义不同：ToolUniverse 说的是「我像 HTTP 一样标准化工具调用」，SCP 说的是「我要做科学界的 HTTP，让不同机构的 agent 和仪器互联」。**SCP 的野心更大，也更依赖生态采纳 —— 协议的价值完全取决于有多少人实现它。**

### 差异三：工具发现机制

| | 时机 | 方法 | 后果 |
| --- | --- | --- | --- |
| Biomni | **循环开始前一次性** | LLM 从编号列表选下标 | 工具集全程固定，猜错没救；但模型第 0 轮就看到全部资源（工具+数据+软件+know-how），全局规划视野最完整 |
| ToolUniverse | **推理中随时** | 三种检索器（向量/LLM/关键词）**本身就是工具** | 能应对计划外分支，context 占用极小（Compact Mode 省 99%）；但 turn-0 不知道自己有什么能力 |
| SCP | 平台侧 | Hub 注册表 `GET /tools` | 中心化目录，跨机构可发现；智能编排在闭源 Hub |

这是「**静态全局视野 vs 动态按需发现 vs 中心化注册**」三条路。没有绝对优劣：任务边界清晰时前者规划更好，任务会分叉时后者更稳，跨组织协作时只有后者可行。

### 差异四：状态与执行模型

这是三者差异最大的地方，而且**决定了它们能做什么任务**。

- **Biomni：有状态，进程内。** 持久 REPL，变量跨轮存活。最大好处是**廉价的错误恢复** —— 第 12 步崩了，前 11 步的中间结果还在内存里，改一行重跑就行。以及**上下文压缩** —— 10 GB 矩阵留在解释器里，进 context 的只有 `print(shape)` 那一行。代价是超时控制被迫用 threading（`utils.py:183` 注释写明了取舍）、状态污染难 debug、`exec` 全权限没法靠进程隔离做沙箱。
- **ToolUniverse：无状态，请求响应。** 每次工具调用独立。简单、安全、可水平扩展，但拿不到上面任何好处。20+ 步的数据密集分析结构上做不了。
- **SCP：有状态，但状态在「实验」上而非「内存」上。** 它建模的是实验生命周期（注册→规划→执行→监控→归档）、`request_id` 追踪、`messageStatus` 中间态、`DeviceStatus` 设备状态。这是**跨小时甚至跨天**的持久化状态，粒度完全不同 —— Biomni 的持久性是「一个 Python 进程里的变量」，SCP 的持久性是「一个实验的全部历史」。

一句话：**Biomni 管的是分钟级的内存状态，SCP 管的是天级的实验状态，ToolUniverse 不管状态。**

### 差异五：干湿实验（只有 SCP 真做了）

Biomni 有 `lab_automation.py`（3 个工具）和 `protocols.py`（4 个工具），但那是**生成协议文本**，不是控制设备。ToolUniverse 论文提到 robotics 和 lab automation 作为工具类别，但我在代码里没找到设备控制实现。

**只有 SCP 有真实的设备控制代码**：MQTT 长连接、指令下发、状态回传、设备孪生、中间进度推送。选 MQTT 而非 HTTP 是关键信号 —— 那是工业物联网的标准做法，说明他们真的在连仪器，不是在做 demo。

如果你的目标包含自动化实验（合成机器人、自动表征、闭环优化），**SCP 是三者里唯一在这条路上的**。

### 差异六：学科覆盖

| | 生物医学 | 化学 | 物理 | 材料 | 数学 |
| --- | :---: | :---: | :---: | :---: | :---: |
| Biomni | ~100% | 少量（rdkit/openmm/vina 分子层面） | — | — | — |
| ToolUniverse | 绝大部分 | 有（pubchem 21 / chembl 29 / chem_tool） | — | — | — |
| **SCP** | 45.9% | **11.6%** | **21.1%** | **8.7%** | **8.0%** |

Biomni 和 ToolUniverse 都是**从生物医学出发**的项目，虽然都自称通用（"general-purpose" / "AI scientists"），实际工具分布高度集中。SCP 是唯一**从一开始就按多学科铺**的。

代价是深度：SCP 每个学科的平均工具数远不及 Biomni 在生物上的密度，而且工具质量无法从代码核实。**广度换深度。**

### 差异七：治理、权限与多机构协作

- **Biomni**：只有 license 分层（commercial mode 把 76 个数据集裁到 41 个）。单机单用户假设。
- **ToolUniverse**：39 个 secret 条目 + 45 条 API key 目录，即「你自己管好你的 key」。论文把 human-in-the-loop 和 safety 列为工具类别，属应用层安全。
- **SCP**：`server/auth/` 有完整 OAuth 风格实现（authorize / token / register / revoke / metadata handlers + bearer/client auth middleware），`hub/permission/` 做权限服务，论文强调「基于实验的细粒度认证授权」和「审计追踪」。

这条差异反映的是**假想用户不同**：Biomni 假设一个博士生在自己机器上跑；ToolUniverse 假设一个开发者接 Claude Code；**SCP 假设多个机构共享昂贵仪器，需要谁能用什么、谁在什么时候用了什么都可查。** 后者是机构级基础设施才要考虑的问题。

---

## 重点：ToolUniverse 和 SCP 到底差在哪

这是本文的主问题。两者表面上都说自己是「统一科学工具的协议/生态」，容易混淆。差异有五层，从表到里：

### 1. 层级不同：工具层 vs 协议层

ToolUniverse 的重心是**攒工具**（37.8 万行代码，2687 条规格，其中 263 个纯 JSON），协议是为了让工具能被调用而顺手定的。

SCP 的重心是**定标准**（自有代码仅 3171 行），工具是别人（各机构的 SCP Server）来接的。

**一个在做内容，一个在做管道。** 37.8 万行 vs 3171 行这个对比比任何描述都直观。

### 2. 工具的所有权：库 vs 服务

| | ToolUniverse | SCP |
| --- | --- | --- |
| 工具在哪 | `pip install` 后在你本地（633 个 JSON） | 上海 AI Lab 服务器 |
| 要不要认证 | 部分工具需要各自的第三方 API key | **需要 SCP-HUB-API-KEY**（190/207 skills） |
| 能否自托管 | ✔ 全部 | ✘ 只能自建 Server 接入，用不了平台上的 2200+ |
| 能否读源码 | ✔ 每个工具的实现都能看 | ✘ 远端黑盒 |

**这是最实质的差异。** ToolUniverse 是开源库，SCP 是开源协议 + 闭源服务。对「我要搞清楚这个工具算的对不对」这种科研刚需，两者体验完全不同。

### 3. 对 MCP 的态度：用它 vs 改它

ToolUniverse **使用** MCP（`mcp[cli]` + `fastmcp` 做依赖，交付一个标准 MCP server），所以 Claude Code、Gemini CLI、opencode 都能直接接。

SCP **fork** 了 MCP（69/88 文件沿用，`mcp`→`scp` 重命名），造了个新协议。好处是能加 MCP 没有的东西（设备、实验上下文、异步中间态）；代价是**生态兼容性** —— 标准 MCP 客户端不认识 SCP。有意思的是它的 skills 里用的其实是 `from mcp.client.streamable_http import ...`，即在 HTTP 传输层仍与 MCP 兼容，只是加了 `SCP-HUB-API-KEY` 头。

**换句话说：ToolUniverse 押注 MCP 生态会赢；SCP 押注科学场景需要自己的标准。**

### 4. 能力边界：干实验 vs 干湿闭环

ToolUniverse 完全在数字世界：查数据库、跑 ML 模型、调 API、算 ADMET。

SCP 跨到了物理世界：MQTT 控设备、设备孪生、中间进度、实验状态机。

**如果只做计算，SCP 的 `lab/` 那 2157 行对你毫无价值，而那正是它 68% 的自有代码。** 反过来，如果要做自动化实验，ToolUniverse 结构上帮不了你。

### 5. 学科面：生物医学 vs 全学科

ToolUniverse 出自 Zitnik Lab（生物医学信息学），工具从 FDA 药品标签（152）、OpenTargets（72）、RCSB PDB（39）、ChEMBL（29）铺开，是**药物发现导向**的。

SCP 非生物学科占约 54%，物理 21% 是第二大类。

### 一句话总结这两者

> **ToolUniverse 是「给 AI 科学家一个巨大的、可自托管、可审计的生物医学工具箱」；SCP 是「给全世界的实验室定一个能连接仪器、管得住权限、跨机构协作的科学互联网标准」。**
>
> 前者的成功指标是工具数量和质量，后者的成功指标是**有多少机构愿意实现这个协议**。

它们甚至可以叠加：一个机构完全可以把 ToolUniverse 包成一个 SCP Server 接进 SCP Hub。

---

## 对材料化学方向的意义

承接 [Biomni 架构笔记第四部分](./biomni-architecture-notes.zh.md#第四部分能否改成材料化学方向)。加入 SCP 这个参照后，判断有变化：

### SCP 是三者里唯一对材料友好的

物理 21.1% + 化学 11.6% + 力学与材料 8.7% ≈ **41% 的工具与材料化学直接相关**，代表工具类型 README 列的是「量子计算、材料模拟、分子对接、反应预测、有限元分析、分子动力学」—— 这正是材料化学要用的东西。

Biomni 和 ToolUniverse 在这方面几乎是空的（Biomni 只有分子层面的 rdkit / openmm / vina；ToolUniverse 有 pubchem / chembl / chem_tool 但仍是药化导向）。

### 但先去核实工具质量

我无法从代码验证那 2200+ 工具的实际质量、可用性和维护状态（它们在远端）。建议实际动作：

1. 去 [SCP 工具清单](https://yankai96.github.io/SCP_Tool_List/) 看物理/化学/材料那 41% 具体是什么 —— 是不是真接了 Materials Project、VASP、LAMMPS、pymatgen 这类，还是一堆薄封装的计算器。
2. 注意 `skills/` 里 207 个 skill 仍以药物发现（71）和基因组学（41）为主，**说明成熟的工作流仍在生物侧**，材料侧可能只有裸工具没有 skill。
3. 注册拿 Key 实测几个材料工具。

### 三条可行路线（更新版）

| 路线 | 做法 | 适合 |
| --- | --- | --- |
| **A. 接 SCP** | 拿 API Key 用现成的物理/化学/材料工具，需要什么自己写 SCP Server 接入 | 想快速拿到多学科工具，且不介意依赖托管平台；**如果规划里有自动化实验，这是唯一选择** |
| **B. Biomni 底盘 + 自建工具** | 复用 LangGraph 循环 + 持久 REPL + 数据湖机制，重写领域内容（改 `utils.py:846` 硬编码列表、重写 `env_desc.py`、新写 conda env） | 要做**计算密集的多步材料分析**（DFT 后处理、跨数据库联合建模），且要发论文讲 curated environment |
| **C. ToolUniverse 范式自建** | 学它的声明式 JSON spec 模式，把 Materials Project / OPTIMADE / NOMAD / OQMD 的 REST API 写成纯 JSON 工具 | 工具开发效率最高。材料数据库 API 化程度高（有 OPTIMADE 统一标准），**263 个 `BaseRESTTool` 那套零代码模式在材料领域尤其划算** |

### 我的建议没变，但更有底气了

**把材料领域工具写成 MCP server。** 现在有四个理由：

1. Biomni 有 `add_mcp()` 能吃
2. opencode / Claude Code / Cursor 原生支持
3. ToolUniverse 本身就是 MCP server，同层可组合
4. **SCP 的 skills 在传输层用的也是标准 MCP streamable HTTP**，只是多了个 Key 头，所以一个标准 MCP server 要接进 SCP 生态的改造量很小

四条路都通 MCP。**在协议之争分出胜负之前，把资产押在 MCP 这个交集上是风险最低的。**

关于数据湖那一条也基本不变：材料领域有 [OPTIMADE](./optimade-materials-data-notes.zh.md) 跨数据库统一查询标准（实测 28 家 provider / 45 个子库 / 2726 万条结构，MP / OQMD / AFLOW / NOMAD / COD 都实现了），**结构数据可以直接走活 API，跳过 Biomni 11 GB 数据湖那套最重的工作**。这一点上 ToolUniverse 和 SCP 的「无本地数据、全走 API」路线更适合材料领域。

但有个必须打的折扣：**OPTIMADE 只标准化了结构与成分，没有标准化物性**——实测 MP 经 OPTIMADE 暴露的 27 个字段里没有形成能、也没有带隙，那些得回退到 `mp-api`。所以性质层仍要逐个对接原生 API，批量筛选时大概仍需一份小型缓存。**是数量级减负，不是归零。**

---

## 选型决策

按你的实际目标选，不要按项目名气选：

```
你的目标是什么？
│
├─ 「我要做数据密集的多步计算分析」（scRNA-seq 全流程 / DFT 批量后处理 /
│   跨数据集联合建模，20+ 步，中间结果很贵）
│      → Biomni。持久 REPL 的廉价错误恢复和上下文压缩是结构性优势，
│        另两家给不了。生物领域直接用；其他领域复用底盘重写内容。
│
├─ 「我要查询、检索、预测」（找靶点 / 查文献 / 算 ADMET / 验专利）
│      → ToolUniverse。工具规模大一个量级，pip install 即用，
│        接 Claude Code 或 Gemini CLI 直接干活，且可自托管可审计。
│
├─ 「我要非生物学科的工具」（物理 / 化学 / 材料 / 力学）
│      → SCP。三者中唯一真多学科（非生物约 54%）。但先核实工具质量。
│
├─ 「我要连实验设备做干湿闭环」
│      → SCP，唯一选项。MQTT 设备控制 + 设备孪生 + 实验生命周期。
│
├─ 「我要跨机构共享仪器，需要权限和审计」
│      → SCP。另两家根本没设计这个问题。
│
├─ 「我要在内网/保密环境用，或需要完全离线复现」
│      → Biomni 或 ToolUniverse。SCP 的工具在托管平台，做不到。
│
└─ 「我要给自己的领域造一个 agent」
       → 分两步：
         ① 工具写成标准 MCP server（这是真资产，与框架无关）
         ② 循环层按上面的表选，或者直接用 opencode / Claude Code
```

**最容易被忽略的一点**：这三个东西**可以叠加使用**，而且叠加往往比单选更优。

- Biomni 的 `add_mcp()` 可以消费 ToolUniverse 的工具（架构上可行，**我未实测**；已知障碍是 Biomni 的 prompt-based retriever 面对 2000+ 工具会爆 context，得改成调用 `Tool_Finder`）。
- 一个机构可以把 ToolUniverse 包成 SCP Server 接进 SCP Hub。
- opencode / Claude Code 可以同时挂 ToolUniverse 和你自己的 MCP server。

---

## 附录：验证方法与关键代码位置

### 本文结论是怎么验证的

想自己复核，可以照做：

```bash
# 1. clone 两个对比对象
git clone --depth 1 https://github.com/mims-harvard/ToolUniverse.git tu
git clone --depth 1 https://github.com/InternScience/scp.git scp

# 2. 证明 SCP 是 MCP SDK 的 fork
pip download "mcp==1.9.4" --no-deps -d mcp1 && (cd mcp1 && unzip -q *.whl -d sdk)
(cd scp/src/scp   && find . -name "*.py" | sed 's|^\./||' | sort) > scp_files.txt
(cd mcp1/sdk/mcp  && find . -name "*.py" | sed 's|^\./||' | sort) > mcp_files.txt
comm -23 scp_files.txt mcp_files.txt   # → 只剩 hub/ 和 lab/ 共 19 个文件
comm -12 scp_files.txt mcp_files.txt | wc -l   # → 69 个沿用

# 3. SCP 自有代码量
(cd scp/src/scp && find hub lab -name "*.py" | xargs wc -l | tail -1)   # → 3171
(cd scp/src/scp && find .       -name "*.py" | xargs wc -l | tail -1)   # → 13569

# 4. SCP 仓库里没有工具
(cd scp && find . -path ./.git -prune -o -name "*.json" -print | wc -l)  # → 0
(cd scp && grep -rl "SCP-HUB-API-KEY" skills/ | wc -l)                   # → 190 / 207

# 5. ToolUniverse 工具规格数
cd tu && python3 -c "
import json,glob
t=0
for f in glob.glob('src/tooluniverse/data/*.json'):
    try: d=json.load(open(f))
    except: continue
    if isinstance(d,list): t+=len(d)
print(t)"                                                                # → 2687

# 6. ToolUniverse 没有 agent 循环
(cd tu && grep -rln "langgraph\|StateGraph" src/ | wc -l)                # → 0
```

### 关键代码位置

**Biomni**（本仓库）—— 完整速查表见 [架构笔记附录](./biomni-architecture-notes.zh.md#附录关键代码位置速查)

| 关注点 | 位置 |
| --- | --- |
| LangGraph 图定义 | `biomni/agent/a1.py:1606` |
| 持久 REPL | `biomni/tool/support_tools.py:6-7` |
| threading 超时及原因注释 | `biomni/utils.py:183` |
| 加新领域必改的硬编码列表 | `biomni/utils.py:846` |
| MCP 双向支持 | `a1.py:350`（`add_mcp`）、`a1.py:1995`（`create_mcp_server`） |

**ToolUniverse**

| 关注点 | 位置 |
| --- | --- |
| MCP server | `src/tooluniverse/smcp.py` |
| 三种工具检索器 | `tool_finder_{embedding,llm,keyword}.py` |
| 检索器工具规格 | `src/tooluniverse/data/finder_tools.json` |
| 通用 REST handler（263 个工具共用） | `src/tooluniverse/base_rest_tool.py` |
| 工具自我生成流水线 | `src/tooluniverse/data/tool_discovery_agents.json`、`compose_tool.py` |
| Claude Code plugin | `plugin/`（agents / commands / hooks / settings.json） |

**SCP**

| 关注点 | 位置 | 备注 |
| --- | --- | --- |
| **设备控制（MQTT）** | `src/scp/lab/cloud/mqtt.py` | 24 KB，`send_device_control()` / `wait_for_status_update()` |
| **设备动作注册** | `src/scp/lab/lab_operator/base.py` | `@scp_register()` 装饰器、`dispatch_device_actions()` |
| **设备状态与中间态** | `src/scp/lab/lab_operator/types.py` | `DeviceStatus` 枚举、`ActionResult.messageStatus` |
| Hub（注册表 + 异步网关） | `src/scp/hub/hub_server.py` | Flask，`/register_server`、`/tools/call_tool` |
| 权限 | `src/scp/hub/permission/permission_server.py`、`src/scp/server/auth/` | OAuth 风格 handlers + middleware |
| MCP fork 的证据 | `src/scp/types.py:17`、`pyproject.toml:111`、`LICENSE` | 注释指向 modelcontextprotocol/specification；LICENSE 署 Anthropic |
| 沿用 MCP 的骨架 | `src/scp/server/fastmcp/`、`client/`、`shared/`、`server/lowlevel/` | 69/88 文件与 mcp 1.9.4 同路径 |
| Skills（Anthropic 格式） | `skills/*/SKILL.md` | 207 个，190 个需 API Key |
| 中文说明 | `SCP中文手册.md` | 比英文 README 更详细地描述了 Hub 的编排设计 |

### 已声明的未验证项

诚实起见，以下结论**无法从代码核实**，引用时请注意：

1. SCP 的 **2200+ 工具**（论文写 1600+，README 写 2200+）及其**学科分布**—— 工具在托管平台，仓库里为 0。
2. SCP 论文与中文手册描述的 **Hub 智能编排能力**（意图分解、top-k 方案按依赖/延迟/风险/成本排序、AI 治理模块的冲突检测与资源预测）—— 开源 Hub 里我没找到对应实现，应在闭源服务中。
3. **「Biomni loop + ToolUniverse MCP」的组合** —— 只做了架构可行性判断，未实际跑通。
4. ToolUniverse 与 SCP 各工具的**实际质量、准确性和维护状态** —— 未做功能测试。
5. 本文的 Biomni 数据基于 `main` @ `400c1f3`；ToolUniverse 与 SCP 基于 2026-08 的 `--depth 1` clone。三个项目都在快速迭代，数字会变。

---

## 核心结论汇总

1. **三者不在同一层，不是竞品。** Biomni 是 agent（②④⑤层），ToolUniverse 是工具库（③④层），SCP 是协议 + 平台（③⑥层）。把它们当竞品比是最常见的误解。
2. **SCP 的参考实现是 MCP SDK 1.9.4 的 fork**：88 个 py 文件里 69 个沿用，自有代码 3171 行（占 23%），全部集中在 `hub/`（1014 行）和 `lab/`（2157 行）。LICENSE 仍署 Anthropic。
3. **SCP 真正的贡献是 `lab/`** —— 三者中唯一有真实物理设备控制代码的（MQTT、设备孪生、中间态推送）。选 MQTT 而非 HTTP 说明是真在连仪器。这也是 MCP 请求-响应模型装不下科学实验的直接证据。
4. **SCP 仓库里工具数为 0**，2200+ 在托管平台，190/207 skills 需 API Key。它是开源协议 + 闭源服务；ToolUniverse 是彻底的开源库（2687 条规格全在本地）。**这是两者最实质的差异。**
5. **ToolUniverse 在做内容，SCP 在做管道**：37.8 万行 vs 3171 行自有代码。ToolUniverse 押注 MCP 生态会赢，SCP 押注科学场景需要自己的标准。
6. **SCP 是唯一真多学科的**（非生物约 54%，物理 21% / 化学 12% / 材料 9%），对材料化学方向最友好，但工具质量待核实，且成熟 skills 仍集中在生物侧。
7. **状态模型三条路**：Biomni 管分钟级内存状态（持久 REPL），SCP 管天级实验状态（生命周期），ToolUniverse 不管状态。这决定了各自能做什么任务。
8. **结论不变：把领域工具写成标准 MCP server。** Biomni、opencode / Claude Code、ToolUniverse、SCP 四条路都通 MCP，这是风险最低的资产形态。
