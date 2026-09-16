# 14 API 契约与错误码

本文档定义 PotatoLink MVP 的 REST API 边界、通用数据格式、首批接口和稳定错误码。机器可读 OpenAPI 3.1 草案位于 [`api/openapi-v1.yaml`](../api/openapi-v1.yaml)。

| 项目 | 内容 |
|---|---|
| 版本 | v0.1 |
| 状态 | 技术设计草案，工程初始化前复核 |
| 基础路径 | `/api/v1` |
| 数据格式 | HTTPS + JSON，文件流接口除外 |

## 1. 接口边界

| 前缀 | 使用方 | 身份要求 | 可访问内容 |
|---|---|---|---|
| `/public` | 小程序、俄语 H5、公开分享页 | 默认无需登录 | 企业、基地、品种、公开供应和批次摘要 |
| `/auth` | 小程序、H5、管理后台 | 按接口要求 | 微信登录、邮箱验证码、内部登录和刷新令牌 |
| `/secure` | 报价/订单客户 | 限时安全令牌，必要时二次验证 | 指定客户的报价、订单和授权文件 |
| `/admin` | 管理后台、内部销售小程序能力 | Bearer Token + 权限 | 客户、供应、询价、报价、订单、文件、配置和报表 |

公开、客户私有和内部管理模型分别定义。后端不得因为字段名称相同就直接把数据库实体作为 API 响应返回。

## 2. 通用协议

### 2.1 字段与精度

- JSON 字段使用 `camelCase`。
- 19 位雪花 ID 在请求、响应和 OpenAPI 中统一为字符串，格式为 `^[1-9][0-9]{18}$`。
- 金额、单价、重量和其他需要精确计算的小数统一按十进制字符串传输；后端转换为 `BigDecimal`。
- 时间使用 ISO 8601 并带时区，例如 `2026-09-15T14:30:00+08:00`。
- 日期使用 `YYYY-MM-DD`。
- 币种使用 ISO 4217 代码，MVP 默认 `CNY`。
- 国家使用 ISO 3166-1 alpha-2 代码，例如 `CN`、`RU`、`KZ`。
- 语言使用 BCP 47 标签，例如 `zh-CN`、`ru-RU`。

### 2.2 成功响应

```json
{
  "code": "OK",
  "message": "success",
  "data": {},
  "requestId": "01J..."
}
```

创建成功通常返回 HTTP `201`，普通成功返回 `200`，无响应体的删除或取消操作可以返回 `204`。文件下载和二维码图片直接返回文件流，不套响应信封。

### 2.3 错误响应

```json
{
  "code": "VALIDATION_FAILED",
  "message": "Request validation failed",
  "requestId": "01J...",
  "details": [
    {
      "field": "items[0].quantityKg",
      "reason": "must be greater than zero"
    }
  ]
}
```

`message` 用于面向当前语言的可读提示，客户端业务判断只能依赖稳定的 `code`。生产环境不返回堆栈、SQL、内部类名和第三方密钥信息。

### 2.4 分页

MVP 管理列表使用页码分页：

- 请求：`page` 从 1 开始，`pageSize` 默认 20、最大 100。
- 响应：`items`、`page`、`pageSize`、`total`。
- 排序仅允许接口声明的白名单字段，禁止把客户端字符串直接拼接到 SQL。

公开供应和品种列表也使用相同结构。大数据量审计和事件列表以后可增加游标分页，不改变现有接口。

### 2.5 幂等

以下写操作必须携带 `Idempotency-Key`：

- 游客提交询价。
- 客户接受报价。
- 创建订单或样品申请。
- 确认、提交、释放或履约库存占用。
- 触发文件导出。

同一调用方、同一接口和同一幂等键重复请求时，后端返回第一次成功结果；请求体不同则返回 `IDEMPOTENCY_KEY_CONFLICT`。幂等记录默认保存 24 小时，交易创建类记录可保存更长时间。

### 2.6 请求追踪与语言

- 客户端可以传 `X-Request-Id`；缺失时由网关或后端生成。
- 所有响应返回 `X-Request-Id` 并在响应体提供 `requestId`。
- 客户端通过 `Accept-Language` 请求中文或俄文；不支持的语言回退到系统默认语言。
- 时间判断使用服务端时间，客户端倒计时只作展示。

## 3. 认证和访问控制

### 3.1 内部账号

