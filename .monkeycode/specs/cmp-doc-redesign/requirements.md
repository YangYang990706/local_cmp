# Requirements Document — Local_CMP 连接管理平台

## Introduction

Local_CMP 是一个面向企业客户的物联网连接管理平台。平台通过对接上游资源方（POD、BICS、CITIC），向下游企业提供统一的 SIM/eSIM 生命周期管理、套餐订阅、话单查询及计费服务。平台提供"可视化 Portal + 开放式 API"双通道交互模式。

本需求文档依据原始 `CMP需求说明(V1.0.2).docx` 重新设计，采用 EARS 规范重写所有需求条目，补充缺失细节，统一术语和文档层次。

## Glossary

| 术语 | 定义 |
|------|------|
| **账户 (Account)** | 资产容器，所有资产严格归属特定账户，账户间逻辑隔离 |
| **用户 (User)** | 操作执行者，必须挂载于账户下 |
| **权限 (Role)** | 赋予用户的操作授权，由一组权限点组成 |
| **资源方 (Resource Provider)** | 提供底层网络资源与技术能力的供应商，如 POD、BICS、CITIC |
| **资产 (Asset)** | 系统中管理的 SIM 卡、eSIM 设备、Profile 的统称 |
| **套餐 (Plan/Product)** | 商用套餐，定义容量、价格、计费周期等 |
| **话单 (CDR)** | Call Detail Record，网络使用记录 |
| **一级企业客户** | 直接与平台签约的企业客户 |
| **二级企业客户** | 由一级企业客户通过 API 或 Portal 自建的子客户 |
| **root 账户** | 平台超级管理员账户，具备全局视图和全局管理能力 |
| **ICCID** | Integrated Circuit Card Identifier，SIM 卡唯一标识 |
| **EID** | eSIM 设备唯一标识 |
| **MSISDN** | 移动用户号码 |

---

## Requirements

### REQ-1: 账户管理

**User Story:** AS 平台管理员，I want 管理企业账户，so that 企业客户可以隔离管理各自的资产。

#### Acceptance Criteria

1. WHEN 管理员创建账户，THE 系统 SHALL 为账户生成全局唯一编码。
2. WHEN 管理员停用账户，THE 系统 SHALL 禁止该账户下新增用户、分配资产、发起套餐订阅类操作。
3. WHEN 管理员请求删除账户，THE 系统 SHALL 校验账户满足以下全部条件后方可删除：无归属资产、无有效用户、无未完成任务、无未结算账单。
4. WHILE 账户状态为停用，THE 系统 SHALL 拒绝该账户下所有写操作请求。
5. THE 系统 SHALL 确保每个资产归属且仅归属一个账户，不允许资产跨账户直接共享。

---

### REQ-2: 用户管理

**User Story:** AS 账户管理员，I want 管理账户下的用户，so that 不同操作人员可以按授权访问平台。

#### Acceptance Criteria

1. WHEN 创建用户，THE 系统 SHALL 将用户归属到指定账户，且一个用户仅允许归属一个账户。
2. THE 系统 SHALL 对用户密码进行不可逆加密存储，不允许明文落库。
3. WHEN 用户修改密码，THE 系统 SHALL 校验新密码与历史前三次密码不重复。
4. WHEN 用户状态由启用切换为停用，THE 系统 SHALL 使该用户所有现有 token 立即失效。
5. WHEN 用户处于停用状态，THE 系统 SHALL 拒绝该用户登录 Portal 以及所有 API 调用。
6. THE 系统 SHALL 将用户邮箱用于密码重置、告警通知和系统通知。

---

### REQ-3: 权限管理

**User Story:** AS 平台管理员，I want 为不同角色分配权限，so that 用户只能在授权范围内操作。

#### Acceptance Criteria

