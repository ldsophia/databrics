# Informatica MDM Golden Record 回写 Salesforce：CDI REST API 实施指南

> 版本：v2.0｜2026-10-26｜语言：中文  
> 范围：**Informatica IDMC MDM 已产出 Golden Record → 普通 IDMC CDI Mapping → Salesforce 原生 REST API → Account/Contact**。  
> 注意：示例字段、ID、域名、Swagger 都是可调整的**实现样例**，不是实际 Salesforce 租户的配置；在 Sandbox 验证后再上线。

## 1. 业务目标和设计决策

目标：将 MDM 里已完成匹配、合并、存活规则（survivorship）的 **Golden Record** 作为可信主数据，同步更新 Salesforce 客户记录，而不是从 Salesforce 再拉取评分。

```text
Informatica MDM (Golden Records)
    │  已发布/批准的客户与变更时间戳
    ▼
CDI Source（MDM 允许的导出接口、数据库视图、文件或 API）
    ▼
EXP_PREPARE_SF：字段标准化、清洗、空值策略、External ID
    ▼
FILTER / ROUTER：只处理待同步且有效的 Golden Records
    ▼
REST V2 Business Service + Web Services Transformation
    │ Salesforce sObject REST API PATCH / Account / MDM_Golden_ID__c / {id}
    ▼
Audit & Error Target（成功、失败、重试、Salesforce Id、同步水位）
```

本指南优先说明**明确要求使用 Salesforce 提供的 REST API** 的实现。技术选型如下：

| 方案 | 适用情况 | 特点 |
|---|---|---|
| **A：REST V2 Target** | 只需将 Golden Record 写入 Salesforce，不依赖 API 返回做复杂后续运算 | Mapping 更直观；**须做 POC 核实**动态 URL/path 和错误记录输出是否满足需求 |
| **B：REST V2 Business Service + Web Services Transformation（Midstream）** | 需要解析 REST 响应、错误输出、继续分流审计 | **本文主要展开**；PATCH、动态 External ID 路径、204 无响应体等在目标 IDMC 版本必须试验验证 |
| **C：Salesforce Connector** | 未被要求必须通过原生 REST V2 接口；偏向大量标准对象读写 | Salesforce 专用连接器通常更容易维护，需核实自身 License/功能；不等同于本文 REST V2 |
| **D：批量 API（sObject Collections / Bulk API）** | 数万/数十万 Golden Records | 适合批量，200 条以内的 Collections Upsert 是本文补充方案；将数组组装到 CDI REST V2 需单独 POC |

**最重要的业务决定：**
1. **Update only**：Salesforce 对应记录必须已经存在；不能意外创建新的 Account。使用 Salesforce `?updateOnly=true`。适合严格回写场景。
2. **Upsert**：没找到就新建，找到就更新。适合 MDM 管控 Salesforce 客户全生命周期。本文以 Upsert 演示，同时给出 updateOnly 模式。
3. 匹配键建议 **MDM Golden Record ID（稳定、不可复用）**，**不要**用客户名称或 Email 做唯一识别。

## 2. Salesforce 目标数据模型与前置准备

本例将 MDM 黄金客户写入 Salesforce 标准对象 **Account**；Contact 可沿用同样模式但字段名及父子关系需单独设计。

Salesforce 管理员在 **Setup → Object Manager → Account → Fields & Relationships** 中准备：

| Salesforce API 字段 | 类型/属性 | 示例 | 用途 |
|---|---|---|---|
| `MDM_Golden_ID__c` | Text，**External ID = true；Unique = true** | `GR-0001001` | 单条 Upsert 的稳定匹配键 |
| `Name` | Text，标准字段 | `Acme Global Ltd` | Golden Name |
| `Phone` | Phone | `+1 415 555 0101` | 标准化电话 |
| `Website` | URL | `https://acme.example` | 网站 |
| `BillingStreet` | Textarea/地址分量 | `100 Main Street` | 开票地址 |
| `BillingCity` | Text | `San Francisco` | 城市 |
| `BillingState` | Text | `CA` | 州/省（注意 State & Country/Territory Picklists） |
| `BillingPostalCode` | Text | `94105` | 邮编 |
| `BillingCountry` | Text | `United States` | 国家（picklist 开启时需按 Salesforce 配置转换） |
| `MDM_Last_Sync__c` | DateTime，自定义可选 | ISO 8601 UTC | 最后一次成功回写时间（仅成功后写入/或集中审计） |
| `MDM_Source_Version__c` | Text，自定义可选 | `2026-10-26T10:00:00Z` | 源变更版本/防循环参考 |

