# Requirements Document — Local_CMP 连接管理平台

## Introduction

Local_CMP 是一个面向企业客户的物联网连接管理平台。平台通过对接上游资源方（POD、BICS、CITIC），向下游企业提供统一的 SIM/eSIM 生命周期管理、套餐订阅、话单查询及计费服务。平台提供"可视化 Portal + 开放式 API"双通道交互模式。

本需求文档依据原始 `CMP需求说明(V1.0.2).docx` 第三章（账户与权限管理）重新设计账户、用户、权限模块，基于原始文档的账户层级体系、用户字段定义和权限矩阵，补齐层级链路、权限细化和 API 设计。

## Glossary

| 术语 | 定义 |
|------|------|
| **账户 (Account)** | 资产容器，所有资产严格归属特定账户，账户间逻辑隔离。分为 root、reseller、customer 三种类型。除 root 外，每个账户必须有且仅有一个归属上级账户，形成严格的树状层级结构 |
| **root 账户** | 平台超级管理员账户（GD 内部），编码前缀 `ACC_ROOT`，具备全局视图和全局管理能力。root 是账户树根节点，无上级账户 |
| **reseller 账户** | 独立结算的企业主体，编码前缀 `ACC_ENT_reseller`，可创建下级 reseller 和 customer 账户并划拨资产，链路上限不限 reseller 层级，但叶子必须是 customer |
| **customer 账户** | 终端企业客户，编码前缀 `ACC_ENT_customer`，不可创建任何下级账户（customer 始终是叶子节点） |
| **账户层级 (Account Level)** | 账户在树状结构中的层级位置，root=0，一级 reseller=1，二级 reseller/customer=2，以此类推，无硬性上限，但 customer 始终为叶子节点 |
| **用户 (User)** | 操作执行者，必须挂载于一个账户下。用户之间不存在上下级关系，不同账户下的用户彼此完全隔离，用户仅与归属账户相关。登录由用户凭据（用户名+密码）完成 |
| **权限 (Role)** | 赋予用户的操作授权，由一组权限组组成，命名规范为 `ADMIN_资产类型` 或 `read_资产类型`。权限是用户级别的，不同用户可以拥有不同权限组合 |
| **资源方 (Resource Provider)** | 提供底层网络资源与技术能力的供应商，如 POD、BICS、CITIC |
| **资产 (Asset)** | 系统中管理的 SIM 卡、eSIM 设备、Profile 的统称 |
| **套餐 (Plan/Product)** | 商用套餐，定义容量、价格、计费周期等 |
| **话单 (CDR)** | Call Detail Record，网络使用记录 |
| **一级企业客户** | 直接与平台签约的 reseller 或 customer 类型的一级账户 |
| **二级企业客户** | 由 reseller 通过 API 或 Portal 创建的下级账户 |
| **ICCID** | Integrated Circuit Card Identifier，SIM 卡唯一标识 |
| **EID** | eSIM 设备唯一标识 |
| **MSISDN** | 移动用户号码 |

---

## Requirements

### REQ-1: 账户管理

**User Story:** AS 平台管理员，I want 管理多层级企业账户体系，so that 平台支持 root、reseller、customer 三种账户类型，reseller 可创建下级 reseller 和 customer 账户，形成无硬性链长限制的树状层级，叶子节点必须为 customer。

#### REQ-1-1: 账户类型与层级

| 账户类型 | 编码 | 层级 | 说明 |
|----------|------|------|------|
| root | ACC_ROOT | 0 | 平台顶级管理机构（GD 内部），统筹全局资源，无上级账户 |
| reseller | ACC_ENT_reseller | 1 ~ N | 独立结算的企业主体，可创建下级 reseller 和 customer 账户并划拨资产 |
| customer | ACC_ENT_customer | 2 ~ N | 终端企业客户，不可创建任何下级账户（始终为叶子节点） |

1. THE 系统 SHALL 支持 root、reseller、customer 三种账户类型。
2. THE 系统 SHALL 确保除 root 外，每个账户有且仅有一个归属上级账户（parent_account_id 必填），形成严格的树状层级结构。
3. WHEN 账户类型为 customer，THE 系统 SHALL 禁止该账户创建任何下级账户（customer 始终为叶子节点）。
4. WHEN 创建下级账户，THE 系统 SHALL 校验上级账户类型为 reseller；若上级为 root，可创建一级 reseller 或 customer；若上级为 reseller，可创建下级 reseller 或 customer，无 hard limit on 链长，但 customer 必须为叶子。
5. THE 系统 SHALL 确保每个账户有全局唯一编码，按类型前缀生成：ACC_ROOT / ACC_ENT_{type}。

