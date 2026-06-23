# Requirements Document — CMP-NG 连接管理平台 V3

## Introduction

CMP-NG 是一个面向企业客户的下一代物联网连接管理平台。平台以"多资源方热插拔 + 强大计费引擎 + 海量设备管理"为核心竞争力，通过对接多种上游网络供应商，向下游企业提供统一的 SIM / eSIM / Profile 生命周期管理、弹性套餐订阅、实时话单采集、多维计费结算、设备群组运维、智能告警与分析决策能力。

平台采用"可视化 Portal + 开放式 API + Webhook 事件推送"三通道交互模式。

## Glossary

| 术语 | 定义 |
|------|------|
| **租户 (Tenant)** | 顶级企业客户或 reseller，拥有独立的账户体系、资产空间和计费结算 |
| **子账户 (Sub-Account)** | 租户下的二级组织单元，用于部门级资产隔离和独立计费 |
| **用户 (User)** | 操作执行者，归属特定租户或子账户 |
| **角色 (Role)** | 权限集合，支持 RBAC 模型 |
| **资源方 (Resource Provider)** | 网络供应商，通过标准化适配器接入平台 |
| **资源方适配器 (Provider Adapter)** | 封装资源方 API 差异的标准化插件，支持热插拔 |
| **资产 (Asset)** | SIM 卡、eSIM 设备、Profile 的统称 |
| **设备群组 (Device Group)** | 资产的逻辑分组，用于批量管理和策略下发 |
| **套餐模板 (Plan Template)** | 预定义的计费规则模板，支持参数化配置 |
| **套餐实例 (Plan Instance)** | 绑定到资产的套餐模板实例，包含生效时间、价格快照等 |
| **计费引擎 (Billing Engine)** | 独立微服务，负责多维度计费计算和账单生成 |
| **话单 (CDR)** | 网络使用记录，含流量、短信、语音等维度 |
| **策略 (Policy)** | 自动化规则，基于条件触发资产操作或告警 |
| **Webhook** | 平台事件推送机制，允许外部系统订阅资产变更、告警等事件 |

---

## Requirements

### REQ-1: 多租户与账户体系

**User Story:** AS 平台运营方，I want 管理多层级租户体系，so that 不同企业客户之间的数据和资产完全隔离。

#### Acceptance Criteria

1. THE 系统 SHALL 支持多租户架构，每个租户拥有独立的数据空间和结算账户。
2. WHEN 创建租户，THE 系统 SHALL 为租户生成全局唯一编码并初始化默认管理员角色。
3. THE 系统 SHALL 支持租户下创建多级子账户，层级深度不超过 3 级。
4. WHEN 查询或操作资产数据，THE 系统 SHALL 将范围限定在当前租户及其子账户内。
5. THE 系统 SHALL 支持 reseller 租户通过 API 管理其名下的二级企业客户。
6. WHEN 停用租户，THE 系统 SHALL 级联冻结该租户及其子账户下的所有写操作。
7. THE 系统 SHALL 支持租户级别的品牌定制（Logo、主题色、Portal 标题），实现白标 Portal 能力。

---

### REQ-2: 用户与权限管理

**User Story:** AS 租户管理员，I want 管理组织内用户和角色权限，so that 不同人员按授权范围操作平台。

#### Acceptance Criteria

1. THE 系统 SHALL 基于 RBAC 模型管理权限，支持角色自定义和权限点组合。
2. THE 系统 SHALL 预设以下标准角色模板：超级管理员、资产管理员、只读用户、计费管理员、审计员。
3. WHEN 为用户分配角色，THE 系统 SHALL 将权限范围限定在用户所属租户或子账户内。
4. THE 系统 SHALL 支持权限点细化到页面级别和 API 级别。
5. WHEN 用户状态由启用切换为停用，THE 系统 SHALL 立即吊销该用户所有活跃 token。
6. THE 系统 SHALL 对用户密码进行不可逆加密存储。
7. WHEN 用户修改密码，THE 系统 SHALL 校验新密码与历史前五次密码不重复。
8. THE 系统 SHALL 支持双因素认证（TOTP），租户管理员可强制启用。

---

### REQ-3: 资源方管理与适配器框架

**User Story:** AS 平台管理员，I want 通过标准化适配器快速接入新的网络供应商，so that 平台可以灵活扩展上游资源。

#### Acceptance Criteria

