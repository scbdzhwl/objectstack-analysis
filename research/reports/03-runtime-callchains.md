---
id: RES-03
title: "ObjectStack 首批程序机制研究：完整数据入口与独立 ObjectQL 嵌入"
type: mechanism-study
source_repo: scbdzhwl/objectstack-analysis
source_commit: 6c5d31fac7763e7aaebc5e853c17727698e8126a
status: research-draft
updated: 2026-11-16
---

# RES-03｜数据能力双入口：对象定义、运行时桥接、ObjectQL 与责任边界

> **本批身份**：A0 研究文件可读取性检查 + A1 有限功能/消费者地图 + B1 第一条**部分闭合**的对象读写机制研究。属于六份研究报告中第三份的第一段实质内容，**不是全量 ObjectStack 程序设计结论**。
>
> **证据与限制**：在 GitHub 通过指定 commit 阅读源码、示例、测试文件和官方架构描述；没有本地 clone、安装依赖、编译、启动系统、运行测试、独立 Agent/IDE 指令自动加载验证及目标 Java 工程实际采用。可直接核实的代码结构属 G1；局部机制已进行静态论证，但授权/事务完整链未闭合，不提升到“端到端 G2/G3”。背景与状态见 [研究设计](../DESIGN.md) 和 [执行计划](../PLAN.md)。

## 0. 研究问题、观察单位与结论摘要

**问题**：ObjectStack 允许同一对象定义在完整平台与不加载 Kernel 的 ObjectQL 嵌入中使用。哪些能力属于核心数据引擎？哪些由 HTTP/协议/宿主提供？如何检验多入口访问中的授权、输入契约、事务/版本控制与错误语义不漂移？

**初步结论（限定在已读源码）**：

1. **轻量入口确实有代码消费者**：`examples/embed-objectql` 直接使用 `@objectstack/objectql/core` 创建引擎、内存驱动、注册对象、写入三条记录并查询活动客户；它不需要 Kernel/元数据管理协议。相关测试源文件存在，**本轮没有执行**。
2. **完整运行时的数据入口有独立的传输门**：`/data` 处理函数检查匿名身份、环境归属、对象 API 暴露及请求 Schema；`callData` 优先调用协议服务的 `createData/getData/updateData/deleteData`，缺少相应协议能力时才使用 ObjectQL 回退。
3. **ObjectQL 公共类已拆为 Facet 委托**：`find()` → `FindFacet.find()`，`insert()` → `InsertFacet.insert()`，所以只读 `engine.ts` 的入口不等于研究了实现。
4. **边界并不等价**：轻量示例没有传入用户执行身份或启动平台授权服务；完整 HTTP 路径明确具有身份和 API 暴露门。不能由此直接推断轻量模式存在漏洞，也不能宣称“核心引擎自动提供与平台相同的完整业务授权”。
5. **版本/错误语义不能静默降级**：在 `callData` 的 update/delete 回退分支，若带了 `expectedVersion` 但缺少能处理它的协议实现，源码选择拒绝（501），而不是无条件放行。这是具备复用潜力的机制，但需要逐入口测试与目标场景验证。

本次只覆盖功能/机制的一小部分。全局架构、全部角色/权限、REST 各入口、数据库驱动真实行为、历史因果、Java 生态对照和现有 DES-ARC 裁决仍未完成。

## 1. A0：研究文件恢复检查（只读）

本会话通过 GitHub 实际读取 `research/README.md`、`DESIGN.md`、`PLAN.md`、`BASELINE.md`、`AGENTS.md`、`METHOD.md`，核对入口、版本、下一批研究任务和只修改 `research/` 的边界，六项文本定位条件均成立。

**不能得出的结论**：这不是重新启动的新 AI 代理，也不是 Codex/IDE 的自动指令继承测试；尚无跨会话加载记录或行为测试。其阻断范围仅限“自动生效已经证明”这一声明，**不阻断有界的 GitHub 只读源码研究**。

本次主分支在启动时为 `c25f9250b819d19ee4cfe168f53cb9a59dd41d1e`，且此前 PR #3 已合并；源代码研究仍固定为 `6c5d31fac7763e7aaebc5e853c17727698e8126a`，防止上游同步后新旧版本混用。

## 2. A1：有限能力、角色和消费者关系图

