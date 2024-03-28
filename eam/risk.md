# 风控通知设计&实现

该需求是希望能在 Quantweb 平台上监控触发风控的通知消息, 以便能有效掌握风控情况并为后续决策作支撑.

目前, 永权会将风控通知推送到 Kafka 对应的主题, 一共有四个主题, `rms_order` 保存的是所有经过风控系统的委托单, `rms_reject_cancel_order` 保存的是触发风控的撤单, `rms_reject_order` 保存的是触发风控的下单, `rms_heartbeat` 保存的是心跳包. 各个 Topic 的内容格式如下所示.

- rms_heartbeat: 时间字符串 格式为 `{时:分:秒}` 如 `{11:37:12}`
- rms_order: json 对象，包含的字段有 create_date(年月日组成的数字, 如 20230922), order_id, order_time(11 位时间戳), rsp_time(11 位时间戳), fk_time(11 位时间戳), account, market, code, biz, price, vol, matchprice, matchvol, eam_id, status, msg
- rms_reject_cancel_order: 同 rms_order
- rms_reject_order: 同 rms_order

我需要编写后端去消费 Kafka 中的消息并进行统计汇总（账号、风控类型）, 汇总信息放到 redis 中. 而每条记录也需要存储到数据库中. 前端页面需要展示风控的汇总消息. 前后端交互采用 websocket.

## 准备

Kafka 有两套环境, 生产环境 192.168.1.131, 仿真环境 192.168.1.205, 用 Offset Explorer 去连接时 将 Host 改成对应的 IP, 其他的参数配置为 `Kafka Cluster Version: 2.8, Zookeeper Port: 2181`

## 附录

### 风控名词解析

todo

### 风控错误码