**账户层级示例：**

```
国 (root) ── 省 (reseller) ── 市 (reseller) ── 县 (reseller) ── 乡 (reseller) ── 村 (customer)
                    └── 直辖市 (reseller) ── 区 (reseller) ── 街道 (customer)
```

```
捷德 (root) ── A (reseller) ── B (reseller) ── C (reseller) ── D (customer)
```

6. WHEN 创建下级账户，上级账户可以是本条链路中任意 reseller 或 root 类型账户。例如：创建 C 账户时，上级账户可选 B（直接上级），也可由捷德（root）或 A（reseller）直接创建，但 parent_account_id 必须填写 B（即账户的真实直接上级）。

#### REQ-1-2: 账户生命周期

1. WHEN 管理员创建账户，THE 系统 SHALL 设置账户类型（reseller/customer）并生成全局唯一编码。
2. WHEN 停用 reseller 账户，THE 系统 SHALL 级联禁止该 reseller 及其所有下级账户的新增用户、分配资产、发起套餐订阅类操作。
3. WHEN 管理员请求删除账户，THE 系统 SHALL 校验账户满足以下全部条件后方可删除：无归属资产、无有效用户、无未完成任务、无未结算账单、无下级账户。
4. WHILE 账户状态为停用，THE 系统 SHALL 拒绝该账户下所有写操作请求。
5. THE 系统 SHALL 确保每个资产归属且仅归属一个账户，不允许资产跨账户直接共享。
6. WHEN 创建下级账户，THE 系统 SHALL 记录 parent_account_id，形成账户树状层级结构。
7. THE 系统 SHALL 在查询下级账户数据时支持按账户层级树向上递归汇总（仅限 reseller 查看名下所有下级客户的聚合数据）。

---

### REQ-2: 用户管理

**User Story:** AS 账户管理员，I want 管理账户下的用户，so that 不同操作人员可以按授权访问平台。

#### REQ-2-1: 用户字段定义（基于 V1.0.2 第三章 TABLE 2）

| 序号 | 字段 | 说明 | 是否必选 |
|------|------|------|----------|
| 1 | 用户名 | 用户登录标识，账户内唯一 | 是 |
| 2 | 密码 | 须加密存储，不可明文落库 | 是 |
| 3 | 邮箱 | 密码重置、告警通知、系统通知 | 是 |
| 4 | 电话号码 | 联系方式 | 否 |
| 5 | 状态 | 0-启用，1-停用 | 否（默认启用） |
| 6 | 归属账户 | 用户所属账户 ID | 是 |
| 7 | 权限 | 用户被授予的权限组 | 否 |

#### REQ-2-2: Acceptance Criteria

1. WHEN 创建用户，THE 系统 SHALL 将用户归属到指定账户，且一个用户仅允许归属一个账户。
2. THE 系统 SHALL 对用户密码进行不可逆加密存储，不允许明文落库。
3. WHEN 用户修改密码，THE 系统 SHALL 校验新密码与历史前三次密码不重复。
4. WHEN 用户状态由启用切换为停用，THE 系统 SHALL 使该用户所有现有 token 立即失效。
5. WHEN 用户处于停用状态，THE 系统 SHALL 拒绝该用户登录 Portal 以及所有 API 调用。
6. THE 系统 SHALL 使用用户邮箱执行密码重置、告警通知和系统通知。
7. THE 系统 SHALL 支持为用户分配多个权限组，权限组可灵活组合。
8. THE 系统 SHALL 确保用户之间不存在上下级关系：所有用户地位平等，仅通过所归属的账户和所拥有的权限组区分操作能力。
9. WHEN 用户 A 归属账户 X，用户 B 归属账户 Y（X != Y），THE 系统 SHALL 使 A 与 B 完全隔离：A 无法查看或操作 B 的信息，也无法查看或操作 Y 账户的资产，反之亦然。
10. WHEN 用户登录系统，THE 系统 SHALL 根据其归属账户和拥有的权限组确定可操作的数据范围：数据范围限定为该用户归属账户及其下级账户（若上级账户用户拥有管理权限）。