**重要**：Upsert URI 的 External ID value 在 URL path 中；请求 JSON Body **通常不应重复提供** `MDM_Golden_ID__c`，以避免键冲突或覆盖。Upsert 若没有匹配，会尝试创建 Account：确保 `Name` 等 Salesforce 必填字段已提供。Unique External ID 可防重复；若 External ID 非 Unique，Salesforce 可能要求额外权限且有歧义风险。

生产配置须同时检查：对象 CRUD 权限、字段级安全（FLS）、API Enabled、验证规则、必填字段、Flows/Triggers、重复规则、Record Type、国家/州 Picklist，以及 Salesforce Integration User 的最小授权。

## 3. MDM Source 与 Golden Record 字段映射

CDI 从**已授权且受支持**的 MDM Golden Record 导出方式读取数据（具体取决于 MDM 产品版本及部署架构，如 MDM SaaS API/发布输出、主数据集成目标、可访问的受支持读取接口）。不要未经许可直接查询 MDM SaaS 内部数据库。以下示例用逻辑 Source `MDM_GOLDEN_CUSTOMER_EXPORT` 代指已有的可读 Golden Record 数据源。

### 3.1 模拟 Source Row

| GOLDEN_ID | GOLDEN_NAME | PHONE_E164 | WEBSITE | ADDRESS_LINE1 | CITY | STATE | POSTAL_CODE | COUNTRY | CHANGE_TS | GOLDEN_STATUS |
|---|---|---|---|---|---|---|---|---|---|---|
| GR-0001001 | Acme Global Ltd | +14155550101 | https://acme.example | 100 Main Street | San Francisco | CA | 94105 | United States | 2026-10-26T10:00:00Z | APPROVED |

### 3.2 Mapping 字段设计

| MDM Source | Salesforce REST JSON / Path | 规则 |
|---|---|---|
| `GOLDEN_ID` | URL path `{goldenId}` 对应 `MDM_Golden_ID__c` | 必填；URI 编码；稳定且唯一 |
| `GOLDEN_NAME` | `Name` | 非空、长度校验 |
| `PHONE_E164` | `Phone` | 字符串、标准格式 |
| `WEBSITE` | `Website` | 有效 URL；空值策略 |
| `ADDRESS_LINE1` | `BillingStreet` | 标准化 |
| `CITY` | `BillingCity` | 标准化 |
| `STATE` | `BillingState` | 受 State/Country 配置影响 |
| `POSTAL_CODE` | `BillingPostalCode` | 用字符串，不可去前导零 |
| `COUNTRY` | `BillingCountry` | 与 Salesforce Picklist 对齐 |
| `CHANGE_TS` | 审计表及可选 `MDM_Source_Version__c` | 支持增量、去重、循环抑制 |
| `GOLDEN_STATUS` | 过滤条件 | 仅 APPROVED/ACTIVE 等治理认可记录 |

只在 Golden Record 自身发生有效变化时回写；MDM merge/unmerge 后 Golden ID 生存规则、源系统引用、Salesforce Account 去重/合并处理需要先定义。

**空值策略必须业务确认**：字段缺失/不明确时**省略 JSON property** 通常意味着不更新；发送 JSON `null` 往往意味着尝试清空 Salesforce 字段。不要不加区分把所有 Null 全写入；不同记录动态省略字段对 REST V2 Swagger 映射是单独的实现测试点。

## 4. Salesforce OAuth 2.0 与访问权限

### 4.1 Salesforce 侧（推荐 External Client App）

Salesforce 官方说明自 Spring '26 起新建 Connected Apps 受到限制，新集成建议使用 **External Client App (ECA)**；既有 Connected Apps 仍可能继续使用。