1. THE 系统 SHALL 预设标准权限组，权限组命名遵循规范：`动作_资产类型` 或 `管理_资产类型`。
2. THE 系统 SHALL 支持将多个权限组组合分配给用户。
3. WHEN 用户执行任何操作，THE 系统 SHALL 校验用户拥有对应权限。
4. WHEN 用户权限为 ADMIN_ALL 或 read_ALL 之外的角色，THE 系统 SHALL 将操作范围限制在用户所属账户内。
5. WHEN root 账户用户访问系统，THE 系统 SHALL 提供全局视图和全局管理能力。
6. WHEN 一级企业客户（reseller）通过 API 操作二级企业客户，THE 系统 SHALL 限制操作范围仅为其名下的二级客户数据。

#### 权限矩阵

权限组命名遵循 `动作_资产类型` 或 `管理_资产类型` 规范。以下为系统预设权限组定义：

| 权限组 | 命名 | 说明 |
|--------|------|------|
| ADMIN_ALL | 全局管理 | root 专属，平台全局视图与全部管理能力 |
| read_ALL | 全局只读 | 所有账户数据的只读访问 |
| read_SIM | SIM 只读 | 本账户下 SIM 列表和详情查看 |
| manage_SIM | SIM 管理 | SIM 生命周期管理（状态修改、套餐操作等） |
| delete_SIM | SIM 删除 | 本账户下 SIM 资产删除 |
| read_eSIM | eSIM 只读 | 本账户下 eSIM 列表和详情查看 |
| manage_eSIM | eSIM 管理 | eSIM 生命周期管理 |
| delete_eSIM | eSIM 删除 | 本账户下 eSIM 资产删除 |
| read_Profile | Profile 只读 | 本账户下 Profile 列表和详情查看 |
| manage_Profile | Profile 管理 | Profile 状态管理 |
| delete_Profile | Profile 删除 | 本账户下 Profile 删除 |
| read_Plan | 套餐只读 | 本账户下可订阅套餐列表查看 |
| subscribe_Plan | 套餐订阅 | 套餐订阅/暂停/恢复/退订/更换 |
| read_CDR | 话单只读 | 本账户下话单数据查看 |
| read_Bill | 账单只读 | 本账户下账单数据查看 |
| adjust_Bill | 账单调整 | 账单金额调整（管理员操作） |
| manage_Account | 账户管理 | 创建/停用/配置二级企业账户 |
| manage_User | 用户管理 | 本账户下用户创建/停用/密码重置 |
| manage_Role | 权限管理 | 本账户下用户权限分配 |
| manage_Resource | 资源方管理 | 资源方配置的增删改查 |

**页面与操作权限映射：**

| 页面/模块 | 操作 | 所需权限组 |
|-----------|------|-----------|
| 管理员首页 | 查看资产列表 | ADMIN_ALL |
| 管理员首页 | 删除资产 | ADMIN_ALL + delete_SIM / delete_eSIM / delete_Profile |
| 企业首页 | 查看仪表盘 | read_SIM / read_eSIM / read_Profile / read_CDR / read_Bill |
| SIM 列表 | 查看 | read_SIM |
| SIM 详情 | 查看详情 | read_SIM |
| SIM 详情 | 修改状态/名称 | manage_SIM |
| SIM 详情 | 获取话单 | read_CDR |
| SIM 详情 | 删除 | delete_SIM |
| eSIM 列表 | 查看 | read_eSIM |
| eSIM 详情 | 查看详情/操作 | manage_eSIM |
| Profile 列表 | 查看 | read_Profile |
| Profile 详情 | 查看详情/修改状态 | manage_Profile |
| 套餐列表 | 查看可选套餐 | read_Plan |
| 套餐列表 | 订阅/暂停/恢复/退订/更换 | subscribe_Plan |
| 话单（月/天/套餐） | 查看 | read_CDR |
| 实时话单 | 查看 | read_CDR |
| 账单 | 查看 | read_Bill |
| 账单 | 调整 | adjust_Bill |
| 用户管理 | CRUD | manage_User |
| 权限分配 | 分配 | manage_Role |
| 资源方管理 | CRUD | manage_Resource |
| 库存管理 | 查看报表 | ADMIN_ALL |