管理后台登录成功后获得短期访问令牌和可刷新令牌。每个 `/admin` 接口同时校验账号状态、角色权限和必要的数据范围。

### 3.2 微信客户

小程序把微信临时登录 `code` 发送给 `POST /auth/wechat/login`。后端与微信服务交换身份后创建或关联客户，不接收前端自行声明的 `openid` 作为可信身份。

### 3.3 俄语 H5

游客可浏览并提交询价。报价和订单通过限时安全链接进入；链接令牌通过请求头 `X-Secure-Access-Token` 传递。需要二次验证时，通过邮箱验证码换取短期访问会话。

安全令牌必须使用密码学安全随机值，数据库只保存哈希。雪花 ID、报价编号和订单编号均不能替代授权令牌。

## 4. 公开接口

| 方法 | 路径 | 用途 |
|---|---|---|
| GET | `/public/company` | 企业品牌、简介和公开联系方式 |
| GET | `/public/bases` | 已发布基地列表 |
| GET | `/public/bases/{id}` | 基地详情和公开媒体 |
| GET | `/public/varieties` | 品种列表与筛选 |
| GET | `/public/varieties/{id}` | 品种详情、优缺点和适用用途 |
| GET | `/public/supply-batches` | 公开供应列表，不含精确库存和内部价格 |
| GET | `/public/supply-batches/{id}` | 公开批次详情 |
| GET | `/public/trace/{batchNo}` | 批次公开溯源摘要 |
| POST | `/public/inquiries` | 游客或已识别客户提交询价 |
| POST | `/public/sample-requests` | 提交样品申请 |
| POST | `/public/recommendations/commodity` | 商品薯规则选品 |
| POST | `/public/recommendations/seed` | 种薯规则选品 |

公开供应响应只返回人工维护的供应状态和允许展示的数量区间。`exactAvailableKg`、指导价、成本、内部备注和未审核文件不属于公开模型。

询价请求按 `businessType` 区分为商品薯和种薯两个 schema。商品薯必须提供用途、规格、水洗状态、包装、数量和交期；种薯必须提供种植国家/地区、面积、土壤、灌溉、重茬、播种/采收计划、规格、数量和交期。品种字段允许选择具体品种或明确选择“需要推荐”，二者至少满足一项。

## 5. 认证与安全访问接口

| 方法 | 路径 | 用途 |
|---|---|---|
| POST | `/auth/wechat/login` | 微信临时 code 登录或关联客户 |
| POST | `/auth/admin/login` | 内部账号登录 |
| POST | `/auth/token/refresh` | 刷新内部访问令牌 |
| POST | `/auth/email-codes` | 向安全链接关联邮箱发送验证码 |
| POST | `/auth/email-codes/verify` | 验证后建立短期安全会话 |
| GET | `/secure/quotations/{quotationNo}` | 查看指定报价当前授权版本 |
| POST | `/secure/quotation-versions/{id}/accept` | 接受有效报价并记录客户反馈；待订单模块交付后再自动创建待锁定订单 |
| POST | `/secure/quotation-versions/{id}/reject` | 拒绝报价 |
| POST | `/secure/quotation-versions/{id}/request-revision` | 请求修改报价 |
| GET | `/secure/orders/{orderNo}` | 查看授权订单和公开履约节点 |
| GET | `/secure/files/{id}` | 获取授权文件的限时下载地址或文件流 |

客户接受报价时，后端必须重新校验版本状态、有效期、客户授权和幂等键。当前阶段先将报价版本更新为 `ACCEPTED` 并记录反馈和审计；订单模块交付后，再由同一动作原子创建 `PENDING_LOCK` 订单及 `PENDING_CONFIRMATION` 占用任务，不直接扣减库存。

## 6. 管理接口

### 6.1 内容与供应

| 方法 | 路径 | 用途 |
|---|---|---|
| GET/POST | `/admin/bases` | 查询或创建基地 |
| GET/PUT | `/admin/bases/{id}` | 基地详情或修改 |
| GET/POST | `/admin/varieties` | 查询或创建品种 |
| GET/PUT | `/admin/varieties/{id}` | 品种详情或修改 |
| POST | `/admin/varieties/{id}/submit-review` | 提交内容审核 |
| POST | `/admin/varieties/{id}/publish` | 发布审核通过内容 |
| GET/POST | `/admin/supply-batches` | 查询或创建供应批次 |
| GET/PUT | `/admin/supply-batches/{id}` | 批次详情或修改 |
| POST | `/admin/supply-batches/{id}/submit-review` | 提交批次审核 |
| POST | `/admin/supply-batches/{id}/publish` | 发布批次 |
| POST | `/admin/supply-batches/{id}/unpublish` | 暂停或取消公开 |
| POST | `/admin/supply-batches/{id}/inventory-adjustments` | 受控库存调整并写流水 |

