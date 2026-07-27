# Informatica IDMC Business 360 与 Cloud Data Integration 资产清单导出方案

> 版本：1.0  
> 编写日期：2026-07-26  
> 适用范围：Informatica Intelligent Data Management Cloud（IDMC）、Business 360 / MDM SaaS、Cloud Data Integration（CDI）  
> 目标：导出 Project、Folder、Business Entity、Relationship、Mapping、Mapping Task、Taskflow 等资产的名称、完整路径和更新时间，并支持按指定日期时间进行增量导出。

---

## 1. 需求说明

本方案解决以下两类需求：

1. **完整清单导出**
   - 列出 IDMC 组织中当前用户有权限读取的 Project、Folder 和指定资产类型。
   - 输出资产名称、完整路径、资产类型、资产 ID、最后更新人和最后更新时间。
   - 同时覆盖 Business 360 / MDM SaaS 与 Cloud Data Integration。

2. **按时间增量导出**
   - 下次运行时，只导出从某个 UTC 日期时间之后发生更新的资产。
   - 支持手工指定时间范围。
   - 支持自动保存上一次成功运行的 watermark。
   - 使用重叠时间窗口和去重机制，降低边界时间或短暂索引延迟造成漏数的风险。

---

## 2. 推荐结论

### 2.1 主方案

推荐使用：

```text
IDMC Platform REST API v3
GET <baseApiUrl>/public/core/v3/objects
```

`objects` 资源适合本需求，原因包括：

- 可以统一列出 Project、Folder、CDI 资产和 MDM SaaS 资产。
- 返回资产的全局 ID、完整路径、类型、最后更新人和最后更新时间。
- 支持按 `type`、`location`、`updateTime` 和 `updatedBy` 过滤。
- 支持 `limit` 和 `skip` 分页。
- 同一套程序可以同时维护完整清单和增量清单。
- 不需要下载完整的资产 ZIP package，因此比迁移型 export API 更轻量。

### 2.2 不建议作为主方案的方法

| 方法 | 适合场景 | 不作为主方案的原因 |
|---|---|---|
| Business 360 或 CDI UI 手工查看 | 临时核查少量资产 | 不便自动化，难以稳定进行按时间增量导出 |
| Platform v3 Export API | 环境迁移、备份、部署 | 返回的是资产 package，目标不是轻量的名称/路径清单 |
| Asset Management CLI | CI/CD、批量迁移 | 更适合资产 package 操作，按 `updateTime` 清单查询不如 `objects` API 直接 |
| 未公开的浏览器内部 API | 临时技术调查 | 不属于稳定的公共接口，月度升级后 URL 或 JSON 结构可能变化 |
| 直接解析 export ZIP | Match 等特殊元数据调查 | 成本较高，应只作为补充方案 |

---

## 3. 总体架构

```mermaid
flowchart LR
    A[Scheduler / Manual Run] --> B[Python Inventory Script]
    B --> C[Platform REST API v3 Login]
    C --> D[GET /public/core/v3/objects]
    D --> E[Loop Asset Types]
    E --> F[limit/skip Pagination]
    F --> G[Normalize ID, Type, Name, Path]
    G --> H[CSV and JSON Output]
    G --> I[Deduplicate by ID + updateTime]
    I --> J[Save Watermark After Full Success]
    D -. Optional .-> K[GET /objects/{id}/references]
    A -. Optional .-> L[Platform v2 Audit Log]
    L --> M[CREATE / DELETE Events]
```

---

## 4. 可列出的资产类型

Informatica 当前 Platform REST API v3 文档将以下类型列为 `objects` 资源支持的类型。

### 4.1 Project 与 Folder

| 业务名称 | API type |
|---|---|
| Project | `PROJECT` |
| Folder | `FOLDER` |

### 4.2 Cloud Data Integration

| 业务名称 | API type |
|---|---|
| Mapping | `DTEMPLATE` |
| Mapping Task | `MTT` |
| Taskflow | `TASKFLOW` |
| Linear Taskflow | `WORKFLOW` |
| Synchronization Task | `DSS` |
| Masking Task | `DMASK` |
| Replication Task | `DRS` |
| Data Integration Mapplet | `DMAPPLET` |
| PowerCenter Mapplet | `MAPPLET` |
| Business Service Definition | `BSERVICE` |
| Hierarchical Schema | `HSCHEMA` |
| PowerCenter Task | `PCS` |
| Fixed-width Configuration | `FWCONFIG` |
| Saved Query | `CUSTOMSOURCE` |
| Mass Ingestion Task | `MI_TASK` |
| User-defined Function | `UDF` |

### 4.3 MDM SaaS / Business 360

| 业务名称 | API type |
|---|---|
| Business Entity | `MDM_BUSINESS_ENTITY` |
| Reference Entity | `MDM_REFERENCE_ENTITY` |
| Hierarchy | `MDM_HIERARCHY` |
| Relationship | `MDM_RELATIONSHIP` |
| Job Definition | `MDM_JOB_DEFINITION` |
| Authorization | `MDM_AUTHORIZATION` |
| Business Event | `MDM_BUSINESS_EVENT` |
| Report Set | `MDM_REPORT_SET` |
| Report | `MDM_REPORT` |
| Dynamic Pool | `MDM_DYNAMIC_POOL` |
| Application | `MDM_APPLICATION` |
| Source System | `MDM_SRC_SYSTEM` |
| Application Component | `MDM_APP_COMPONENT` |
| Application Page | `MDM_APP_PAGE` |

> 建议不要一次假定所有 IDMC service 类型都存在于当前组织。程序应允许通过配置选择实际需要的类型。

---

## 5. Match Model / Match Rules 的特殊处理

当前 `objects` API 的公开 MDM SaaS type 列表中没有单独列出：

```text
MDM_MATCH_MODEL
MDM_MATCH_RULE
```

因此不能假定以下查询可用：

```http
GET <baseApiUrl>/public/core/v3/objects?q=type=='MDM_MATCH_MODEL'
```

### 5.1 推荐处理方法