---

### REQ-4: 资源方管理

**User Story:** AS 平台管理员，I want 管理上游资源方配置，so that 平台可以对接多个网络供应商。

#### Acceptance Criteria

1. WHEN 管理员添加资源方，THE 系统 SHALL 记录资源方类型（流量型/资源池）、API 密钥、密码及有效期信息。
2. THE 系统 SHALL 对 api_password 与 key 进行加密存储。
3. WHEN 页面展示资源方配置，THE 系统 SHALL 对 api_password 与 key 进行脱敏显示。
4. THE 系统 SHALL 每天凌晨通过定时任务扫描所有状态为"启用"的资源方。
5. WHEN 当前日期超过 `last_pw_update_time + pw_validity_period - 3天`，THE 系统 SHALL 触发邮件通知拥有 ADMIN_ALL 权限的用户，并将该资源方密码状态标记为"即将过期"。
6. WHEN 资源方密码状态为"即将过期"，THE 系统 SHALL 在 Portal 顶部横幅展示提示信息："资源方XXX的API密码将在3天后过期，请及时更新"。
7. WHEN 资源方状态为"停用"，THE 系统 SHALL 拒绝所有面向该资源方的业务调用。
8. WHEN 资源方密码状态为"已过期"(pw_status=2)，THE 系统 SHALL 拒绝调用依赖该密码的资源方接口。
9. WHEN 修改资源方配置，THE 系统 SHALL 将变更记录写入审计日志。
10. WHEN 管理员执行资源方启停用操作，THE 系统 SHALL 弹出二次确认对话框。

---

### REQ-5: Portal — 管理员首页

**User Story:** AS 平台管理员（GD内部人员），I want 登录后直接看到资产列表，so that 可以快速管理和操作资产。

#### Acceptance Criteria

1. WHEN 管理员登录 Portal，THE 系统 SHALL 默认展示资产管理模块的资产列表。
2. THE 系统 SHALL 优先展示 SIM 资产，如无 SIM 资产则按 eSIM、Profile 顺序回退。
3. THE 系统 SHALL 在资产列表提供单条删除和批量删除功能。

---

### REQ-6: Portal — 企业用户首页

**User Story:** AS 企业用户，I want 登录后看到仪表盘和统计图表，so that 可以直观了解资产使用状况。

#### Acceptance Criteria

1. WHEN 企业用户登录 Portal，THE 系统 SHALL 展示仪表盘页面，包含各类可动态配置的统计图表。
2. THE 系统 SHALL 在连接管理模块下提供话单统计二级页面，支持按月、按天、按套餐维度展示。

---

### REQ-7: Portal — SIM 资产管理

**User Story:** AS 资产管理员，I want 查看和管理 SIM 资产，so that 可以掌握卡片状态并执行生命周期操作。

#### Acceptance Criteria

1. WHEN 用户访问 SIM 列表页，THE 系统 SHALL 使用可配置表头展示资产数据，字段至少包括：ICCID、项目名称、归属用户、MSISDN、状态、商用套餐名称、创建时间、激活时间、最后同步时间、绑定套餐时间、卡片型号、卡片类型。
2. THE 系统 SHALL 支持的 SIM 卡片类型包括：插拔卡、工规卡、车规卡。
3. WHEN 用户进入单资产详情页，THE 系统 SHALL 展示以下信息块：
   - 资产信息：归属企业客户、资产名称、项目名称、MSISDN、商用套餐名称
   - 链接信息：状态、资产创建时间、绑定套餐时间、最近激活时间、最后登网时间、登录网络名称
4. WHEN 用户点击"查看资产详情"按钮，THE 系统 SHALL 向资源方同步查询 card info 并更新本地数据。
5. THE 系统 SHALL 在单资产详情页提供以下操作按钮：修改资产状态、修改资产名称、获取话单。

---

### REQ-8: Portal — eSIM 资产管理