### 6.2 客户与询价

| 方法 | 路径 | 用途 |
|---|---|---|
| GET | `/admin/customers` | 客户列表 |
| GET/PUT | `/admin/customers/{id}` | 客户详情或更新 |
| POST | `/admin/customers/{id}/transfer` | 转移负责人并留痕 |
| POST | `/admin/customers/merge` | 合并重复客户 |
| GET | `/admin/inquiries` | 询价列表、负责人和 SLA 筛选 |
| GET | `/admin/inquiries/{id}` | 询价、明细、客户和跟进详情 |
| POST | `/admin/inquiries/{id}/assign` | 分配主负责人和备用负责人 |
| POST | `/admin/inquiries/{id}/follow-ups` | 添加有效跟进 |
| POST | `/admin/inquiries/{id}/confirm-requirements` | 确认需求 |
| POST | `/admin/inquiries/{id}/mark-lost` | 标记未成交及原因 |
| POST | `/admin/inquiries/{id}/reopen` | 按权限重新打开 |

### 6.3 报价

| 方法 | 路径 | 用途 |
|---|---|---|
| POST | `/admin/inquiries/{id}/quotations` | 从询价创建报价 |
| GET | `/admin/quotations/{id}` | 报价当前版本详情 |
| GET | `/admin/quotations/{id}/versions` | 报价全部历史版本 |
| POST | `/admin/quotations/{id}/versions` | 从驳回或客户调整请求创建新版本 |
| POST | `/admin/quotations/{id}/approval` | 审批通过或驳回当前版本 |
| POST | `/admin/quotations/{id}/send` | 标记当前版本已发送 |
| POST | `/admin/quotations/{id}/secure-links` | 撤销旧链接并生成新的限时客户链接 |
| POST | `/admin/quotations/{id}/secure-links/revoke` | 撤销该报价的有效客户链接 |
| GET | `/admin/quotation-versions/{id}/pdf` | 获取报价 PDF |

### 6.4 库存与订单

| 方法 | 路径 | 用途 |
|---|---|---|
| GET | `/admin/inventory-reservations` | 待确认、临时锁定和正式占用列表 |
| POST | `/admin/inventory-reservations/{id}/confirm-temp-lock` | 原子确认临时锁定 |
| POST | `/admin/inventory-reservations/{id}/commit` | 合同或定金确认后转正式占用 |
| POST | `/admin/inventory-reservations/{id}/release` | 释放未履约占用 |
| POST | `/admin/inventory-reservations/{id}/fulfill` | 出库并扣减实物库存 |
| GET | `/admin/orders` | 订单列表 |
| GET | `/admin/orders/{id}` | 订单、分配、合同和履约详情 |
| POST | `/admin/orders/{id}/confirm` | 确认合同/定金及订单 |
| POST | `/admin/orders/{id}/suspend` | 暂停订单并记录原因 |
| POST | `/admin/orders/{id}/cancel` | 取消并释放未履约库存 |
| POST | `/admin/orders/{id}/events` | 新增履约节点 |

### 6.5 样品、文件和运营

| 方法 | 路径 | 用途 |
|---|---|---|
| GET | `/admin/sample-requests` | 样品申请列表 |
| GET | `/admin/sample-requests/{id}` | 样品详情 |
| POST | `/admin/sample-requests/{id}/reviews` | 销售、质量或合规审核 |
| POST | `/admin/sample-requests/{id}/dispatch` | 扣减批次并登记发出 |
| POST | `/admin/sample-requests/{id}/receive` | 登记签收 |
| POST | `/admin/sample-requests/{id}/follow-ups` | 登记回访 |
| POST | `/admin/files/uploads` | 获取受控上传凭证或上传文件 |
| POST | `/admin/batches/{id}/documents` | 关联批次文件 |
| POST | `/admin/documents/{id}/review` | 审核、驳回或撤销文件 |
| GET/POST | `/admin/routes` | 查询或创建运输路线 |
| GET/PUT | `/admin/routes/{id}` | 路线版本详情或修改 |
| GET | `/admin/reports/overview` | 管理看板指标 |
| POST | `/admin/exports` | 创建受控异步导出任务 |
| GET | `/admin/exports/{id}` | 查询导出状态和授权下载 |