1. 用 `MDM_BUSINESS_ENTITY` 列出 Business Entity。
2. 用 `MDM_RELATIONSHIP` 列出业务模型中的 Relationship。
3. 将 Match Model 视为 Business Entity 的附属配置，而不是已确认的独立 Platform asset。
4. 如果必须单独获得 Match Model 名称、版本、状态或最后更新时间：
   - 先向 Informatica Support 确认当前 POD 和 release 是否提供受支持的 Business 360 service-specific metadata API。
   - 或使用 Business 360 / Platform Export API 导出相关 Business Entity 及 dependencies，再分析 export package。
5. 不建议将浏览器 Developer Tools 中看到的内部 endpoint 作为生产清单接口。

### 5.2 重要风险

即使 Business Entity 被成功列出，也不能在没有验证的情况下保证：

- 只修改 Match Rule 时，Business Entity 的 `updateTime` 一定发生变化。
- Match Model 在所有版本中都有独立的 FRS object ID。
- Match Model 的路径能从 `objects` API 直接取得。

如果 Match 资产是合规审计的强制范围，必须在测试组织中验证上述行为，并得到 Informatica 对公共接口支持范围的确认。

---

## 6. 身份认证

### 6.1 REST API v3 Login

请求：

```http
POST https://<cloud-provider>-<region>.informaticacloud.com/saas/public/core/v3/login
Content-Type: application/json
Accept: application/json

{
  "username": "<IDMC username>",
  "password": "<IDMC password>"
}
```

响应中需要取得：

- `sessionId`
- `baseApiUrl`

后续请求使用：

```http
INFA-SESSION-ID: <sessionId>
```

并以返回的 `baseApiUrl` 组成 v3 URL：

```text
<baseApiUrl>/public/core/v3/<resource>
```

### 6.2 POD 与 Login URL

不要把示例 POD URL直接用于所有环境。Login URL 取决于组织所在 POD。

推荐做法：

- 从管理员处取得组织 POD。
- 将完整 login URL 放在环境变量 `INFA_LOGIN_URL` 中。
- 不把用户名和密码写入源代码或 Git repository。
- 生产环境优先使用专用 service account。
- service account 应对目标 Project / Folder 至少具有 Read 权限。

---

## 7. 完整清单导出

### 7.1 基本请求

列出 Business Entity：

```http
GET <baseApiUrl>/public/core/v3/objects
    ?q=type=='MDM_BUSINESS_ENTITY'
    &limit=200
    &skip=0
```

列出 Relationship：

```http
GET <baseApiUrl>/public/core/v3/objects
    ?q=type=='MDM_RELATIONSHIP'
    &limit=200
    &skip=0
```

列出 Mapping：

```http
GET <baseApiUrl>/public/core/v3/objects
    ?q=type=='DTEMPLATE'
    &limit=200
    &skip=0
```

列出 Mapping Task：

```http
GET <baseApiUrl>/public/core/v3/objects
    ?q=type=='MTT'
    &limit=200
    &skip=0
```

列出 Taskflow：

```http
GET <baseApiUrl>/public/core/v3/objects
    ?q=type=='TASKFLOW'
    &limit=200
    &skip=0
```

### 7.2 为什么逐个 type 查询

建议一个 type 一次查询，而不是构造复杂的多类型表达式：

- 易于发现哪个 service 或 type 查询失败。
- 易于单独重试。
- 易于统计每类资产数量。
- 避免依赖未确认的 `OR` 查询语法。
- 方便后续按 service 分类输出。

### 7.3 分页

单次响应最多包含 200 个 assets。

分页方式：

```text
limit=200&skip=0
limit=200&skip=200
limit=200&skip=400
...
```

停止条件建议使用：

```text
本页 objects 数量 < limit
```

不要完全依赖响应中的 `count` 作为停止条件。官方文档说明，当结果量较大时，`count` 在短时间内可能不精确；实际 fetch list 仍会按请求参数返回数据。

### 7.4 权限范围

`objects` API 不会返回当前登录用户无 Read 权限的资产。

因此：

```text
API 返回清单
≠
组织中绝对全部资产
```

它表示：

```text
当前 service account 在当前组织中可以读取的资产
```

上线前应建立权限覆盖检查：

- 目标 Project 是否全部授权。
- Folder 是否存在单独 ACL。
- service account 是否能进入 Business 360 和 CDI。
- API 清单数量是否与 UI 管理员视图抽样一致。

---

## 8. 响应字段与输出模型

`objects` API 的主要返回字段包括：

```json
{
  "count": 4,
  "objects": [
    {
      "id": "1a3TnUrT2cfiwQGtkWQEUy",
      "path": "ProjectA/FolderA/Mapping1",
      "type": "DTEMPLATE",
      "description": "Example mapping",
      "updatedBy": "user@example.com",
      "updateTime": "2026-07-01T10:30:00Z"
    }
  ]
}
```

### 8.1 推荐 CSV 字段

| 字段 | 说明 |
|---|---|
| `service` | `Platform`、`CDI` 或 `MDM SaaS` |
| `requested_type` | 程序查询时使用的 type |
| `returned_type` | API 实际返回的 type |
| `asset_id` | FRS/global asset ID |
| `asset_name` | 从完整路径最后一段解析出的名称 |
| `full_path` | API 返回的完整路径 |
| `project_name` | 路径第一段 |
| `parent_folder_path` | Project 与资产名称之间的路径 |
| `description` | 资产描述 |
| `updated_by` | 最后更新用户 |
| `update_time_utc` | 最后更新时间 |
| `inventory_mode` | `full`、`initial`、`incremental` 或 `manual_range` |
| `query_start_utc` | 增量窗口开始时间 |
| `query_end_utc` | 增量窗口结束时间 |
| `inventory_run_time_utc` | 清单程序执行时间 |

### 8.2 名称与路径拆分

例如：

```text
full_path = Customer360/Model/Person
```

解析为：

```text
project_name       = Customer360
parent_folder_path = Model
asset_name         = Person
```

对于 Project：

```text
full_path = Customer360
```

解析为：

