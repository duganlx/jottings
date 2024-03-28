# 结算明细报表

## 计算公式汇总

### 当日盈亏

最新计算如下

```text
==== 资产维度 ====
日末资产 = 总资产 - 总负债 + 资金转出 + 证券转出
日初资产 = 日初总资产 - 日初总负债 + 资金转入 + 证券转入
当日盈亏 = 日末资产 - 日初资产
当日盈亏% = (日末资产 / 日初资产 - 1) * 100%


==== 市值维度 ====
日末资产 = 总资产 - 总负债 + 资金转出 + 证券转出
日初资产 = 日初总资产 - 日初总负债 + 资金转入 + 证券转入
当日盈亏 = 日末资产 - 日初资产
日初市值 = 日初持仓市值 + 日初证券负债
# 如果日末资产 <= 0，则当日盈亏% = 0
当日盈亏% = (当日盈亏 / 日初市值) * 100%
```

历史版本

```
==== 市值维度-v1.0 ====
日末资产 = 总资产 - 总负债 + 资金转出 + 证券转出
日初资产 = 日初总资产 - 日初总负债 + 资金转入 + 证券转入
当日盈亏 = 日末资产 - 日初资产
日初市值 = 日初持仓市值 - 日初证券负债
# 如果日末资产 <= 0，则当日盈亏% = 0
当日盈亏% = (当日盈亏 / 日初市值) * 100%

tips: 该版本在多空头下会存在问题。在空头情况下，日初证券负债也算是日初市值，不应该减去

==== 区分是否为T0下 T0 交易 ====
当日盈亏 = 卖出市值 + 净出入金 - 买入市值 + 资金转出 - 资金转入
当日盈亏% = 当日盈亏 / 买入市值
```

### 当日盈亏(对冲)

最新计算如下，需要注意的是在这三种情况下，*成立以来*的第一天的`当日盈亏%(对冲) = 当日盈亏%(对冲) * 0`，原因是成立第一天还在建仓，没有日初持仓市值和日初证券负债。

```text
==== 对冲类型:指数 ====
基准指数pnl% = dm_histdata.bar_day取close和preclose按照pnl公式计算
当日盈亏(对冲) = (日初持仓市值 + 日初证券负债) * 基准指数pnl%
当日盈亏%(对冲) = 基准指数pnl%


==== 对冲类型:公司基准 ====
指数pnl% = dm_histdata.bar_day取close和preclose按照pnl公式计算
设定pnl% = x% / 243 (x为一平核定每年一个对冲成本的参数：10%(2021), 4%(2022), 3%(2023), 3%(2023))
当日盈亏(对冲) = (日初持仓市值 + 日初证券负债) * (指数pnl% + 设定pnl%)
当日盈亏%(对冲) = 指数pnl% + 设定pnl%


==== 对冲类型:主力合约 ====
pnl% = ads_eqw.ads_ic889中的 pnl_close
当日盈亏(对冲) = (日初持仓市值 + 日初证券负债) * pnl%
当日盈亏%(对冲) = pnl%
```

历史版本

```text
==== 虚拟期值-v1.1 ====
对冲张数 = 200
对冲票数 = round(日初持仓市值 / (基准指数的昨日收盘价 * 对冲张数))
当日盈亏(对冲) = 对冲票数 * (基准指数的昨日收盘价 * 对冲张数) * 基准指数pnl%
当日盈亏%(对冲) = 基准指数pnl%


==== 指数-v1.0 ====
基准指数pnl% = dm_histdata.bar_day取close和preclose按照pnl公式计算
当日盈亏(对冲) = (日初持仓市值 + 日初证券负债) * 基准指数pnl%
当日盈亏%(对冲) = 当日盈亏(对冲) / 日初资产
-> 日初资产 (资产维度) = 日初总资产 - 日初总负债 + 资金转入 + 证券转入
-> 日初资产 (市值维度) = 日初持仓市值 - 日初证券负债 + 证券转入


==== 虚拟期值-v1.0 ====
对冲张数 = 200
对冲票数 = round(日初持仓市值 / (基准指数的昨日收盘价 * 对冲张数))
当日盈亏(对冲) = 对冲票数 * (基准指数的昨日收盘价 * 对冲张数) * 基准指数pnl%
当日盈亏%(对冲) = 当日盈亏(对冲) / 日初资产
-> 日初资产 (资产维度) = 日初总资产 - 日初总负债 + 资金转入 + 证券转入
-> 日初资产 (市值维度) = 日初持仓市值 - 日初证券负债 + 证券转入
```