## 7. 状态修改原则

- 普通 `PUT` 仅修改处于可编辑状态的资源字段。
- 审批、发布、发送、接受、锁定、释放、取消等业务动作使用明确的 `POST .../{action}` 接口。
- 动作接口由后端检查当前状态和允许的目标状态，客户端不能提交任意 `status`。
- 冲突返回 HTTP `409` 和稳定业务错误码。
- 每次成功动作写审计日志；库存、订单和报价动作同时写 Outbox 事件。

## 8. 第一批开发接口

工程初始化后优先实现以下端到端切片：

1. `GET /public/company`。
2. `GET /public/varieties` 与 `GET /public/varieties/{id}`。
3. `GET /public/supply-batches` 与详情接口。
4. `POST /public/inquiries`。
5. `POST /auth/admin/login`。
6. `GET /admin/inquiries` 与 `GET /admin/inquiries/{id}`。
7. `POST /admin/inquiries/{id}/assign`。
8. `POST /admin/inquiries/{id}/follow-ups`。

这组接口形成“展示—供应—询价—后台接收—分配—跟进”的第一个可验收闭环。机器可读 OpenAPI 草案首先完整描述这一组接口，其余接口按开发阶段逐步补齐 schema。

## 9. HTTP 状态码

| HTTP | 使用场景 |
|---:|---|
| 200 | 查询或动作成功 |
| 201 | 创建资源成功 |
| 204 | 无响应体的成功操作 |
| 400 | JSON、参数格式或字段校验失败 |
| 401 | 未认证、令牌无效或已过期 |
| 403 | 已认证但无操作或数据权限 |
| 404 | 资源不存在，或为避免泄露而不向当前用户暴露 |
| 409 | 状态冲突、重复接受、库存不足、幂等冲突 |
| 410 | 安全链接、报价或文件授权已明确失效 |
| 413 | 上传文件或请求体过大 |
| 422 | 格式正确但违反业务规则 |
| 429 | 请求过于频繁或验证码限流 |
| 500 | 未预期内部错误 |
| 503 | 必要依赖暂不可用 |

## 10. 通用错误码

### 10.1 基础、认证与权限

| 错误码 | HTTP | 含义 |
|---|---:|---|
| `VALIDATION_FAILED` | 400 | 字段格式或必填项不正确 |
| `MALFORMED_REQUEST` | 400 | JSON 或请求结构无法解析 |
| `UNSUPPORTED_LOCALE` | 400 | 不支持的语言参数 |
| `UNAUTHENTICATED` | 401 | 尚未认证 |
| `TOKEN_INVALID` | 401 | 令牌无效 |
| `TOKEN_EXPIRED` | 401 | 令牌已过期 |
| `EMAIL_CODE_INVALID` | 401 | 邮箱验证码错误 |
| `EMAIL_CODE_EXPIRED` | 401 | 邮箱验证码过期 |
| `FORBIDDEN` | 403 | 无操作权限 |
| `DATA_SCOPE_FORBIDDEN` | 403 | 无该客户或业务数据范围 |
| `RESOURCE_NOT_FOUND` | 404 | 资源不存在或当前用户不可见 |
| `STATE_CONFLICT` | 409 | 当前状态不允许此操作 |
| `IDEMPOTENCY_KEY_REQUIRED` | 400 | 缺少必要幂等键 |
| `IDEMPOTENCY_KEY_CONFLICT` | 409 | 同一幂等键对应不同请求 |
| `RATE_LIMITED` | 429 | 请求频率超过限制 |
| `INTERNAL_ERROR` | 500 | 未预期内部错误 |
| `DEPENDENCY_UNAVAILABLE` | 503 | 数据库、存储或外部服务暂不可用 |

### 10.2 客户、供应和询价

