# MDM SaaS → CDI Egress → Salesforce Account 去重与软删除实施方案（修订版 V2）

**修订日期：2026-10-26**  
**状态：实施设计 / 上线前需业务及连接器验证**  
**限制：不允许使用数据库表或持久化 Survivor 控制表；每小时运行一次。**

## 1. 目标和边界

- 来源：`SRC_ORG_xrefLinks` 仅有 `BusinessID`、`SourceSystem`、`SourcePKey`。
- `SourcePKey` 在 Salesforce 来源中存放 Salesforce Account.Id（15/18 位字母数字标识符）；同一 BusinessID 也可能有 SAP、Oracle 等其他来源。
- 同一个 BusinessID 的多个 Salesforce Account，仅保留一个；其他 **仍有效** 的 Account 执行 Salesforce 标准 **Delete（软删除）**。
- 已被删除的 Salesforce Account 不重复提交 Delete。
- 不更新 Salesforce 标准 `IsDeleted`：它是只读状态字段，删除必须使用 **Delete** 操作。
- 不在此 Mapping 中删除 MDM Crosswalk；不删除非 Salesforce 来源。
- **安全假设**：业务明确同意按 `MAX(SourcePKey)` 选择 Survivor。该排序只是技术约定，**不代表最新创建、最近修改、最可信或真实 Golden Record**。

> **重要上线限制：** 无外部持久化状态、MDM 又不给出权威 Survivor ID 时，无法保证 Xref 集合改变后 Survivor 永远不变。例如新增一个排序更大的 Salesforce ID 会令 MAX 改选新 Survivor。**若业务要求 Survivor 绝对不可变，必须由 MDM 提供权威字段/关系，或者改用持久化状态；本设计不能保证这一点。** 对 Xref 变化、历史删除及人工恢复应建立运维审批机制。

## 2. 为什么选 Lookup，而非另加一条 Salesforce Source + Joiner

- 使用两个 Salesforce Lookup 查询单条 Account 当前可见/有效状态，映射较直观：一个核对 **Survivor Account**，一个核对 **待处理 Account**。
- **仍然需要一个 CDI Joiner**：它是用来把 `AGG_MAX` 的分组 MAX 值回连到原始 **所有 Xref 行**，不是为了访问 Salesforce。
- 批量数据量非常大时，可以评估 Salesforce Source + Joiner 的批量读取设计，与 Lookup 的 API 调用、缓存、时间、平台限额作实测对比；Lookup 未必更快。
- Informatica Salesforce Connector 的官方资料说明 Salesforce Lookup 可用于 Mapping，但 **Lookup 只支持 Salesforce Standard API**；必须核对本租户的 Connection / Agent / Mapping 模式能否使用该能力。
- 普通 Salesforce 查询通常排除软删除记录；Lookup 返回 NULL 只能视为“当前查不到”，**不能证明 `IsDeleted=TRUE`**（也可能是权限、错误 ID、永久删除或故障）。发生 Lookup 错误必须 fail closed，不得将错误视为 NULL 并继续 Delete。

## 3. 总体架构

```text
SRC_ORG_xrefLinks
       |
FLT_SFDC_Source          -- 仅 Salesforce 来源
       |
       +------------------------------+
       |                              |
       v                              v
AGG_MAX_SourcePKey                 原始 Xref 流（保留每一行）
  Group By BusinessID                   |
  MAX(SourcePKey)                       |
       |                              |
       +-----------> JNR_Xref_Max <---+
                       |
                 EXP_KeyAndFlags
                       |
                 LKP_Survivor_Active   -- Account.Id = MaxSourcePKey
                       |
                 LKP_Current_Active    -- Account.Id = SourcePKey
                       |
                 EXP_ShouldDelete
                       |
                 RTR_Delete_Exceptions
                    /          \
          DELETE_CANDIDATE      SKIP / audit / error
                 |
           TGT_SFDC_Account
              Delete (soft)
```

**Joiner 仅负责“分组结果回连原始行”**；Salesforce Active Check 使用 Lookup。任何已存在的 Root / Phone / Address 主 Egress 写回链路应与本删除流程解耦，避免错误分支影响正常 Upsert。

## 4. 字段、比较规则及去重前提

| 字段 | 推荐类型 | 规则 |
|---|---|---|
| `BusinessID` | String | 必填。聚合 Group By；不能跨 BusinessID 去重。 |
| `SourceSystem` | String | 与 MDM 实际 SFDC SourceSystem Code 精确匹配。 |
| `SourcePKey` | String(18+) | 必填；不能 `TO_DECIMAL`。建议上游统一为 Salesforce 18 位 ID。 |
| `o_SurvivorId` | String(18) | `MAX(SourcePKey)`，仅当业务正式批准此固定规则。 |
| `o_SurvivorFoundId` | String(18) | Lookup 返回的 Salesforce Account.Id；空表示未查询到。 |
| `o_CurrentFoundId` | String(18) | 另一个 Lookup 返回的 Salesforce Account.Id。 |
| `o_ShouldDelete` | Integer / Boolean | 仅在两次 Lookup 成功且当前 ID != Survivor ID 时为 true。 |