```text
project_name       = Customer360
parent_folder_path =
asset_name         = Customer360
```

---

## 9. 按指定日期时间增量导出

### 9.1 单一开始时间

列出指定时间之后更新的 Mapping Task：

```http
GET <baseApiUrl>/public/core/v3/objects
    ?q=type=='MTT' and updateTime>=2026-07-01T00:00:00Z
    &limit=200
    &skip=0
```

### 9.2 封闭开始、开放结束的时间窗口

推荐使用：

```text
updateTime >= start_time
updateTime <  end_time
```

例如：

```http
GET <baseApiUrl>/public/core/v3/objects
    ?q=type=='MDM_BUSINESS_ENTITY'
       and updateTime>=2026-07-01T00:00:00Z
       and updateTime<2026-07-26T12:00:00Z
    &limit=200
    &skip=0
```

使用半开区间 `[start, end)` 的优点：

- 窗口边界定义清晰。
- 下一窗口可以从上次 `end_time` 开始。
- 避免两个相邻窗口同时包含同一个结束时刻。

### 9.3 按最后更新用户过滤

如果还需要限定最后更新人：

```http
GET <baseApiUrl>/public/core/v3/objects
    ?q=type=='DTEMPLATE'
       and updatedBy=='user@example.com'
       and updateTime>=2026-07-01T00:00:00Z
       and updateTime<2026-07-26T12:00:00Z
```

注意：

- `updatedBy` 表示**最后更新人**。
- 它不等于**最初创建人**。

---

## 10. Watermark 设计

### 10.1 状态文件

建议使用 JSON 状态文件：

```json
{
  "lastSuccessfulRunUtc": "2026-07-26T02:15:30Z"
}
```

### 10.2 第一次成功运行

```text
query_start = 不设置
query_end   = 本次运行开始时的 UTC 时间
```

先导出当前完整资产清单，所有类型均成功后保存：

```text
lastSuccessfulRunUtc = query_end
```

### 10.3 后续自动增量运行

```text
previous_watermark = lastSuccessfulRunUtc
query_start        = previous_watermark - overlap
query_end          = 本次运行开始时的 UTC 时间
```

推荐：

```text
overlap = 60 seconds
```

查询：

```text
updateTime >= query_start
updateTime <  query_end
```

### 10.4 为什么需要 overlap

网络重试、索引更新和时间边界处理可能产生以下风险：

- 某资产更新时间正好等于上一次 watermark。
- 分页过程中资产又被更新。
- 后端索引短暂延迟。
- 不同服务对时间写入和索引可见时间存在轻微差异。

使用 60 秒重叠窗口后，通过以下键去重：

```text
asset_id + updateTime
```

### 10.5 什么时候更新 watermark

只在以下条件全部满足时更新：

- Login 成功。
- 所有配置的 asset type 查询成功。
- 所有分页完成。
- CSV/JSON 文件成功落盘。
- 去重完成。
- 没有未处理异常。

任何 type 失败时：

```text
不要推进 watermark
```

否则下一次运行可能永久跳过失败窗口中的资产。

---

## 11. 删除、创建人和历史事件

### 11.1 删除对象

`GET /public/core/v3/objects` 返回当前存在且当前用户能读取的资产。

已经删除的资产不会继续出现在当前清单中，因此仅靠 `updateTime` 增量无法可靠输出删除事件。

如需追踪删除：

- 使用 Platform REST API v2 Audit Log。
- 识别 `event=DELETE`。
- 保存 `entryTimeUTC`、`username`、`objectId`、`objectName`、`category` 和 `eventParam`。
- 将 audit delta 与当前资产 inventory 合并。

Audit Log endpoint：

```http
GET <serverUrl>/api/v2/auditlog?batchId=<batchId>&batchSize=<batchSize>
icSessionId: <version-2-session-id>
```

注意：

- v2 API 使用 `serverUrl` 和 `icSessionId`，不是 v3 的 `baseApiUrl` 和 `INFA-SESSION-ID`。
- Audit Log 对不同 IDMC service 和资产类别的覆盖应在实际组织中验证。
- MDM SaaS 特定对象是否全部产生所需 audit category，应进行测试。

### 11.2 “我创建的资产”

`objects` list response 公开字段主要是：

```text
updatedBy
updateTime
```

没有可直接用于统一筛选的标准 `createdBy` / `createTime` 字段。

因此：

| 实际需求 | 可行方法 |
|---|---|
| 我可以读取的资产 | 使用 `objects` API |
| 最后由我更新的资产 | 使用 `updatedBy=='username'` |
| 最初由我创建的资产 | 结合 Audit Log 的 `CREATE` 事件 |
| 已被删除的资产 | 结合 Audit Log 的 `DELETE` 事件 |

---

## 12. 资产依赖关系

不要混淆以下两个概念：

1. **Business 360 Relationship**
   - 资产类型为 `MDM_RELATIONSHIP`。
   - 表示业务模型中的关系定义。

2. **Platform asset dependency/reference**
   - 使用 `/objects/{objectId}/references`。
   - 表示技术资产“uses”或“usedBy”的依赖关系。

请求示例：

```http
GET <baseApiUrl>/public/core/v3/objects/<objectId>/references
    ?refType=uses
    &limit=50
    &skip=0
```

或：

```http
GET <baseApiUrl>/public/core/v3/objects/<objectId>/references
    ?refType=usedBy
    &limit=50
    &skip=0
```

可选用途：

- Mapping 使用哪些 Connection 或其他资产。
- 哪些 Mapping Task 使用某个 Mapping。
- 哪些 Taskflow 调用某个 Mapping Task。
- 变更影响分析。
- 删除前依赖检查。

依赖接口单页上限与主 `objects` 接口不同，应使用该接口文档规定的分页值。

---

## 13. 可直接使用的 Python 示例

下面脚本实现：

- v3 login。
- 多资产类型循环。
- `limit=200`、`skip` 分页。
- 完整导出。
- 自动 watermark 增量。
- 手工时间范围导出。
- `updatedBy` 和 `location` 可选过滤。
- 429 和常见 5xx 重试。
- CSV 与 JSON 输出。
- `asset_id + updateTime` 去重。
- 只有完全成功后才保存 watermark。