### 当日超额

```text
alpha = 当日盈亏 - 当日盈亏(对冲)
alpha% = 当日盈亏% - 当日盈亏%(对冲)
```

### 累计计算

```text
累计* = 昨日累计* + 当日*
```

### 是否参与计算

```text
持仓变动% = (持仓市值 - 证券负债) / (日初持仓市值 - 日初证券负债) - 1

持仓变动% <= 设定阈值%  ==>  参与计算
持仓变动% > 设定阈值%  ==>  参与计算
```

## 数据来源汇总

按产品中，左侧树形列表：dim_datahub.dim_unit_account_product

### 数据表

资产单元打标表 ads_eqw.ads_unit_label_value[deal_date, au_code, label, value]：保存资产单元在每一个交易日的标签数据 `(deal_date, au_code, label) -> value`

dim_datahub.dim_jhlun_product [productName, productFullName, fundRecordNumber, parentProductId, productCode, setDate, productStatus, productStrategyTyp, pbInternalProductCode]：保存了产品的信息

ads_eqw.ads_statement_status [au_code, account_name_cn, settle_status, statement_status]：对账单存续状态，针对资金账号维度

命名规范：`ads_模块_业务类型_数据口径`

- 模块：account 账户, product 产品, unit 单元
- 业务类型：balance 资产信息, position 持仓信息, transit 划拨信息
- 数据口径：raw 原始数据, pending 内部清算, checking 结算单, confirm 复核后

资产信息表 ads_eqw.ads_account_balance_pending, ads_eqw.ads_unit_balance_pending;

- au_name 资产单元名称, au_code 资产单元, currency 币种,
- net_asset 净资产, total_asset_initial 日初总资产, total_asset 总资产, equity_initial 日初市值, equity 持仓市值, equity_in_transit 在途市值, fund_initial 交易日开始时的可用资金, fund_deposit_withdraw 净出入金, fund_available 可用资金, fund_in_transit 在途资金, fund_frozen 冻结资金, balance 资金余额, total_liability 总负债, cash_debt 资金负债, security_debt 证券负债, commission 手续费, create_time 成交时间, update_time 更新时间, settle_time 清算时间, trade_date 交易日, fund_deposit 入金, fund_withdraw 出金, total_liability_initial 日初总负债, cash_debt_initial 日初资金负债, security_debt_initial 日初证券负债, type 账户类型, equity_deposit 转入证券市值, equity_withdraw 转出证券市值, net_equity_traded 净买入证券市值, equity_buy 买入证券市值, equity_sell 卖出证券市值

持仓信息表 ads_eqw.ads_account_position_pending, ads_eqw.ads_unit_position_pending

成交信息表 ads_eqw.ads_account_trade_pending, ads_eqw.ads_unit_trade_pending

资金划转信息表 ads_eqw.ads_account_fund_transit_pending, ads_eqw.ads_unit_fund_transit_pending, ads_eqw.ads_account_fund_transit_checking

持仓划转信息表 ads_eqw.ads_account_equity_transit_pending, ads_eqw.ads_unit_equity_transit_pending, ads_account_equity_transit_checking

内部清算-结算单对账结果评分表 ads_eqw.ads_account_settle_balance_org_checking, ads_eqw.ads_account_settle_position_checking

一致性校验表 dwd_tradedata.dwd_diff_trading, dwd_tradedata.dwd_diff_goal_position, dwd_paper.dwd_diff_trading, dwd_paper.dwd_diff_goal_position