---

### REQ-3: 权限管理

**User Story:** AS 平台管理员，I want 为不同角色分配权限，so that 用户只能在授权范围内操作。权限是赋予用户的数据操作授权，不是账户级别的配置。同一个账户下的不同用户可以拥有不同的权限组组合（例如"国"这个 root 账户下可以有国务院（ADMIN_ALL）、财政部（ADMIN_Billing + read_SIM）、工信部（ADMIN_SIM + ADMIN_eSIM）等多个用户，各自权限不同）。

#### REQ-3-1: 权限组定义（基于 V1.0.2 第三章 TABLE 3，命名规范：`操作_资产类型`）

权限组分为全局权限、全权管理权限（ADMIN_前缀）和只读权限（read_前缀）三类。以下为系统预设权限组：

| 权限组编码 | 权限说明 | 适用资产范围 |
|-----------|----------|--------------|
| ADMIN_ALL | 所有权限，全局视图与全部管理能力 | ALL（仅限 root 账户使用） |
| read_ALL | 所有只读权限，全局数据只读访问 | ALL |
| read_only | 账户下所有信息的查询权限（数据级只读，不包含管理操作） | 账户、用户、SIM、eSIM、Profile、CDR、Bill |
| ADMIN_SIM | SIM 全权管理（查看、状态修改、名称修改、删除） | SIM |
| read_SIM | SIM 只读（列表查看和详情查看） | SIM |
| ADMIN_eSIM | eSIM 全权管理（查看、业务操作、删除） | eSIM |
| read_eSIM | eSIM 只读（列表查看和详情查看） | eSIM |
| ADMIN_PROFILE | Profile 全权管理（查看、状态修改、删除） | Profile |
| read_PROFILE | Profile 只读（列表查看和详情查看） | Profile |
| ADMIN_CDR | 话单全权管理（话单查看、话单补采） | CDR |
| read_CDR | 话单只读（话单数据查看） | CDR |
| ADMIN_Billing | 账单全权管理（账单查看、账单调整） | Bill |
| read_Billing | 账单只读（账单数据查看） | Bill |
| ADMIN_Plan | 套餐全权管理（套餐查看、订阅/暂停/恢复/退订/更换） | Plan |
| read_Plan | 套餐只读（可选套餐列表查看） | Plan |
| ADMIN_Account | 账户管理（创建/停用/配置下级企业账户） | Account（reseller 限于名下子账户） |
| ADMIN_User | 用户管理（账户下用户的创建/停用/密码重置） | User（限于本账户） |
| ADMIN_Role | 权限管理（账户下用户的权限分配） | Role（限于本账户） |
| ADMIN_Resource | 资源方管理（资源方配置的增删改查） | Resource Provider |

#### REQ-3-2: 页面与操作权限矩阵（基于 V1.0.2 第三章 TABLE 4）

| 操作 | read_only | ADMIN_SIM | read_SIM | ADMIN_eSIM | read_eSIM | ADMIN_PROFILE | read_PROFILE | ADMIN_CDR | read_CDR | ADMIN_Billing | read_Billing | ADMIN_ALL | read_ALL |
|------|-----------|-----------|----------|------------|-----------|---------------|--------------|-----------|----------|---------------|--------------|-----------|----------|
| 账户查看 | Y | N | N | N | N | N | N | N | N | N | N | Y | Y |
| 用户查看 | Y | N | N | N | N | N | N | N | N | N | N | Y | Y |
| 创建二级企业客户 | N | N | N | N | N | N | N | N | N | N | N | Y | N |
| SIM 列表/详情查看 | Y | Y | Y | N | N | N | N | N | N | N | N | Y | Y |
| SIM 状态修改 | N | Y | N | N | N | N | N | N | N | N | N | Y | N |
| SIM 名称修改 | N | Y | N | N | N | N | N | N | N | N | N | Y | N |
| SIM 删除 | N | Y | N | N | N | N | N | N | N | N | N | Y | N |
| eSIM 列表/详情查看 | Y | N | N | Y | Y | N | N | N | N | N | N | Y | Y |
| eSIM 业务操作 | N | N | N | Y | N | N | N | N | N | N | N | Y | N |
| eSIM 删除 | N | N | N | Y | N | N | N | N | N | N | N | Y | N |
| Profile 列表/详情查看 | Y | N | N | N | N | Y | Y | N | N | N | N | Y | Y |
| Profile 业务操作 | N | N | N | N | N | Y | N | N | N | N | N | Y | N |
| Profile 删除 | N | N | N | N | N | Y | N | N | N | N | N | Y | N |
| 话单查询 | Y | N | N | N | N | N | N | Y | Y | N | N | Y | Y |
| 话单补采 | N | N | N | N | N | N | N | Y | N | N | N | Y | N |
| 账单查询 | N | N | N | N | N | N | N | N | N | Y | Y | Y | Y |
| 账单调整 | N | N | N | N | N | N | N | N | N | Y | N | Y | N |
| 套餐查看 | N | N | N | N | N | N | N | N | N | N | N | Y | Y |
| 套餐订阅/操作 | N | N | N | N | N | N | N | N | N | N | N | Y(需ADMIN_Plan)* | N |
| 资源方管理 | N | N | N | N | N | N | N | N | N | N | N | Y | N |

