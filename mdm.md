# 金融行业 MDM 与用户画像参考资料

## 一、先明确项目边界：MDM 不等于用户画像

金融客户数据项目建议拆成三个相互连接、但职责不同的能力：

| 能力 | 主要回答的问题 | 典型输出 |
|---|---|---|
| Customer MDM | 这个客户究竟是谁？ | 统一客户 ID、Golden Record、自然人/企业/家庭/关联方关系、数据来源和生效时间 |
| Customer 360 / Profile | 这个客户现在是什么状态？ | 产品持有、资产负债、交易行为、渠道行为、生命周期、客户价值、风险状态 |
| Analytics / Decisioning | 下一步应采取什么行动？ | 客群划分、流失预测、Next Best Action、营销推荐、反欺诈、AML 预警 |

MDM 的核心是匹配、去重、合并和治理客户主记录；完整的 Customer 360 还需要数据采集、统一、分析、激活和治理。

因此，不建议把交易流水、实时点击、模型分数全部直接放入 MDM 主记录，而应通过统一客户 ID 连接画像库、特征库和明细数据平台。

参考：

- [IBM — What is Master Data Management?](https://www.ibm.com/think/topics/master-data-management)

---

## 二、优先推荐的专业书籍

### 1. 数据治理与 MDM 体系基础

| 推荐书籍 | 推荐理由 |
|---|---|
| [DAMA-DMBOK: Data Management Body of Knowledge](https://dama.org/learning-resources/dama-data-management-body-of-knowledge-dmbok/) | 建议作为项目总框架。重点阅读 Data Governance、Data Quality、Metadata、Reference & Master Data、Data Architecture 等章节。 |
| [Master Data Management and Data Governance, 2nd Edition](https://www.oreilly.com/library/view/master-data-management/9780071744584/) — Alex Berson、Larry Dubov | 系统讲解 MDM、数据治理、客户数据整合、架构和实施方法。虽然出版时间较早，但概念体系仍适合作为入门教材。 |
| [Master Data Management in Practice: Achieving True Customer MDM](https://onlinelibrary.wiley.com/doi/book/10.1002/9781118269053) — Dalton Cervo、Mark Allen | 对实际项目非常实用，覆盖客户 MDM 立项、Owner 和 Data Steward、数据质量、访问管理、指标体系、Customer 360 和组织变革。 |
| [Enterprise Master Data Management: An SOA Approach to Managing Core Information](https://www.amazon.com/Enterprise-Master-Data-Management-Information/dp/0132366258) — Allen Dreibelbis 等 | 适合数据架构师阅读，重点理解 MDM Hub、服务化、业务系统与主数据平台的交互模式。 |

### 2. 客户匹配、去重和实体解析

| 推荐书籍 | 推荐理由 |
|---|---|
| [Entity Resolution and Information Quality](https://shop.elsevier.com/books/entity-resolution-and-information-quality/talburt/978-0-12-381972-7) — John R. Talburt | 金融 Customer MDM 的关键技术书。重点学习确定性匹配、概率匹配、Fellegi–Sunter 模型、身份解析、关联分析和信息质量。 |
| [Multi-Domain Master Data Management](https://books.google.com/books/about/Multi_Domain_Master_Data_Management.html?id=y-ScBAAAQBAJ) — Mark Allen、Dalton Cervo | 适合项目从客户主数据扩展到机构、产品、账户、渠道、员工、合作方和参考数据时阅读。 |

### 3. Customer 360、画像和分析模型

| 推荐书籍 | 推荐理由 |
|---|---|
| [Customer 360: How Data, AI, and Trust Change Everything](https://onlinelibrary.wiley.com/doi/book/10.1002/9781394308668) — Martin Kihn、Andrea Chen Lin | 连接 Customer 360、AI、隐私、信任、组织和技术平台，适合项目负责人、产品负责人和数据负责人。 |
| [Customer Data Platforms: Use People Data to Transform the Future of Marketing Engagement](https://books.google.com/books?q=Customer+Data+Platforms+Martin+Kihn+Christopher+O%27Hara) — Martin Kihn、Christopher B. O’Hara | 帮助理解 CDP、身份统一、客户分群和触达激活。需要注意，CDP 通常不能替代金融机构的操作型 MDM、KYC 主档和数据治理。 |
| [The Data Warehouse Toolkit, 3rd Edition](https://www.kimballgroup.com/data-warehouse-business-intelligence-resources/books/data-warehouse-dw-toolkit/) — Ralph Kimball、Margy Ross | 用于设计画像分析模型。重点关注客户维度、账户与交易事实表、桥接表、多值属性、层级、快照以及 Slowly Changing Dimensions。 |

### 建议阅读顺序

1. DAMA-DMBOK
2. Master Data Management in Practice
3. Entity Resolution and Information Quality
4. The Data Warehouse Toolkit
5. Customer 360
6. Customer Data Platforms

这样可以避免一开始就被某个厂商的产品功能带偏。

---

## 三、推荐文章与架构资料

### MDM 基础

1. [IBM — What is Master Data Management?](https://www.ibm.com/think/topics/master-data-management)  
   适合快速建立 MDM、主数据域、匹配合并、治理和 Golden Record 的基本认识。

2. [Informatica — Master Data Management](https://www.informatica.com/products/master-data-management.html)  
   可了解多域 MDM、统一主记录、数据质量以及 MDM 和业务系统之间的关系。

3. [Informatica — Customer 360](https://www.informatica.com/products/master-data-management/customer-360.html)  
   重点关注客户数据匹配、合并、治理、关系管理和统一视图。

### Customer 360 与用户画像架构

4. [AWS — Create an end-to-end data strategy for Customer 360](https://aws.amazon.com/blogs/big-data/create-an-end-to-end-data-strategy-for-customer-360-on-aws/)  
   很适合用来设计项目蓝图。文章将 Customer 360 拆分为数据采集、统一、分析、激活和数据治理五个部分，并提供参考架构。

5. [Google Cloud — Unlocking personalization for financial services customers](https://cloud.google.com/transform/customer-data-platform-financial-services-value-personalization-privacy)  
   专门讨论金融机构如何统一分散的客户数据、进行智能分群，同时处理安全、隐私和个性化之间的关系。

6. [Snowflake — Customer 360 in Financial Services](https://www.snowflake.com/en/solutions/industries/financial-services/customer-360-in-financial-services/)  
   关注金融 Customer 360 在获客、产品适配、投资规划、Next Best Action 和个性化方面的应用。

7. [Databricks — Data Management in Financial Services](https://www.databricks.com/resources/ebook/s/data-management-in-financial-services)  
   可参考金融机构如何统一主数据、实时数据和分析数据，建立可信数据基础。

### 金融机构案例

8. [Citizens Bank — Cloud MDM and single customer view](https://www.informatica.com/customer-success-stories/citizens.html)  
   案例重点是银行如何建立单一客户视图、支持实时个性化以及缩短数据接入周期。

9. [Google Cloud — Comprehensive customer financial profiles](https://cloud.google.com/blog/products/data-analytics/build-comprehensive-customer-financial-profiles-with-elastic-cloud-and-google-cloud)  
   展示如何把大量银行交易转化为客户资金、商户、收入支出和异常行为等画像信息。

10. [Google Cloud — Macquarie Bank case study](https://cloud.google.com/customers/macquarie-bank)  
    可参考数字银行如何利用数据和云平台提供更相关、个性化的金融服务。

---

## 四、推荐视频和 Webinar

1. [What is Master Data Management?](https://www.youtube.com/watch?v=l83bkKJh1wM)  
   快速理解 MDM 如何形成个人、地点或事物的统一视图。

2. [Informatica — What is Master Data Management?](https://video.informatica.com/detail/video/5281221321001/what-is-master-data-management)  
   从业务和 IT 协作、准确性、Data Stewardship 和语义一致性角度解释 MDM。

3. [Databricks — Reshaping Retail Banking with Personalization](https://www.databricks.com/resources/webinar/reshaping-retail-banking-with-personalization)  
   直接对应零售银行客户画像，涵盖交易数据、人口属性、Customer 360、CLV 和个性化。

4. [Snowflake — Delivering Personalized Banking Experiences with Customer 360](https://www.snowflake.com/en/resources/podcast/delivering-personalized-experiences-with-customer-360-degree-views-7/)  
   介绍零售银行如何打通数据孤岛并建立 Customer 360。

5. [How Techcombank Scales AI Banking to 16M Customers](https://www.youtube.com/watch?v=keMlAuCerCE&vl=en)  
   金融机构数据平台、规模化个性化和 AI 银行实践案例。

6. [Databricks for Financial Services — Video Playlist](https://www.youtube.com/playlist?list=PLTPXxbhUt-YWiWhb0wmfjW9uS0v-59e1T)  
   包含银行个性化、数据产品、金融犯罪、治理和客户分析案例。

---

## 五、金融用户画像建议的数据域

建议不要从“要做多少个标签”开始，而是先建立以下数据域。

### 1. 身份与 KYC 主档

- 统一客户 ID
- 姓名、证件、联系方式和地址
- 客户类型及客户状态
- KYC 等级、身份核验状态
- 数据来源、可信度、生效和失效时间
- 合并、拆分和人工复核记录

### 2. 客户关系图谱

- 自然人与企业关系
- 家庭、共同地址和共同联系方式
- 法人、实际控制人、受益所有人
- 代理人、授权人和联系人
- 账户、银行卡、合同、设备与客户之间的关系

关系图谱不能仅凭“相同手机号或地址”自动判定为同一客户；需要区分“同一实体”和“存在关联关系”。

### 3. 产品与资产负债画像

- 存款、贷款、信用卡、理财、保险和证券持有
- 余额、额度、期限、收益和成本
- 产品组合、交叉持有和产品渗透率
- 资产管理规模、负债规模和净资产估算

### 4. 交易行为画像

- Recency、Frequency、Monetary
- 收入和支出稳定性
- 商户类别、交易渠道和时间分布
- 资金流入流出特征
- 异常交易和行为变化

在部分投资客户分群场景中，交易频率和交易量等行为变量可能比静态 KYC 属性更能解释实际行为，因此画像不应只依赖客户基本属性。

参考：

- [Behavioral variables for customer segmentation](https://arxiv.org/abs/2005.03625)

### 5. 数字渠道与服务画像

- App、Web、柜面、客服和客户经理互动
- 登录、浏览、搜索、申请和放弃行为
- 投诉、咨询、服务满意度
- 渠道偏好及触达响应

### 6. 风险与合规画像

- 客户风险等级
- 欺诈和 AML 风险信号
- 信用风险及还款行为
- 制裁、PEP 和负面信息匹配状态
- 模型版本、评分时间和决策依据

### 7. 客户价值与生命周期

- 获客渠道和获客成本
- 生命周期阶段
- 收入贡献和服务成本
- CLV、流失概率和活跃度
- Next Best Product / Next Best Action

### 8. 授权、隐私和使用限制

- 营销授权
- 数据共享授权
- 允许使用的目的
- 数据保留期限
- 敏感属性使用限制
- 自动决策和人工复核要求

---

## 六、金融行业必须同时参考的治理资料

1. [World Economic Forum — Appropriate Use of Customer Data in Financial Services](https://www.weforum.org/publications/the-appropriate-use-of-customer-data-in-financial-services/)  
   讨论金融客户数据使用中的客户控制、安全、个性化、高级分析和数据可携带性。

2. [FATF — Guidance on Digital Identity](https://www.fatf-gafi.org/en/publications/Financialinclusionandnpoissues/Digital-identity-guidance.html)  
   适合设计数字身份、客户身份核验、CDD 和持续尽职调查相关主数据。

3. [BIS — BCBS 239 Executive Summary](https://www.bis.org/fsi/fsisummaries/rdarr.htm)  
   银行项目应关注风险数据的准确性、完整性、及时性、可追溯性和聚合能力。

这些属于国际参考框架，不能替代公司所在司法辖区对个人信息、银行监管、征信、营销、模型风险和自动化决策的专项法律审查。

---

## 七、推荐的项目技术分层

```text
核心银行 / CRM / 信贷 / 卡系统 / 理财 / 保险 / App / 客服 / 外部 KYC
                         │
                  数据标准化与质量校验
                         │
        Customer MDM / Identity Resolution Hub
      统一客户 ID、Golden Record、关系、合并拆分、审计
                         │
         ┌───────────────┴───────────────┐
         │                               │
交易明细、行为事件、历史快照        Profile / Feature Layer
Data Lake / Warehouse             标签、指标、特征、模型分数
         │                               │
         └───────────────┬───────────────┘
                         │
         Customer 360 API / Profile Service
                         │
营销、客户经理、客服、风控、AML、推荐、经营分析
```

其中：

- **MDM 保存相对稳定、可治理的客户事实。**
- **画像层保存可重新计算、有明确时间窗口的标签和指标。**
- **特征层保存模型训练与推理使用的特征。**
- **明细平台保留交易和行为事件，避免 MDM 膨胀。**
- 所有标签和模型分数都应带有计算时间、观察窗口、规则或模型版本、数据来源、责任人和允许用途。

---

## 八、优先阅读建议

最值得优先阅读的三项是：

1. **DAMA-DMBOK**
2. **Master Data Management in Practice**
3. **AWS Customer 360 架构文章**

它们分别解决：

- 数据治理总体框架
- MDM 项目落地方法
- Customer 360 技术分层与架构设计
