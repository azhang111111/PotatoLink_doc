# 2026-09-30 审查修复接口补充

本补充对应 2026-09-29 审查中的 R01–R09。前后端需配套更新，以下三个列表的 `data` 由数组改为分页对象。

## 列表与客户入口

| 接口 | 参数与响应 |
|---|---|
| `GET /customer/inquiries` | `page=1&pageSize=20`；`data={items,page,pageSize,total}` |
| `GET /customer/orders` | 同上；仅包含账号拥有的 CRM 联系人记录 |
| `GET /admin/quotations` | 同上；可选 `status`、`inquiryId`，先按内部用户权限筛选 |
| `GET /customer/inquiries/{inquiryNo}` | 账号鉴权；`data={inquiry,quotations}`，仅列出已发送的当前报价版本 |
| `POST /customer/quotations/{quotationNo}/access` | 账号鉴权并检查归属；生成一小时报价访问链接，不撤销销售分享链接 |

页码最小为 1，页大小为 1–100，默认 20。客户可以从“我的询价”进入详情并继续查看报价；两个历史列表均提供加载更多。内部询盘、订单和报价采用数据库分页，工作台采用按权限筛选的汇总 SQL。订单列表不加载条目、分配历史和物流事件；完整数据继续通过订单详情接口返回。

账号直接查看的询盘没有分享链接到期时间，`inquiry.trackingExpiresAt` 为 `null`；分享链接访问仍返回实际到期时间。账号报价访问继续使用现有安全报价页及接受、拒绝、申请调整接口，其账号和链接权限检查保持有效。

## 库存更新

批次创建仍接受 `onHandKg` 初始化库存。编辑资料、审核、发布及下架只写批次资料和流程状态，忽略更新请求中的库存值。

新增 `POST /admin/supply-batches/{id}/inventory-adjustments`，仅 ADMIN、OPERATIONS 可调用：

```json
{
  "expectedOnHandKg": "30000.000",
  "onHandKg": "31000.000",
  "reason": "仓库盘点补录"
}
```

后端锁定库存行并比对当前实物库存与 `expectedOnHandKg`。不匹配返回 `STATE_CONFLICT`，新值低于当前临时占用和正式占用之和返回 `INVENTORY_INSUFFICIENT`。成功后增加库存版本并写入 `MANUAL_ADJUSTMENT` 流水，记录变化量、可售量前后值、原因及操作者。找不到库存返回 `RESOURCE_NOT_FOUND`。

## 交易一致性

报价审批、发送、客户接受、拒绝、申请调整和版本修订使用同一报价父行锁。在锁内重新读取当前状态；事务覆盖状态修改及创建订单。发送命令重放不会改写已经成交的询盘，版本修订重放不会再次撤销后续版本的链接。

后台每个尚未完成的路径和请求体组合使用同一个随机幂等键，失败重试、401 自动重试和同标签页刷新后保留该键。成功响应完成该意图，下一次提交生成新键；退出和新登录清理历史意图。

报价单价和指导价最多两位小数，数量最多三位小数；超出精度返回校验错误。明细金额按两位小数四舍五入，金额超出 `NUMERIC(18,2)` 可表示范围时拒绝。新询盘、报价和订单编号使用“前缀-日期-完整19位ID”，已有编号继续有效。

新增 Flyway `V021__workspace_read_indexes.sql`，为分页、归属筛选和趋势查询增加索引；不修改历史迁移。

## 回归入口

前端在仓库根目录执行 `pnpm test`、`pnpm typecheck`、`pnpm build`。

后端默认 `mvn test` 不运行数据库集成组。集成组要求一个独立测试数据库，用户为 `fix_test`，采用测试认证；通过环境变量启用：

```powershell
$env:POTATOLINK_TEST_DATABASE_URL='jdbc:postgresql://127.0.0.1:55439/potatolink_fix_test'
mvn test
```

集成组只接受回环地址和名称以 `_fix_test` 结尾的数据库，创建 `workspace_regression` schema，并在每个用例前清理测试数据。覆盖 105 条历史记录、内部和客户权限、批次旧快照保存、库存盘点流水及真实双连接报价状态竞争。不得把该变量指向业务数据库。