**强制校验**：仅使用一个标准化的 18 位 ID 表达式进行聚合、Lookup、比较、Target 映射；15 位 ID 转 18 位应通过可靠的标准化方式，不能简单 `UPPER()` 或猜测补位。Salesforce ID 的大小写可能有意义；确认 Lookup 的大小写比较与 ID 表达一致。

对于同一个 `BusinessID + SourceSystem + SourcePKey` 的重复完全相同行：先做去重，或在输出 Delete 前使用去重 Transformation，确保一个 Account.Id 每个 Task 至多提交一次 Delete。不要只使用 MAX 值来假设没有同键重复。

## 5. Transformation 逐步配置

### 5.1 `FLT_SFDC_Source` (Filter)

条件示例：

```text
SourceSystem = 'SFDC'
```

`'SFDC'` 是示例，替换为 MDM 实际代码。排除空 BusinessID、空 SourcePKey，异常行另行审计或报错；不要自动删除。

### 5.2 `AGG_MAX_SourcePKey` (Aggregator)

从 Filter 分出第一路，配置：

| 配置 | 值 |
|---|---|
| Group By | `BusinessID`（过滤 SFDC 后） |
| Output `o_SurvivorId` | `MAX(SourcePKey)` |
| Output `o_XrefCount` | `COUNT(SourcePKey)` |

`MAX(String)` 是 CDI 支持的聚合函数。它采用字符串排序，不可作为 Account 创建先后依据。聚合输入是 **完整 Salesforce Xref 数据**，不能先过滤已删除记录，否则 Survivor 每小时可能改变。

### 5.3 `JNR_Xref_Max` (Joiner)

Master：Aggregator 输出；Detail：来自 Filter 的原始所有 SFDC Xref 行。

| 配置 | 值 |
|---|---|
| Join Type | Normal / Inner Join |
| Join Condition | `AGG.BusinessID = DETAIL.BusinessID` |
| Output | `BusinessID`, `SourcePKey`, `o_SurvivorId`, `o_XrefCount` |

因为 Filter 已限制到同一个 SFDC 来源，按 BusinessID 即可；如果未来扩展到多个目标 Salesforce Org，聚合及 Join 必须同时包括该 Org / SourceSystem 分区键。

### 5.4 `EXP_KeyAndFlags` (Expression)

创建：

```text
o_IsSurvivor = IIF(SourcePKey = o_SurvivorId, 1, 0)
```

注意不要在此直接将 `IsDeleted` 设为 TRUE；Salesforce Target 通过 `Delete` 实现标准删除状态。

### 5.5 `LKP_Survivor_Active`（第一 Salesforce Lookup）

| 属性 | 值 |
|---|---|
| Lookup connection/object | Salesforce / Account |
| Condition | `Account.Id = o_SurvivorId` |
| Return | `Account.Id` 重命名为 `o_SurvivorFoundId` |
| Query semantics | **只查询活动/非删除记录**；在租户中验证普通 Salesforce Lookup 是否默认不包含删除记录 |
| No Match | 输出 NULL、后续 **不删除该 BusinessID 的任何 Account** |

这是**最重要的安全检查**：即使完整 Xref 里 MAX 所代表的 Account 上次已经被删除，也不能自动选“下一个 MAX”去删其余记录。

### 5.6 `LKP_Current_Active`（第二 Salesforce Lookup）

| 属性 | 值 |
|---|---|
| Lookup connection/object | Salesforce / Account |
| Condition | `Account.Id = SourcePKey` |
| Return | `Account.Id` 重命名为 `o_CurrentFoundId` |
| No Match | Skip / audit；**不发送 Delete** |

Lookup 命中证明当前普通查询可见，避免已软删除 ID 重复被删；但若因权限不可见，需要额外监控。两次 Lookup 可以重复访问 Survivor，吞吐量较低时可先接受，后续再优化。

### 5.7 `EXP_ShouldDelete` (Expression)

建议定义数值或布尔输出：

```text
o_ShouldDelete = IIF(
    NOT ISNULL(o_SurvivorFoundId)
    AND NOT ISNULL(o_CurrentFoundId)
    AND SourcePKey <> o_SurvivorId
    AND o_XrefCount > 1,
    1,
    0
)
```

表达式需按 CDI 校验器实际语法调整；上述逻辑不是让 Lookup 错误自动回退为 NULL。错误必须记录并中止该 BusinessID 处理。