**User Story:** AS 资产管理员，I want 查看和管理 eSIM 资产及其 Profile，so that 可以管理 eSIM 设备的全部配置。

#### Acceptance Criteria

1. WHEN 用户访问 eSIM 列表页，THE 系统 SHALL 使用可配置表头展示资产数据，字段至少包括：EID、eSIM 名称、归属用户（一级企业客户）、Profile 数量、启用 Profile ID（ICCID）、状态、Profile 类型（bootprofile/virtual）、激活日期。
2. WHEN 用户进入 eSIM 单资产详情页，THE 系统 SHALL 展示以下信息块：
   - eSIM 基本信息
   - eSIM Profiles 列表
   - 启用 Profile 信息
   - 链接信息
   - 设备信息
   - 操作按钮

---

### REQ-9: Portal — Profile 资产管理

**User Story:** AS 资产管理员，I want 查看和管理 Profile 资产，so that 可以追踪每个 Profile 的归属和状态。

#### Acceptance Criteria

1. WHEN 用户访问 Profile 列表页，THE 系统 SHALL 使用可配置表头展示资产数据，字段至少包括：ICCID、资产名称、归属企业用户、项目名称、MSISDN、eSIM Profile 状态、归属网络、链接状态、所属 EID、套餐名称。
2. WHEN 用户进入 Profile 单资产详情页，THE 系统 SHALL 展示以下信息块：
   - 资产信息：归属企业客户、资产名称、项目名称、MSISDN、商用套餐名称
   - 链接信息：状态、资产创建时间、绑定套餐时间、最近激活时间、最后登网时间、登录网络名称
3. THE 系统 SHALL 在单资产详情页提供以下操作按钮：修改资产状态、修改资产名称、获取话单。

---

### REQ-10: 套餐管理

**User Story:** AS 企业用户，I want 查看可选套餐并为资产订阅套餐，so that 资产可以获得网络服务。

#### Acceptance Criteria

1. WHEN 企业用户通过 Portal 或 API 查询套餐列表，THE 系统 SHALL 返回每个套餐的名称和编码。
2. THE 系统 SHALL 使用的商用套餐编码取值于资源方侧套餐 ID。
3. THE 系统 SHALL 保持平台侧套餐名称与资源方侧套餐名称一致。
4. THE 系统 SHALL 在数据库中为套餐价格、套外价格、容量、更新周期和计费周期字段创建联合索引。
5. WHEN 企业用户为资产订阅套餐，THE 系统 SHALL 记录套餐首次绑定时间，用于后续计费周期计算。

---

### REQ-11: 话单管理

**User Story:** AS 企业用户，I want 查看资产的话单数据，so that 可以了解网络使用情况和费用明细。

#### Acceptance Criteria

1. THE 系统 SHALL 通过 API 实时采集资源方话单数据。
2. THE 系统 SHALL 将话单分两类处理：累计话单和实时话单。

#### REQ-11-1: 累计话单

1. THE 系统 SHALL 通过定时任务向资源方同步话单，以资产为索引按天入库。
2. WHEN 用户查询月累计话单，THE 系统 SHALL 按资产累加"累计话单表"中单天数据，整合后展示。
3. WHEN 用户查询天累计话单，THE 系统 SHALL 以资产 ID 为索引展示"累计话单表"中每个资产单天的话单列表。
4. WHEN 用户查询套餐累计话单，THE 系统 SHALL 按套餐 ID 聚合累计话单数据，整合后展示。
5. THE 系统 SHALL 在累计话单列表页支持按单个表头排序。

#### REQ-11-2: 实时话单

1. WHEN 用户按时间段查询实时话单，THE 系统 SHALL 调用资源方 API 获取详细话单，展示返回报文中的完整数据。
2. THE 系统 SHALL 对实时话单不做入库持久化。
3. THE 系统 SHALL 以资产为索引、按时间降序排列实时话单。

---

### REQ-12: 计费管理