dm_histdata.bar_day

---

### 按投资经理

在按投资经理的结算明细报表页面中, 左侧的目录树为三级结构, 各级的关系是 _基金经理-产品-资产单元_. 实现上是先取 `dim_datahub.dim_unit_account_product` 表, 该表有 _资产单元_ 和 _产品_ 之间的映射关系, 接着利用 `ads_eqw.ads_unit_label_value` 表可以取得 _资产单元_ 和 _基金经理_ 之间的映射关系, 通过用户中心的接口`/api/uc/v1/users` 可以请求得到所有用户的信息. 具体如下所示.

- dim*unit_account_product 表: \_unit_code* 资产单元编码, _unit_name_ 资产单元名称, unit*type 资产单元类型, account_code 资金账号编码, account_name 资金账号名称, account_type 资金账号类型, \_product_inner_code* 产品内部编码, fund*record_number 产品协会编号, \_product_short_name* 产品名称简称, product_full_name 产品名称全称, product_type 产品类型, etl_time 数据入库时间. 目前, 仅仅展示 `unit_type=[1, 3]` 的, 跟凯强确认了下 该字段存在三种取值 `1 普通资产单元, 2 默认资产单元, 3 客户资产单元`.

- ads*unit_label_value 表: deal_date 日期, \_au_code* 资产单元, label 标签, _value_ 标签内容. 设置 `label = 'manager'`, `au_code - value` 就是资产单元和基金经理的映射. 需要注意的是因为存在日期的维度, 存在一个资产单元在不同的日期隶属于不同的基金经理的情况, 该情况在展示上就是每个基金经理都会有该资产单元.

- /api/uc/v1/users 接口: _id_, _userName_, nickName, email, mobile, avatar, status, ext, roles, sex, depts, qywxId, createAt. `status=0` _应该_ 是属于正常状态.

具体实现逻辑是当取得这三张表的数据, 根据 ads_unit_label_value 表生成一个 `Map<基金经理, 资产单元[]>`, 使用 dim_unit_account_product 表生成一个 `Map<资产单元, 产品>`, 根据前面两个 Map 可以产生 `Map<基金经理, Map<产品, 资产单元[]>>`, 利用用户中心 userName 取得基金经理信息进行绑定.

```js
// products 使用 product_inner_code 作为 key
InvestMgrItem{userId: string; userName: string; nickName: string; category: 'investMgr'; name: string; products: Record<string, InvestMgrProduct>}
// units 使用 unit_code 作为 key
InvestMgrProduct{category: 'investProduct'; type: string; name: string; fullName: string; code: string; units: Record<string, StlUnitItem>}
StlUnitItem{category: 'unit'; type: string; name: string; fullName: string; code: string; isDefault: boolean}

// 关联关系
// ads_unit_label_value.au_code --- dim_unit_account_product.
```

---

## 典型场景分析

### 场景 1：成立以来下的选择项目切换

在资产概要页面中，当时间范围选择"成立以来"时，会自行判断选择项目的*起始日期*，然后取数据时会取相应日期范围内的数据进行展示。此时切换选择的项目时，会出现页面数据加载后会出现一次突变。出现数据突变的原因是触发了两次获取数据的 useEffect，因为两次取数据的时间范围不相同，所以取得的数据也不相同。而进一步分析，之所以会触发两次 useEffect，是因为当切换项目时，会直接触发取数据的 useEffect，而此时时间范围还是上一次旧数据；与此同时，项目切换因为*起始日期*不同，会再一次触发取数据的 useEffect；当第一次取数据的 useEffect 完成时页面就完成渲染了，第二次取数据的 useEffect 完成会再次渲染页面，用户观感上就是突然间数据突变了。

目前的解决方案是采用在取数据的 useEffect 中加入一段代码去判断当时间范围为"成立以来"时先发一次请求去拿到正确的*起始日期*后去发送请求。这样即可让两次发送的请求携带参数是完全一致从而规避掉页面突变的问题。该解决办法是目前设计下的最优解法。