可选辅助 `o_SkipReason`：

```text
IIF(ISNULL(o_SurvivorFoundId), 'SURVIVOR_NOT_ACTIVE',
  IIF(ISNULL(o_CurrentFoundId), 'CURRENT_NOT_FOUND',
    IIF(SourcePKey = o_SurvivorId, 'SURVIVOR',
      IIF(o_XrefCount <= 1, 'SINGLE_XREF', 'DELETE_CANDIDATE'))))
```

### 5.8 `RTR_Delete_Exceptions` (Router)

- `DELETE_CANDIDATE`：`o_ShouldDelete = 1`。
- `SKIP`：未命中、单条、Survivor 自身、其它异常，不进入 Salesforce Delete。
- 如连接器支持另行记录错误文件，优先启用 Task Error / Success 日志；**不要求使用数据库、表、永久文件存储**。如果企业不允许文件审计，则至少保留 Mapping Task 原生日志与监控指标。

### 5.9 `TGT_SFDC_Account` (Target)

| 属性 | 值 |
|---|---|
| Connection | Salesforce，目标 Org |
| Object | Account |
| Operation | **Delete**，不是 Update 或 Hard Delete |
| Field Mapping | `SourcePKey` → `Account.Id` |
| `IsDeleted` 字段 | **不映射** |
| 错误处理 | 对删除失败、权限失败、锁、依赖对象限制等启用可观测的错误日志与告警 |

确认 Bulk API 设置中 **Hard Delete 关闭**。普通 Delete 把记录标记为已删除（典型情况下进入 Recycle Bin）；Hard Delete 无法按软删除恢复。

## 6. 分组示例与重复运行

完整 Xref（其中 SAP 不参与）：

| BusinessID | SourceSystem | SourcePKey | 初始 Salesforce 状态 |
|---|---|---|---|
| B001 | SFDC | 001AAA | Active |
| B001 | SFDC | 001BBB | Active |
| B001 | SFDC | 001CCC | Active |
| B001 | SAP | SAP100 | 无关 |
| B002 | SFDC | 001DDD | Active |

示意 ID 为简写；真实环境需要有效 18 位 ID。

B001 中 `MAX(SourcePKey) = 001CCC`。第一次执行删除 001AAA、001BBB，保留 001CCC。第二次执行，MAX 仍来自**完整 Xref** 且仍为 001CCC；Survivor Lookup 命中，001AAA/001BBB 的 Current Lookup 不命中，因此不再提交 Delete。B002 只有一条，保留。

**异常保护**：如果 001CCC 被手工删掉，而 B001 的 Xref 仍包含它，Survivor Lookup 查不到，B001 的所有 Delete 都被禁止，并发出告警。不会自动把 001BBB 升格为 Survivor。

## 7. 不使用数据库表时无法自动消除的风险

1. **Xref 新增改变 MAX**：第一次 MAX 为 ID A；若新 Xref 加入一个更大的 ID B，下次 MAX 变成 B，可能把原保留 A 删除。双 Lookup 无法检测“之前的 Survivor 是 A”。**此时需要业务承诺 Xref 的 Survivor 规则/成员不会无审批变更，或改用 MDM 权威 Survivor 属性。**
2. **Xref 从源端移除**：如果原 MAX 不再出现在本次 Xref 中，也可能改选次大 ID。
3. **MDM 合并/拆分**：BusinessID 合并会改变分组，必须进行特殊评审。
4. **已删除后再 Undelete**：下一次可能被重新删除；如不希望如此，需要业务级恢复豁免信息（现有三个字段不能表达）。
5. **SFDC Lookup 请求失败/部分失败**：必须停止发送 Delete，不可降级为简单 MAX 删其余记录。
6. **Lookup 与 Delete 间竞争条件**：查询后有人可能修改/删除/恢复记录；应确保 Mapping Task 不重叠运行并监控 Salesforce 端执行结果。
7. **多个任务并行修改同一 Account**：必须协调 Egress 普通 Account 写回与 Account Delete 的执行顺序，避免后续 Upsert 重建或写回已删除 ID。
8. **引用与业务影响**：Account 删除可能影响 Contact、Opportunity、共享规则、自动化以及 CDC / MDM Ingress；业务必须评审。

**上线 Gate（必须满足）：**

- [ ] 数据治理/业务所有者批准 `MAX(SourcePKey)` 决策及“Xref 集合变化可能改选 Survivor”的风险，或提供 MDM 权威 Survivor 标识。
- [ ] 确认 Salesforce Lookup 使用 Standard API，普通查询不会返回已删除记录；校验连接器缓存和大小写匹配。
- [ ] 确认所有 SourcePKey 是同一个 Salesforce Org 中可匹配的完整 Account.Id，并规范 15/18 位处理。
- [ ] 运行 Dry Run：`DELETE_CANDIDATE` 输出先审阅，不执行 Target Delete。
- [ ] 业务批准删除历史和关联对象处理策略；禁止 Hard Delete。
- [ ] Mapping Task 不允许重叠执行；错误即终止或跳过，不得“失败继续删”。