> *注：ADMIN_Plan 为扩展权限组，支持常规非 root 账户管理本账户下套餐订阅操作。

#### REQ-3-3: Acceptance Criteria

1. THE 系统 SHALL 预设上述标准权限组，权限组命名遵循规范：`ADMIN_资产类型` 或 `read_资产类型`。
2. THE 系统 SHALL 支持将多个权限组组合分配给用户。
3. WHEN 用户执行任何操作，THE 系统 SHALL 校验用户拥有对应权限组。
4. WHEN 用户权限非 ADMIN_ALL 或 read_ALL，THE 系统 SHALL 将操作范围限制在用户所属账户内。
5. WHEN root 账户用户访问系统，THE 系统 SHALL 提供全局视图和全局管理能力。
6. WHEN 上级账户的用户操作下级账户数据，THE 系统 SHALL 允许其查询/管理名下所有下级账户（按账户层级树向下递归）的数据。
7. WHEN 下级账户的用户操作数据，THE 系统 SHALL 仅允许操作本账户数据，不可向上访问上级账户数据。
8. THE 系统 SHALL 确保同一个账户下的不同用户可以拥有不同的权限组组合，实现同一账户下不同角色（如财务、运维、管理）的权限隔离。
9. WHEN 用户仅拥有 read_only 权限，THE 系统 SHALL 允许其查看账户下所有信息但不允许任何管理操作。

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

### REQ-14: API — 对接资源方（基于 POD API v3.6）

**User Story:** AS CMP 开发者，I want 调用资源方 API 完成业务操作，so that 平台可以代理执行资产和 eSIM 的完整生命周期管理。

#### REQ-14-1: Token 获取与鉴权

| 接口 | 方法 | 路径 | 说明 |
|------|------|------|------|
| 登录获取 Token | POST | `/auth/token` | 使用 username 和 password 获取 token |
| 刷新 Token | POST | `/auth/token/refresh` | 使用 refreshToken cookie 刷新 session token |
| 密码重置 | POST | `/auth/recover-password` | 使用 username 或 email 发送密码重置邮件 |
| 修改密码 | POST | `/auth/change-password` | 修改用户密码 |

1. THE 系统 SHALL 调用 `/auth/token` 使用预置的 username 和 password 获取 token。
2. THE 系统 SHALL 将返回的 token 值作为后续接口 `Authorization` header 的值。
3. THE 系统 SHALL 将返回的 `permissions.accountId` 作为后续接口的 `accountId` 参数。
4. THE 系统 SHALL 在 token 有效期内（25 分钟）复用 token。
5. IF token 过期返回 401，THE 系统 SHALL 自动重新获取 token 后重试。
6. THE 系统 SHALL 支持通过 `/auth/token/refresh` 接口在 token 过期前主动刷新。

#### REQ-14-2: SIM 资产管理（POD /assets 模块，37 个端点）

**SIM 列表与查询（7 个端点）：**