### 场景 2：资产概要中的数据计算

资产概要中的数据计算有当日盈亏、当日盈亏（对冲）、当日超额以及这三个的累计值。当日盈亏计算分为*资产*和*市值*两个维度。当日盈亏（对冲）根据对冲类型不同也不一样，对冲类型有*指数*，*公司基准*和*主力合约*三种，*虚拟期指*这个对冲类型被移除了。另外，这些计算指标有两种量纲，人民币（元/万/亿）和百分比（%）。所以这几个指标在页面展示时就有 12 种（2x2x3）情况了。原始数据的来来源也各有不同，当日盈亏的数据来自于资产信息表（balance）；当日盈亏（对冲）则根据类型不同，数据来源也不同，指数和公司基准要指数日 K 数据（bar_day），主力合约需要 ic889（ads_ic889）。

目前在代码实现上，会一次性请求所有需要的数据并计算出所有情况的结果。这样在页面切换时即可直接渲染。变量命名方式采用`对冲类型_指标_[市值/资产]`，比如，主力合约在市值维度下的当日超额，其变量命名为`zlhy_alpha_sz`。在进行切换时，s2 表格展示对应情况下的*指标*即可。这样设计的在应对每种情况计算公式各不相同，或者不同的情况下展示的数据指标有差异时（之前的虚拟期指就需要单独展示多一列数据对冲张数`xnqz_ticket`）表现很好。但从目前的情况来看，所有的情况下展示的*指标*完全一致。在这种情况下，缺点似乎更为明显：1. 代码冗余度高，每多一个指标就得按照排列组合增加多个字段，且维护较为繁琐；2. 初次加载速度慢，但在后续切换情况时很快。如果后续开发中仍然保持目前的情况（*指标*基本一致），可以考虑改成每次只计算展示的那种情况下的指标，切换时再计算新的情况，变量也只需要保存一份。

### 场景 3：左侧项目切换与右侧联动

左侧树形列表可以选择不同的项目（按产品：产品/资金账号/资产单元；按投资经理：投资经理/产品/资产单元；按策略：策略/产品/资产单元），接着右侧就会展示该项目的结算明细。另外，左侧项目的选择不仅仅有单选，还存在复选的情况，就是查看复选的几个组合起来的结算明细。复选的限制在不同的 Tab 页并不相同：按产品中，并没有任何限制，用户可以任意组合产品、资金账号、资产单元；按投资经理中，只允许复选同属于一个投资经理下的产品和资产单元；按策略中，也只允许复选同属于一个策略下的产品和资产单元。在页面实现上，左侧树形列表和右侧结算明细分属于两个组件，左侧组件会将选择的项目以*对象*数组的形式发送给右侧组件，所以该*对象*数组必须精确表达出选中的项目，项目选择的场景有如下所示。在按投资经理和按策略下选择产品/资产单元时，传给右侧组件的对象数组仅仅有产品/资产单元，并不知道是哪个投资经理或哪个策略，所以会再传一个*根项目*的参数给右侧组件。由于按产品中复选可以跨越多个产品，所以在该场景下*根项目*是传空。右侧组件在收到了选中*对象*数组和*根项目*的参数信息后，就有足够的信息去获取结算数据进行展示。

```text
项目选择的场景
== 按产品 ==
1. 单选 => 产品，资金账号，资产单元
2. 选择多个产品
3. 选择几个产品和几个资金账号
4. 选择几个产品和几个资金账号和几个资产单元
5. 选择几个产品和几个资产单元
6. 选择多个资金账号
7. 选择几个资金账号和几个资产单元
8. 选择多个资产单元

== 按投资经理 ==
1. 单选 => 投资经理，产品，资产单元
2. 在单个投资经理下，选择多个产品
3. 在单个投资经理下，选择几个产品和几个资产单元
4. 在单个投资经理下，选择多个资产单元

== 按策略 ==
1. 单选 => 策略，产品，资产单元
2. 在单个策略下，选择多个产品
3. 在单个策略下，选择几个产品和几个资产单元
4. 在单个策略下，选择多个资产单元
```