保存为：

```text
idmc_asset_inventory.py
```

```python
#!/usr/bin/env python3
"""
IDMC Business 360 / CDI asset inventory exporter.

Required environment variables:
    INFA_LOGIN_URL
    INFA_USERNAME
    INFA_PASSWORD

Install:
    python -m pip install requests

Examples:
    # First/full export and initialize watermark
    python idmc_asset_inventory.py --full

    # Incremental export based on saved watermark
    python idmc_asset_inventory.py

    # Ad hoc time window; does not advance saved watermark
    python idmc_asset_inventory.py \
        --from-time 2026-07-01T00:00:00Z \
        --to-time   2026-07-26T00:00:00Z

    # Only selected types
    python idmc_asset_inventory.py \
        --types MDM_BUSINESS_ENTITY,MDM_RELATIONSHIP,DTEMPLATE,MTT,TASKFLOW

    # Filter by user who last updated assets
    python idmc_asset_inventory.py \
        --updated-by user@example.com
"""

from __future__ import annotations

import argparse
import csv
import json
import os
import sys
import time
from datetime import datetime, timedelta, timezone
from pathlib import Path
from typing import Any, Iterable

import requests


DEFAULT_ASSET_TYPES = [
    # Platform structure
    "PROJECT",
    "FOLDER",

    # MDM SaaS / Business 360
    "MDM_BUSINESS_ENTITY",
    "MDM_REFERENCE_ENTITY",
    "MDM_HIERARCHY",
    "MDM_RELATIONSHIP",
    "MDM_JOB_DEFINITION",
    "MDM_AUTHORIZATION",
    "MDM_BUSINESS_EVENT",
    "MDM_REPORT_SET",
    "MDM_REPORT",
    "MDM_DYNAMIC_POOL",
    "MDM_APPLICATION",
    "MDM_SRC_SYSTEM",
    "MDM_APP_COMPONENT",
    "MDM_APP_PAGE",

    # Cloud Data Integration
    "DTEMPLATE",
    "MTT",
    "TASKFLOW",
    "WORKFLOW",
    "DSS",
    "DMASK",
    "DRS",
    "DMAPPLET",
    "MAPPLET",
    "BSERVICE",
    "HSCHEMA",
    "PCS",
    "FWCONFIG",
    "CUSTOMSOURCE",
    "MI_TASK",
    "UDF",
]

MDM_TYPES = {
    value for value in DEFAULT_ASSET_TYPES if value.startswith("MDM_")
}

CDI_TYPES = {
    "DTEMPLATE",
    "MTT",
    "TASKFLOW",
    "WORKFLOW",
    "DSS",
    "DMASK",
    "DRS",
    "DMAPPLET",
    "MAPPLET",
    "BSERVICE",
    "HSCHEMA",
    "PCS",
    "FWCONFIG",
    "CUSTOMSOURCE",
    "MI_TASK",
    "UDF",
}

RETRYABLE_STATUS_CODES = {429, 500, 502, 503, 504}
PAGE_LIMIT = 200
DEFAULT_OVERLAP_SECONDS = 60


def utc_now() -> datetime:
    return datetime.now(timezone.utc).replace(microsecond=0)


def parse_iso_datetime(value: str) -> datetime:
    """Parse ISO-8601 and normalize to UTC."""
    normalized = value.strip()
    if normalized.endswith("Z"):
        normalized = normalized[:-1] + "+00:00"

    parsed = datetime.fromisoformat(normalized)
    if parsed.tzinfo is None:
        raise ValueError(
            f"Datetime must include a timezone or Z suffix: {value}"
        )

    return parsed.astimezone(timezone.utc).replace(microsecond=0)


def iso_z(value: datetime | None) -> str:
    if value is None:
        return ""
    return value.astimezone(timezone.utc).strftime("%Y-%m-%dT%H:%M:%SZ")


def require_env(name: str) -> str:
    value = os.getenv(name)
    if not value:
        raise RuntimeError(f"Missing required environment variable: {name}")
    return value


def validate_filter_value(value: str | None, field_name: str) -> None:
    """
    Keep the example conservative.
    Reject apostrophes rather than guessing Informatica query escaping rules.
    """
    if value and "'" in value:
        raise ValueError(
            f"{field_name} contains an apostrophe, which this example "
            "does not automatically escape."
        )


def request_with_retry(
    session: requests.Session,
    method: str,
    url: str,
    *,
    max_attempts: int = 6,
    **kwargs: Any,
) -> requests.Response:
    last_error: Exception | None = None

    for attempt in range(1, max_attempts + 1):
        try:
            response = session.request(method, url, timeout=60, **kwargs)

            if response.status_code in RETRYABLE_STATUS_CODES:
                retry_after = response.headers.get("Retry-After")
                if retry_after and retry_after.isdigit():
                    delay = int(retry_after)
                else:
                    delay = min(2 ** (attempt - 1), 30)

                if attempt == max_attempts:
                    response.raise_for_status()

                print(
                    f"Retryable HTTP {response.status_code}; "
                    f"sleeping {delay}s before retry {attempt + 1}/"
                    f"{max_attempts}",
                    file=sys.stderr,
                )
                time.sleep(delay)
                continue

            response.raise_for_status()
            return response

        except (requests.RequestException, ValueError) as exc:
            last_error = exc
            if attempt == max_attempts:
                break

            delay = min(2 ** (attempt - 1), 30)
            print(
                f"Request failed: {exc}; sleeping {delay}s before retry "
                f"{attempt + 1}/{max_attempts}",
                file=sys.stderr,
            )
            time.sleep(delay)

    raise RuntimeError(f"Request failed after retries: {last_error}")


def login_v3(
    session: requests.Session,
    login_url: str,
    username: str,
    password: str,
) -> tuple[str, str]:
    response = request_with_retry(
        session,
        "POST",
        login_url,
        headers={
            "Content-Type": "application/json",
            "Accept": "application/json",
        },
        json={
            "username": username,
            "password": password,
        },
    )
    payload = response.json()

    user_info = payload.get("userInfo") or {}
    session_id = user_info.get("sessionId") or payload.get("sessionId")

    base_api_url = payload.get("baseApiUrl")
    if not base_api_url:
        for product in payload.get("products") or []:
            candidate = product.get("baseApiUrl")
            if candidate:
                base_api_url = candidate
                break

    if not session_id:
        raise RuntimeError(
            "Login succeeded but no sessionId was found in the response."
        )
    if not base_api_url:
        raise RuntimeError(
            "Login succeeded but no baseApiUrl was found in the response."
        )

    return str(session_id), str(base_api_url).rstrip("/")


def load_state(path: Path) -> dict[str, Any]:
    if not path.exists():
        return {}

    with path.open("r", encoding="utf-8") as handle:
        payload = json.load(handle)

    if not isinstance(payload, dict):
        raise ValueError(f"State file must contain a JSON object: {path}")
    return payload


def save_state_atomic(path: Path, payload: dict[str, Any]) -> None:
    path.parent.mkdir(parents=True, exist_ok=True)
    temp_path = path.with_suffix(path.suffix + ".tmp")

    with temp_path.open("w", encoding="utf-8") as handle:
        json.dump(payload, handle, indent=2, ensure_ascii=False)
        handle.write("\n")

    temp_path.replace(path)


def build_query(
    asset_type: str,
    *,
    start_time: datetime | None,
    end_time: datetime | None,
    updated_by: str | None,
    location: str | None,
) -> str:
    conditions = [f"type=='{asset_type}'"]

    if start_time:
        conditions.append(f"updateTime>={iso_z(start_time)}")
    if end_time:
        conditions.append(f"updateTime<{iso_z(end_time)}")
    if updated_by:
        conditions.append(f"updatedBy=='{updated_by}'")
    if location:
        conditions.append(f"location=='{location}'")

    return " and ".join(conditions)


def fetch_assets_for_type(
    session: requests.Session,
    *,
    base_api_url: str,
    session_id: str,
    asset_type: str,
    start_time: datetime | None,
    end_time: datetime | None,
    updated_by: str | None,
    location: str | None,
) -> list[dict[str, Any]]:
    url = f"{base_api_url}/public/core/v3/objects"
    query = build_query(
        asset_type,
        start_time=start_time,
        end_time=end_time,
        updated_by=updated_by,
        location=location,
    )

    headers = {
        "Accept": "application/json",
        "INFA-SESSION-ID": session_id,
    }

    all_objects: list[dict[str, Any]] = []
    skip = 0

    while True:
        response = request_with_retry(
            session,
            "GET",
            url,
            headers=headers,
            params={
                "q": query,
                "limit": PAGE_LIMIT,
                "skip": skip,
            },
        )
        payload = response.json()
        objects = payload.get("objects") or []

        if not isinstance(objects, list):
            raise RuntimeError(
                f"Unexpected objects response for type {asset_type}: "
                f"{type(objects).__name__}"
            )

        all_objects.extend(objects)
        print(
            f"{asset_type}: fetched {len(objects)} objects "
            f"(running total {len(all_objects)})"
        )

        # Do not depend only on count because the documented count can lag.
        if len(objects) < PAGE_LIMIT:
            break

        skip += len(objects)

    return all_objects


def classify_service(asset_type: str) -> str:
    upper = asset_type.upper()
    if upper in MDM_TYPES:
        return "MDM SaaS"
    if upper in CDI_TYPES:
        return "CDI"
    return "Platform"


def split_path(full_path: str) -> tuple[str, str, str]:
    parts = [part for part in full_path.split("/") if part]

    if not parts:
        return "", "", ""

    asset_name = parts[-1]
    project_name = parts[0]
    parent_folder_path = "/".join(parts[1:-1]) if len(parts) > 2 else ""

    return asset_name, project_name, parent_folder_path


def normalize_object(
    obj: dict[str, Any],
    *,
    requested_type: str,
    inventory_mode: str,
    query_start: datetime | None,
    query_end: datetime,
    run_time: datetime,
) -> dict[str, Any]:
    full_path = str(obj.get("path") or "")
    asset_name, project_name, parent_folder_path = split_path(full_path)

    returned_type = str(obj.get("type") or requested_type)

    return {
        "service": classify_service(returned_type),
        "requested_type": requested_type,
        "returned_type": returned_type,
        "asset_id": str(obj.get("id") or ""),
        "asset_name": asset_name,
        "full_path": full_path,
        "project_name": project_name,
        "parent_folder_path": parent_folder_path,
        "description": str(obj.get("description") or ""),
        "updated_by": str(obj.get("updatedBy") or ""),
        "update_time_utc": str(obj.get("updateTime") or ""),
        "inventory_mode": inventory_mode,
        "query_start_utc": iso_z(query_start),
        "query_end_utc": iso_z(query_end),
        "inventory_run_time_utc": iso_z(run_time),
    }


def deduplicate_rows(
    rows: Iterable[dict[str, Any]]
) -> list[dict[str, Any]]:
    deduped: dict[tuple[str, str, str], dict[str, Any]] = {}

    for row in rows:
        # Include type as a defensive fallback if an ID is unexpectedly absent.
        key = (
            str(row.get("asset_id") or ""),
            str(row.get("update_time_utc") or ""),
            str(row.get("returned_type") or ""),
        )
        deduped[key] = row

    return sorted(
        deduped.values(),
        key=lambda row: (
            str(row.get("service") or ""),
            str(row.get("returned_type") or ""),
            str(row.get("full_path") or ""),
        ),
    )


def write_csv(path: Path, rows: list[dict[str, Any]]) -> None:
    path.parent.mkdir(parents=True, exist_ok=True)

    fieldnames = [
        "service",
        "requested_type",
        "returned_type",
        "asset_id",
        "asset_name",
        "full_path",
        "project_name",
        "parent_folder_path",
        "description",
        "updated_by",
        "update_time_utc",
        "inventory_mode",
        "query_start_utc",
        "query_end_utc",
        "inventory_run_time_utc",
    ]

    # utf-8-sig makes Chinese content easier to open in Excel.
    with path.open("w", encoding="utf-8-sig", newline="") as handle:
        writer = csv.DictWriter(handle, fieldnames=fieldnames)
        writer.writeheader()
        writer.writerows(rows)


def write_json(path: Path, rows: list[dict[str, Any]]) -> None:
    path.parent.mkdir(parents=True, exist_ok=True)
    with path.open("w", encoding="utf-8") as handle:
        json.dump(rows, handle, indent=2, ensure_ascii=False)
        handle.write("\n")


def parse_args() -> argparse.Namespace:
    parser = argparse.ArgumentParser(
        description="Export IDMC Business 360 and CDI asset inventory."
    )

    mode = parser.add_mutually_exclusive_group()
    mode.add_argument(
        "--full",
        action="store_true",
        help="Ignore the previous watermark and export all current assets.",
    )
    mode.add_argument(
        "--from-time",
        help=(
            "Ad hoc UTC/ISO start time, for example "
            "2026-07-01T00:00:00Z. Does not advance saved watermark."
        ),
    )

    parser.add_argument(
        "--to-time",
        help=(
            "Optional UTC/ISO exclusive end time. "
            "Defaults to the program start time."
        ),
    )
    parser.add_argument(
        "--types",
        help=(
            "Comma-separated asset types. "
            "Defaults to the configured Business 360 and CDI list."
        ),
    )
    parser.add_argument(
        "--updated-by",
        help="Optional userName filter for the user who last updated assets.",
    )
    parser.add_argument(
        "--location",
        help="Optional exact project/folder location filter.",
    )
    parser.add_argument(
        "--overlap-seconds",
        type=int,
        default=DEFAULT_OVERLAP_SECONDS,
        help="Watermark overlap for automatic incremental mode.",
    )
    parser.add_argument(
        "--state-file",
        default="inventory_state.json",
        help="Path to the watermark state file.",
    )
    parser.add_argument(
        "--output-dir",
        default="inventory_output",
        help="Directory for CSV and JSON output.",
    )

    return parser.parse_args()


def main() -> int:
    args = parse_args()

    validate_filter_value(args.updated_by, "updated-by")
    validate_filter_value(args.location, "location")

    if args.overlap_seconds < 0:
        raise ValueError("--overlap-seconds cannot be negative")

    login_url = require_env("INFA_LOGIN_URL")
    username = require_env("INFA_USERNAME")
    password = require_env("INFA_PASSWORD")

    state_path = Path(args.state_file)
    output_dir = Path(args.output_dir)

    run_time = utc_now()
    query_end = (
        parse_iso_datetime(args.to_time)
        if args.to_time
        else run_time
    )

    state = load_state(state_path)
    explicit_start = (
        parse_iso_datetime(args.from_time)
        if args.from_time
        else None
    )

    if args.full:
        inventory_mode = "full"
        query_start = None
        advance_state = True
    elif explicit_start:
        inventory_mode = "manual_range"
        query_start = explicit_start
        advance_state = False
    elif state.get("lastSuccessfulRunUtc"):
        inventory_mode = "incremental"
        previous = parse_iso_datetime(
            str(state["lastSuccessfulRunUtc"])
        )
        query_start = previous - timedelta(
            seconds=args.overlap_seconds
        )
        advance_state = True
    else:
        inventory_mode = "initial"
        query_start = None
        advance_state = True

    if query_start and query_start >= query_end:
        raise ValueError("query start must be earlier than query end")

    if args.types:
        asset_types = [
            item.strip().upper()
            for item in args.types.split(",")
            if item.strip()
        ]
    else:
        asset_types = DEFAULT_ASSET_TYPES

    if not asset_types:
        raise ValueError("No asset types were selected")

    print(f"Mode: {inventory_mode}")
    print(f"Query start: {iso_z(query_start) or '(not set)'}")
    print(f"Query end:   {iso_z(query_end)}")
    print(f"Types:       {len(asset_types)}")

    with requests.Session() as http:
        session_id, base_api_url = login_v3(
            http,
            login_url,
            username,
            password,
        )

        normalized_rows: list[dict[str, Any]] = []

        # Any exception stops the program before state advancement.
        for asset_type in asset_types:
            objects = fetch_assets_for_type(
                http,
                base_api_url=base_api_url,
                session_id=session_id,
                asset_type=asset_type,
                start_time=query_start,
                end_time=query_end,
                updated_by=args.updated_by,
                location=args.location,
            )

            for obj in objects:
                normalized_rows.append(
                    normalize_object(
                        obj,
                        requested_type=asset_type,
                        inventory_mode=inventory_mode,
                        query_start=query_start,
                        query_end=query_end,
                        run_time=run_time,
                    )
                )

    rows = deduplicate_rows(normalized_rows)

    timestamp = run_time.strftime("%Y%m%dT%H%M%SZ")
    csv_path = output_dir / f"idmc_asset_inventory_{timestamp}.csv"
    json_path = output_dir / f"idmc_asset_inventory_{timestamp}.json"

    write_csv(csv_path, rows)
    write_json(json_path, rows)

    if advance_state:
        save_state_atomic(
            state_path,
            {
                "lastSuccessfulRunUtc": iso_z(query_end),
                "lastMode": inventory_mode,
                "lastOutputCsv": str(csv_path),
                "lastOutputJson": str(json_path),
                "lastRowCount": len(rows),
                "assetTypes": asset_types,
            },
        )

    print(f"Rows written: {len(rows)}")
    print(f"CSV:  {csv_path}")
    print(f"JSON: {json_path}")
    if advance_state:
        print(f"State advanced to: {iso_z(query_end)}")
    else:
        print("State was not changed for this ad hoc manual range.")

    return 0


if __name__ == "__main__":
    try:
        raise SystemExit(main())
    except Exception as exc:
        print(f"ERROR: {exc}", file=sys.stderr)
        raise SystemExit(1)
```

