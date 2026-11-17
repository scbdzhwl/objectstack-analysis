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

## 7. 首批记录时的未闭合事项（2026-11-16；后续见 §8—§10）

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

---

## 8. B1-2｜第二批实证：协议数据门、授权中间件和数据操作的实际连接（2026-11-16）

> **增量研究**：本节继承 §0—§7 的相同源码基线 `6c5d31fac7763e7aaebc5e853c17727698e8126a`，补全上次停在 `callData → protocol` 的关键连接，并开始深入真实授权执行点。以下所有“确定”均指**静态源码及明确调用关系已核实**，不是本次启动系统的观察。曾有历史修复的说法只有在独立核查提交/测试后才能升级为历史事实。
>
> **技术问题**：数据请求的 `ExecutionContext`（执行身份/组织/授权上下文）在哪里构造、何处可信？对象级权限、行级 RLS（Row-Level Security）、字段级 FLS（Field-Level Security）以及租户隔离由谁实际决策？在钩子改变写入数据或缺少插件的组合中，安全保证成立的前提是什么？

### 8.1 从请求到数据库的分层路径：已找到调用边界

~~~mermaid
flowchart TD
  IN["HTTP /data 请求"] --> ROUTE["HttpDispatcher / DomainRoute"]
  ROUTE --> ID["resolveExecutionContext / request-scoped authz"]
  ID --> DOM["handleDataRequest: 匿名、环境、API 暴露、请求形状"]
  DOM --> BRIDGE["action-execution.callData"]
  BRIDGE --> P{"有相应 protocol 能力?"}
  P -->|是| CRUDF["ObjectStackProtocolImplementation → DataCrud / DataWrites"]
  P -->|否| FALLBACK["ObjectQL fallback：只提供有限契约"]
  CRUDF --> ENGINE["ObjectQL.find / insert / update / delete"]
  FALLBACK --> ENGINE
  ENGINE --> MWARE["已注册的 ObjectQL 中间件链"]
  MWARE --> DOOR["引擎规则、钩子、范围/图像检查"]
  DOOR --> DRIVER["IDataDriver / 数据库"]
  DRIVER --> RES["结果整理、字段处理、错误响应"]
~~~

**图的前提**：它表示此次核实到的 `HttpDispatcher` 数据域与特定协议路径。REST Server、MCP、声明式 API 与其他宿主并不保证按完全相同的图装配。尤其 `SecurityPlugin` 是通过注册注入而不是被数据引擎硬编码；未安装/启动成功时不能照搬图中的“已注册的安全中间件”这一假设。