### 场景 4：标签表的设计使用

标签表指的是 ads_eqw.ads_unit_label_value 存放了资产单元在每一个交易日的标签数据，数据表一共有四列：交易日 deal_date, 资产单元 au_code, 标签名 label, 标签值 value。所以如果想获取某个标签值 value，需要提供三个参数 deal_date, au_code, label。

产品在运行过程中存在不同交易日归属于不同的投资经理和不同交易日归属于不同的策略类型。所以，在按投资经理或按策略展示时，就需要去标签表中取得每一个资产单元在每一天的投资经理标签 manager 或策略标签 strategy 的数据进行过滤才能得到正确的统计结果。一些使用到的 SQL 语句如下：

```sql
-- 获取按策略下每个资产单元的最早时间
select au_code, `value`, min(deal_date) as deal_date from ads_eqw.ads_unit_label_value where label = 'strategy' group by au_code, `value`

-- 获取每个资产单元的最新结算时间
select au_code, max(deal_date) as deal_date from ads_eqw.ads_unit_label_value group by au_code, `value`
```

### 场景 5：左侧项目列表设计

## 附录

### UniverseData 对象属性

| 字段名                                          | 中文名                   | 备注                                                        |
| ----------------------------------------------- | ------------------------ | ----------------------------------------------------------- |
| auCode                                          | 资金账号                 |                                                             |
| auName                                          | 资金账号名称             |                                                             |
| tradeDate                                       | 交易日                   | 日期基准字段，13 位时间戳                                   |
| currency                                        | 币种                     | 人民币 CNY                                                  |
| totalAssetInitial                               | 日初总资产               |                                                             |
| totalAsset                                      | 总资产                   |                                                             |
| equityInitial                                   | 日初持仓市值             |                                                             |
| equity                                          | 持仓市值                 |                                                             |
| fundInitial                                     | 日初资金                 |                                                             |
| balance                                         | 资金余额                 |                                                             |
| totalLiabilityInitial                           | 日初总负债               |                                                             |
| totalLiability                                  | 总负债                   | 净资产 = 总资产 - 总负债                                    |
| cashDebtInitial                                 | 日初资金负债             |                                                             |
| cashDebt                                        | 资金负债                 |                                                             |
| securityDebtInitial                             | 日初证券负债             |                                                             |
| securityDebt                                    | 证券负债                 |                                                             |
| netEquityTraded                                 | 净买入市值               |                                                             |
| equityBuy                                       | 买入市值                 |                                                             |
| equitySell                                      | 卖出市值                 |                                                             |
| fundDepositWithdraw                             | 净出入金                 | 资金转入 - 资金转出                                         |
| fundDeposit                                     | 资金转入                 |                                                             |
| fundWithdraw                                    | 资金转出                 |                                                             |
| equityDeposit                                   | 证券转入                 |                                                             |
| equityWithdraw                                  | 证券转出                 |                                                             |
| commission                                      | 手续费                   |                                                             |
| settleTime                                      | 清算时间                 |                                                             |
| equityInTransit                                 | 在途市值                 |                                                             |
| fundAvailable                                   | 可用资金                 |                                                             |
| fundInTransit                                   | 在途资金                 |                                                             |
| fundFrozen                                      | 冻结资金                 |                                                             |
| type                                            | 账户类型                 |                                                             |
| createTime                                      | 成交时间                 |                                                             |
| updateTime                                      | 更新时间                 |                                                             |
| isT0                                            | 是否 T+0                 |                                                             |
| isValid                                         | 是否有效                 |                                                             |
| totalAssetPnl                                   | 当日盈亏                 | 共有三种情况，参看下方公式 _当日盈亏_                       |
| totalAssetPnlCum                                | 累计盈亏                 | 参看下方公式汇总 _累计盈亏_                                 |
| prevTotalAssetPnlCum                            | 昨日累计盈亏             | 计算 _累计盈亏_ 时使用                                      |
| totalAssetPnlPercentage                         | 当日盈亏%                | 共有三种情况，参看下方公式 _当日盈亏%_                      |
| totalAssetPnlCumPercentage                      | 累计盈亏%                | 参看下方公式汇总 _累计盈亏_                                 |
| prevTotalAssetPnlCumPercentage                  | 昨日累计盈亏%            | 计算 _累计盈亏_ 时使用                                      |
| verifyTotalAssetInitial                         | 核算字段: 日初总资产     | 日初持仓市值 + 日初资金余额                                 |
| isOkTotalAssetInitial                           | 验证字段结果: 日初总资产 | `verifyTotalAssetInitial == totalAssetInitial`              |
| verifyTotalAsset                                | 核算字段: 总资产         | 持仓市值 + 在途市值 + 资金余额                              |
| isOkTotalAsset                                  | 验证字段结果: 总资产     | 如果核算的结果和取数回来的结果一致，则为 true；反之为 false |
| verifyTotalLiability                            | 核算字段: 总负债         | 资金负债 + 证券负债                                         |
| isOkTotalLiability                              | 验证字段结果: 总负债     | 如果核算的结果和取数回来的结果一致，则为 true；反之为 false |
| banchmarkPnlPercentage                          | 基准盈亏%                |                                                             |
| banchmarkPnlCumPercentage                       | 基准累计盈亏%            | 参看下方公式汇总 _累计盈亏_                                 |
| benchmarkPreClose                               | 基准指数昨收             | bar_day 表的字段 pre_close                                  |
| zs_totalAssetPnlHedge                           | 当日盈亏(对冲)           | `zs` 开头表示对冲类型为 指数                                |
| zs_totalAssetPnlHedgeCum                        | 累计盈亏(对冲)           | 累计盈亏(对冲) = 昨日累计盈亏(对冲) + 当日盈亏(对冲)        |
| zs_prevTotalAssetPnlHedgeCum                    | 昨日累计盈亏(对冲)       | 计算 _累计盈亏(对冲)_ 时使用                                |
| zs_alpha                                        | 当日超额                 | 当日盈亏 - 当日盈亏(对冲, 对冲类型: 指数)                   |
| zs_alphaCum                                     | 累计超额                 | 累计超额 = 昨日累计超额 + 当日超额\*                        |
| zs_prevAlphaCum                                 | 昨日累计超额             | 计算累计超额时使用                                          |
| zs_totalAssetPnlHedgePercentage                 | 当日盈亏%(对冲)          | 所除分母维度是总资产，参看下方公式汇总 _当日盈亏%(对冲)_    |
| zs_totalAssetPnlHedgeCumPercentage              | 累计盈亏%(对冲)          | 累计盈亏%(对冲) = 昨日累计盈亏%(对冲) + 当日盈亏%(对冲)     |
| zs_prevTotalAssetPnlHedgeCumPercentage          | 昨日累计盈亏%(对冲)      | 计算累计盈亏%(对冲)时使用                                   |
| zs_totalAssetPnlHedgePercentage_rcccsz          | 当日盈亏%(对冲)          | 所除分母维度是总市值，参看下方公式汇总 _当日盈亏%(对冲)_    |
| zs_totalAssetPnlHedgeCumPercentage_rcccsz       | 累计盈亏%(对冲)          |                                                             |
| zs_prevTotalAssetPnlHedgeCumPercentage_rcccsz   | 昨日累计盈亏%(对冲)      |                                                             |
| zs_alphaPercentage                              | 当日超额                 |                                                             |
| zs_alphaCumPercentage                           | 累计超额                 |                                                             |
| zs_prevAlphaCumPercentage                       | 昨日累计超额             |                                                             |
| zs_alphaPercentage_rcccsz                       | 当日超额                 |                                                             |
| zs_alphaCumPercentage_rcccsz                    | 累计超额                 |                                                             |
| zs_prevAlphaCumPercentage_rcccsz                | 昨日累计超额             |                                                             |
| xnqz_ticket                                     | 张数                     | `xnqz` 开头表示对冲类型为 虚拟期指                          |
| xnqz_totalAssetPnlHedge                         | 当日盈亏(对冲)           |                                                             |
| xnqz_totalAssetPnlHedgeCum                      | 累计盈亏(对冲)           |                                                             |
| xnqz_prevTotalAssetPnlHedgeCum                  | 昨日累计盈亏(对冲)       |                                                             |
| xnqz_alpha                                      | 当日超额                 |                                                             |
| xnqz_alphaCum                                   | 累计超额                 |                                                             |
| xnqz_prevAlphaCum                               | 昨日累计超额             |                                                             |
| xnqz_totalAssetPnlHedgePercentage               | 当日盈亏%(对冲)          |                                                             |
| xnqz_totalAssetPnlHedgeCumPercentage            | 累计盈亏%(对冲)          |                                                             |
| xnqz_prevTotalAssetPnlHedgeCumPercentage        | 昨日累计盈亏%(对冲)      |                                                             |
| xnqz_totalAssetPnlHedgePercentage_rcccsz        | 当日盈亏%(对冲)          |                                                             |
| xnqz_totalAssetPnlHedgeCumPercentage_rcccsz     | 累计盈亏%(对冲)          |                                                             |
| xnqz_prevTotalAssetPnlHedgeCumPercentage_rcccsz | 昨日累计盈亏%(对冲)      |                                                             |
| xnqz_alphaPercentage                            | 当日超额%                |                                                             |
| xnqz_alphaCumPercentage                         | 累计超额%                |                                                             |
| xnqz_prevAlphaCumPercentage                     | 昨日累计超额%            |                                                             |
| xnqz_alphaPercentage_rcccsz                     | 当日超额%                |                                                             |
| xnqz_alphaCumPercentage_rcccsz                  | 累计超额%                |                                                             |
| xnqz_prevAlphaCumPercentage_rcccsz              | 昨日累计超额%            |                                                             |