---

## 14. 运行配置

### 14.1 安装依赖

```bash
python -m pip install requests
```

### 14.2 Linux / macOS 环境变量

```bash
export INFA_LOGIN_URL="https://<cloud-provider>-<region>.informaticacloud.com/saas/public/core/v3/login"
export INFA_USERNAME="service-account@example.com"
export INFA_PASSWORD="<secret>"
```

### 14.3 Windows PowerShell 环境变量

```powershell
$env:INFA_LOGIN_URL = "https://<cloud-provider>-<region>.informaticacloud.com/saas/public/core/v3/login"
$env:INFA_USERNAME = "service-account@example.com"
$env:INFA_PASSWORD = "<secret>"
```

### 14.4 第一次完整运行

```bash
python idmc_asset_inventory.py --full
```

结果：

```text
inventory_output/
  idmc_asset_inventory_20260726T021530Z.csv
  idmc_asset_inventory_20260726T021530Z.json

inventory_state.json
```

### 14.5 后续自动增量

```bash
python idmc_asset_inventory.py
```

程序自动读取：

```text
inventory_state.json
```

并使用：

```text
lastSuccessfulRunUtc - 60 seconds
```

作为查询开始时间。

### 14.6 手工指定时间范围

```bash
python idmc_asset_inventory.py \
  --from-time 2026-07-01T00:00:00Z \
  --to-time   2026-07-26T00:00:00Z
```