1. 准备专用 **Salesforce Integration User**，仅授予 Account 更新/创建所需对象和字段权限，不要使用个人账号或赋予无关高权限。
2. 在 Salesforce Setup 的 **External Client App Manager** 创建 External Client App，启用 OAuth，选择满足需要的 API scope（如 `api`）。
3. 为 headless server-to-server 集成优先评估 **OAuth 2.0 Client Credentials**，配置 **Run As** integration user；具体 ECA 策略视 Salesforce 租户而定。
4. 安全保存 Client ID/Client Secret 到合规 Secret Store，区分 Sandbox 与 Production。
5. 通过 OAuth Token Endpoint 申请 Access Token，REST 请求带 `Authorization: Bearer <token>`。
6. 访问 API 时使用登录响应或已确认的 Salesforce Org Instance/My Domain URL，不要把 OAuth 登录 Host 当作 REST 数据实例 Host。

例：

```bash
curl -X POST 'https://YOUR_DOMAIN.my.salesforce.com/services/oauth2/token' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data-urlencode 'grant_type=client_credentials' \
  --data-urlencode 'client_id=YOUR_CLIENT_ID' \
  --data-urlencode 'client_secret=YOUR_CLIENT_SECRET'
```

此 curl 是 Salesforce 标准 OAuth 用法的概念性测试命令，需依据 org Domain 和 ECA policy 验证。

### 4.2 IDMC REST V2 Connection

`Administrator → Connections → New Connection → REST V2`：

| 配置项 | 推荐值/说明 |
|---|---|
| Name | `CONN_RESTV2_SF_ACCOUNT` |
| Runtime Environment | 可以访问 Salesforce 的 Secure Agent |
| Swagger / OpenAPI specification | 下文 Swagger 2.0 JSON，Secure Agent 可读的路径或支持的 URL |
| Authentication | REST V2 在**你的具体版本**支持并能完成 Salesforce OAuth 的认证配置，或采用安全的 token provisioning/connection 策略 |
| API Base URL | Salesforce My Domain / instance，HTTPS |
| Test | 从 Secure Agent 实测网络、TLS、认证、权限 |

**兼容性警示：** Salesforce 的 OAuth Client Credentials 是否能直接在当前 REST V2 Connection UI 中原生自动换取/刷新 token，**不能只凭 Salesforce 端支持就推断 CDI 端支持**。创建前通过 IDMC REST V2 OAuth 配置文档及租户 UI POC 校验。若不满足，应选择受支持的连接器/安全 token 管理方案，而不是将 token 硬编码到每一行 Mapping。

## 5. Salesforce REST API：单条 External ID Upsert（推荐先 POC）

### 5.1 Endpoint

```http
PATCH https://YOUR_DOMAIN.my.salesforce.com/services/data/vXX.X/sobjects/Account/MDM_Golden_ID__c/GR-0001001
Authorization: Bearer <ACCESS_TOKEN>
Content-Type: application/json
Accept: application/json
```

其中 `vXX.X` 为 Salesforce org 已启用和验证的 API 版本；`GR-0001001` 为当前 Source Row 的 `GOLDEN_ID`，需动态传入。

**只允许更新已有账户：**

```http
PATCH /services/data/vXX.X/sobjects/Account/MDM_Golden_ID__c/GR-0001001?updateOnly=true
```

Salesforce 官方明确说明 `updateOnly=true` 防止找不到 External ID 时自动创建记录。

### 5.2 响应特征——不能假定全部是 200 + JSON

- 新建：通常 `HTTP 201 Created`，有 JSON body，例如 `id / success / errors / created`。
- 更新：在 API 46.0+ 的官方示例中可能返回 `HTTP 200 OK` 和 JSON（较早版本可能 `204 No Content`）。需按**当前版本实际返回**设计 Schema 和 Fault Handling。
- 失败：HTTP 4xx/5xx，典型错误体是 JSON **数组** `[ {"message": ..., "errorCode": ..., "fields": [...] } ]`。
- 不应在 Mapping 中固定要求每次响应都包含相同 JSON 节点；REST V2 响应解析行为以 POC 为准。

来源：Salesforce `Upsert Records Using sObject Rows by External ID` 和 `Insert or Update (Upsert) a Record Using an External ID`。

## 6. CDI Mapping：REST V2 Midstream 详细步骤

目标 Mapping：`M_MDM_GOLDEN_TO_SF_ACCOUNT_REST`

```text
SRC_MDM_GOLDEN
   -> EXP_MAP_SF_ACCOUNT
   -> FIL_VALID_GOLDEN
   -> WS_SF_ACCOUNT_UPSERT   [REST V2 Business Service]
   -> EXP_NORMALIZE_RESPONSE
   -> RTR_SUCCESS_FAILURE
       ├─ TGT_MDM_SF_SYNC_AUDIT
       └─ TGT_MDM_SF_SYNC_ERROR
```