| 错误码 | HTTP | 含义 |
|---|---:|---|
| `CONTACT_METHOD_REQUIRED` | 422 | 手机或邮箱至少填写一项 |
| `CUSTOMER_MERGED` | 409 | 当前客户已经合并到其他客户 |
| `VARIETY_NOT_PUBLISHED` | 404 | 品种未发布或不可见 |
| `SUPPLY_BATCH_NOT_AVAILABLE` | 409 | 批次暂停、售罄或不接受当前业务 |
| `BUSINESS_TYPE_MISMATCH` | 422 | 商品薯、种薯类型不一致 |
| `MIXED_VARIETY_IN_ITEM` | 422 | 单条明细包含多个品种 |
| `MOQ_NOT_MET` | 422 | 未达到订单或单品种起订量 |
| `SPECIAL_REVIEW_REQUIRED` | 422 | 特殊需求只能进入人工审核 |
| `INQUIRY_ALREADY_CLOSED` | 409 | 询价已关闭 |
| `INQUIRY_OWNER_REQUIRED` | 422 | 当前动作前必须分配负责人 |
| `INQUIRY_STATUS_INVALID` | 409 | 询价状态不允许当前动作 |

### 10.3 报价、库存和订单

| 错误码 | HTTP | 含义 |
|---|---:|---|
| `QUOTATION_APPROVAL_REQUIRED` | 409 | 报价必须先审批 |
| `QUOTATION_VERSION_IMMUTABLE` | 409 | 已发送版本不可修改 |
| `QUOTATION_NOT_SENT` | 409 | 报价尚未正式发送 |
| `QUOTATION_EXPIRED` | 410 | 报价已经过期 |
| `QUOTATION_ALREADY_ACCEPTED` | 409 | 报价已接受或已经生成订单 |
| `QUOTATION_ACCESS_DENIED` | 403 | 当前安全会话无权查看报价 |
| `INVENTORY_INSUFFICIENT` | 409 | 一个或多个批次可售库存不足 |
| `INVENTORY_RESERVATION_EXPIRED` | 409 | 临时锁定已经失效 |
| `INVENTORY_RESERVATION_CONFLICT` | 409 | 占用已被其他操作改变 |
| `ORDER_STATUS_INVALID` | 409 | 订单状态不允许当前动作 |
| `ORDER_ACCESS_DENIED` | 403 | 当前安全会话无权查看订单 |
| `ROUTE_NOT_ACTIVE` | 422 | 所选路线未启用 |
| `CONTRACT_REQUIRED` | 422 | 当前动作需要已归档合同或定金确认 |

### 10.4 样品、文件和通知

| 错误码 | HTTP | 含义 |
|---|---:|---|
| `SAMPLE_VARIETY_LIMIT_EXCEEDED` | 422 | 一次样品超过 3 个品种 |
| `SAMPLE_REVIEW_REQUIRED` | 409 | 销售、质量或合规审核未完成 |
| `SAMPLE_STATUS_INVALID` | 409 | 样品状态不允许当前动作 |
| `DOCUMENT_NOT_VERIFIED` | 403 | 文件尚未通过审核 |
| `DOCUMENT_EXPIRED` | 410 | 文件已过期 |
| `DOCUMENT_ACCESS_DENIED` | 403 | 当前客户无文件访问授权 |
| `FILE_TYPE_NOT_ALLOWED` | 422 | 文件类型不允许 |
| `FILE_TOO_LARGE` | 413 | 文件超过大小限制 |
| `NOTIFICATION_SEND_FAILED` | 503 | 通知暂时发送失败，业务数据已保存 |
| `EXPORT_PERMISSION_REQUIRED` | 403 | 无数据导出权限 |

## 11. 错误码治理

- 错误码一经被已发布客户端使用，不改变原含义。
- 新增业务错误码时同时更新本文、OpenAPI 和前端错误码映射。
- 不使用中文或俄文文本作为错误码。
- 同一业务冲突优先返回最具体的错误码，未知情况才使用 `STATE_CONFLICT`。
- 字段错误放入 `details`，不得为了单个字段不断新增错误码。
- 内部异常统一记录 `requestId`，面向客户只返回安全提示。

## 12. OpenAPI 与代码同步

- `api/openapi-v1.yaml` 是工程初始化前的机器可读契约草案。
- 后端实现后，契约测试必须验证路径、字段、状态码和字符串 ID。
- 前端类型从已审核的 OpenAPI 生成或同步，禁止三端各自手写不同字段名称。
- 不兼容变更进入新的 API 版本；新增可选字段和新接口可在 `v1` 内兼容扩展。
- 每次合并接口变更时，同时提交实现、测试和契约更新。