| 步骤 | 已见源码事实 | 重要边界 |
|---|---|---|
| 1. 数据域挂载 | `HttpDispatcher` 注册 `createDataDomain(this.domainDeps)`。[http-dispatcher.ts L940–965](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/runtime/src/http-dispatcher.ts#L940-L965) | 表示这一 dispatcher 的路线，不等于所有请求都走它 |
| 2. 请求身份 | `resolveExecutionContext` 调用共享的 `resolveRequestExecutionContext`；后者的授权解析关联 `resolveAuthzContext`；同一执行上下文组装提供认证/访客不同入口。[resolve-execution-context.ts L150–247](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/runtime/src/security/resolve-execution-context.ts#L150-L247)、[resolve-authz-context.ts L569–760](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/core/src/security/resolve-authz-context.ts#L569-L760)、[assemble-execution-context.ts L429–456](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/core/src/security/assemble-execution-context.ts#L429-L456) | **身份解析并不等于所有业务权限已经被执行** |
| 3. HTTP 数据门 | `handleDataRequest` 检查匿名请求、目标环境；POST 还检查对象 API 暴露及 `CreateDataRequestSchema`。[data.ts L178–219](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/runtime/src/domains/data.ts#L178-L219)、[data.ts L488–516](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/runtime/src/domains/data.ts#L488-L516) | 请求 Schema 校验不是业务规则或 RLS |
| 4. 桥接选择 | `callData` 优先请求 `protocol` 对应方法；缺失相应方法时查找 `ObjectQL` 能力。[action-execution.ts L612–689](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/runtime/src/action-execution.ts#L612-L689) | 协议路径与回退路径的响应、并发条件、可用能力并非天然等价 |
| 5. 协议实现 | `ObjectStackProtocolImplementation.findData/getData/createData` 委托 `ProtocolDataCrudFacet` / `protocol-data-writes`，再执行 `host.engine.find/findOne/insert`。[protocol.ts L5148–5200](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/metadata-protocol/src/protocol.ts#L5148-L5200)、[protocol-data-crud.ts L691–742](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/metadata-protocol/src/protocol-data-crud.ts#L691-L742)、[protocol-data-writes.ts L43–113](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/metadata-protocol/src/protocol-data-writes.ts#L43-L113) | 类头与注释只是入口，程序主体在已拆出的模块 |
| 6. 中间件执行 | `ObjectQL.executeWithMiddleware` 依据匹配对象筛选已注册的中间件，依次调用，最内层才执行真实操作。[engine-middleware-and-context.ts L104–167](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/objectql/src/engine-middleware-and-context.ts#L104-L167) | 没有注册的安全插件不能靠这个机制“自动补齐” |
| 7. 授权插件 | `SecurityPlugin.start` 对缺少对象引擎或中间件支持的组合会提前返回；正常启动时注册安全服务和 `authzMiddleware`。[security-plugin.ts L1878–1896](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/plugins/plugin-security/src/security-plugin.ts#L1878-L1896)、[security-plugin.ts L2280–2341](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/plugins/plugin-security/src/security-plugin.ts#L2280-L2341) | 只有读取有效运行组合及其启动结果，才能确认某部署**实际受该授权层保护** |

### 8.2 一个特别重要的信任边界：不得接受客户端伪造的执行上下文

`ProtocolDataCrudFacet.findData` 首先展开来自网络查询的 `request.query`，随即 **无条件删除** `options.context`，之后只有当协议请求的 `request.context` 已由可信上游提供时，才重新放入引擎请求；其余查询参数继续解析成有限的查询 AST。[protocol-data-crud.ts L68–122](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/metadata-protocol/src/protocol-data-crud.ts#L68-L122)

这不是抽象原则，而是可直接定位的代码顺序：

~~~text
request.query                       // 网络输入：不可信
  → {...request.query}
  → delete options.context          // 不允许请求参数冒充系统执行上下文
  → if(request.context !== undefined) options.context = request.context
  → 规范化 where / orderBy / fields / page window
  → engine.find(object, findQuery)  // 携带独立解析出的上下文
~~~

**安全含义**：若客户端通过请求体或参数放入 `context: {isSystem:true}`，这一层不应该把它当可信的授权旁路。真正的可信前提是**上游调用方必须自己可信地产生 `request.context`**，不能从网络正文直接复制过来。与之配合，`SecurityPlugin` 的执行中间件把 `context.isSystem` 当系统操作分支，只对部分受限自动化写入保留额外检查。[security-plugin-authz-middleware.ts L462–558](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/plugins/plugin-security/src/security-plugin-authz-middleware.ts#L462-L558)

`getData` 的 `request.context` 同样通过专门参数传给引擎，而不是从 `query` 形状捡取。[protocol-data-crud.ts L825–900](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/metadata-protocol/src/protocol-data-crud.ts#L825-L900)。这是两个具体方法的源码结论；不能据此跳到“全部接口都不存在上下文伪造风险”。

### 8.3 授权为什么是多个裁决层，而非一次 boolean 检查

**(1) 主体及权限集合解析**：`resolveAuthzContext` 的职责是把用户、API Key、会话、组织归属、岗位、权限集等解析为请求授权信息；`SecurityPlugin` 的中间件再从操作上下文解析有效权限集与被委托者。[resolve-authz-context.ts L569–760](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/core/src/security/resolve-authz-context.ts#L569-L760)、[security-plugin-authz-middleware.ts L700–813](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/plugins/plugin-security/src/security-plugin-authz-middleware.ts#L700-L813)。

**(2) 对象级 RBAC（Role-Based Access Control）**：安全中间件调用 `PermissionEvaluator.checkObjectPermission(operation, object, ...)` 判断操作许可；无法解析对象安全声明（posture）的情况下，它有拒绝逻辑。[security-plugin-authz-middleware.ts L973–1054](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/plugins/plugin-security/src/security-plugin-authz-middleware.ts#L973-L1054)、[security-plugin-authz-middleware.ts L700–813](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/plugins/plugin-security/src/security-plugin-authz-middleware.ts#L700-L813)。这与 HTTP 的 `apiEnabled` 或对象 API 暴露控制是**两个不同问题**：一个问“该接口是否对外提供此动作”，另一个问“该主体是否允许执行此业务对象动作”。

**(3) 行级 RLS**：对于带 `ast` 的查询，安全中间件经 `computeLayeredRlsFilter` 组合平台租户层与业务策略，必要时叠加受委托者及父子对象读取范围，最终 AND 到查询条件里，随后引擎将已成形的过滤条件交给驱动。[security-plugin-authz-middleware.ts L2480–2610](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/plugins/plugin-security/src/security-plugin-authz-middleware.ts#L2480-L2610)、[engine-find.ts L419–552](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/objectql/src/engine-find.ts#L419-L552)。**RLS 不是“查出全部行再在页面隐藏”**。

**(4) 按 ID 更新与删除**：这种操作可能由 `driver.update(object,id,...)` / `delete` 执行而没有读取查询 AST；所以安全中间件对指定 ID 的写入先检查对应的行级写入范围，必要时在调用者上下文下重新读取目标记录，并区分记录不可读、无写权限和存储异常。[security-plugin-authz-middleware.ts L1115–1355](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/plugins/plugin-security/src/security-plugin-authz-middleware.ts#L1115-L1355)。这与对查询 AST 注入 `where` 过滤的机制不同，不能只测试列表权限就声称写权限完整。

**(5) 字段级 FLS**：除了字段写入限制与隐藏字段不得用于过滤/聚合的检查之外，安全中间件在执行后遮蔽返回记录，覆盖读取和写入回显，避免“不能读取的字段通过 PATCH 返回体泄露”。[security-plugin-authz-middleware.ts L2611–2664](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/plugins/plugin-security/src/security-plugin-authz-middleware.ts#L2611-L2664)。字段输出过滤与前置授权是不同责任点。

**(6) 委托执行与系统身份**：安全代码包含 `isSystem` 旁路、特定自动化系统写入限制，以及代表他人执行时的权限与行范围组合；这不是一个简单的“管理员绕过所有安全”的模型。具体限制见 [security-plugin-authz-middleware.ts L462–558](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/plugins/plugin-security/src/security-plugin-authz-middleware.ts#L462-L558)、[security-plugin-authz-middleware.ts L1115–1355](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/plugins/plugin-security/src/security-plugin-authz-middleware.ts#L1115-L1355)、[security-plugin-authz-middleware.ts L2480–2610](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/plugins/plugin-security/src/security-plugin-authz-middleware.ts#L2480-L2610)。本批没有核实所有 MCP/Action/Flow 到这套执行上下文的传递，不宣称所有委托入口已统一。

### 8.4 写入安全的时间维度：为什么必须在 Hook 之后再裁决一次

**发现的专业机制**：只对用户提交的原始数据检查租户 `organization_id`，不够。因为 `beforeInsert/beforeUpdate` 可以生成、修改字段，**最终实际写入的数据与最初送入的数据可能不同**。

对应实现包含两个互补检查：

1. 中间件在 `next()` 之前检查调用者的原始写入数据是否符合租户范围；即使后续 Hook 会替它改成合法值，也不应使原来的非法输入获得许可。
2. 通过 `OperationContext.postHookWriteImageCheck` 把**最终准备写入的记录图像（post-hook image）**交给安全执行层检查。业务规则 `check` 与租户层可以在同一校验钩子上组合；引擎必须实际调用它，若应调用却没有调用，中间件不得把结果认定为已获得授权担保。[security-plugin-authz-middleware.ts L2370–2448](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/plugins/plugin-security/src/security-plugin-authz-middleware.ts#L2370-L2448)、[security-plugin-authz-middleware.ts L2611–2664](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/plugins/plugin-security/src/security-plugin-authz-middleware.ts#L2611-L2664)。

**已经补齐到真正的引擎调用点**：插入流程在 `beforeInsert` 钩子后筛选仍有效的行，并设置 `postHookWriteImageCheck.honoured = true`，再调用 `evaluate(live)`；按 ID 更新用“先前记录 + 最终变化量”执行同一校验；按谓词更新则对匹配记录逐条构造最终图像并裁决。对应 [engine-insert.ts L1019–1037](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/objectql/src/engine-insert.ts#L1019-L1037)、[engine-update.ts L1559–1575](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/objectql/src/engine-update.ts#L1559-L1575)、[engine-update.ts L1984–2007](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/objectql/src/engine-update.ts#L1984-L2007)。因此这里不仅是插件“声明要检查”，而是已经定位到引擎准备调用驱动前的实际校验调用。**但 `honoured` 只表示校验点运行，不表示写入在任何驱动或外部宿主中都已经原子回滚**；真正的事务语义仍需验证。

`tenant-wall-post-hook-image.test.ts` **测试源码**覆盖独立插入、批量插入、按 ID/按谓词更新、越组织、`group` 范围、系统旁路以及“宿主未执行图像校验”的情形。[tenant-wall-post-hook-image.test.ts L260–368](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/plugins/plugin-security/src/tenant-wall-post-hook-image.test.ts#L260-L368)。这为后续真实运行提供了针对性测试入口；目前只核查了测试设计，未实际执行通过。

**架构判断（静态 G2 局部）**：这一设计的可迁移价值在于**定义安全不变量的最终裁决位置**，而不是简单复制一个 `beforeSave` 监听器。任何可改变结果的钩子、默认值计算、状态流转或自动化动作，都应纳入最终约束验证的观察范围；具体应在目标事务的哪个阶段执行，还要验证数据库一致性与失败回滚。

### 8.5 错误语义、乐观并发与事务：不能过早下结论

| 关注点 | 源码证据与当前确定事实 | 必须保留的未知 |
|---|---|---|
| 不合法请求 | `POST /data` 在调用写入桥前按 `CreateDataRequestSchema` 检查；有两个入口的请求形状对照测试。[data.ts L488–516](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/runtime/src/domains/data.ts#L488-L516)、[data-write-request-gate.test.ts L350–400](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/runtime/src/domains/data-write-request-gate.test.ts#L350-L400) | 未执行测试，也未证明其他 API 的完全一致性 |
| 业务写入失败 | `createData` 调用 `engine.insert`，有 `droppedFields/warnings` 等响应和内部字段去除。[protocol-data-writes.ts L43–113](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/metadata-protocol/src/protocol-data-writes.ts#L43-L113) | Hook、事件、汇总重算造成的失败/部分成功尚未闭合 |
| 版本冲突 | `updateData/deleteData` 在写前调用 `assertVersionMatch`；其实现先判断 token 格式、按 ID 查询版本，必要时返回冲突。[protocol-data-writes.ts L114–172](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/metadata-protocol/src/protocol-data-writes.ts#L114-L172)、[protocol-data-writes.ts L263–320](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/metadata-protocol/src/protocol-data-writes.ts#L263-L320)、[protocol-record-reads.ts L203–263](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/metadata-protocol/src/protocol-record-reads.ts#L203-L263) | 这是一条**读版本→再写入**的可见调用顺序；尚未证明最终存储更新是否包含原子版本条件 |
| SQL 更新 | ObjectQL 的按 ID 分支可以调用 `driver.update(object,id,data,...)`；另有 `writePrecondition` 时才选择 `updateByIdExpecting`。[engine-update.ts L1613–1624](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/objectql/src/engine-update.ts#L1613-L1624) | `protocol.expectedVersion` 是否与 `writePrecondition` 关联、其他保护机制是否覆盖并发窗口，尚需追踪；不据此认定已存在数据竞争缺陷 |
| 跨数据源事务 | `enforceTransactionOrigin` 对引擎拥有的事务来源做跨数据源写入裁决，业务写入与某些系统账本行为有不同处理。[engine-execute-transactions.ts L205–280](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/objectql/src/engine-execute-transactions.ts#L205-L280) | 所有驱动、嵌入模式、外部事务和补偿流程未验证 |

**关于 OCC（Optimistic Concurrency Control，乐观并发控制）的对抗假设 H-01**：

> 两个请求 A、B 同时读取旧版本 v1，通过协议的版本比较；如果 A 更新成功后 B 的存储写入没有原子地携带 v1 前置条件，B 可能覆盖 A。必须用确定的代码连接、实际驱动条件更新或可重复的并发测试来证明该窗口被关闭，或者证明问题实际成立。

目前只有“协议中的版本检查与实际写入分开出现”这一**源码事实**。不能因找不到一段代码就断言没有其他执行层补偿，也不能凭用例名称直接宣称并发控制成立。`protocol.occ-version-token-instant.test.ts` 主要提供版本 token 解析/比较的测试证据入口，不等同于双请求交错更新的原子性测试。[protocol.occ-version-token-instant.test.ts L290–422](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/metadata-protocol/src/protocol.occ-version-token-instant.test.ts#L290-L422)。

### 8.6 历史资料第一次实质交叉核对：设计声明与实现证据如何配对

**ADR-0049：安全属性必须有真实消费者。** 正式决策文本提出安全属性只能处于“已被执行、明确标记实验性、从协议移除”三类状态，并指出仅能解析却无法执行的权限配置会制造虚假的安全承诺。[0049-no-unenforced-security-properties.md L1–85](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/docs/adr/0049-no-unenforced-security-properties.md#L1-L85)。这份 ADR 是作者对原则、缺陷和历史处置的**原始声明**；本批已从 `SecurityPlugin` 注册和中间件判断处找到部分实际消费者，但尚未完成逐一验证所有安全属性的实际覆盖。

**ADR-0076：ObjectQL 轻量核心与完整平台分层。** ADR 说明引擎与元数据管理协议的拆分，并对 D1/D2/D3 等决定分别记录实施/未实施/推迟；本批已以 `ObjectQL/core` 示例与数据协议拆出文件实证部分边界。[0076-objectql-core-tiering.md L1–73](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/docs/adr/0076-objectql-core-tiering.md#L1-L73)。其中 D3 的 profile/capability 挂载在该 ADR 版本内被记作未实现，应视为**该历史 ADR 的状态声明**，不得未经当前代码和真实装配检验就断言所有运行组合没有安全保障。

同一条研究链因此出现两种不同的专业知识：一是“可执行的安全契约应有真实消费者”，二是“数据引擎与授权能力的组合必须明确说明宿主的责任”。两者有关联，但不能压缩成一句“用插件做权限”。

## 9. 工程知识候选 K-02｜安全不变量在多入口及写入生命周期中的执行位置

| 项目 | 可用于目标工程的候选知识（非现行公司规范） |
|---|---|
| 核心问题 | 同一个数据对象既有 HTTP/内部服务/自动化/AI 工具入口，又可能经过 Hook/规则/计算改变最终存储内容，单点校验易产生授权/业务语义偏差 |
| 知识类型 | 系统架构、程序设计、身份与租户契约、RLS/FLS、异常与对抗测试、历史失败经验 |
| 必须分开的职责 | **入口**可信身份、输入形状和对外暴露；**领域/执行层**操作与业务不变量；**数据访问安全层**对象、行、字段和租户范围；**最终存储前**对 Hook 后结果与事务约束作最终裁决 |
| 机制使用建议 | 对每个实际入口记录其可信 `ExecutionContext` 产生者、授权执行者、写入图像改变者、最后一个约束执行点、错误/回滚的归属；不能只验证 Controller 上的注解 |
| 可复用测试结构 | (a) 参数中伪造 `isSystem`；(b) 对象允许读但不能改；(c) 能改对象但不能改指定行；(d) 能编辑但不能读取敏感字段，禁止写入响应回显；(e) Hook 将 `organization_id` 改到无权限组织；(f) 系统/自动化旁路；(g) 缺授权服务或部分中间件时的启动/拒绝；(h) 两请求交错的条件更新 |
| 适用边界 | 适用于多入口、多租户、可扩展 Hook/Action/Flow 的系统；小型单租户嵌入可选不同授权模型，但必须**明确由谁承担且通过验证** |
| 不可照搬 | ObjectStack 的 TypeScript `ExecutionContext`、`registerMiddleware`、`PostHookWriteImageCheck` 不是 Java 标准结构；不能据此引入第二套 Spring Security 或与现有数据权限体系并存的平行规则 |
| 证据状态 | ObjectStack 固定版源码的已定位机制与测试设计；**G1 + 局部 G2 静态分析**，未独立运行、未证明所有入口完整；**未被任何现有 Java 项目实际采用** |
| 下一消费动作 | 在现有 Java 权限与对象数据设计任务中，以“入口→执行身份→授权→状态变更→持久化→响应/失败”的同一对象为单位生成差异与测试矩阵；取得现行 DES-ARC 正式版本后再比较，不自动提交修改 |

**参考入口**：§8.2—§8.5 的源码与测试链接；尤其关注 [protocol-data-crud.ts L68–122](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/metadata-protocol/src/protocol-data-crud.ts#L68-L122)、[security-plugin-authz-middleware.ts L973–1054](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/plugins/plugin-security/src/security-plugin-authz-middleware.ts#L973-L1054)、[security-plugin-authz-middleware.ts L2480–2610](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/plugins/plugin-security/src/security-plugin-authz-middleware.ts#L2480-L2610)、[security-plugin-authz-middleware.ts L2370–2448](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/plugins/plugin-security/src/security-plugin-authz-middleware.ts#L2370-L2448)、[tenant-wall-post-hook-image.test.ts L260–368](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/plugins/plugin-security/src/tenant-wall-post-hook-image.test.ts#L260-L368)。

## 10. 本批对抗检查、完成边界与下一步

| 对抗问题 | 证据基础 | 本次裁决 |
|---|---|---|
| 客户端塞入 `query.context.isSystem` 是否直接获得平台权限？ | `findData` 清除输入 context、仅重放请求专属 context；中间件把可信 `isSystem` 作为特殊分支 | 已核实一个**防混淆代码边界**；其他入口未验证 |
| 只做 API 暴露检查是否足以防越权？ | 引擎另有对象权限、RLS、字段及租户执行机制 | **不足**，不能用“接口有 gate”替代底层策略执行；具体部署必须核实插件已装配 |
| 一条 INSERT 在 Hook 后更改组织字段怎么办？ | 安全中间件安装 post-hook 检查；测试源码覆盖多写入方式 | 机制有实际源码和测试；测试执行及驱动行为本轮未验证 |
| 系统上下文是否无条件跨一切权限？ | `isSystem` 有旁路，但针对部分自动化写入有先行限制 | 无法用单一 boolean 说明所有安全与业务约束 |
| OCC 检查是否等于原子比较更新？ | 版本探测函数与存储更新分离；部分引擎更新有独立 `writePrecondition` 分支 | **H-01 待证**，不能断言安全或漏洞 |
| 所有入口是否都必须经过 `SecurityPlugin`？ | 插件通过引擎注册中间件，缺所需服务可不注册；`core` 有独立嵌入 | **取决于部署与可信宿主契约**；待验证默认组合、必选能力和启动行为 |
| ADR 声称修复是否可以直接当历史实证？ | ADR 是明确的作者决策与状态说明，源码有部分对应执行 | 仍需逐项核实相关提交 diff、真正运行测试与影响范围 |

**当前状态**：这是第二批**静态源码机制研究**，不是完整的 B1 端到端验收。已核对协议方法→ObjectQL 实现→已注册安全中间件→读/写与响应处理的主要连接；数据库真实 SQL、所有驱动的多租户约束、外部 REST/MCP 装配、业务角色全景及并发实测仍未完成。

**下一项优先任务**：沿 `writePrecondition` 与协议 `expectedVersion` 的实际连接和驱动执行条件检验 H-01；并对 `SecurityPlugin` 缺席/失败、租户图像检查、字段读写过滤开展**真实的定向测试**（在能运行固定源码的独立工作区内）；然后再选 Java 成熟项目与现有设计进行同约束比较。不要继续增加管理模板。