### 6.1 准备 Swagger 文件

- 使用本文**最后**提供的完整 Swagger 2.0 JSON 示例，保存为 `salesforce-account-upsert-swagger.json`。
- Swagger `host` 应换成你的 Salesforce instance，`basePath` 换成实际版本 `/services/data/vXX.X`。
- path 参数 `{goldenId}` 负责不同 Golden Record 的 URL；`operationId` 设置为 `upsertAccountByGoldenId`。
- **提醒**：Swagger 2.0 描述 `PATCH` 和 URL Path Parameter 没有问题，但 REST V2 Business Service 能否在你的 CDI 租户将该 Path Parameter 从上游每行字段映射出来，必须实测验证；如果 UI 不暴露参数，改用 REST V2 Target、Salesforce Connector 或受支持的 API 集成方式，不要声称已经实现动态调用。
- response 结构应以 Salesforce org 实际 API 版本的 200/201/204/Error 响应为准；示例用 200/201 成功和 default error 定义。

### 6.2 建 REST V2 Connection 和 Business Service

1. `Administrator → Connections → New Connection`，Type `REST V2`，创建 `CONN_RESTV2_SF_ACCOUNT`。
2. 指定 Swagger JSON（Secure Agent 可访问位置）、认证和 Runtime Environment，并测试。
3. `Data Integration → New → Components → Business Service`。
4. Name `BS_SF_ACCOUNT_UPSERT`，选 REST V2 Connection，再选 `upsertAccountByGoldenId` Operation。
5. 保存后检查 Request/Response 树结构是否包括 `goldenId`、Body 中 Account 字段和正常返回/错误输出。

### 6.3 Source 和 Expression

1. Source 绑定可访问的 MDM Golden Record 发布数据，筛选已批准且待同步记录。
2. `EXP_MAP_SF_ACCOUNT` 将 MDM Field 映射到 Salesforce Account 字段。校验 `GOLDEN_ID`、`Name`、地址结构、长度等。
3. 对每条记录预先求得可审计 `SYNC_RUN_ID`、`MDM_CHANGE_TS` 和源数据 Hash/Version。
4. 如果 MDM 导出中某一 Golden ID 存在多条版本记录，**先按 Change Timestamp 去重**，一次同步只发送最新有效版本。

### 6.4 Web Services Transformation

1. Mapping Designer 中添加 `Web Services`，命名 `WS_SF_ACCOUNT_UPSERT`。
2. 在 **Web Service** 区域选择 `BS_SF_ACCOUNT_UPSERT`。
3. 在 **Request Mapping** 将：

| Incoming Field | REST V2 请求节点 |
|---|---|
| `GOLDEN_ID` | path `goldenId` |
| `SF_ACCOUNT_NAME` | body `Name` |
| `SF_PHONE` | body `Phone` |
| `SF_WEBSITE` | body `Website` |
| `SF_BILLING_STREET` | body `BillingStreet` |
| `SF_BILLING_CITY` | body `BillingCity` |
| `SF_BILLING_STATE` | body `BillingState` |
| `SF_BILLING_POSTAL_CODE` | body `BillingPostalCode` |
| `SF_BILLING_COUNTRY` | body `BillingCountry` |

4. **Response Mapping**：如果当前 REST V2 版本将 `id, success, created, errors[]` 输出到成功 Group，将这些字段映射到审计输出；Fault Group 分开处理。**不要凭空假定 status code 已作为可映射字段暴露**，请检查真正的运行日志、Group 和错误端口。
5. 如果 `PATCH` 的某类成功响应为空（204）或 Salesforce 错误响应是数组，调整 Swagger / Mapping 后用真实响应验证；对无法直接映射 HTTP 状态的情况，通过 Mapping Task/日志/受支持的异常通道确保不会把失败当成功。
6. `EXP_NORMALIZE_RESPONSE` / Router 整理成功和错误，并将结果落入持久 Audit/Error Targets。若 Web Services Transformation 为嵌套 Group 输出，需要正确建立关联键，防止输入 Golden ID 与 API 返回脱钩。

### 6.5 Mapping Task 与运行参数