1. THE 系统 SHALL 定义统一的资源方适配器接口（Provider Adapter Interface），包含以下能力：获取 Token、获取套餐列表、资产订阅/暂停/恢复/退订/更换套餐、获取话单、查询资产详情。
2. WHEN 接入新的资源方，THE 系统 SHALL 允许通过实现适配器接口并配置注册来上线，无需修改核心业务代码。
3. THE 系统 SHALL 支持资源方适配器的热加载与热卸载，不影响已运行的其他资源方。
4. THE 系统 SHALL 对每个资源方的 API 密钥、密码进行加密存储和传输。
5. THE 系统 SHALL 为每个资源方维护独立的连接池和超时配置。
6. WHEN 资源方的 API 密码即将过期（距离过期日不足 7 天），THE 系统 SHALL 向超级管理员发送邮件和站内通知。
7. WHEN 资源方状态为停用或密码已过期，THE 系统 SHALL 拒绝所有面向该资源方的业务调用，并返回明确错误。
8. THE 系统 SHALL 记录资源方每次 API 调用的请求/响应摘要到审计日志。
9. THE 系统 SHALL 为资源方提供健康检查端点，定期探测可用性，连续失败 3 次后自动标记为"异常"并告警。

---

### REQ-4: 资产生命周期管理 — SIM

**User Story:** AS 资产管理员，I want 管理 SIM 卡资产的完整生命周期，so that 可以追踪每张卡从入库到退网的全程状态。

#### Acceptance Criteria

1. THE 系统 SHALL 支持的 SIM 资产状态流转：待激活 → 已激活 → 使用中 → 暂停 → 已停用 → 已退网。
2. WHEN 用户访问 SIM 列表页，THE 系统 SHALL 支持以下筛选维度：状态、租户/子账户、套餐、卡片类型、激活时间范围、标签。
3. THE 系统 SHALL 为每个 SIM 资产维护完整的状态变更历史。
4. WHEN 用户进入 SIM 详情页，THE 系统 SHALL 展示资产信息、链接信息、套餐订阅历史、话单摘要、告警记录、操作日志六个信息区。
5. THE 系统 SHALL 支持批量操作：批量状态变更、批量套餐订阅、批量导出。
6. THE 系统 SHALL 支持 SIM 资产的自定义标签（Tag）管理，用于灵活分组和检索。

---

### REQ-5: 资产生命周期管理 — eSIM

**User Story:** AS 资产管理员，I want 管理 eSIM 设备及其 Profile，so that 可以掌握 eSIM 设备的多 Profile 配置。

#### Acceptance Criteria

1. THE 系统 SHALL 在 eSIM 详情页展示设备基本信息、Profile 列表、当前激活 Profile、设备网络信息。
2. WHEN eSIM 设备绑定或解绑 Profile，THE 系统 SHALL 更新 Profile 与 EID 的关联关系。
3. THE 系统 SHALL 支持的 eSIM 资产状态流转：待激活 → 已激活 → 使用中 → 已停用。
4. THE 系统 SHALL 为 eSIM 资产维护 Profile 变更历史。

---

### REQ-6: 资产生命周期管理 — Profile

**User Story:** AS 资产管理员，I want 管理 Profile 的独立生命周期，so that 可以追踪每个 Profile 的归属、网络和套餐状态。

#### Acceptance Criteria

1. THE 系统 SHALL 支持的 Profile 状态流转：已下载 → 已安装 → 已启用 → 已停用 → 已删除。
2. WHEN 查询 Profile，THE 系统 SHALL 展示 Profile 归属的 EID、当前套餐、网络状态。
3. THE 系统 SHALL 为 Profile 维护完整的状态变更和套餐变更历史。

---

### REQ-7: 设备群组管理

**User Story:** AS 运维人员，I want 将资产按业务逻辑分组管理，so that 可以对一组设备执行批量操作和统一策略。

#### Acceptance Criteria

1. THE 系统 SHALL 支持创建设备群组（Device Group），支持手动添加资产和按标签/状态动态匹配资产。
2. WHEN 配置动态匹配规则，THE 系统 SHALL 实时更新群组成员列表。
3. THE 系统 SHALL 支持对群组执行批量操作：批量套餐变更、批量状态变更、批量策略下发。
4. THE 系统 SHALL 在群组详情页展示群组内资产的聚合统计（在线数、总流量、总费用）。

---

### REQ-8: 套餐模板与订阅管理

**User Story:** AS 产品经理，I want 定义灵活的套餐模板，so that 可以快速创建不同计费规则的商用套餐。

#### Acceptance Criteria

1. THE 系统 SHALL 支持套餐模板的参数化配置，包含：基础价格、计费周期、容量、套外资费、多维度计费项（流量/短信/语音）。
2. THE 系统 SHALL 支持阶梯计费模板：不同用量区间对应不同单价。
3. THE 系统 SHALL 支持共享流量池套餐：多个资产共享同一流量池。
4. WHEN 创建套餐实例（绑定到资产），THE 系统 SHALL 记录当时的套餐参数快照，确保历史账单可追溯。
5. THE 系统 SHALL 在套餐变更时保留历史订阅记录。
6. THE 系统 SHALL 支持套餐的生效时间定时和立即生效两种模式。

---

### REQ-9: 话单采集与处理