**User Story:** AS 企业用户，I want 系统自动计算费用，so that 可以获得准确的账单。

#### Acceptance Criteria

1. THE 系统 SHALL 采用"套餐内费用 + 套餐外费用"模式计算总费用，计算公式为：`总费用 = 套餐价格 + (套外价格 x 超出容量)`。
2. THE 系统 SHALL 按计费周期进行累计容量计算：当累计使用量未超过套餐容量时只计取套餐费（周期结束时结算），超出部分按套外价格计费。
3. WHEN 当前时间在 `套餐首次绑定时间` 与 `计费周期范围` 之外，THE 系统 SHALL 触发下一个计费周期。
4. THE 系统 SHALL 在跨周期计费时按累计容量判断是否超出套餐，不重置累计值。

---

### REQ-13: API — 外部接口

**User Story:** AS 一级企业客户开发者，I want 通过 API 集成连接管理能力，so that 可以将平台功能嵌入自有业务系统。

#### Acceptance Criteria

1. THE 系统 SHALL 为每个一级企业客户提供独立的用户名和密码用于 API 鉴权。
2. THE 系统 SHALL 使用 token 作为 API 请求的认证 header。
3. THE 系统 SHALL 对一级企业客户开放以下 API 能力：自建二级企业客户、归属资产管理、归属资产业务操作、商用套餐列表查询、话单和账单查询。
4. THE 系统 SHALL 以 Swagger 模式提供 API 文档。
5. WHEN 客户通过统一接口发起请求，THE 系统 SHALL 根据请求中的参数区分归属资源方并执行相应操作。

---

### REQ-14: API — 对接资源方（以 POD 为例）

**User Story:** AS CMP 开发者，I want 调用资源方 API 完成业务操作，so that 平台可以代理执行资产管理。

#### Acceptance Criteria

#### REQ-14-1: Token 获取

1. THE 系统 SHALL 调用资源方 `/auth/token` 接口，使用预置的 username 和 password 获取 token。
2. THE 系统 SHALL 将返回的 token 值作为后续接口 `Authorization` header 的值。
3. THE 系统 SHALL 将返回的 `permissions.accountId` 作为后续接口的 `accountId` 参数。
4. THE 系统 SHALL 在 token 有效期内（25 分钟）复用 token。
5. IF token 过期返回 401，THE 系统 SHALL 自动重新获取 token 后重试。

#### REQ-14-2: 套餐列表

1. THE 系统 SHALL 调用资源方 `/products` 接口（GET 方法）获取套餐列表。
2. THE 系统 SHALL 取返回结果中的 `resellerProductId` 作为套餐 ID。

#### REQ-14-3: 套餐订阅（激活）

1. THE 系统 SHALL 调用资源方 `/assets/{iccid}/subscribe` 接口订阅套餐。
2. THE 系统 SHALL 在请求中传递以下参数：`subscriberAccountId`、`productId`、`carrier`、`poolId`。

#### REQ-14-4: 暂停使用

1. THE 系统 SHALL 调用资源方 `/assets/{iccid}/suspend` 接口暂停资产网络服务。

#### REQ-14-5: 恢复使用

1. THE 系统 SHALL 调用资源方 `/assets/{iccid}/unsuspend` 接口恢复已暂停的资产。

#### REQ-14-6: 停止订阅

1. THE 系统 SHALL 调用资源方 `/assets/{iccid}/unsubscribe` 接口退订套餐。

#### REQ-14-7: 更换套餐

1. THE 系统 SHALL 调用资源方 `/assets/{iccid}/resubscribe` 接口更换订阅套餐。
2. THE 系统 SHALL 在请求中传递 `subscriberAccountId`、`productId`、`startTime` 参数。

#### REQ-14-8: 获取话单

1. THE 系统 SHALL 调用资源方 `/cdr` 接口获取话单数据。
2. THE 系统 SHALL 使用返回结果中的 `roundedBytes` 字段作为流量累计值。

---

### REQ-15: 卡产品管理