- Mapping Task：`MT_MDM_GOLDEN_TO_SF_ACCOUNT_REST`。
- 推荐参数化 API Host、源水位、batch run id、运行环境；不要在 Mapping 中硬编码 Client Secret/Token。
- Sandbox 首先做 1 行、10 行、几百行验证，然后才扩大数量。
- 核查 Session Log、Source/Target Row Count、Salesforce Account 实际字段、审计表成功失败条数及 API 消耗。
- 根据 Salesforce 限流和 Secure Agent 承载能力调节并发，不要默认“一条记录 = 一次 API 调用”在百万级适用。

## 7. 批量方案：Salesforce sObject Collections Upsert

Salesforce 官方支持 `PATCH /services/data/vXX.X/composite/sobjects/Account/MDM_Golden_ID__c` **单请求最多 200 条同对象记录**；REST API v46.0+，Body 的 `records` 是数组，每条必须包括 `attributes.type`，不能设置 `Id`；响应 UpsertResult 数组顺序与提交顺序一致。

适用于 Golden Records 数量大、需要减少 HTTP 请求数的情况。但 CDI REST V2 Mapping 中如何**将 200 个 Golden Rows 聚合到一个复杂 JSON Array**，以及如何逐条拆解响应，需具体设计并 POC，不应认为 Web Services Transformation 自动帮你批处理。

示例：

```http
PATCH https://YOUR_DOMAIN.my.salesforce.com/services/data/vXX.X/composite/sobjects/Account/MDM_Golden_ID__c
Authorization: Bearer <ACCESS_TOKEN>
Content-Type: application/json
```

```json
{
  "allOrNone": false,
  "records": [
    {
      "attributes": {"type": "Account"},
      "MDM_Golden_ID__c": "GR-0001001",
      "Name": "Acme Global Ltd",
      "Phone": "+14155550101"
    },
    {
      "attributes": {"type": "Account"},
      "MDM_Golden_ID__c": "GR-0001002",
      "Name": "Beta Manufacturing Inc",
      "Phone": "+14155550102"
    }
  ]
}
```

成功的 HTTP 200 **不代表每条记录成功**，必须解析每条记录的 `success`、`errors` 并回写审计。超大规模数据应额外评估 Salesforce Bulk API 2.0、专用 Salesforce Connector 和集成平台能力。

## 8. 增量同步、审计与重试

### 8.1 水位/增量

- 仅同步 Golden Record 自上次成功水位后发生的**有效变化**，最好以 MDM 发布事件或源变更流水为依据。
- 水位推进应基于**持久化已处理状态**，不能在 API 局部失败时将所有源记录标为同步完成。
- 使用 Golden ID + Source Version/Change TS 作为同步唯一工作项。重跑相同版本 Upsert 应是幂等的。

### 8.2 审计表建议

`MDM_SF_SYNC_AUDIT (RUN_ID, GOLDEN_ID, MDM_VERSION, SF_OBJECT, SF_RECORD_ID, ACTION, ATTEMPT, RESULT, HTTP_STATUS_IF_AVAILABLE, ERROR_CODE, ERROR_MESSAGE, REQUEST_HASH, PROCESSED_AT)`。

不要把 Access Token、Client Secret、完整 PII Request Body 记录到无权限控制的 Session Log。敏感字段脱敏，审计保留策略遵循组织合规要求。

### 8.3 错误分类

| 类型 | 处理策略 |
|---|---|
| 401 | 检查 token 生命周期/权限，受控重新认证，不无限重试 |
| 403 | 权限/FLS/用户政策，人工排查 |
| 404 + `updateOnly=true` | Salesforce 未有匹配记录；业务决定补建、补关联或人工核对 |
| 400 校验错误 / `INVALID_FIELD_FOR_INSERT_UPDATE` | 数据或权限/字段错误，修复数据后重试 |
| 300 多匹配 | External ID 唯一性问题，停止并修复 |
| 429 或流量限制错误 | 根据实际 Salesforce 返回与限流策略指数退避、抖动和限速 |
| 5xx / 网络暂态 | 有上限的指数退避；记录结果未定状态，防重复业务副作用 |

**不要**无脑重放全部失败。应先分类永久性错误与可重试暂态错误。

### 8.4 防止双向同步循环