| 接口 | 方法 | POD 路径 | 说明 |
|------|------|----------|------|
| 资产列表 | GET | `/assets` | 资产列表查询，支持按 iccid/imsi/msisdn/name/status/type 等 25+ 查询参数筛选 |
| 资产详情 | GET | `/assets/{iccid}` | 获取单个资产完整信息 |
| 资产诊断 | GET | `/assets/{iccid}/diagnostic` | 网络状态检查、最后连接、数据传输状态 |
| 位置查询 | GET | `/assets/{iccid}/location` | 获取资产当前位置 |
| 会话记录 | GET | `/assets/{iccid}/sessions` | 获取资产历史会话 |
| SM-DS 服务器 | GET | `/assets/{iccid}/sm-ds-servers` | 获取 SM-DS 服务器列表（Consumer eSIM Profile 专用） |
| SGP.22 事件日志 | GET | `/assets/{iccid}/esim-events` | 获取 SGP.22 事件日志（Consumer eSIM Profile 专用） |

**SIM 生命周期管理（6 个端点）：**

| 接口 | 方法 | POD 路径 | 说明 |
|------|------|----------|------|
| 创建资产 | POST | `/assets` | 创建新资产（须提供 accountId、iccid、carriers） |
| 更新分组名称 | PUT | `/assets/{iccid}/groupname` | 更新资产分组名称 |
| 设置 dpProfileType | PUT | `/assets/{iccid}/dpprofiletype` | 设置 dpProfileType 分类标记 |
| 删除外部资产 | DELETE | `/assets/{iccid}/external` | 删除外部 SGP.32 eSIM Profile |
| 转移资产 | POST | `/assets/{iccid}/transfer` | 转移资产到其他账户 |
| 归还资产 | POST | `/assets/{iccid}/return` | 归还资产到上级账户 |

**套餐操作（6 个端点）：**

| 接口 | 方法 | POD 路径 | 说明 |
|------|------|----------|------|
| 激活并订阅 | PUT | `/assets/{iccid}/subscribe` | 激活资产并订阅套餐（需 productId） |
| 退订 | PUT | `/assets/{iccid}/unsubscribe` | 退订套餐 |
| 终止 | PUT | `/assets/{iccid}/terminate` | 立即终止套餐（与 unsubscribe 不同，terminate 立即生效） |
| 更换套餐 | PUT | `/assets/{iccid}/resubscribe` | 移除旧订阅并创建新订阅 |
| 暂停 | PUT | `/assets/{iccid}/suspend` | 暂停资产网络服务 |
| 恢复 | PUT | `/assets/{iccid}/unsuspend` | 恢复已暂停资产 |

**SIM 运维管理（6 个端点）：**

| 接口 | 方法 | POD 路径 | 说明 |
|------|------|----------|------|
| 设置告警 | PUT | `/assets/{iccid}/alerts` | 设置用量告警（data alerts + sms alerts） |
| 网络刷新 | POST | `/assets/{iccid}/purge` | 强制更新位置，重置网络连接 |
| 发送短信 | POST | `/assets/{iccid}/sms` | 向 SIM 发送短信（最多 160 字符） |
| 设置用量限制 | POST | `/assets/{iccid}/limit` | 设置 data/datalimit 或 smslimit |
| 设置标签 | POST | `/assets/{iccid}/tags` | 设置带索引的自定义标签 |
| 重新分配 IP | POST | `/assets/{iccid}/reallocate-ip` | 重新分配固定 IP（支持单资产和批量） |

**SIM 高级操作（7 个端点）：**

| 接口 | 方法 | POD 路径 | 说明 |
|------|------|----------|------|
| MSISDN 交换 | POST | `/assets/{iccid}/swapMSISDN` | 将虚拟 MSISDN 迁至另一 SIM |
| 修改 SID | POST | `/assets/{iccid}/sid` | 修改服务标识 |
| 下载 IMSI | POST | `/assets/{iccid}/download-imsi` | 下载新 IMSI 到 SIM（切换运营商） |
| 禁用 IMSI | POST | `/assets/{iccid}/disable-imsi` | 禁用 SIM 上的辅助 IMSI |
| 删除 IMSI | POST | `/assets/{iccid}/delete-imsi` | 从 SIM 上删除 IMSI |
| 启用 STK 菜单 | POST | `/assets/{iccid}/enable-stk-menu` | 启用 Multi-IMSI STK 菜单 |
| 禁用 STK 菜单 | POST | `/assets/{iccid}/disable-stk-menu` | 禁用 Multi-IMSI STK 菜单 |

**通话与 Consumer eSIM Profile 操作（5 个端点）：**