该模式默认不推进自动 watermark，适合：

- 补跑历史窗口。
- 验证某一天的变更。
- 审计抽取。
- 故障后重放。

### 14.7 只导出重点类型

```bash
python idmc_asset_inventory.py \
  --types PROJECT,FOLDER,MDM_BUSINESS_ENTITY,MDM_RELATIONSHIP,DTEMPLATE,MTT,TASKFLOW
```

### 14.8 限定某个 Project / Folder location

```bash
python idmc_asset_inventory.py \
  --location "Customer360/Model"
```

`location` 是精确 Project/Folder path 过滤条件。需要确认目标资产的实际 location 路径与大小写。

---

## 15. 调度建议

### 15.1 推荐频率

根据资产变更频率选择：

| 场景 | 建议频率 |
|---|---|
| 日常资产治理 | 每日一次 |
| 活跃开发环境 | 每小时或每 2–4 小时 |
| Release 审计 | 部署前、部署后各一次 |
| 仅月度盘点 | 每月一次完整导出 |

### 15.2 调度工具

可使用：

- Linux cron
- Windows Task Scheduler
- Jenkins
- GitHub Actions self-hosted runner
- Azure DevOps Pipeline
- Control-M
- 企业内部调度平台

### 15.3 并发控制

同一个组织和同一个 state file 不应同时运行多个 inventory process。