| 关注对象 | 定义/入口 | 已找到的实际消费者 | 已有的证据 | 不能代表 |
|---|---|---|---|---|
| Account 对象/字段 | `ObjectSchema.create`：name（必填）、industry、active | 轻量 `runEmbeddedEngine()` | [示例源码](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/examples/embed-objectql/src/index.ts#L17-L58) | 完整 CRM 业务定义或身份权限模型 |
| 数据引擎注册/初始化 | `ObjectQL`、`registerDriver`、`init`、`registry.registerObject` | 同一轻量示例 | [示例](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/examples/embed-objectql/src/index.ts#L37-L58)、[引擎注册委托](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/objectql/src/engine.ts#L2231-L2271) | 每一种驱动的建连/同步/事务可靠性 |
| 数据 CRUD | `engine.insert`、`engine.find` | 轻量示例与它的 Vitest 用例 | [嵌入测试](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/examples/embed-objectql/test/embed.test.ts) | 测试已执行通过或生产可用 |
| HTTP 数据域 | `handleDataRequest`（/data） | `HttpDispatcher` 域处理调用，向 `actionExec.callData` 委托 | [数据域](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/runtime/src/domains/data.ts#L166-L219)、[调用桥](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/runtime/src/action-execution.ts#L612-L690) | 每种 REST/MCP 自定义端点均经过此域 |
| 元数据种类覆盖清单 | `KIND_COVERAGE` 对齐 `DEFAULT_METADATA_TYPE_REGISTRY` | Showcase 的覆盖检查测试 | [清单](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/examples/app-showcase/src/coverage.ts#L45-L220)、[覆盖测试](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/examples/app-showcase/test/coverage.test.ts) | 全部种类的运行时功能/业务效果均已验证 |
| 对象权限/读取范围 | HTTP 匿名与 API 暴露门，ObjectQL 读取阶段安全相关组合逻辑 | /data 入口、引擎读取流程 | [HTTP 入口](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/runtime/src/domains/data.ts#L188-L220)、[读取执行](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/objectql/src/engine-find.ts#L419-L554) | 所有入口、所有对象与所有租户权限已一致 |

固定版本 `KIND_COVERAGE` 的**元数据种类清单**有 **32** 项：`demonstrated` 25 项、`waived` 7 项（picklist、external_catalog、queue、assignment_rule、agent、tool、skill）。此统计是对该清单的静态字段读取，**不是已实现、已测试通过、生产支持率**。其中 picklist 明确说明其运行时读取缺口；其他 `waived` 原因各异，不可统一说成“未实现”。[来源](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/examples/app-showcase/src/coverage.ts#L45-L223)。

在本批范围中，“用户角色”须明确分开：轻量示例只是程序调用者；`/data` 入口出现执行身份和系统上下文；Showcase 的 position、permission 等有清单和声明文件。尚未恢复岗位、授权集合、数据范围、系统执行身份之间的完整关系，不能以代码演示中出现 `account` 就制造销售人员权限模型。

## 3. B1-1：双入口的已确认执行片段

~~~mermaid
flowchart TD
  A["同源业务对象 Schema"]
  A --> L["独立嵌入：ObjectQL/core"]
  A --> H["完整系统：HTTP /data"]
  L --> LR["registerDriver / init / registerObject"]
  LR --> LL["insert / find → 引擎 Facet"]
  H --> G["匿名/环境/暴露/请求校验"]
  G --> BR["action-execution.callData"]
  BR --> P{"协议服务提供操作？"}
  P -->|是| PP["protocol.createData / getData / ..."]
  P -->|否| F["ObjectQL 回退（有限能力）"]
  LL --> D["引擎→驱动（内存或其他）"]
  F --> D
  PP --> U["协议实现下游：本批待继续追踪"]
~~~

图中的 Kernel 装配、完整运行时协议实现和各驱动的内部执行**未完全追踪**。图只表示当前可在源码中定位的分支，不能替代真实运行时序图。

### 路径 L｜独立嵌入

1. 定义 `Account`：`ObjectSchema.create({ name: 'account', fields: ... })`。
2. `new ObjectQL()`；`registerDriver(new InMemoryDriver({ persistence:false }), true)`；`await engine.init()`；`engine.registry.registerObject(Account, 'example-embed')`。
3. 插入 Acme、Globex、Initech；调用 `find('account', { where: {active:true}, orderBy:... })`。
4. 对应测试**源码**断言返回 Acme 和 Initech，并断言 core 导出 ObjectQL 而不导出 ObjectQLPlugin 等符号。

直接证据：[示例源码](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/examples/embed-objectql/src/index.ts#L17-L60)、[测试代码](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/examples/embed-objectql/test/embed.test.ts#L1-L25)、[轻量导出](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/objectql/src/core.ts#L1-L65)。

`core-boundary.ratchet.test.ts` 声明限制 core 入口引入 plugin/kernel-factory/metadata-protocol，源码检查的是本包特定可达依赖与禁用包名，作者同时指出另有构建后跨包闭包检查；**没有实际运行这两种门禁**。[来源](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/objectql/src/core-boundary.ratchet.test.ts#L1-L70)。

### 路径 H｜完整运行时数据域

1. `handleDataRequest` 的早期步骤检查匿名身份（必要时 401）、请求路径、在多租户宿主中的环境解析；这只是该数据域的入口语义，不等于所有数据入口都执行本函数。[来源](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/runtime/src/domains/data.ts#L178-L220)。
2. `POST /data/:object` 先调用 API 暴露门，再以 `CreateDataRequestSchema.safeParse` 检查装配好的请求，随后进入 `callDataWrite(() => callData(...))`，正常返回 201。[`/data` 源码](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/runtime/src/domains/data.ts#L488-L516)。
3. `actionExec.callData` 调用对象级 API 暴露门，尝试获取 `protocol` 与 `objectql` 服务；create 优先 `protocol.createData`，缺失时如存在引擎 `insert` 则执行回退，最终还处理写入响应的内部字段暴露。[来源](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/runtime/src/action-execution.ts#L612-L693)。
4. `update/delete` 在协议不存在、调用者带 `expectedVersion` 的情形明确拒绝 501，避免对乐观并发条件静默降级。[来源](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/runtime/src/action-execution.ts#L700-L791)。
5. `/data` 写入还有一类“数据已落地但父级汇总计算失败”的错误分支：当异常携带已挣得的成功响应时，入口按该响应返回并记录汇总错误，避免调用方因误判写入失败而盲目重试。证据是代码分支与注释，不是本轮运行复现。[来源](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/runtime/src/domains/data.ts#L57-L83)。

### 引擎实际程序层：不能止步于 `engine.ts`

- `ObjectQL.find` 委托 `FindFacet.find`；Facet 对查询别名、未知选项、投影、字段可评价性进行处理，构造 AST 后执行中间件/钩子，并在驱动 `find` 前组合所需读取范围表达式；随后处理声明字段、公式/关联、afterFind 和敏感字段遮蔽。[公开入口](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/objectql/src/engine.ts#L4297-L4314)、[Facet 实现](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/objectql/src/engine-find.ts#L419-L644)。
- `ObjectQL.insert` 委托 `InsertFacet.insert`，先检查写入可用性和事务来源、规范化数据，再通过中间件进入默认值/钩子/验证/driver.create 等阶段。可见证据包括调用边界、部分声明字段与引用验证、批次写入差异；**尚未审查全部异常、并发和补偿路径**。[公开入口](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/objectql/src/engine.ts#L4690-L4696)、[Facet 开头](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/objectql/src/engine-insert.ts#L312-L370)、[存储位置](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/objectql/src/engine-insert.ts#L1322-L1490)。
- `ObjectQL` 的 ambient transaction 与跨数据源限制在引擎中有显式设计说明；本批已看见 insert 的 `enforceTransactionOrigin` 调用，但还未闭合该函数对不同驱动的所有裁决/测试，暂不宣称跨数据源事务已被完整保障。[相关入口](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/objectql/src/engine.ts#L390-L433)。

## 4. 安全与业务边界：三个不得混写的判断

**F1｜API 暴露并非全部授权**：`refuseUnexposedDataAction` 检查对象 API 暴露，处理系统上下文、缺失声明默认值和管理 API-key 的额外限制；数据域还有匿名与环境门。它们不能替代记录级权限（RLS）、字段级权限（FLS）、租户隔离、身份信任链或副作用授权。协议层源码与消费者仍须进一步追踪。[来源](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/runtime/src/action-execution.ts#L559-L608)。

**F2｜特定读取表达式的 fail-closed 不等于普遍拒绝**：`FindFacet.find` 在调用驱动前组合 `ReadableIds` 叶节点；安全解析器对这一**特定表达式**在安全服务不可用时将结果收缩为 `$in: []`。若查询里没有这种叶节点，该组合器会直接返回 AST，不能据此断言“任何未注册安全服务的普通查询都被禁止”。[查找执行](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/objectql/src/engine-find.ts#L529-L554)、[安全组合器](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/objectql/src/engine-security-resolvers.ts#L167-L247)。

**F3｜独立嵌入的调用责任需外部明确**：轻量示例直接调用引擎，没有模拟用户身份、组织租户、API 暴露或 HTTP 安全。因此必须为使用它的宿主单独回答“如何验证可信身份、权限、组织边界、失败行为和审计”。这是一条设计/测试候选，**不是已经证明的 ObjectStack 安全漏洞**。

## 5. 质量、历史和标准评价（本轮范围内）

- **程序质量正面机制候选**：对可见能力进行显式分层；入口请求合同先校验；回退路径拒绝其不能遵守的版本条件；读取后保证返回数组形状；敏感字段遮蔽和可见字段过滤。它们的完整性依赖后续端到端取证及运行测试。
- **需要对抗的风险**：多个 HTTP/API/MCP 入口可能存在授权门与错误码差异；轻量宿主不应默认继承完整平台安全能力；schema 字段与真实持久化/运行时消费者可能脱节；批量写和附属汇总部分失败必须避免误重试。
- **历史线索，不是已经复核的因果结论**：源码注释指向 ADR-0076、#23491、#23462、#28782、#29986 等修订；本轮未独立读取对应 diff/Issue 和测试运行输出，不能由注释直接证明引入原因和最终修复效果。
- **标准应用**：参照 ISO/IEC/IEEE 42010 的不同关注点/执行视图，ISO/IEC 25010 的功能适合性、安全性、可靠性、维护性，ISO/IEC/IEEE 29148 的可验证请求契约；只是评价维度，不宣称软件或报告已取得标准符合性/认证。

## 6. 提取的第一项知识候选：面向 Java 多入口数据能力

**候选编号 K-01｜共享对象契约与能力分层，不默许安全或并发语义在入口降级**

- **目标消费者**：Java/Spring 后端功能设计、数据服务/统一授权的程序设计、HTTP/消息/后台作业/AI 工具多入口测试。
- **要解决的问题**：同一领域对象通过多种入口访问时，输入合同、执行身份、记录/字段/租户范围、版本前置条件及异常响应可能不一致；抽象数据引擎和 HTTP 运行时为不同责任层。
- **可借鉴的已见机制**：业务 Schema 可由轻量内核与完整平台复用，但可用能力由宿主组合决定；完整 /data 路径有独立入站校验；协议优先与有限回退明确区分；处理不了的 `expectedVersion` 明确拒绝。
- **使用方法（候选，不是正式规定）**：设计时列出不同入口的“身份来源、授权决策权威、业务不变量、输入/输出、事务/重试语义、审计所有者”；检查相同业务动作是否有不同通道；若某个通道无法保证关键前置条件，应明确拒绝或限制能力，而非默认放行。
- **不适用/限制**：不可把 ObjectStack 的 TypeScript API、注册器或运行时结构直接照搬成 Java Engine；不能以本例证明业务状态、一致性、租户与权限模型完整；Java 具体实现需结合现有有效架构与授权契约。
- **验证建议**：构造相同对象的 HTTP、内部服务、后台任务（如有）、AI 工具（如有）用例，比较身份/权限/乐观并发/请求参数/失败码；对无权限、跨租户、空字段、错误版本和部分成功设计负向测试；逐入口保留真实执行证据。
- **证据**：本报告 §2—§4 固定源码及测试来源；当前最多为 **G1（已采集源码证据，局部机制分析）**，不能升格为 G3 或生产已采用；目标公司 DES-ARC 最新版本尚未读取，未作差异裁决。
- **采用状态**：候选，尚未交付任何真实 Java 工程消费者，无收益测量。
- **维护/许可**：ObjectStack 固定源码基线（Apache-2.0 说明需在真实代码采用前核查具体版本、第三方许可和复制条件）；本文仅提炼一般设计机制与测试思路。

## 7. 当前缺口与下一批可执行任务

| 未闭合事项 | 为什么关键 | 下一次证据入口与方式 |
|---|---|---|
| 实际 HTTP Server 挂载及身份分发 | 目前验证的是 `handleDataRequest` 与部分桥接函数，不是所有 API/REST/MCP 实际装配 | 从 HTTP Dispatcher、RestServer 路由和身份装配追到此域；比较暴露门、认证及错误映射 |
| 协议 createData/findData 真实实现及 RLS/FLS/租户约束 | `callData` 源码注释不能直接证明所有授权正确 | 固定源码定位 ObjectStackProtocolImplementation 和权限插件、驱动访问，追踪拒绝条件与测试 |
| ObjectQL 轻量模式的权限责任 | 示例无身份上下文，无法推定全部引擎默认安全语义 | 追注册的中间件/服务与 context 生成；用独立运行/安全测试验证“无服务”场景 |
| 事务与失败的实测语义 | write/hook/batch/summary、回退和跨驱动可能改变副作用 | 实际运行针对性 Vitest、SQLite/驱动集成，记录命令、环境与结果 |
| 历史因果与 ADR 决策 | 注释和 Issue 编号只是线索 | 核读对应 Issue、ADR、修订 diff，区分维护者目标、变更事实和独立推断 |
| Java 与 DES-ARC 消费 | 当前只有初步候选，没有目标系统证据 | 取得 Java 方案正式版本后按同问题/同约束比较，必要时转化为测试或设计变更；不得虚构已采用 |
| 研究标准在新 Agent 的真实生效 | 本轮只是同会话的文本读取成功 | 在新的 Work/Codex 运行环境检查 root 与 nested AGENTS 的加载、操作边界与恢复行为 |

**停止判定**：本次可交付有界的 G1 源码机制事实和 K-01 候选，不能宣布 B1 端到端研究完成，更不能宣布 G3–G5。下一批优先补齐协议授权路径，而不是新增研究规范或重复本篇概述。