| 接口 | 方法 | POD 路径 | 说明 |
|------|------|----------|------|
| 快捷拨号 | POST | `/assets/{iccid}/quick-dial` | 添加快捷拨号条目 |
| MT 语音呼叫 | POST | `/assets/{iccid}/dial` | 发起 MT 语音呼叫（P2P 唤醒） |
| Download & Confirm | POST | `/assets/{iccid}/download-confirm` | 启动 Consumer eSIM Profile 下载并确认 |
| 取消下载 | POST | `/assets/{iccid}/cancel-relaxed` | 取消待处理下载订单（宽松模式） |
| 同步重建 | POST | `/assets/{iccid}/rebuild` | 从 Provider 同步并重建 Consumer eSIM Profile |

#### REQ-14-3: eSIM 资产管理（POD /esims 模块，34 个端点）

**eSIM 列表与查询（4 个端点）：**

| 接口 | 方法 | POD 路径 | 说明 |
|------|------|----------|------|
| eSIM 列表 | GET | `/esims` | eSIM 列表查询，支持按 eid/name/groupName/accountId/profileIccid 等 30+ 查询参数筛选 |
| eSIM 详情 | GET | `/esims/{eid}` | 获取 eSIM 完整信息（含 Profile 列表、启用 Profile、标签） |
| 同步 eUICC 信息 | GET | `/esims/{eid}/euiccsync-euicc-info` | 从 eIM 同步 eUICC 元数据和 Profile 信息（SGP.32） |
| 事件列表 | GET | `/esims/{eid}/events` | eSIM 事件列表（SGP.32，支持 JSON/CSV 导出） |

**eSIM 生命周期管理（4 个端点）：**

| 接口 | 方法 | POD 路径 | 说明 |
|------|------|----------|------|
| 创建 eSIM | POST | `/esims` | 创建 eSIM（须提供 accountId、eid、profiles 数组） |
| 设置标签 | POST | `/esims/{eid}/tags` | 设置 eSIM 自定义标签 |
| 转移 eSIM | POST | `/esims/{eid}/transfer` | 转移 eSIM 到其他账户 |
| 归还 eSIM | POST | `/esims/{eid}/return` | 归还 eSIM 到上级账户 |

**启用 Profile 套餐操作（5 个端点）：**

| 接口 | 方法 | POD 路径 | 说明 |
|------|------|----------|------|
| 订阅套餐 | PUT | `/esims/{eid}/subscribe` | 激活启用 Profile 并订阅套餐 |
| 退订套餐 | PUT | `/esims/{eid}/unsubscribe` | 启用 Profile 退订 |
| 更换套餐 | PUT | `/esims/{eid}/resubscribe` | 启用 Profile 更换订阅套餐 |
| 暂停 | PUT | `/esims/{eid}/suspend` | 暂停启用 Profile |
| 恢复 | PUT | `/esims/{eid}/unsuspend` | 恢复暂停的启用 Profile |

**eSIM 运维管理（5 个端点）：**

| 接口 | 方法 | POD 路径 | 说明 |
|------|------|----------|------|
| 设置告警 | PUT | `/esims/{eid}/alerts` | 设置启用 Profile 用量告警 |
| 网络刷新 | POST | `/esims/{eid}/purge` | 启用 Profile 网络刷新 |
| 发送短信 | POST | `/esims/{eid}/sms` | 向启用 Profile 发送短信 |
| 设置限制 | POST | `/esims/{eid}/limit` | 设置启用 Profile 用量限制 |
| eSIM 审计 | POST | `/esims/{eid}/audit` | 审计 eSIM 及所有已下载 Profile |

**eSIM Profile 操作（SGP.02 M2M + SGP.22 Consumer，4 个端点）：**

| 接口 | 方法 | POD 路径 | 说明 |
|------|------|----------|------|
| 下载 Profile | POST | `/esims/{eid}/download-profile` | 下载并安装 Profile 到 eSIM（SGP.02 ES2+ / SGP.22 ES9+） |
| 启用 Profile | POST | `/esims/{eid}/enable-profile` | 启用已安装的 Profile（Disabled → Enabled） |
| 禁用 Profile | POST | `/esims/{eid}/disable-profile` | 禁用已安装的 Profile（Enabled → Disabled） |
| 删除 Profile | POST | `/esims/{eid}/delete-profile` | 删除 eSIM 上已安装的 Profile |

**SGP.32 IoT eSIM 专用（12 个端点）：**