**User Story:** AS 系统运维，I want 高效采集和处理海量话单数据，so that 计费和统计分析有准确的数据基础。

#### Acceptance Criteria

1. THE 系统 SHALL 支持双通道话单采集：定时批量同步（每 5 分钟至每天可配）和准实时查询（按需调用资源方 API）。
2. THE 系统 SHALL 将批量采集的话单持久化到话单存储，并支持按资产 ID 和日期分区。
3. THE 系统 SHALL 以 `roundedBytes` 作为流量计费基准值。
4. THE 系统 SHALL 对话单采集任务执行结果进行记录，失败时自动重试 3 次。
5. THE 系统 SHALL 支持话单补采功能，允许指定时间范围和资产范围重新采集历史话单。
6. WHEN 单次话单采集的数据量超过阈值（可配置），THE 系统 SHALL 自动分页拉取。

---

### REQ-10: 计费引擎

**User Story:** AS 财务人员，I want 系统自动完成多维计费计算，so that 可以实现灵活的定价策略和精准的账单生成。

#### Acceptance Criteria

1. THE 系统 SHALL 支持以下计费模式：固定费用、按用量阶梯计费、共享流量池计费、多维度组合计费（流量+短信+语音）。
2. THE 系统 SHALL 的计费引擎作为独立微服务运行，支持水平扩展。
3. WHEN 触发计费计算，THE 系统 SHALL 按以下顺序执行：获取套餐快照 → 获取周期内话单 → 按计费维度聚合 → 判断阶梯区间 → 计算费用 → 生成账单明细。
4. THE 系统 SHALL 支持月度、季度、年度和自定义计费周期。
5. THE 系统 SHALL 对计费计算过程做幂等保护：同一计费周期同一资产的账单仅生成一次。
6. THE 系统 SHALL 支持账单预览功能：在正式出账前生成预估账单供审核。
7. WHEN 账单审核不通过，THE 系统 SHALL 允许管理员调整账单金额并记录调整原因。
8. THE 系统 SHALL 支持多币种计费和汇率转换（通过可配置的汇率服务）。

---

### REQ-11: 实时监控与告警

**User Story:** AS 运维人员，I want 实时监控资产状态和用量，so that 可以在异常发生时第一时间响应。

#### Acceptance Criteria

1. THE 系统 SHALL 在仪表盘展示实时监控面板，包含：在线资产数、离线资产数、今日总流量、今日总费用、活跃告警数。
2. THE 系统 SHALL 支持配置告警规则，触发条件包括但不限于：资产离线超过 N 分钟、单日流量超过阈值、月度费用超过预算、资源方不可用。
3. WHEN 告警规则被触发，THE 系统 SHALL 通过以下渠道发送通知：站内消息、邮件、Webhook 推送。
4. THE 系统 SHALL 支持告警分级（严重/警告/提示）和告警静默时段配置。
5. THE 系统 SHALL 维护告警历史记录，支持按时间、资产、告警级别筛选。

---

### REQ-12: 数据分析与报表

**User Story:** AS 业务决策者，I want 查看多维度的数据分析报表，so that 可以基于数据做出运营决策。

#### Acceptance Criteria

1. THE 系统 SHALL 提供以下预设报表：资产概览报表、话单趋势报表、费用分析报表、资源方质量报表、租户活跃度报表。
2. THE 系统 SHALL 支持报表的自定义维度筛选和时间范围选择。
3. THE 系统 SHALL 支持报表导出为 Excel 和 PDF 格式。
4. THE 系统 SHALL 支持报表的定时生成和邮件投递（日报/周报/月报可配）。

---

### REQ-13: API 网关与开放接口

**User Story:** AS 企业客户开发者，I want 通过丰富的 API 集成平台能力，so that 可以将连接管理嵌入自有业务系统。

#### Acceptance Criteria

1. THE 系统 SHALL 对外提供 RESTful API，遵循 OpenAPI 3.0 规范。
2. THE 系统 SHALL 为每个租户提供独立的 API 凭证（Access Key + Secret Key）。
3. THE 系统 SHALL 对 API 请求进行租户级限流，默认 1000 次/分钟，支持按租户自定义。
4. THE 系统 SHALL 开放以下 API 能力域：资产管理、套餐管理、话单查询、账单查询、设备群组管理、告警查询、Webhook 订阅管理。
5. THE 系统 SHALL 提供 API 调试控制台（Swagger UI），租户可直接在 Portal 内测试 API。
6. THE 系统 SHALL 为每个 API 请求记录访问日志（调用方、接口、耗时、响应码）。

---

### REQ-14: Webhook 事件推送

**User Story:** AS 企业客户开发者，I want 订阅平台事件，so that 我的系统可以实时响应资产变更和告警。

#### Acceptance Criteria

