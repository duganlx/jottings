# Gsf

接收实盘数据配置如下

```ts
'/grpc_arrow': {
  target: 'http://eqw.eam.com',
  changeOrigin: true,
  pathRewrite: { '^': '' },
},
```

ads_eqw.ads_gsf_gsfServices [host 主机 ip, port 端口, alias 别名, del_at 删除时间]：记录 gsf 的服务列表

左侧金葵花服务列表对于选中的服务，会开启一个定时器每 5 秒取一次 Balance 数据，每一个 gsf 服务一条。

```
请求头携带信息如下
X-Gsf-Host: <host>:<port>
Content-Type: application/grpc-web
X-Au-Code: <auCode>  (有些有)
```

grpc 请求

- 获取 Balances 信息 `service_name: 'Oms', data_name: 'get_balances', format: 'mapArray'`
- 获取持仓信息 `service_name: 'Oms', data_name: 'get_position', format: 'mapArray'`

```typescript
//
interface GsfClientInfo {
  host: string;
  port: string | number;
  alias?: string;
  balances?: BalanceItem[];
}

interface BalanceItem {
  auCode: string; // 资金账号
  auName: string; // 资金账号名称
  currency: string; // 币种
  type: number; // 账户类型
  netAsset: number; // 净资产
  totalAssetInitial: number; // 日初总资产
  totalAsset: number; // 总资产
  equityInitial: number; // 日初持仓市值
  equity: number; // 持仓市值
  equityInTransit: number; // 在途市值
  equityDeposit: number; // 转入证券市值
  equityWithdraw: number; // 转出证券市值
  netEquityTraded: number; // 净买入证劵市值
  equityBuy: number; // 买入证劵市值
  equitySell: number; // 卖出证劵市值
  fundInitial: number; // 日初资金
  fundDepositWithdraw: number; // 净出入金
  fundDeposit: number; // 入金（银证转入）
  fundWithdraw: number; // 出金（银证转出）
  fundAvailable: number; // 可用资金
  fundInTransit: number; // 在途资金
  fundFrozen: number; // 冻结资金
  balance: number; // 资金余额/总资金
  totalLiabilityInitial: number; // 日初总负债
  totalLiability: number; // 总负债
  cashDebtInitial: number; // 日初资金负债
  cashDebt: number; // 资金负债
  securityDebtInitial: number; // 日初证券负债
  securityDebt: number; // 证券负债
  commission: number; // 手续费
  createTime: number; // BigInt
  updateTime: number; // BigInt
  settleTime: number; // BigInt
  tradeDate: number; // BigInt
  totalMarket?: number; // 总市值, 业务自定义计算
}
```

## 备注

### 解决：`zmq4.go:1167:34: could not determine kind of name for C.zmq_msg_group`

```bash
git clone https://github.com/zeromq/libzmq.git
cd libzmq && git checkout v4.3.5
./autogen.sh && ./configure && make && make install
cp /usr/local/lib/libzmq.so* /lib/x86_64-linux-gnu/
```