推荐：

- 使用文件锁或调度器互斥。
- 每个组织使用独立 state file。
- 每个环境使用独立输出目录。
- 不允许 Dev 与 Prod 共用 watermark。

---

## 16. 生产化增强建议

### 16.1 将 CSV 改为数据库表

建议表结构：

```sql
CREATE TABLE idmc_asset_inventory (
    org_name              VARCHAR(200),
    service               VARCHAR(50),
    asset_type            VARCHAR(100),
    asset_id              VARCHAR(200),
    asset_name            VARCHAR(500),
    full_path             VARCHAR(2000),
    project_name          VARCHAR(500),
    parent_folder_path    VARCHAR(1500),
    updated_by            VARCHAR(500),
    update_time_utc       TIMESTAMP,
    inventory_run_time_utc TIMESTAMP,
    is_current            BOOLEAN,
    PRIMARY KEY (org_name, asset_id, update_time_utc)
);
```

另建当前状态表：

```sql
CREATE TABLE idmc_asset_current (
    org_name              VARCHAR(200),
    asset_id              VARCHAR(200),
    asset_type            VARCHAR(100),
    full_path             VARCHAR(2000),
    updated_by            VARCHAR(500),
    update_time_utc       TIMESTAMP,
    last_seen_time_utc    TIMESTAMP,
    PRIMARY KEY (org_name, asset_id)
);
```

### 16.2 Snapshot 与 Delta 同时保留

推荐同时保留：

1. **Snapshot**
   - 当前所有存在的资产。
   - 适合盘点和路径搜索。

2. **Delta history**
   - 每次发生更新时保留一条记录。
   - 适合审计和趋势分析。

3. **Audit event**
   - CREATE、UPDATE、DELETE 等事件。
   - 用于补足当前态 API 无法表示的删除历史。

### 16.3 定期完整对账

即使有增量机制，也建议每周或每月执行一次完整 snapshot，并与 current table 对账。

可检查：

- 增量漏数。
- 资产删除。
- 权限变化造成的资产消失。
- Project/Folder 移动。
- API type 支持范围变化。
- service account 权限异常。

### 16.4 Rate limit 与重试

官方文档说明 `objects` 资源使用 dynamic rate limit。

生产程序应：

- 对 HTTP 429 执行 backoff。
- 读取 `Retry-After`，如果存在则优先采用。
- 对 500、502、503、504 进行有限次数重试。
- 记录失败的 type、query、skip 和 HTTP response。
- 不在部分成功时推进 watermark。
- 避免不受控的高并发。

---

## 17. 验证测试计划

上线前至少执行以下测试。

### 17.1 完整清单

- 新建 Project。
- 新建 Folder。
- 新建 Mapping、Mapping Task 和 Taskflow。
- 新建 Business Entity 和 Relationship。
- 执行 `--full`。
- 确认 name、path、type 和 updateTime 正确。

### 17.2 增量更新

- 记录当前 watermark。
- 修改一个 Mapping。
- 修改一个 Business Entity。
- 运行增量。
- 确认只返回窗口内更新的资产。

### 17.3 时间边界

- 在 watermark 前后各修改一个资产。
- 使用 60 秒 overlap 运行。
- 确认不会漏掉边界资产。
- 确认去重后没有重复行。

### 17.4 分页