1. THE 系统 SHALL 支持以下事件类型的 Webhook 推送：资产状态变更、套餐订阅变更、话单更新、告警触发、账单生成。
2. WHEN 租户注册 Webhook 端点，THE 系统 SHALL 发送验证请求确认端点可达。
3. THE 系统 SHALL 对 Webhook 推送失败进行重试（指数退避，最多 5 次），连续失败 10 次后自动停用该端点并通知租户。
4. THE 系统 SHALL 支持 Webhook 签名校验（HMAC-SHA256），确保推送来源可信。

---

### REQ-15: 审计与合规

**User Story:** AS 安全审计员，I want 追踪所有关键操作记录，so that 可以满足合规审计要求。

#### Acceptance Criteria

1. THE 系统 SHALL 在每条审计日志中记录：日志 ID、操作人、所属租户、操作对象类型、操作对象 ID、操作类型、请求摘要、响应结果、操作前值、操作后值、来源 IP、User-Agent、Trace ID、创建时间。
2. THE 系统 SHALL 对以下操作进行强制审计：登录/登出、密码修改、用户启停用、租户创建/停用、资源方配置修改、资产状态变更、套餐订阅/变更/退订、账单调整、权限变更、API 凭证重置。
3. THE 系统 SHALL 确保审计日志不可篡改，支持追加写入。
4. THE 系统 SHALL 支持审计日志的按时间范围导出，并保留至少 365 天。

---

### REQ-16: 通知中心

**User Story:** AS 平台用户，I want 通过多种渠道接收重要通知，so that 不会遗漏关键信息。

#### Acceptance Criteria

1. THE 系统 SHALL 支持以下通知渠道：站内消息、邮件、短信（可选）。
2. THE 系统 SHALL 为每个用户提供通知偏好设置，允许选择接收渠道和通知类型。
3. THE 系统 SHALL 在以下场景触发通知：告警触发、账单生成、套餐即将到期、资源方密码即将过期、Webhook 推送异常。

---

### REQ-17: 非功能需求 — 性能

**User Story:** AS 运维工程师，I want 系统支撑海量设备管理，so that 百万级资产规模下仍保持良好响应。

#### Acceptance Criteria

1. THE 系统 SHALL 在百万级资产规模下，资产列表首页加载时间不超过 2 秒。
2. THE 系统 SHALL 对话单采集任务支持水平扩展，通过分布式任务调度实现。
3. THE 系统 SHALL 对高频查询接口（资产列表、话单查询）实现缓存策略，缓存过期时间可配置。
4. THE 系统 SHALL 的计费引擎支持并行计算，单次计费周期可处理 100 万以上资产。

---

### REQ-18: 非功能需求 — 安全

**User Story:** AS 安全工程师，I want 系统具备企业级安全防护能力。

#### Acceptance Criteria

1. THE 系统 SHALL 对所有用户密码和 API 密钥进行不可逆加密存储。
2. THE 系统 SHALL 对传输中的数据使用 TLS 1.3 加密。
3. THE 系统 SHALL 对敏感数据（密钥、密码、手机号）在日志和 API 响应中进行脱敏处理。
4. THE 系统 SHALL 支持 IP 白名单限制 API 访问。
5. THE 系统 SHALL 在检测到暴力破解（同一账户 5 分钟内 10 次失败）后临时锁定账户 30 分钟。

---

### REQ-19: 非功能需求 — 可用性

**User Story:** AS 运维工程师，I want 系统具备高可用和容灾能力。

#### Acceptance Criteria

1. THE 系统 SHALL 的核心服务（API 网关、资产管理、计费引擎）支持多实例部署和自动故障转移。
2. THE 系统 SHALL 在数据库故障时自动切换到备用节点。
3. WHEN 资源方不可用，THE 系统 SHALL 在 Portal 和 API 中返回明确的服务降级提示。
4. THE 系统 SHALL 的定时任务具备断点续传能力。

---

### REQ-20: 非功能需求 — 可观测性

**User Story:** AS 运维工程师，I want 全链路可追踪和监控。

#### Acceptance Criteria

1. THE 系统 SHALL 在每次请求中生成并传递 Trace ID。
2. THE 系统 SHALL 将所有服务日志输出为结构化 JSON 格式，包含 Trace ID、服务名、时间戳、日志级别。
3. THE 系统 SHALL 暴露 Prometheus 指标端点，包含：API 请求量/QPS/延迟分布、数据库连接池状态、JVM 内存/GC 指标、计费任务执行状态。
4. THE 系统 SHALL 支持分布式链路追踪（与 Trace ID 打通），可视化服务间调用链路。

---

## Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| V3.0.0 | 2026-06-23 | AI Coding Agent | 全新架构设计，覆盖旧版全部功能并扩展：多租户、设备群组、计费引擎、实时监控告警、数据分析报表、Webhook、通知中心、可观测性 |
