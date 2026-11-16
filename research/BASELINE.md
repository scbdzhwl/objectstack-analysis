# ObjectStack 研究基线与首轮证据登记

> 核查日期：2026-11-15。以下是对 GitHub API 和选定源码文件的初步读取；**未在本地克隆、编译、运行或测试**。

## 1. 研究快照

- 用户 Fork：https://github.com/scbdzhwl/objectstack-analysis
- 上游：https://github.com/objectstack-ai/objectstack
- 关系：GitHub API 确认 fork=true，parent=objectstack-ai/objectstack。
- 固定快照：**6c5d31fac7763e7aaebc5e853c17727698e8126a**；所有研究源码链接默认按此 commit 固定。
- 核查时，上游 main 为 6e287974e2d2fcff54b67a89d14bbb6bba9c3ead，上游相较 Fork 已前进约 152 次提交；该数字是一次性快照，不自动更新。
- 主分支为 main；许可证：Apache-2.0（具体商业服务和自托管边界应再次核对 LICENSING.md）。
- Git 树扫描约 25,303 个 blob 文件和 1,135 个目录；按文件名规则检测到约 14,436 个测试文件，不等于实际测试执行率、有效覆盖率或全部通过。
- 当前工作目录仅作为研究独立区域；Fork 原始文件和根 AGENTS 都是上游资料。

## 2. 已打开的核心入口及待核查问题

| 原始入口 | 本轮可确认的事实 | 需要继续核查 |
|---|---|---|
| [README.md](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/README.md) | 定位为“可执行业务本体”，声称 AI 编写元数据并由运行时消费 | 宣称功能对应哪些消费者和已执行证据 |
| [ARCHITECTURE.md](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/ARCHITECTURE.md) | 已由长文改成指向当前协议架构文档 | 不能用早期三层架构文档充当现行主权威 |
| [architecture.mdx](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/content/docs/concepts/architecture.mdx) | 以协议分层解释 Data/System/UI/Automation/AI/Marketplace 及 ObjectQL/Kernel/ObjectUI 引擎视图 | 标题“四层”与所列六协议/三引擎需核对口径，不先推断矛盾已成立 |
| [architecture-overview.mdx](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/content/docs/concepts/architecture-overview.mdx) | 声明 Zod→compiled artifact→runtime→REST/UI/MCP，提到 UI 来源曾在其他仓库 | 当前 Fork 包含 ui/，需查明是否迁入、子项目/构建源与叙述是否过期 |
| [docs/NORTH-STAR.md](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/docs/NORTH-STAR.md) | 描述业务/AI 双用户与端到端应用验证目标 | 其中 Roadmap/Definition of Done 属上游作者目标，不等于已验证功能 |
| [AGENTS.md](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/AGENTS.md) | 上游 AI 协作规则，包含构建、代码和开发纪律 | 与研究本身的目标不同；需用 research/AGENTS 区分 |
| [packages/core/src/kernel.ts](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/core/src/kernel.ts) | 存在插件生命周期、服务装配、降级启动信号等实现入口 | 必须追踪每个分支和真实调用者，不依据注释直接断言行为 |
| [packages/objectql/src/engine.ts](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/objectql/src/engine.ts) | 数据引擎入口包含查询、hook、事务及多类子模块 import | 跟踪数据写入、授权门禁、并发和错误语义 |
| [packages/spec/src/data/object.zod.ts](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/packages/spec/src/data/object.zod.ts) | 对象 Schema 与字段、Action、分享范围等定义入口 | 概念声明与 ObjectQL/runtime 实现映射，识别无消费者声明 |
| [examples/app-showcase/src/coverage.ts](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/examples/app-showcase/src/coverage.ts) | 功能元数据种类被标为 demonstrated 或 waived，存在显式缺口 | 测试是否覆盖真实 runtime；waived 的具体原因和追踪状态 |
| [examples/README.md](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/examples/README.md) | Todo、CRM、Showcase、Embed ObjectQL 为本仓库示例；HotCRM 为外部仓库 | 业务设计与框架功能必须分开取证 |

## 3. 当前重要问题（先登记、后查证）

1. 架构资料各自的适用范围、最新状态与冲突：README、NORTH-STAR、architecture、architecture-overview 和 ADR 之间如何裁决？
2. Fork 内 ui/ 的具体来源与构建接入方式，和 overview “UI source outside repo” 表述如何对齐？
3. 展示样例 demonstrated/waived 所对应的真实调用者、测试覆盖及运行能力是什么？
4. Object Schema、权限/Policy、Action、Flow、MCP 是否在所有入口执行同一契约？哪些是不同同名语义？
5. 完整 CRM 业务设计需要另取 HotCRM 固定快照；不能用 app-crm 小示例代替。
6. 用户既有 DES-ARC/SEM 正式最新文件尚未在本仓库提供，跨体系结论不得用历史 L0/L1 代替。

## 3.1 对抗审查中新核实的两个源码边界（2026-11-15）

