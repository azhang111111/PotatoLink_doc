# PotatoLink 中俄薯贸

PotatoLink 是面向俄罗斯市场的马铃薯贸易数字化项目，由佳秀农业发起。平台同时经营商品薯与种薯，通过微信小程序、俄语 H5 和统一管理后台，连接产品展示、智能选品、在线询价、批次溯源与订单协同。

> 当前仓库只存放业务、产品和项目管理文档。前后端代码仓库已经建立，待需求确认后开始开发。

代码仓库已经创建：

- 前端：[PotatoLink_frontend](https://github.com/azhang111111/PotatoLink_frontend)
- 后端：[PotatoLink_backend](https://github.com/azhang111111/PotatoLink_backend)

## 当前状态

- 项目名称：PotatoLink 中俄薯贸
- 业务范围：商品薯与种薯同时经营
- 主要市场：俄罗斯；架构预留哈萨克斯坦等市场
- 当前阶段：需求与技术设计评审
- 第一版目标：形成“展示—选品—询价—报价—批次—订单—交付”的最小业务闭环
- 微信小程序：佳秀农业企业主体申请正在推进

## 文档目录

| 文档 | 内容 |
|---|---|
| [项目概述](docs/00-project-overview.md) | 项目定位、目标、边界与核心原则 |
| [业务需求](docs/01-business-requirements.md) | 商品薯、种薯及跨境贸易业务规则 |
| [产品需求 PRD](docs/02-product-requirements-prd.md) | 三端功能、页面和验收标准 |
| [用户与业务流程](docs/03-user-roles-and-flows.md) | 用户角色、询价、报价、批次与订单流程 |
| [信息架构](docs/04-information-architecture.md) | 小程序、俄语 H5 和后台导航结构 |
| [领域数据模型](docs/05-domain-data-model.md) | 核心实体、字段和状态设计 |
| [MVP 路线图](docs/06-mvp-roadmap.md) | 开发阶段、优先级和里程碑 |
| [内容资料清单](docs/07-content-checklist.md) | 开发和上线需要准备的业务资料 |
| [待确认事项](docs/08-open-questions.md) | 需要业务方决策的问题及确认记录 |
| [中俄术语表](docs/09-zh-ru-glossary.md) | 品种、贸易和产品界面的双语术语草案 |
| [品牌资产与使用方向](docs/10-brand-assets.md) | 佳秀农业现有 Logo、PotatoLink 联合品牌及后续设计要求 |
| [隐私与合规参考](docs/11-compliance-references.md) | 中俄个人信息保护参考和上线复核边界 |
| [技术架构基线](docs/12-technical-architecture.md) | 前后端分离、前后端技术栈及模块边界 |
| [数据库与状态机设计](docs/13-database-and-state-machine.md) | 核心表、数据约束、库存并发和业务状态流转 |
| [API 契约与错误码](docs/14-api-contract-and-error-codes.md) | REST API 分组、接口清单、通用协议和错误码 |

## 已确认的产品决策

1. 产品名称使用“PotatoLink 中俄薯贸”。
2. 商品薯与种薯同时经营，但采用独立产品字段、询价流程和溯源内容。
3. 微信小程序不是唯一客户入口；俄罗斯采购商主要使用俄语 H5。
4. 第一版采用 B2B 询价与人工报价，不做直接在线支付。
5. 第一版服务佳秀农业自营业务，不开放第三方供应商入驻。
6. 批次、报告和证书只有经过后台审核后才能显示“已验证”。
7. 第一版智能选品采用可解释的规则匹配，不依赖 AI 模型。
8. 企业法律全称为“张北县佳秀农业开发有限公司”，对外简称“佳秀农业”。
9. 系统采用前后端分离架构；后端基于 Java 17、Spring Boot 3.5 和模块化单体架构建设。
10. 微信小程序与俄语 H5 使用 uni-app、Vue 3 和 TypeScript；管理后台使用 Vue 3、TypeScript、Vite 和 Element Plus。
11. 前后端分别使用 `PotatoLink_frontend` 和 `PotatoLink_backend` 两个独立 GitHub 仓库。
12. 前后端代码仓库均为私有仓库，当前由项目方自行开发，暂不添加外部开发协作者。
13. 本地开发基础设施使用 Docker Compose，统一运行 PostgreSQL、Redis、MinIO 和测试邮件服务；测试与生产云资源后续确定。
14. 所有业务表主键统一使用 19 位雪花 ID；接口中的 ID 一律按字符串传输，避免前端精度丢失。

## 变更管理

- 业务决策先记录到 `docs/08-open-questions.md`。
- 确认后同步修改相关业务文档、PRD 和数据模型。
- 进入开发后，代码仓库中的接口与实现文档应引用本仓库对应版本。
- 影响范围较大的需求变更应保留决策日期、决策人和原因。