如果 Salesforce → MDM 同时也有入站集成，增加 `originSystem=MDM` 或相关集成上下文/可审计标记；源变更版本、字段拥有权、同步 Hash、去抖窗口与事件过滤结合，避免“MDM 回写 Salesforce → Salesforce Change Event → MDM 重写 → 再回写”的循环。字段拥有权矩阵优先于单纯 `LastModifiedDate` 判断；Salesforce 业务用户仍可拥有某些本地字段。

## 9. 上线前测试清单

- [ ] Salesforce Sandbox 中 `MDM_Golden_ID__c` 已设置 External ID + Unique。
- [ ] ECA/OAuth token 可用，Integration User 具备最小权限。
- [ ] Secure Agent 对 Salesforce My Domain 可达，TLS/Proxy 正常。
- [ ] 单条 PATCH 新建/更新以及 `updateOnly=true` 的缺失记录场景均验证。
- [ ] CDI REST V2 支持该 Swagger Operation 的 PATCH、动态 `{goldenId}`、OAuth 或 token 方案。
- [ ] 实际 200/201/204/4xx Response/错误 Group 映射正确。
- [ ] 金记录主键重复、空 Name、国家代码、字段长度与 Null 规则验证。
- [ ] Salesforce Flow/Trigger、Validation Rules、Duplicate Rules 的副作用验证。
- [ ] 审计、失败重试、幂等处理、同步水位及双向循环控制验证。
- [ ] 对象和 API 调用限额已做容量评估。

## 10. 官方技术文档与视频资源

### Salesforce 官方

