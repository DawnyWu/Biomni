# OPTIMADE 笔记：材料数据的跨库统一查询标准

> 我在[架构笔记](./biomni-architecture-notes.zh.md)和[生态对比](./scientific-agent-ecosystem-comparison.zh.md)里反复用「材料领域有 OPTIMADE，所以可以跳过数据湖」这个论据，这份笔记把它讲清楚，并**修正其中说过头的部分**。
>
> 另一份姊妹文档：[SCP 湿实验设备控制笔记](./scp-lab-device-control-notes.zh.md) —— 材料方向如果要接实验设备（管式炉、XRD、电化学工作站），那里有可参考的设计与要避开的坑。
>
> 本文所有查询结果都是实际发出请求跑过的，不是抄文档。测试时间 2026-08。
>
> - 官网：[optimade.org](https://www.optimade.org/) ｜ 规范：[v1.3.0](https://www.optimade.org/specification/latest/)
> - 论文：[OPTIMADE, an API for exchanging materials data](https://www.nature.com/articles/s41597-021-00974-z)（*Scientific Data*, 2021）
> - 参考实现：[Materials-Consortia/optimade-python-tools](https://github.com/Materials-Consortia/optimade-python-tools)

## 目录

- [一句话](#一句话)
- [它解决什么问题](#它解决什么问题)
- [四个核心设计](#四个核心设计)
- [实测验证](#实测验证)
- [⚠️ 重要限制：标准化的是结构，不是性质](#️-重要限制标准化的是结构不是性质)
- [对我此前结论的修正](#对我此前结论的修正)
- [常用 filter 速查](#常用-filter-速查)
- [生态工具](#生态工具)
- [对做材料 agent 的实践含义](#对做材料-agent-的实践含义)
- [其他局限与注意事项](#其他局限与注意事项)

---

## 一句话

**OPTIMADE（Open Databases Integration for Materials Design）是一套 REST API 规范，让你用同一个 URL、同一套过滤语法去查几十个互不相干的材料数据库。**

它由 Materials-Consortia 维护，本质是「材料数据库界的 SQL 标准 + DNS」——既统一了查询语言，也提供了自动发现服务的注册机制。

实测当前规模（[官方 dashboard](https://www.optimade.org/providers-dashboard/)）：

| | |
| --- | --- |
| 注册 provider | **28** 家（21 家有在线实现） |
| 子数据库 | **45** 个 |
| 可查询结构总数 | **27,266,136** 条 |

覆盖的库包括：Materials Project、OQMD、AFLOW、NOMAD、COD/TCOD（实验晶体学）、JARVIS（NIST）、Alexandria、Materials Cloud、2DMatpedia、MPDS（Pauling File）、Matterverse、CMR（含 C2DB）等。

---

## 它解决什么问题

在 OPTIMADE 之前，想「找出所有含锂的二元氧化物」，你得对每个数据库分别处理：

| 数据库 | 接入方式 | 化学式怎么写 |
| --- | --- | --- |
| Materials Project | `mp-api` Python 包 + API key | `chemsys="Li-O"` |
| OQMD | 自己的 REST，另一套参数 | `composition=Li-O` |
| AFLOW | AFLUX 查询语言（完全不同的语法） | `species(Li,O)` |
| NOMAD | 自己的 REST + Elasticsearch 风格查询 | 又一套 |
| COD | SQL 或自己的 REST | CIF 语义 |

**五个库 = 五份适配代码 + 五套认证 + 五种化学式约定 + 五种返回格式。** 而且这些约定都不是自明的（`Li-O` 还是 `LiO` 还是 `O-Li`？包含三元的吗？），每接一个都要读半天文档试半天。

这个痛点和 [Biomni 数据湖](./biomni-architecture-notes.zh.md#q22-数据湖是什么为什么要有这个东西)要解决的问题**一模一样**——只不过生物领域选择了「预先 ETL 成统一格式落到本地」，材料领域选择了「让上游统一 API」。

**这是两条不同的路：Biomni 在客户端做统一（数据湖），OPTIMADE 在服务端做统一（协议）。** 后者显然更优雅，但需要上游配合，而这在生物领域几乎不可能（几十家机构各自为政），在材料领域却成功了（社区规模小、共识好达成）。

---

## 四个核心设计

### 1. 统一的端点结构

每个实现都必须提供同一组版本化端点：

```
{base_url}/v1/structures        ← 结构条目（核心）
{base_url}/v1/references        ← 文献引用
{base_url}/v1/info              ← 这个实现支持什么
{base_url}/v1/info/structures   ← structures 有哪些可查字段
{base_url}/v1/links             ← 关联的其他数据库
{base_url}/versions             ← 支持哪些大版本（不带版本号）
```

响应遵循 [JSON:API 1.1](https://jsonapi.org/)，所以分页、错误格式、`meta` 结构全都是标准的。

通用查询参数：`filter`、`page_limit`、`page_offset`、`page_number`、`page_above`、`sort`、`response_fields`、`response_format`、`email_address`。

### 2. 统一的过滤语法

这是最核心的一块。规范定义了一套 **EBNF 语法**（参考实现用 [Lark](https://github.com/lark-parser/lark) 解析），支持比较运算、`AND`/`OR`/`NOT`、以及针对列表的量词操作：

```
filter=elements HAS ALL "Li","O" AND nelements=2
filter=nelements>=2 AND nelements<=7
filter=chemical_formula_reduced="O2Si"
filter=chemical_formula_anonymous="A2B"
filter=elements LENGTH 3
filter=NOT elements HAS "Pb"
```

关键在于**这个字符串在所有实现上语义相同**。这正是下面实测要验证的东西。

### 3. 统一的结构数据模型

`structures` 条目的标准字段（实测 MP 暴露 25 个标准字段）：

| 类别 | 字段 |
| --- | --- |
| 标识 | `id`、`immutable_id`、`last_modified`、`type` |
| 成分 | `elements`、`nelements`、`elements_ratios`、`chemical_formula_reduced` / `_descriptive` / `_hill` / `_anonymous` |
| 晶格 | `lattice_vectors`、`dimension_types`、`nperiodic_dimensions`、`nsites` |
| 原子位置 | `cartesian_site_positions`、`species`、`species_at_sites` |
| 对称性 | `space_group_it_number`、`space_group_symbol_hall`、`space_group_symbol_hermann_mauguin`(`_extended`)、`space_group_symmetry_operations_xyz` |
| 其他 | `structure_features`、`assemblies` |

**注意这个清单里有什么、没有什么——这是全文最重要的一点，[后面专门讲](#️-重要限制标准化的是结构不是性质)。**

### 4. 自动发现（两级机制）

这是常被忽略但对 agent 特别有用的部分。整个联邦是**机器可发现**的：

```
第一级：https://providers.optimade.org/providers.json
        ↓ 官方注册表，返回全部 provider 及其 index meta-db 地址
第二级：https://providers.optimade.org/index-metadbs/{provider}/v1/links
        ↓ 每家的 index meta-db，列出它旗下真实的子数据库地址
第三级：https://oqmd.org/optimade/v1/structures
        ↓ 真正干活的端点
```

自定义字段用**命名空间前缀**隔离：`_mp_stability`、`_alexandria_formation_energy_per_atom`、`_nmd_upload_id`。前缀在官方注册表里登记，保证不冲突。

**对 agent 的意义**：整个 action space 是可以在运行时枚举出来的。agent 可以先拉 `providers.json` 看有哪些库，再拉 `/info/structures` 看每个库有哪些可查字段，然后自己决定去哪查——**不需要有人预先给它写一份目录**。这恰好绕开了 [Q2.1 讲的「前提一：科学环境不自我描述」](./biomni-architecture-notes.zh.md#q21-补充这和普通-coding-agent-有什么区别)那个问题：OPTIMADE 让材料数据库变成了**自我描述**的。

---

## 实测验证

### 测试一：同一个 filter 打到六个数据库

用完全相同的过滤字符串 `elements HAS ALL "Li","O" AND nelements=2`（含 Li 和 O 的二元化合物）：

```bash
curl -s -G "{base_url}/v1/structures" \
  --data-urlencode 'filter=elements HAS ALL "Li","O" AND nelements=2' \
  --data-urlencode 'page_limit=1'
```

| 数据库 | 匹配数 | 返回样例 ID | 库的性质 |
| --- | --: | --- | --- |
| Materials Project | 21 | `mp-1097030` | DFT 计算 |
| Alexandria (PBE) | 46 | `agm001907020` | DFT 计算（Marques 组） |
| COD | 27 | `1010064` | **实验**晶体结构 |
| NOMAD | 2370 | `-emRPF-4NEdYwU4K-qaeoqKun7E4` | FAIR 数据仓库 |
| 2DMatpedia | 2 | `2dm-22` | 二维材料 |
| OQMD | ✗ | — | **HTTP 502，服务端故障** |

**六个后端实现完全不同（其中 COD 是实验数据库、NOMAD 是数据仓库、2DMatpedia 是专门的二维材料库），同一个字符串全都认。** 这就是 OPTIMADE 的价值。

匹配数差异巨大（21 vs 2370）反映的是各库的收录范围和去重策略不同，不是 bug。

### 测试二：官方客户端一次聚合查询

```python
from optimade.client import OptimadeClient

c = OptimadeClient(base_urls=[
    "https://optimade.materialsproject.org",
    "https://alexandria.icams.rub.de/pbe",
    "https://www.crystallography.net/cod/optimade",
    "http://optimade.2dmatpedia.org",
    "https://nomad-lab.eu/prod/v1/optimade",
])
print(c.count('elements HAS ALL "Li","O" AND nelements=2'))
```

实际输出（`optimade` 1.5.0，并发查询，几秒返回）：

```json
{
  "structures": {
    "elements HAS ALL \"Li\",\"O\" AND nelements=2": {
      "https://optimade.materialsproject.org": 21,
      "https://alexandria.icams.rub.de/pbe": 46,
      "https://www.crystallography.net/cod/optimade": 27,
      "http://optimade.2dmatpedia.org": 2,
      "https://nomad-lab.eu/prod/v1/optimade": 2370
    }
  }
}
```

> 💡 踩坑记录：`OptimadeClient` 的 `base_urls` 要传**不带版本号**的地址，它自己会拼 `/v1`。传 `.../v1` 会得到 `404 unknown endpoint: 'v1'`。

安装：`pip install "optimade[http_client]"`，也可用命令行 `optimade-get`。

---

## ⚠️ 重要限制：标准化的是结构，不是性质

**这一节修正了我此前的判断，是本文最有价值的部分。**

OPTIMADE 把「**结构与成分**」标准化得很好，但**几乎没有标准化物性字段**。实测各库 `/info/structures` 的字段构成：

| 数据库 | 标准字段 | provider 自定义字段 | 自定义字段示例 |
| --- | --: | --: | --- |
| Materials Project | 25 | **2** | `_mp_chemical_system`、`_mp_stability` |
| Alexandria (PBE) | 20 | 16 | `_alexandria_formation_energy_per_atom`、`_alexandria_hull_distance`、`_alexandria_xc_functional` |
| NOMAD | 25 | **436** | `_nmd_upload_id`、`_nmd_entry_id`、… |

### 关键发现：MP 经 OPTIMADE 不提供形成能和带隙

实测 Materials Project 经 OPTIMADE 暴露的**全部** 27 个字段：

```
_mp_chemical_system, _mp_stability, assemblies, cartesian_site_positions,
chemical_formula_anonymous, chemical_formula_descriptive, chemical_formula_hill,
chemical_formula_reduced, dimension_types, elements, elements_ratios, id,
immutable_id, last_modified, lattice_vectors, nelements, nperiodic_dimensions,
nsites, space_group_it_number, space_group_symbol_hall,
space_group_symbol_hermann_mauguin, space_group_symbol_hermann_mauguin_extended,
space_group_symmetry_operations_xyz, species, species_at_sites,
structure_features, type
```

**清一色是结构、成分、对称性。没有 `formation_energy_per_atom`，没有 `band_gap`，没有弹性常数，没有磁矩。** 唯一沾边的是 `_mp_stability`。

想拿 MP 的形成能，还是得用 `mp-api` 加 API key。

### 而且自定义字段同名可能异义

规范原文（§Database-provider-specific fields）明确允许：

> Providers that serve multiple databases MAY use the same provider-specific field names with different meanings in different databases. For example, a provider may use the field `_exmpl_band_gap` to mean a **computed** band gap in one of their databases, and a **measured** band gap in another database.

也就是说 `_exmpl_band_gap` 这个字段名在同一家的两个库里可以一个是计算值一个是实验值。**跨库聚合性质数据时，字段名不能当语义用。**

规范里连 `band_gap` 都只以 `_exmpl_band_gap`（示例前缀）的形式出现，说明它不是标准字段。

### 所以准确的分层是

| 你想做的事 | OPTIMADE 够用吗 |
| --- | --- |
| 按成分/化学式/元素数筛结构 | ✅ **完美**，跨库统一 |
| 按空间群/对称性筛 | ✅ 标准字段 |
| 拿晶格向量和原子坐标（喂给 DFT 或 ML 势） | ✅ 标准字段 |
| 跨库统计、去重、找新化合物 | ✅ 这是它的主场 |
| 拿形成能 / 带隙 / 弹性模量 / 磁性 | ❌ **要么用各库自定义字段（命名各异、语义可能不同），要么回退到原生 API** |
| 跨库比较同一性质 | ❌ 需要自己做字段映射和语义对齐 |

**一句话：OPTIMADE 是「结构发现层」的标准，不是「性质获取层」的标准。**

---

## 对我此前结论的修正

我在前两份笔记里说过「材料领域有 OPTIMADE，所以**可以完全不做**本地数据湖」。**这个说法太强了。** 修正后的准确版本：

| 环节 | 原判断 | 修正后 |
| --- | --- | --- |
| 结构数据 | 不用自建 | ✅ **成立**。OPTIMADE 免费给你 2726 万条，跨库统一查，无需托管 |
| 性质数据 | 不用自建 | ⚠️ **不成立**。要么逐个对接各家原生 API（`mp-api`、OQMD REST、NOMAD API），要么自建一份性质缓存 |
| 整体工作量 | 「省掉最重的一环」 | 省掉了**结构托管**这一环（确实是最重的），但**性质层仍需要适配层，且可能仍需要一个小型本地缓存** |

所以更诚实的说法是：

> **OPTIMADE 让材料 agent 的数据湖从「11 GB 结构 + 性质的全量副本」缩小到「一层性质 API 适配 + 可选的性质缓存」。是数量级的减负，不是归零。**

而且那个「小型缓存」的必要性和 [Biomni 数据湖的理由二/三](./biomni-architecture-notes.zh.md#q22-数据湖是什么为什么要有这个东西)完全一致：跨库 join 只有本地做得到，以及可复现性需要冻结版本。**你做批量筛选时终究会想把性质拉下来存成 parquet。**

---

## 常用 filter 速查

```bash
# 二元锂氧化物
elements HAS ALL "Li","O" AND nelements=2

# 含 Li 或 Na，但不含 Pb
(elements HAS "Li" OR elements HAS "Na") AND NOT elements HAS "Pb"

# 精确化学式（约简式，元素按字母序）
chemical_formula_reduced="O2Si"

# 匿名化学式：任意 A2B 化学计量的化合物
chemical_formula_anonymous="A2B"

# 元素数在 2~4 之间
nelements>=2 AND nelements<=4

# 只含这几种元素（不多不少之外的）
elements HAS ONLY "Li","Fe","P","O"

# 至少含其中之一
elements HAS ANY "Ga","Ge","As"

# 原子数上限（控制后续计算成本）
nsites<=20

# 指定空间群
space_group_it_number=225

# 二维材料（周期性维度为 2）
nperiodic_dimensions=2

# 配合 provider 自定义字段（注意：不跨库通用）
_alexandria_formation_energy_per_atom<-0.5
```

量词关键字：`HAS`、`HAS ALL`、`HAS ANY`、`HAS ONLY`、`LENGTH`。
注意 `filter` 值里的引号、`<`、`>`、空格**必须 URL 编码**（用 `curl --data-urlencode` 或 `requests` 的 `params=` 自动处理）。

---

## 生态工具

| 工具 | 用途 |
| --- | --- |
| **`optimade` (PyPI)** | 官方 Python 库：pydantic 数据模型、Lark filter 解析器、并发聚合客户端 |
| **`optimade-get`** | 命令行客户端，一条命令查所有 provider |
| **`optimade-validator`** | 合规性验证器，官方 dashboard 就是用它生成的 |
| **`optimade-maker` / `optimake`** | 把静态数据集（一堆 CIF）包成 OPTIMADE API——**自建 provider 用这个** |
| **`providers.json`** | 机器可读的 provider 注册表 |
| **pymatgen 集成** | `pymatgen.ext.optimade.OptimadeRester`，返回 `Structure` 对象 |
| **OPTIMADE Client (Web)** | 浏览器里的聚合搜索 + filter 可视化构建器 |

版本对应关系（注意别装错）：OPTIMADE v1.0 → `optimade<=0.12.9`；v1.1 → `optimade>=0.16,<1.2`；v1.2 → `optimade>=1.2.0`。规范最新是 v1.3.0。

---

## 对做材料 agent 的实践含义

承接[生态对比文档的路线 C](./scientific-agent-ecosystem-comparison.zh.md#三条可行路线更新版)。OPTIMADE 对 agent 特别友好，有三个原因：

**1. 它天然适合做成极薄的 MCP 工具。** 因为查询就是一个 GET 加一个 filter 字符串，返回是标准 JSON——这正是 ToolUniverse 那 263 个 `BaseRESTTool`「纯 JSON 声明、零 Python 代码」模式的理想对象。

**2. 它是自我描述的。** agent 可以运行时拉 `providers.json` 和 `/info/structures` 自己搞清楚有什么可查，不需要有人预先手写目录。这绕开了 [Q2.1 前提一](./biomni-architecture-notes.zh.md#q21-补充这和普通-coding-agent-有什么区别)的困难。

**3. filter 语法对 LLM 友好。** 语法小、正交、接近自然语言（`elements HAS ALL "Li","O"`），比 AFLUX 或 Elasticsearch DSL 好生成得多。

### 一个 MCP server 的设计草案

```
optimade_list_providers()                  → 从 providers.json 拉可用库（带缓存）
optimade_describe(provider)                → /info/structures，返回该库可查字段
optimade_search(filter, providers, limit)   → 聚合查询，返回标准化结构
optimade_get_structure(provider, id)        → 取单条，转 pymatgen Structure / CIF
```

设计要点：

- **`optimade_describe` 是关键**，别省。它让 agent 能自己发现 `_alexandria_formation_energy_per_atom` 这类字段，而不是靠猜。
- **性质查询要单独设计工具**，别混进 `optimade_search`。因为性质不跨库统一（见上文），应该老实提供 `mp_api_query()` 这类 provider 专属工具，并在描述里写清楚「MP 的形成能只能从这里拿，OPTIMADE 没有」。
- **返回结构时转成 pymatgen `Structure`**，因为下游（ASE、CHGNet、VASP 输入生成）都吃这个。
- 这个 server 在 opencode / Claude Code / Biomni（`add_mcp()`）/ SCP 里都能用——延续[「别赌框架，押注 MCP」](./scientific-agent-ecosystem-comparison.zh.md#选型决策)的思路。

---

## 其他局限与注意事项

**1. 可用性风险是真实的。** 我这次测试里 **OQMD 直接 HTTP 502**（`/info` 端点也 502，是服务端故障不是配置问题）。28 家 provider 里官方 dashboard 显示只有 21 家有在线实现。

这是「全走活 API、不做本地缓存」路线的固有代价，也是 [Biomni 选择数据湖的理由四](./biomni-architecture-notes.zh.md#q22-数据湖是什么为什么要有这个东西)（延迟与可复现性）在材料领域的回响：**如果你的论文结果依赖某次线上查询，而那个库明年下线了，结果就不可复现。** 做严肃工作时，把查询结果连同查询时间戳一起存档。

**2. 各库的收录范围、去重策略、计算参数不同。** 同一个 filter 在 MP 返回 21 条、NOMAD 返回 2370 条。跨库聚合时必须自己处理重复（同一个化合物在多个库里）和可比性（不同 XC 泛函算出来的能量不能直接比——注意 Alexandria 专门有 `_alexandria_xc_functional` 字段）。

**3. 覆盖偏计算、偏无机晶体。** 主力是 DFT 计算库和无机晶体结构。有机分子、聚合物、非晶、器件级数据基本不在射程内。CCDC（剑桥有机晶体库）只登记了命名空间前缀，没有开放的 OPTIMADE 端点。

**4. 拿不到「加工-结构-性能」关系。** OPTIMADE 是结构数据库标准，实验条件、合成路径、表征原始数据都不在其中。如果你的研究是实验主导（合成参数 → 性能），OPTIMADE 只能帮你做文献/结构层面的对照。

**5. 大数据量传输要留意。** `cartesian_site_positions` 这类字段对大胞很大，规范有 partial data / 大属性值传输协议来处理，客户端要正确实现。批量拉取记得用 `response_fields` 只要你需要的字段。

---

## 核心结论

1. **OPTIMADE 是材料数据库界的统一查询标准**，28 家 provider / 45 个子库 / 2726 万条结构，同一个 filter 字符串跨库通用（实测 5 个后端完全不同的库全部认同一个查询）。
2. **它和 Biomni 数据湖解决同一个问题，但方向相反**：Biomni 在客户端统一（预先 ETL 落地），OPTIMADE 在服务端统一（让上游实现协议）。后者更优雅，但需要社区共识——这在材料领域成功了，在生物领域没有。
3. **它是自我描述、机器可发现的**（`providers.json` → index meta-db → 子库，加 `/info/structures` 字段自省），所以 agent 能自己枚举 action space，不需要人工写目录。
4. **⚠️ 但它只标准化了结构，没有标准化性质。** 实测 MP 经 OPTIMADE 只暴露结构/成分/对称性 27 个字段，**没有形成能也没有带隙**；性质字段是 provider 私有的（`_alexandria_formation_energy_per_atom`），且规范明确允许同名异义。
5. **因此我此前「材料领域可以完全跳过数据湖」的说法要修正为**：可以跳过**结构数据**的托管（这确实是最重的一环），但性质层仍需逐个对接原生 API，且批量筛选时你终究会想建一份小型性质缓存。**是数量级减负，不是归零。**
6. **对自建 agent**：OPTIMADE 极适合做成薄 MCP 工具（GET + filter 字符串 + 标准 JSON），但**性质查询要单独设计 provider 专属工具**，并在工具描述里写明各自的边界。