- **功能“已展示”的证据强度**：[`examples/app-showcase/test/coverage.test.ts`](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/examples/app-showcase/test/coverage.test.ts) 存在元数据 registry 校核、对应文件存在性及 `waived` 必须提供 Issue 的测试；[`examples/app-showcase/src/coverage.ts`](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/examples/app-showcase/src/coverage.ts) 中 `picklist` 被标为 `waived`。这些静态代码事实不证明被 `demonstrated` 的所有能力已经端到端运行验证。
- **Kernel 适用域**：[`content/docs/concepts/architecture.mdx`](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/content/docs/concepts/architecture.mdx) 声明请求通过 Kernel；[`examples/embed-objectql/README.md`](https://github.com/scbdzhwl/objectstack-analysis/blob/6c5d31fac7763e7aaebc5e853c17727698e8126a/examples/embed-objectql/README.md) 明确描述无需 Kernel 直接嵌入 ObjectQL 引擎。当前只能确认两种描述/形态存在；必须在实际实现和完整部署场景中核查授权、访问及数据边界，**不得直接认定越权漏洞，也不得无条件采信一条总述**。

## 4. 本轮实证完成界限

已确认公开 Fork 仓库元数据、Git 树与少量直接源码。未执行本地命令、依赖安装、静态检查、CI、E2E、功能验证、角色/越权测试及性能实验。报告和任务状态必须沿用这个区分。

## 5. 后续更新纪律

- 更换基线先记录新 commit、差异、受影响结论，不直接覆盖旧结果。
- 每新增一项关键判断，补充定位到 commit+源码范围或复现实验记录。
- 有证据不足就记录“未知/待验证”，不得以目录名、文档宣称或模型经验填空。
## 6. 研究设计输入与权威边界（2026-11-15）

当前 DESIGN.md 综合了用户在会话中的两轮专业反向审查、此前“寻找探宝讨论”所述老资产和外部经验提取目标，以及《Code-Assets-Evolution-Research-2026-11-10》《DES-01-分析利用工作区设计与迁移方案》《PLN-01-分析利用主线与阶段执行计划》等历史材料。

**来源可得性声明**：这些跨对话/文件库历史材料不在当前 Fork 的研究目录下，不能据 DESIGN.md 自称已在此仓库完整取得或复核原件。DESIGN.md 保存为后续工作可读的**综合设计判断**；若后续将这些历史材料作为具体事实证据，需独立登记其准确原件、版本、引用和可读范围。上游 ObjectStack README/AGENTS/架构仍均作为研究对象而不是本研究的最高规则。

最新提交的 research 文件版本与被研究源码固定 SHA 不是同一个概念；即使研究计划/设计继续更新，也不得静默改写代码研究基线。

## 7. 2026-11-16 源码与研究成果版本核对

- 本轮开工时 GitHub main 为 `c25f9250b819d19ee4cfe168f53cb9a59dd41d1e`（用户已将 PR #3 合并，并进一步同步了上游）。这是**研究文件工作树版本**，与本轮被分析的源码基线 `6c5d31fac7763e7aaebc5e853c17727698e8126a` 不同；不得把新 main 的实现混入这份旧快照研究。
- 第一批实际读取的源码文件、符号和验证强度见 [RES-03](reports/03-runtime-callchains.md)；它的代码事实仍依赖固定快照。`examples/app-showcase/src/coverage.ts` 中 32 个元数据类别的 25/7 分类仅指声明式样例覆盖清单，不能解释为执行通过。
- 研究规则的文件可读取性已在本会话核查；**未执行新的独立 Agent 加载或 IDE/Codex 指令作用域实测**。
- 当需要采用最新 ObjectStack 时，先重新固定上游 commit，按影响范围对 RES-03 的 API/Facet/入口逐项差异复核，而不是直接更换本文的 source_commit。

## 8. 2026-11-16 协议与授权机制证据追加

- **源码基线保持**：`6c5d31fac7763e7aaebc5e853c17727698e8126a`。本批研究文件工作基线为 `663a36fc7b0fba24d0e2574e1658e0f3399de7f1`，与被研究源码不是同一身份。PR 审查/合并后应检查研究文件 HEAD，不得改写源码时间线。
- [RES-03 §8—§10](reports/03-runtime-callchains.md) 记录 `ProtocolDataCrudFacet.findData`、`protocol-data-writes`、身份解析、SecurityPlugin 中间件注册/对象授权/RLS/FLS/租户写入，以及引擎更新和 OCC 的静态证据。
- 真实反例：`findData` 删除网络查询参数里的 `context` 再加入可信请求上下文；`SecurityPlugin.start` 不能取得 ObjectQL/metadata 时不注册授权中间件；Hook 后组织字段需要最终图像校验。均须以具体部署、入口与实测补充适用域。
- **未证实假设 H-01**：协议 `assertVersionMatch` 的读取与写入在代码中分步出现，最终是否有数据库原子前置条件尚未验证；不将其标为已确认竞争缺陷。
- 检查只读取测试源码（含 tenant wall post-hook、协议 OCC token、data write request gate），**未实际运行任何测试或新 Agent/IDE 指令自动加载实验**。