| 接口 | 方法 | POD 路径 | 说明 |
|------|------|----------|------|
| 设置立即启用标志 | POST | `/esims/{eid}/immediate-enable-flag` | 设置 IoT eSIM 立即启用标志 |
| Fallback 管理 | POST | `/esims/{eid}/fallback-mgmt` | 管理 fallback profile 属性 |
| 添加 EIM | POST | `/esims/{eid}/ecoadd` | 注册 EIM 到 IoT eSIM |
| 更新 EIM | POST | `/esims/{eid}/ecoupdate` | 更新 IoT eSIM 的 EIM 配置 |
| 删除 EIM | POST | `/esims/{eid}/ecodelete` | 从 IoT eSIM 删除 EIM |
| 列出 EIM | POST | `/esims/{eid}/ecolist` | 列出已注册的所有 EIM |
| 更新 eUICC 元数据 | POST | `/esims/{eid}/euiccupdate-metadata` | 更新 eUICC 元数据 |
| 设置默认 DP 地址 | POST | `/esims/{eid}/euiccset-default-dp-address` | 设置 eUICC 默认 DP 地址 |
| 更新 eUICC 状态 | POST | `/esims/{eid}/euiccupdate-status` | 更新 eUICC 激活状态 |
| 注册 eUICC | POST | `/esims/{eid}/euiccregister` | 注册 eUICC 到 eIM |
| 注销 eUICC | POST | `/esims/{eid}/euiccunregister` | 从 eIM 注销 eUICC |
| 取消待处理操作 | POST | `/esims/{eid}/operationcancel` | 按 transactionId 取消待处理操作 |

#### REQ-14-4: 套餐列表

1. THE 系统 SHALL 调用资源方 `/products` 接口（GET 方法）获取套餐列表。
2. THE 系统 SHALL 取返回结果中的 `resellerProductId` 作为套餐 ID。

#### REQ-14-5: 话单接口

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
| V2.1.0 | 2026-07-13 | AI Coding Agent | 基于原始 V1.0.2 第三章重新设计账户/用户/权限模块：新增账户类型层级体系（root/reseller/customer）、reseller 三层级联链路、权限命名对齐原始 ADMIN_ 前缀、新增 read_only 角色、完善权限矩阵 |
| V2.2.0 | 2026-07-13 | AI Coding Agent | 基于 POD API.txt (API v3.6) 重新设计 REQ-14 对接资源方接口：补齐 assets 模块 37 个端点（含 SIM 全生命周期 + Multi-IMSI + Consumer eSIM Profile）和 esims 模块 34 个端点（含 eSIM 全生命周期 + Profile 操作 + SGP.32 IoT 专用） |
| V2.2.1 | 2026-07-13 | AI Coding Agent | 新增 Profile 导入至删除完整生命周期设计（design.md）：定义 Profile 资产模型（含 ac_code 字段）、状态机（onstock→downloading→disabled→enabled→删除）、各阶段 API 流程与前置条件 |
| V2.2.2 | 2026-07-13 | AI Coding Agent | 基于 POD API.txt AssetSimcard/eSIM schema 重建 Data Models：Asset 表对齐 POD 42 个字段（status/profileState/carriers/lastCall/lastSMS/securityServices）；新增 eSIM/AssetProfile/AssetSetup/AssetAlert/Subscription/SubscriptionBundle 表；Account 表补齐地址/税务/时区/语言等字段 |
| V2.2.3 | 2026-07-13 | AI Coding Agent | 更新 Profile 状态机：delete-profile 完成后根据 repeat_download 参数恢复至 onstock（可重复下载）或进入 terminated（终端不可复用）；Asset 模型新增 repeat_download 字段 |
| V2.2.4 | 2026-07-13 | AI Coding Agent | 简化状态机为线性 onstock→installed→disabled→enabled→disabled→onstock/terminated；installed 可直接 disable；disabled 可重新 enable；onstock 不绑定 EID 可被任意 eSIM 选中下载 |
| V2.3.0 | 2026-07-13 | AI Coding Agent | 重构账户/用户/资产关系逻辑：①除 root 外每个账户必须有归属上级，形成严格树状层级；②reseller 链长无硬性限制（不受三层约束），customer 必为叶子；③用户之间无上下级关系，不同账户用户完全隔离；④权限是用户级别的，同账户下不同用户可拥有不同权限组合；⑤上级账户用户可向下递归管理下级账户资产和用户；⑥登录依赖用户凭据，非账户 |