```cpp
#ifndef JHLERRCODE_HHH_
#define JHLERRCODE_HHH_

const unsigned short CONST_ERROR_UNDEFINE					= -1;//未定义错误

const unsigned short CONST_ERROR_OK                          = 0;    //成功
//通讯层        0x0001-0x01FF   在网络通讯层发生的错误(数据的收发、校验、封包、拆包)
//交易所接口    0x0200-0x05FF   交易所返回错误
//应用          0x0600-0x9FFF   应用程序业务错误码
//数据库        0xA000-0xCFFF   在公司系统内部业务操作上发生的错误
//预留          0xD000-0xFFFF   预留错误码
const unsigned short CONST_ERROR_CONNECT_TIMEOUT             = 0x0001;   //连接响应超时
const unsigned short CONST_ERROR_SEND_FAILED                 = 0x0002;   //发送数据失败

//JHLQuoteApi
const unsigned short CONST_ERROR_QUOTE_UNINIT_PARAM          = 0x0700;   //行情API参数未初始化
const unsigned short CONST_ERROR_QUOTE_INITED                = 0x0701;   //行情转换API已初始化
const unsigned short CONST_ERROR_QUOTE_JOIN_FAILED           = 0x0702;   //行情加入接收组播错误

//QuoteConvertor
const unsigned short CONST_ERROR_CONVERTOR_UNINIT_PARAM      = 0x0800;   //行情转换API参数未初始化
const unsigned short CONST_ERROR_CONVERTOR_INITED            = 0x0801;   //行情转换API已初始化
const unsigned short CONST_ERROR_CONVERTOR_MARKET_PROHIBIT   = 0x0802;   //行情市场索引未授权
const unsigned short CONST_ERROR_CONVERTOR_ETF_UNKNOWFILE    = 0x0803;   //ETF清单无法定位文件
const unsigned short CONST_ERROR_CONVERTOR_ETF_MESSAGE       = 0x0804;   //ETF清单请求消息长度有误
const unsigned short CONST_ERROR_CONVERTOR_ETF_READFILE      = 0x0805;   //ETF清单读取文件失败
const unsigned short CONST_ERROR_CONVERTOR_UNINITED          = 0x0806;   //行情尚未初始化

//0xD000-0xD100柜台业务逻辑
const unsigned short CONST_ERROR_COMMON_NORESULT             = 0xD000;   //没有查询结果
const unsigned short CONST_ERROR_NOT_SEND                    = 0xD001;   //没有送达至柜台
const unsigned short CONST_ERROR_NOT_ENOUGH_MONEY			 = 0xD002;	//买入可用金额不足。可用金额=availbale_amt，委托金额=order_amt
const unsigned short CONST_ERROR_QTY						 = 0xD003;	//委托数量超过数量上限下限。数量=qty
const unsigned short CONST_ERROR_NOT_UNIT_LIMIT				 = 0xD004;	//委托数量不是最小下单量的整数倍。证券=security_code，委托数量=order_qty
const unsigned short CONST_ERROR_NOT_TICK_LIMIT				 = 0xD005;	//订单价格不符合最小价差。 证券=security_code, 价格=price, 价差=tick_limit
const unsigned short CONST_ERROR_NOT_PRICE_ERROR			 = 0xD006;	//价格错误，超过涨跌停。证券=security_code, 价格=price
const unsigned short CONST_ERROR_NOT_ENOUGH_AVAILABLE_QTY	 = 0xD007;	//可卖数量不足。证券=security_code，当前可卖为=available_qty
const unsigned short CONST_ERROR_CANCEL_FAILED				 = 0xD008;	//撤单失败。证券=security_code,委托号=orderid,委托状态为=order_status时，不允许撤单操作
const unsigned short CONST_ERROR_NOT_DBP					 = 0xD009;	//非担保品，不能买入。security_code=002715
const unsigned short CONST_ERROR_OVER_DEBT_QTY				 = 0xD00A;	//还券数量超过负债数量。security_code=601600,负债数量debt_qty=500,偿还数量order_qty=600
const unsigned short CONST_ERROR_OVER_TIME_LOCAL			 = 0xD00B;	//本地网关等待超时。账号=%s，证券=%s，委托数量=%d，价格=%.03f 方向=%d
const unsigned short CONST_ERROR_OVER_NEW_STK_SG			 = 0xD00C;	//超出申购市值额度。可申购额度eanble_qty=51000,order_qty=16000
const unsigned short CONST_ERROR_NOT_ENOUGH_MARGIN			 = 0xD00D;	//保证金不足，不能买入。security_code=002715
const unsigned short CONST_ERROR_NOT_RZ_STK					 = 0xD00E;	//非融资标的，不能买入。security_code=002715
const unsigned short CONST_ERROR_OVER_TIME_COUNTER			 = 0xD00F;	//柜台给的应答是超时。账号=%s，证券=%s，委托数量=%d，价格=%.03f 方向=%d
const unsigned short CONST_ERROR_NOT_SIGN_PROTOCAL			 = 0xD010;	//柜台拒绝该业务。原因:%s
const unsigned short CONST_ERROR_NOT_NORTH_TARGET			 = 0xD011;	//柜台拒绝该业务。原因:%s非北向标的，请更新北向清单,网关会自动通过QFII再次下单
const unsigned short CONST_ERROR_API_INIT_FAILED			 = 0xD012;	//柜台API初始化失败


//风控0xD300-0xD500
const unsigned short CONST_ERROR_FK_SVR_DISCONNECT						 = 0xD300;	//网关风控：与交易网关断开
const unsigned short CONST_ERROR_FK_REJECT_BUY      					 = 0xD301;	//网关风控：证券：#security_code#该账户没有买入权限
const unsigned short CONST_ERROR_FK_REJECT_SELL     					 = 0xD302;	//网关风控：证券：#security_code#该账户没有卖出权限
const unsigned short CONST_ERROR_FK_REJECT_BUY_ST						 = 0xD303;	//网关风控：证券：#security_code#该账户禁止买入ST股票
const unsigned short CONST_ERROR_FK_REJECT_SELL_ST					     = 0xD304;	//网关风控：证券：#security_code#该账户禁止卖出ST股票
const unsigned short CONST_ERROR_FK_BUY_ST_LIMIT    					 = 0xD305;	//网关风控：证券：#security_code#该账户买入ST超限
const unsigned short CONST_ERROR_FK_REJECT_BUY_HUGE_AMT					 = 0xD306;	//网关风控：证券：#security_code#禁止巨额买入，阀值=1002000，委托金额=5000000
const unsigned short CONST_ERROR_FK_REJECT_BUY_REPEAT_ORDER				 = 0xD307;	//网关风控：证券：#security_code#禁止大量重复委托，总笔数=3,总金额=1002000
const unsigned short CONST_ERROR_FK_CANCEL_LIMIT						 = 0xD308;	//网关风控：证券：#security_code#撤单比例超限风控目前比例=42%，阀值=40%
const unsigned short CONST_ERROR_FK_ZT_CANCEL_LIMIT						 = 0xD309;	//网关风控：证券：#security_code#涨停股票撤单笔数超限
const unsigned short CONST_ERROR_FK_SELF_MATCH							 = 0xD310;	//网关风控：禁止自成交
const unsigned short CONST_ERROR_FK_STATION_INFO						 = 0xD311;	//网关风控：站点信息不对
const unsigned short CONST_ERROR_FK_MAX_ORDER_COUNT						 = 0xD312;	//网关风控：达到最大下单数量
const unsigned short CONST_ERROR_FK_USELESS_LIMIT						 = 0xD313;	//网关风控：达到废单最大比例
const unsigned short CONST_ERROR_FK_NET_BUY_AMT_LIMIT					 = 0xD314;	//网关风控：达到净买入额度控制
const unsigned short CONST_ERROR_FK_ORDER_MATCH_LIMIT					 = 0xD315;	//网关风控：委托成交比预警功能
const unsigned short CONST_ERROR_FK_MD5_INFO							 = 0xD316;	//网关风控：没上传MD5码
const unsigned short CONST_ERROR_FK_FIX_INFO							 = 0xD317;	//网关风控：没配置FIX信息
const unsigned short CONST_ERROR_FK_VOL_ERROR							 = 0xD318;	//网关风控：委托数量有误
const unsigned short CONST_ERROR_FK_PRICE_ERROR							 = 0xD319;	//网关风控：委托价格有误
const unsigned short CONST_ERROR_FK_ACCOUNT_NOT_EXIST					 = 0xD320;	//网关风控：账户不存在
const unsigned short CONST_ERROR_FK_REJECT_TRADE    					 = 0xD321;	//网关风控：账户禁止交易
const unsigned short CONST_ERROR_FK_IP_BLACK        					 = 0xD322;	//网关风控：IP黑名单
const unsigned short CONST_ERROR_FK_NOT_IN_IP_WHITE        				 = 0xD323;	//网关风控：不在IP白名单
const unsigned short CONST_ERROR_FK_MAC_BLACK        					 = 0xD324;	//网关风控：MAC黑名单
const unsigned short CONST_ERROR_FK_NOT_IN_MAC_WHITE        			 = 0xD325;	//网关风控：不在MAC白名单
const unsigned short CONST_ERROR_FK_FLOW_CONTROL               			 = 0xD326;	//网关风控：流控
const unsigned short CONST_ERROR_FK_MANIPULATION                		 = 0xD327;	//网关风控：拉抬打压
const unsigned short CONST_ERROR_FK_AUCTION                		         = 0xD328;	//网关风控：集合竞价
const unsigned short CONST_ERROR_FK_ACCOUNT_BUY_BLACK_LIST		         = 0xD329;	//网关风控：账号买入黑名单
const unsigned short CONST_ERROR_FK_ACCOUNT_SELL_BLACK_LIST		         = 0xD330;	//网关风控：账号卖出黑名单
const unsigned short CONST_ERROR_FK_GROUP_BUY_BLACK_LIST		         = 0xD331;	//网关风控：组内买入黑名单
const unsigned short CONST_ERROR_FK_GROUP_SELL_BLACK_LIST		         = 0xD332;	//网关风控：组内卖出黑名单
const unsigned short CONST_ERROR_FK_FUND_BUY_BLACK_LIST		             = 0xD333;	//网关风控：基金买入黑名单
const unsigned short CONST_ERROR_FK_FUND_SELL_BLACK_LIST		         = 0xD334;	//网关风控：基金卖出黑名单
const unsigned short CONST_ERROR_FK_COMPANY_BUY_BLACK_LIST		         = 0xD335;	//网关风控：公司买入黑名单
const unsigned short CONST_ERROR_FK_COMPANY_SELL_BLACK_LIST		         = 0xD336;	//网关风控：公司卖出黑名单
const unsigned short CONST_ERROR_FK_REJECT_BUY_BLOCK    		         = 0xD337;	//网关风控：禁止买入板块
const unsigned short CONST_ERROR_FK_REJECT_SELL_BLOCK       	         = 0xD338;	//网关风控：禁止卖出板块
//const unsigned short CONST_ERROR_FK_MAX_ORDER_BUY             	         = 0xD339;	//网关风控：最大净买入金额
const unsigned short CONST_ERROR_FK_OFFER_REJECT               	         = 0xD340;	//网关风控：举牌拒绝
const unsigned short CONST_ERROR_FK_OFFER_WARNNING             	         = 0xD341;	//网关风控：举牌警告
const unsigned short CONST_ERROR_FK_SELF_MATCH_BETWEEN_GROUP			 = 0xD342;	//网关风控：组间自成交
const unsigned short CONST_ERROR_FK_SELF_MATCH_WITHIN_GROUP			     = 0xD343;	//网关风控：组内自成交
const unsigned short CONST_ERROR_FK_REJECT_REVERSE          		     = 0xD344;	//网关风控：反向交易
const unsigned short CONST_ERROR_FK_DELIST                    		     = 0xD345;	//网关风控：退市股票
const unsigned short CONST_ERROR_FK_CONTINUOUSAUCTION_FAKE_ORDER	     = 0xD346;	//网关风控：连续竞价虚假申报
const unsigned short CONST_ERROR_FK_KEEP_LIMIT_ORDER                     = 0xD347;	//网关风控：连续竞价维持涨跌幅限制价格
const unsigned short CONST_ERROR_FK_LIMIT_ORDER                          = 0xD348;	//网关风控：涨停板大额申报
const unsigned short CONST_ERROR_FK_MAX_BUY_ORDER_AMT                    = 0xD349;	//网关风控：单笔最大委托金额
const unsigned short CONST_ERROR_FK_TOTAL_BUY_ORDER_AMT                  = 0xD350;	//网关风控：累计买入委托金额
const unsigned short CONST_ERROR_FK_STK_NOT_EXIST                        = 0xD351;	//网关风控：STK对象不存在
const unsigned short CONST_ERROR_FK_OPENCALLACUTIN_FALSECLAIMS_ORDER      = 0xD352;	//网关风控：开盘集合竞价虚假申报
const unsigned short CONST_ERROR_FK_LIMIT_FALSECLAIMS_ORDER                 = 0xD353;	//网关风控：涨跌停虚假申报(连续竞价)
const unsigned short CONST_ERROR_FK_OPENCALLACUTIN_MANIPULATION             = 0xD354;	//网关风控：开盘集合竞价拉抬打压
const unsigned short CONST_ERROR_FK_CONTINUOUSAUCTION_MANIPULATION          = 0xD355;	//网关风控：连续竞价拉抬打压
const unsigned short CONST_ERROR_FK_CLOSECALLACUTIN_MANIPULATION            = 0xD356;	//网关风控：开盘集合竞价拉抬打压
const unsigned short CONST_ERROR_FK_CLOSECALLACUTIN_KEEP_LIMIT_ORDER     = 0xD357;	//网关风控：收盘集合竞价维持涨跌幅限制价格


#endif /*JHLERRCODE_HHH_*/
```