1. [Upsert Records Using sObject Rows by External ID — REST API Developer Guide](https://developer.salesforce.com/docs/platform/api-rest/guide/resources-sobject-upsert-patch.html) — PATCH 外部 ID、updateOnly、返回语义。
2. [Insert or Update (Upsert) a Record Using an External ID](https://developer.salesforce.com/docs/atlas.en-us.api_rest.meta/object_reference/dome_upsert.htm) — 新建/更新 JSON 与状态码示例。
3. [Upsert Records Using sObject Collections](https://developer.salesforce.com/docs/platform/api-rest/guide/resources-composite-sobjects-collections-upsert.html) — 200 条批量 Upsert。
4. [Authorization Through External Client Apps or Connected Apps and OAuth 2.0](https://developer.salesforce.com/docs/platform/api-rest/guide/intro-oauth-and-connected-apps.html) — Salesforce OAuth 与 2026 新建应用建议。
5. [Invoke REST APIs with the Salesforce Integration User and OAuth Client Credentials](https://developer.salesforce.com/blogs/2024/02/invoke-rest-apis-with-the-salesforce-integration-user-and-oauth-client-credentials) — Server-to-server 最小权限实践。
6. [Send Multiple Requests Using Composite](https://developer.salesforce.com/docs/platform/api-rest/guide/resources-composite-composite-post.html) — 多资源组合调用（每请求上限与 Collections 不同）。

### Informatica 官方/支持站点

7. [Informatica REST V2 Connector Official Guide (PDF)](https://docs.informatica.com/content/dam/source/GUID-4/GUID-4D4C13E1-7BF0-4267-B56A-E2DFD055F69D/42/en/__%28REST%29V2Connector_en.pdf) — REST V2 Connection / Business Service / Mapping。
8. [REST ENDPOINT IMPLEMENTATION USE CASE — Informatica KB](https://knowledge.informatica.com/s/article/REST-ENDPOINT-IMPLEMENTATION-USE-CASE?language=en_US&type=external) — Midstream Mapping 实施案例。
9. [Informatica KB 000203790](https://knowledge.informatica.com/s/article/000203790?language=en_US) — API 有响应但 Mapping 无数据时，排查 Response Schema。
10. [视频：How to Use REST V2 Connection as Midstream in IICS](https://success.informatica.com/videos/support-videos/lfh0JIV32Lg.html) — Informatica Support 教学。
11. [视频：REST V2 OAuth Connection — Midstream and Source Demo](https://success.informatica.com/videos/support-videos/vdAjwnfPR2o.html) — OAuth 和 REST V2 教学。
12. [YouTube 搜索：Informatica REST V2 Web Services Transformation](https://www.youtube.com/results?search_query=Informatica+REST+V2+Web+Services+Transformation) — 搜索入口，非已核实的特定视频。
13. [YouTube 搜索：Salesforce REST API Upsert External ID](https://www.youtube.com/results?search_query=Salesforce+REST+API+Upsert+External+ID) — 搜索入口，非已核实的特定视频。

> 链接 7–11 沿用上一版本教程的 Informatica 官方文档与视频地址。REST V2 的实际兼容性和 UI 配置需在目标环境核验。这里优先列出英文官方内容。

---

# 11. 完整 API Request / Response JSON 示例（最终实施样例）

以下所有 JSON 均为**示例数据**，域名、ID、字段和版本必须替换为自己的 Salesforce Org 配置。对于单条 Salesforce External ID Upsert，**External ID 位于 URL 中而不是 Request Body 中**。

## 11.1 Scenario A — Golden Record 新建或更新 Salesforce Account（Upsert）

### HTTP Request

```http
PATCH https://YOUR_DOMAIN.my.salesforce.com/services/data/vXX.X/sobjects/Account/MDM_Golden_ID__c/GR-0001001
Authorization: Bearer eyJ...REDACTED
Content-Type: application/json
Accept: application/json
```

### Request JSON

```json
{
  "Name": "Acme Global Ltd",
  "Phone": "+14155550101",
  "Website": "https://acme.example",
  "BillingStreet": "100 Main Street",
  "BillingCity": "San Francisco",
  "BillingState": "CA",
  "BillingPostalCode": "94105",
  "BillingCountry": "United States",
  "MDM_Source_Version__c": "2026-10-26T10:00:00Z"
}
```

`MDM_Source_Version__c` 是可选自定义字段，只有实际创建才保留。示例业务中 Golden ID `GR-0001001` 对应 Salesforce Account 的 `MDM_Golden_ID__c`。

### Response JSON — 新建成功（HTTP 201，示意）

```json
{
  "id": "001XXXXXXXXXXXXXXX",
  "success": true,
  "errors": [],
  "created": true
}
```

### Response JSON — 更新成功（HTTP 200，适用支持该格式的 API 版本，示意）

```json
{
  "id": "001XXXXXXXXXXXXXXX",
  "success": true,
  "errors": [],
  "created": false
}
```

> 其他 API 版本可能使用 `204 No Content` 作为更新成功响应，务必实测并在 REST V2 里针对空响应处理。

## 11.2 Scenario B — 只更新已存在 Salesforce Account（Update Only）

```http
PATCH https://YOUR_DOMAIN.my.salesforce.com/services/data/vXX.X/sobjects/Account/MDM_Golden_ID__c/GR-0001001?updateOnly=true
Authorization: Bearer <ACCESS_TOKEN>
Content-Type: application/json
```

### Request JSON

```json
{
  "Name": "Acme Global Limited",
  "Phone": "+14155550199",
  "BillingCity": "San Francisco"
}
```

这个请求仅修改提交的字段；不存在的 External ID 不应被自动创建。返回状态码与错误体应依目标 API 版本和 org 实测处理。

## 11.3 Scenario C — Validation Error（示例）

### Response JSON — HTTP 400

```json
[
  {
    "message": "Required fields are missing: [Name]",
    "errorCode": "REQUIRED_FIELD_MISSING",
    "fields": ["Name"]
  }
]
```

这是说明错误体结构的例子，具体 message/errorCode/HTTP code 以 Salesforce 实际响应为准。

## 11.4 Scenario D — 批量 Golden Records Upsert（最多 200 条）

### HTTP Request

```http
PATCH https://YOUR_DOMAIN.my.salesforce.com/services/data/vXX.X/composite/sobjects/Account/MDM_Golden_ID__c
Authorization: Bearer <ACCESS_TOKEN>
Content-Type: application/json
```

### Request JSON

```json
{
  "allOrNone": false,
  "records": [
    {
      "attributes": { "type": "Account" },
      "MDM_Golden_ID__c": "GR-0001001",
      "Name": "Acme Global Ltd",
      "Phone": "+14155550101",
      "BillingCity": "San Francisco"
    },
    {
      "attributes": { "type": "Account" },
      "MDM_Golden_ID__c": "GR-0001002",
      "Name": "Beta Manufacturing Inc",
      "Phone": "+14155550102",
      "BillingCity": "Seattle"
    }
  ]
}
```

### Response JSON — HTTP 200（示意；逐项结果）

```json
[
  {
    "id": "001XXXXXXXXXXXXAAA",
    "success": true,
    "errors": [],
    "created": false
  },
  {
    "id": "001XXXXXXXXXXXXBBB",
    "success": true,
    "errors": [],
    "created": true
  }
]
```

务必检查数组里的**每个元素**，HTTP 200 不表示全部成功。

## 11.5 Swagger 2.0 JSON — 用于 REST V2 Business Service 的单条 PATCH 示例

> 这是规范层面的**起点模板**，不是保证 IDMC 直接导入后成功运行的 Salesforce 官方 Swagger。尤其须在 POC 中验证 `PATCH`、动态 path 参数、成功时 200/201/204 的返回兼容性，以及错误响应数组。API Host 和 Version 可修改。

```json
{
  "swagger": "2.0",
  "info": {
    "title": "Salesforce Account Upsert by MDM Golden ID",
    "version": "1.0.0"
  },
  "host": "YOUR_DOMAIN.my.salesforce.com",
  "basePath": "/services/data/vXX.X",
  "schemes": ["https"],
  "consumes": ["application/json"],
  "produces": ["application/json"],
  "paths": {
    "/sobjects/Account/MDM_Golden_ID__c/{goldenId}": {
      "patch": {
        "operationId": "upsertAccountByGoldenId",
        "summary": "Upsert Salesforce Account by MDM Golden Record ID",
        "parameters": [
          {
            "name": "goldenId",
            "in": "path",
            "required": true,
            "type": "string"
          },
          {
            "name": "account",
            "in": "body",
            "required": true,
            "schema": {
              "$ref": "#/definitions/AccountUpsertRequest"
            }
          }
        ],
        "responses": {
          "200": {
            "description": "Updated",
            "schema": {
              "$ref": "#/definitions/UpsertResponse"
            }
          },
          "201": {
            "description": "Created",
            "schema": {
              "$ref": "#/definitions/UpsertResponse"
            }
          },
          "204": {
            "description": "Updated; no response body"
          },
          "default": {
            "description": "Salesforce REST API error",
            "schema": {
              "type": "array",
              "items": {
                "$ref": "#/definitions/SalesforceError"
              }
            }
          }
        }
      }
    }
  },
  "definitions": {
    "AccountUpsertRequest": {
      "type": "object",
      "required": ["Name"],
      "properties": {
        "Name": { "type": "string" },
        "Phone": { "type": "string" },
        "Website": { "type": "string" },
        "BillingStreet": { "type": "string" },
        "BillingCity": { "type": "string" },
        "BillingState": { "type": "string" },
        "BillingPostalCode": { "type": "string" },
        "BillingCountry": { "type": "string" },
        "MDM_Source_Version__c": { "type": "string" }
      }
    },
    "UpsertResponse": {
      "type": "object",
      "properties": {
        "id": { "type": "string" },
        "success": { "type": "boolean" },
        "created": { "type": "boolean" },
        "errors": {
          "type": "array",
          "items": { "type": "object" }
        }
      }
    },
    "SalesforceError": {
      "type": "object",
      "properties": {
        "message": { "type": "string" },
        "errorCode": { "type": "string" },
        "fields": {
          "type": "array",
          "items": { "type": "string" }
        }
      }
    }
  }
}
```

**updateOnly 的 Swagger 变体**：可以在 `parameters` 增加一个可选的 query Boolean `updateOnly`，由 Mapping 填 `true`；也可以为“仅更新”使用单独固定 Endpoint 定义。必须确认 REST V2 将 Query 参数暴露给 Mapping 并按预期序列化。

## 11.6 curl 端到端测试

先通过合规机制得到真实 `SF_ACCESS_TOKEN`，**不要**写进共享日志/版本仓库：

```bash
curl --request PATCH \
  --url 'https://YOUR_DOMAIN.my.salesforce.com/services/data/vXX.X/sobjects/Account/MDM_Golden_ID__c/GR-0001001' \
  --header "Authorization: Bearer ${SF_ACCESS_TOKEN}" \
  --header 'Content-Type: application/json' \
  --data '{
    "Name": "Acme Global Ltd",
    "Phone": "+14155550101",
    "BillingCity": "San Francisco"
  }'
```

按上述顺序完成 Salesforce Sandbox curl → Secure Agent Connectivity → REST V2 Connection → Business Service → 1-row CDI Mapping POC，任何一步失败都先定位权限、Schema、URL 和认证问题，再扩大规模。