## 8. 测试计划（Sandbox）

| 用例 | 输入/状态 | 预期 |
|---|---|---|
| 单一 SFDC ID | 一个 BusinessID 只有一条 | 0 个 Delete |
| 多条 SFDC ID | 3 条有效，同 BusinessID | 仅非 MAX 的 2 条 Delete |
| 跨来源 | B001 同时有 SFDC / SAP / Oracle | 仅 SFDC 参与计算和删除 |
| 第二次运行 | 上轮已删除非 MAX | 0 个重复 Delete |
| Survivor 不可见/已删 | MAX ID 的 Lookup 无结果 | 整组 0 个 Delete + 异常 |
| 某待删 ID 不可见 | Current Lookup 无结果 | 对该 ID 不提交 Delete |
| 同 ID 重复行 | 两条完全相同 Xref | 不允许重复 Delete 同一 ID |
| Lookup API 故障 | 超时/错误/限流 | 中止或安全跳过；无误删 |
| 新增更大的 ID | Xref 新增更大的 SourcePKey | **风险用例：识别 Survivor 漂移并禁止直接生产自动上线** |
| ID 15/18 位混用 | 两种格式 | 完整标准化，否则不执行 Delete |
| Salesforce 恢复 | 已删除 Account 被 Undelete | 与业务策略一致；无豁免机制时可能再次 Delete |

## 9. 运维和每小时调度

- CDI Mapping Task 配置每小时运行；确保执行窗口不重叠。
- 监控 Xref 总数、SFDC Xref 数、每组最大值、待删除数、实际删除成功/失败数、Survivor 缺失组数、Current Lookup 未命中数。
- 对 `SURVIVOR_NOT_ACTIVE`、异常多的 Delete 候选、突增失败数触发告警。
- 上线先限制小批量并采用 Dry Run；通过 Salesforce Sandbox 完整重跑两次确认幂等性。
- Salesforce 删除属于 **软删除**；不要启用 Hard Delete。

## 10. 英文官方技术参考（2026-10-26 检索核验）

1. Informatica [Aggregate functions](https://docs.informatica.com/integration-cloud/data-integration/current-version/function-reference/functions/function_overview/aggregate_functions.html)：确认 `MAX(String)` 支持。
2. Informatica [Salesforce Connector Guide](https://docs.informatica.com/content/dam/source/GUID-F/GUID-F3409595-C41F-486D-ACD0-D4C59E670B4D/8/en/CDI__SalesforceConnectorGuide_en.pdf)：说明 Mapping Salesforce Lookup，且 Lookup 仅适用于 Salesforce Standard API。**注意此 PDF 为历史版本；运行前以租户当前版本为准。**
3. Informatica [Salesforce field mapping / Delete ID](https://docs.informatica.com/integration-cloud/cloud-data-integration-connectors/current-version/salesforce-connector/synchronization-tasks-with-salesforce/field-mappings-in-salesforce-synchronization-tasks.html)：Delete 需要匹配目标 ID；该章节描述 Synchronization Task，Mapping Target 仍须在当前连接器中验证。
4. Informatica [Salesforce Connector / Bulk API and hard delete](https://docs.informatica.com/content/dam/source/GUID-F/GUID-F3409595-C41F-486D-ACD0-D4C59E670B4D/23/en/CDI__SalesforceConnector_en.pdf)：区别软删除与 Hard Delete。
5. Salesforce [QueryAll REST API](https://developer.salesforce.com/docs/platform/api-rest/guide/resources-queryall.html)：可读取仍保留的软删除记录；永久清除后不可查询。
6. Salesforce [Query vs QueryAll](https://developer.salesforce.com/docs/platform/api-rest/guide/dome-queryall.html)：普通 Query 自动排除删除记录。

---

### 最终决策

**当前最简可实现方案**：`FLT_SFDC_Source → AGG(MAX SourcePKey) + Joiner(回连所有 Xref) → LKP_Survivor_Active → LKP_Current_Active → EXP/Router → Salesforce Account Delete`。不使用任何数据库控制表；使用 Salesforce Lookup 做每小时幂等过滤；保留完整 Xref 决定固定候选。

**但它不是严格意义上永久稳定的 Survivor 管理系统**：如需处理任意 Xref 变化时仍维持历史选择，必须引入 MDM 权威 Survivor 规则或某种持久化的业务状态。没有这些信息，不能安全声称实现了完全自动且永久稳定的去重。