**User Story:** AS 平台管理员，I want 管理卡产品类型，so that 平台可以支持不同类型的 eSIM 规格。

#### Acceptance Criteria

1. THE 系统 SHALL 支持 SGP.22 和 SGP.32 两种卡产品类型。
2. THE 系统 SHALL 为卡产品类型提供基础 CRUD 管理能力。

---

### REQ-16: 库存管理

**User Story:** AS 平台管理员，I want 查看资产库存报表，so that 可以掌握采购、分配和剩余情况。

#### Acceptance Criteria

1. THE 系统 SHALL 提供库存报表功能，按资产类型（SIM、eSIM、Profile）分类统计。
2. THE 系统 SHALL 在库存报表中展示以下维度的统计：采购总量、已分配给各企业客户的数量、剩余可用数量。

---

### REQ-17: 审计日志

**User Story:** AS 安全审计员，I want 追踪所有关键操作，so that 可以进行合规审计和问题追溯。

#### Acceptance Criteria

1. THE 系统 SHALL 在每条审计日志中记录以下字段：日志 ID、操作人、操作账户、操作对象类型、操作对象 ID、操作类型、请求摘要、响应结果、操作前值、操作后值、来源 IP、traceId、创建时间。
2. THE 系统 SHALL 对以下操作进行强制审计：登录/登出、修改密码、用户启停用、账户创建/停用、资源方配置修改、资产状态修改、套餐订阅/暂停/恢复/退订/更换套餐、话单补采、账单调整。
3. THE 系统 SHALL 同时记录业务日志和码号资源日志。

---

### REQ-18: 非功能需求 — 安全

**User Story:** AS 安全工程师，I want 系统具备基础安全防护能力，so that 数据和操作安全得到保障。

#### Acceptance Criteria

1. THE 系统 SHALL 对用户密码进行不可逆加密存储。
2. THE 系统 SHALL 对资源方密钥和密码进行加密存储。
3. THE 系统 SHALL 对 API token 进行过期校验。
4. THE 系统 SHALL 对页面和 API 响应中的敏感字段进行脱敏展示。
5. WHEN 执行任何写操作，THE 系统 SHALL 校验当前用户的操作权限。

---

### REQ-19: 非功能需求 — 性能

**User Story:** AS 运维工程师，I want 系统满足基本性能指标，so that 用户体验不受性能问题影响。

#### Acceptance Criteria

1. THE 系统 SHALL 对高频查询（如资产列表、话单查询）的数据库表建立合理索引。
2. THE 系统 SHALL 对实时话单查询进行分页限制，避免一次性返回全量数据。

---

### REQ-20: 非功能需求 — 可用性

**User Story:** AS 运维工程师，I want 系统具备容错和告警能力，so that 异常情况能被及时发现和处理。

#### Acceptance Criteria

1. WHEN 定时任务执行失败，THE 系统 SHALL 支持自动重试机制。
2. WHEN 关键任务（话单采集、计费计算）执行失败，THE 系统 SHALL 触发告警通知。
3. WHEN 资源方不可用，THE 系统 SHALL 在 Portal 和 API 中返回明确的错误信息。

---

### REQ-21: 非功能需求 — 可维护性

**User Story:** AS 运维工程师，I want 系统具备可追踪和可配置能力，so that 问题排查和日常运维便捷。

#### Acceptance Criteria

1. THE 系统 SHALL 在全链路日志中包含 traceId 字段。
2. THE 系统 SHALL 确保外部接口调用日志与内部业务日志可通过 traceId 关联。
3. THE 系统 SHALL 支持关键字典项（如资产状态、资源方类型等）可配置。

---

## Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| V2.0.0 | 2026-06-23 | AI Coding Agent | 基于 V1.0.2 原始文档完整重设计，采用 EARS 规范，新增 REQ-18~21 非功能需求详情 |
| V2.0.1 | 2026-06-23 | AI Coding Agent | 补全权限矩阵定义（权限组列表 + 页面操作映射） |