说明

- isT0: 根据`ads_eqwads_unit_label_value`表中 label 为 strategy，value 为 T0 和 T1 的记录。如果当天有 T1 的记录，则直接判定为 _非 T0_；否则根据当天是否有 T0 记录进行判定。
- banchmarkPnlPercentage: 数据取 `dm_histdata.bar_day`，按照公式 pnl% = (当日收盘价 - 昨日收盘价) / 昨日收盘价 \* 100% 计算得到
- isValid: 头尾如果出现 `[持仓市值, 证券负债, 手续费]` 都为 0，则判定为无效数据，中间部分如果连续三天出现这三个字段为 0 的话，也判定为无效数据

### 对象构建

```typescript
type SettleUnitItem = {
  category: "unit" | "account" | "product";
};
```

### SQL

```sql
-- ## 表 dim_unit_account_product ##
-- 查询 实盘单元
SELECT * FROM dim_datahub.dim_unit_account_product WHERE account_is_real = '0';
-- 查询 仿真单元
SELECT * FROM dim_datahub.dim_unit_account_product WHERE account_is_real = '1';

-- 查询 产品页中 “存续” 的产品
SELECT DISTINCT product_inner_code FROM dim_datahub.dim_unit_account_product WHERE unit_status = '1' AND account_is_real = '0';
SELECT count(*) AS total FROM (SELECT DISTINCT product_inner_code FROM dim_datahub.dim_unit_account_product WHERE unit_status = '1' AND account_is_real = '0');
-- 查询 产品页中 “存续” 的资金账号
SELECT DISTINCT account_code, account_name, product_inner_code, product_short_name FROM dim_datahub.dim_unit_account_product WHERE unit_status = '1' AND account_is_real = '0'
-- 查询 产品页中 “存续” 的资产单元
SELECT DISTINCT unit_code, unit_name, account_code, account_name, product_inner_code, product_short_name FROM dim_datahub.dim_unit_account_product WHERE unit_status = '1' AND account_is_real = '0'
```