- 选择资产数量超过 200 的类型。
- 验证 `skip=0`、`skip=200`、`skip=400`。
- 确认最终唯一 asset ID 数量符合预期。

### 17.5 权限

- 用管理员账号运行。
- 用 service account 运行。
- 比较差异。
- 确认差异来自 ACL，而不是 API 漏数。

### 17.6 删除

- 创建测试 Mapping Task。
- 完整导出。
- 删除该资产。
- 再次执行 `objects` 清单。
- 确认当前清单中已不存在。
- 检查 Audit Log 是否包含 `DELETE` 事件。

### 17.7 Match 配置

- 修改 Match Model / Match Rule。
- 检查关联 Business Entity 的 `updateTime` 是否变化。
- 检查 export package 是否包含所需 Match 元数据。
- 将结果提交 Informatica Support 确认公共 API 支持范围。

---

## 18. 常见问题

### Q1：能否一次查询所有类型？

可以不设置 `type` 来取得组织资产，但不建议作为生产主方式，因为：

- 结果量可能很大。
- 无法方便判断某种类型是否失败。
- 不利于分类统计和重试。
- 新增 service type 可能使输出范围意外扩大。

推荐按配置的 type 逐个查询。

### Q2：`path` 是否已经包含名称？

是。官方 response 将 `path` 定义为包含 Project、Folder 和 object name 的完整路径。

### Q3：为什么还要保存 asset ID？

路径和名称可能发生变化，asset ID 更适合：

- 去重。
- 追踪 rename。
- 追踪 move。
- 查询 dependencies。
- 作为 export API 输入。

### Q4：可以只用最后一条返回记录的 updateTime 当 watermark 吗？

不建议。分页排序不应被当作可靠 watermark 顺序。

应使用：

```text
本次运行开始时确定的 query_end
```

并在全部成功后保存它。

### Q5：如何捕获删除？

使用 Audit Log，或定期完整 snapshot 做集合差异比较。

### Q6：如何准确找“我创建的”？

使用 Audit Log 的 `CREATE` event 和 `username`，而不是 `updatedBy`。

### Q7：为什么不直接导出所有资产 ZIP？

你的需求是名称和路径清单。ZIP export 会增加：

- job 提交与轮询。
- package 下载。
- ZIP 解压。
- 不同 asset schema 解析。
- dependencies 去重。

只有在 Match 等元数据无法通过公共 list API 获得时，才建议把 export package 作为补充。

---

## 19. 官方参考链接

以下均为 Informatica 官方英文文档。

1. **REST API v3 Login**  
   <https://docs.informatica.com/integration-cloud/data-ingestion-and-replication/current-version/rest-api-reference/platform-rest-api-version-3-resources/login.html>

2. **Platform REST API v3 Objects Overview**  
   <https://docs.informatica.com/integration-cloud/data-integration/current-version/rest-api-reference/platform-rest-api-version-3-resources/objects.html>

3. **Finding an Asset：type、location、updateTime、updatedBy、limit、skip、返回字段**  
   <https://docs.informatica.com/integration-cloud/data-ingestion-and-replication/current-version/rest-api-reference/platform-rest-api-version-3-resources/objects/finding-an-asset.html>

4. **Finding Asset Dependencies**  
   <https://docs.informatica.com/ipaas/b2b-gateway/current-version/rest-api-reference/platform-rest-api-version-3-resources/objects/finding-asset-dependencies.html>

5. **Starting an Export Job**  
   <https://docs.informatica.com/integration-cloud/data-integration/current-version/rest-api-reference/platform-rest-api-version-3-resources/exporting-objects/starting-an-export-job.html>

6. **Getting Export Job Status**  
   <https://docs.informatica.com/integration-cloud/data-integration/current-version/rest-api-reference/platform-rest-api-version-3-resources/exporting-objects/getting-the-export-job-status.html>

7. **Platform REST API v2 Audit Logs**  
   <https://docs.informatica.com/integration-cloud/data-integration/current-version/rest-api-reference/platform-rest-api-version-2-resources/audit-logs.html>

8. **Date/Time Values**  
   <https://docs.informatica.com/integration-cloud/data-ingestion-and-replication/current-version/rest-api-reference/informatica-intelligent-cloud-services-rest-api/date-time-values.html>

9. **Object IDs**  
   <https://docs.informatica.com/integration-cloud/data-integration/current-version/rest-api-reference/informatica-intelligent-cloud-services-rest-api/object-ids.html>

10. **Data Integration REST API Overview**  
    <https://docs.informatica.com/integration-cloud/data-integration/current-version/rest-api-reference/data-integration-rest-api.html>

---

## 20. 最终实施建议

建议分三个阶段实施。

### 阶段一：最小可行版本

范围：

```text
PROJECT
FOLDER
MDM_BUSINESS_ENTITY
MDM_RELATIONSHIP
DTEMPLATE
MTT
TASKFLOW
WORKFLOW
```

实现：

- 完整 CSV。
- 手工 `--from-time`。
- API 分页。
- service account 权限验证。

### 阶段二：自动增量

增加：

- JSON watermark。
- 60 秒 overlap。
- CSV/JSON 双输出。
- 调度。
- 429/5xx 重试。
- 失败不推进 watermark。
- 每周完整 snapshot。

### 阶段三：审计与 Match 补充

增加：

- Audit Log CREATE/DELETE。
- Current 与 history 数据库表。
- Asset dependency。
- Match Model export package 调查。
- 与 Informatica Support 确认 Match metadata 公共 API。
- 变更影响分析和治理报表。

---

## 21. 一句话总结

对于 Business 360 和 Cloud Data Integration 的资产名称/路径清单，最佳主方案是：

```text
Platform REST API v3 objects
+ 按 type 分页查询
+ updateTime 半开区间增量
+ persisted watermark
+ overlap and deduplication
+ periodic full reconciliation
```

而 Match Model、删除事件和最初创建人，应分别使用受支持的 Business 360 metadata/export 能力以及 Platform Audit Log 作为补充。

https://success.informatica.com/videos/support-videos/P5NTX-AP8cc.html

