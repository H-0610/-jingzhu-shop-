# 景筑小铺：免费试运行版本

Node.js 24 + PostgreSQL。保留分规格库存、订单查询、人工核款后自动发货、每2秒商品同步。
网站自身不要求ChatGPT账号、不设置地区白名单；实际地区可用性仍需外部网络验证。

## 部署与费用

render.yaml使用新加坡Free Web Service，无付费磁盘或Render数据库；订单和图片写入独立Neon免费数据库。
部署前必须确认Render账单未绑定付款方式，或存在已核实的零支出硬限制。Free计算套餐不保证流量和构建超额免费。
未绑卡时，Render超额会暂停服务或构建。Neon必须保持Free套餐，不自动升级。
Render空闲15分钟休眠，首次再访问可能等待约1分钟；Neon免费计算或流量额度用完可能暂停到下个周期。
此方案适合试运行，不承诺全天候可用，也不承诺所有亚洲运营商均能访问。

## 启动

需要Node.js 24。运行npm ci --omit=dev，然后node server.mjs。

- DATABASE_URL：秘密PostgreSQL连接地址。Neon应用使用pooler连接，表结构初始化使用对应直连；强制验证TLS证书。
- REQUIRE_POSTGRES=true：免费云端必须配置；缺少DATABASE_URL就拒绝启动，防止误用临时SQLite保存订单。
- INITIAL_ADMIN_PASSWORD：首次空库初始化密码，12–128位；导入已有数据后沿用原密码摘要。
- HOST=0.0.0.0；PORT由平台提供。
- PUBLIC_BASE_URL：自定义域名时设置；Render默认读取RENDER_EXTERNAL_URL。
- TRUST_PROXY_HOPS：默认0。受控代理后配置正确跳数；Render示例为1，部署时需核实。
- DATABASE_PATH：仅限本地或自有持久磁盘的SQLite兼容模式，免费云端不使用。

/healthz检查启动完成的进程，不反复查询数据库消耗空闲计算额度。数据库失效时业务请求返回503。

## 私有数据迁移

源码seed是空店，不含真实商品、订单、收款信息或管理员密码。
完整备份格式：{"state":{"settings":{},"products":[],"orders":[]},"assets":[]}。
必须保留adminPasswordHash、orderKeyHash、reservedCards、reservedContent和在用图片。
切换前暂停旧入口新增订单，取得最终备份，避免两套库存同时销售。

设置DATABASE_URL后运行：

```
node tools/postgres-backup.mjs import PRIVATE_BACKUP.json
node tools/postgres-backup.mjs export NEW_PRIVATE_BACKUP.json
```

导入只接受空库，事务失败回滚；导出拒绝覆盖现有文件。
备份、.env和密码不能放public目录或提交GitHub。
SQLite导入兼容：node tools/import-state.mjs PRIVATE_BACKUP.json NEW_DATABASE.sqlite。

## 规格库存与采购合计

每个规格分别选择发货方式。独立内容模式每行一件，录入20条不同内容即可销售20件；数字库存自动随内容条数变化，不能用一个数字代替缺少的货品内容。
可在规格下点击“导入 TXT 追加库存”，导入UTF-8文本（每行一件，1MB以内）；空行忽略，重复内容跳过，保存商品后才更新前台。删除剩余内容会相应减少可售库存。
购买20件时只预留该规格的20条内容，确认到账后交付全部20条；其他规格不扣减，不混用内容。
固定内容和人工交付模式可直接修改库存数字。
前台卡片与列表会按输入数量立即更新总价，并显示件数、单价；数量大于1时高亮该规格。后台价格或库存更新后，前台自动重新计算，结算金额仍由服务端核定。

## 测试与运行限制

### 客户钱包

前台 `/#wallet`：客户注册 / 登录，查看余额、累计充值、采购净支出、充值申请及钱包流水。账号可跨设备登录；注册时提供一次性显示的恢复码，恢复密码后轮换恢复码并使旧登录失效。
充值使用现有商家微信 / 支付宝收款链接，实际款项直接到商家原账户。客户提交“已付款”后不增加余额；后台 `admin#wallet` 核对到账，填写真实平台交易单号并确认后，充值才入账。同平台交易单号不能重复用于多笔充值。
余额支付在同一次版本条件更新中完成扣款、库存预留与自动交付。独立内容 / 固定内容立即完成；人工交付订单已支付但待商家交付，关闭时退回钱包余额且释放库存。已完成订单不可直接关闭，不提供客户转账 / 提现功能。
金额按整数分记账，请求编号防重复扣款；余额不足或数据保存失败时扣款、订单与库存一起回滚。余额和流水不能由前台或普通客户直接修改。
客户登录采用 HttpOnly / SameSite=Strict Cookie，线上 HTTPS 使用 Secure；密码 PBKDF2-HMAC-SHA256 600000 次，独立盐值，恢复码只存摘要。修改资金的客户请求验证同源及专用请求头，后台充值审核沿用管理员鉴权。
单次充值 1–10000 元，每个客户最多 5 笔待处理充值，账户余额上限 1000000 元（应用配置限额）。现有 JSON 状态总容量仍为 1.8 MB，包含商品、订单、客户和账目；接近上限会拒绝新增写入。扩大经营前需迁移为按行存储，不能通过删除钱包流水释放容量。
完整备份和导入包含客户凭据摘要、余额、充值和流水；导入检查余额与流水一致性。不要把备份上传到公开仓库。
密码存储参数参考：[OWASP Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html#pbkdf2)。

npm test：本地PostgreSQL引擎、HTTP、迁移和防误配置检查。
npm run test:cloud：使用.env.qa的TEST_DATABASE_URL，仅连接独立QA分支，不删除云端数据。
已验证双服务并发抢库存、规格内容隔离、旧编辑拒绝覆盖、确认幂等、取消返还一次和重启查单。
上线仍需目标手机、电脑、微信内置浏览器实测；静态收款码必须商家核实到账后才发货。
当前保留约1.8MB聚合商品/订单状态上限，接近上限前应归档升级。

官方说明：https://render.com/docs/free 、https://neon.com/docs/introduction/plans
