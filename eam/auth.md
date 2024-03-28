# 用户中心

## 权限管理三期

> 2024-02-07

权限设计二期主要完成了针对金葵花服务的权限配置，其业务流程如下

1. 管理员先新建金葵花服务，接着配置哪些用户可以访问该金葵花服务（选择服务模块为 accessToken）。
2. 用户在 QuantWeb 中的访问令牌中，可以选择生成能访问哪些金葵花服务的访问令牌。
3. 在 nacos 中按照金葵花服务名保存一个 token 能够访问的文件 authorized_keys.yml。（金葵花服务会实时读取最新 authorized_keys.yml 来进行权限控制）

设计实现上可以分为两个部分“用户资源访问控制” 和 “令牌资源访问控制”。由于令牌由用户产生，所以对用户的资源访问控制能够影响到令牌的资源访问。权限管理二期已经完成了逻辑闭环，具体可以参看*附录-权限设计二期*。涉及的表如下

- gsfsrv{id, namespace, namespace_id, group, comment} 保存金葵花服务信息
- authcode{id, appid, appsecret, name, userid, remark, aucodes 可访问服务列表，json 格式`{obj,mod,act}`} 保存生成的令牌信息
- casbin_rbac{p, sub, obj, mod, act} | {g, sub, group} 用户资源访问控制信息，其中有用户组、资源组的概念，方便对用户进行资源的批量授权。
- casbin_authcode{p, sub, obj, act} 令牌资源访问控制信息

二期设计有几点原则假设

1. 生成的令牌的可访问服务不允许自动扩大授权范围（可访问服务选择“\*(用户所有)”场景，即代表此刻的用户所有）

在权限管理三期中，希望能够对金葵花监控页面中的服务也能控制是否能够访问，以及在结算明细报表中，按产品/按投资经理/按策略中能够控制用户能够展示的单元。这些属于 QuantWeb 页面上的展示数据权限。页面访问性只需要分为可访问和不可访问两种，不存在更细粒度的访问方式（如读/写等）。另外，页面的路由最多分为三级，页面内存在 tab 页（比如结算明细报表中的“按产品/按投资经理/按策略”；金葵花监控页的“生产/访问/测试”），tab 页下可能有多个单元 unit 的选择（比如结算明细报表中按产品 tab 中的“产品/资金账号/资产单元”；金葵花监控页的生产 tab 中有金葵花服务）。每个 unit 都有一堆 tab 页（称之为 模块 mod）可以查阅（比如金葵花监控中的“目标/持仓/委托/成交/划拨/配置”；结算明细表中的“资产概要/持仓明细/资产对账/持仓对账等”）。采取该种设计需要添加多一张表 casbin_quantweb，其结构为 casbin_quantweb {p, sub, url, tab, unit, mod, act} | {g, sub, group}。

再进一步抽象，其实 casbin_quantweb 中 url+tab+unit 可以看成是 casbin_rbac 中的 obj 字段，如此以来就可以将这两表进行整合。原本的 casbin_rbac 的 obj 字段内容格式直接就是金葵花服务名（类似：KlindDs:v1.0.3:192.168.15.42:prod），那么现在 obj 字段需要采用两套命名（其中`<>`表示变量）：`GSF:<服务名>:<版本号>:<ip>:<环境>` 和 `QWEB:<url>:<tab>:<unit>`。举例如下：

```
# 金葵花服务 GSF:<服务名>:<版本号>:<ip>:<环境>
GSF:KlindDs:v1.0.3:192.168.15.42:prod

# quantweb页面权限 QWEB:<url>:<tab>:<unit>
结算明细报表按产品中产品 QWEB:/operation/settlement/detail:product:DRWGJH
结算明细报表按产品中资金账号 QWEB:/operation/settlement/detail:product:0650020666
结算明细报表按产品中资产单元 QWEB:/operation/settlement/detail:product:DRWGJHZS_01
结算明细报表

```

## 附录

### nacos-sdk-go 使用问题&解决

##### 1. 定位问题 nacos-sdk-go 调用注册到 nacos 的服务器 http 接口时报 `code = 503 reason = NODE_NOT_FOUND message = error: code = 503 reason = no_available_node message =  metadata = map[] cause = <nil> metadata = map[] cause = <nil>`

客户端通过 nacos 去调用服务器的 http 方式接口时，会出现问题 `code = 503 reason = NODE_NOT_FOUND message = error: code = 503 reason = no_available_node message =  metadata = map[] cause = <nil> metadata = map[] cause = <nil>`，问题定位如下。解决办法有两种，第一种是采用 grpc 去访问（推荐）；第二种是手动取服务节点转换成最终的 url(比如 `http://127.0.0.1:8000`)去访问。

```go
// client/main.go
conn, err := transhttp.NewClient( // NewClient returns an HTTP client.
  context.Background(),
  transhttp.WithEndpoint("discovery:///srv1.http"), // 服务名
  transhttp.WithDiscovery(r), // r: nacos registry
)

// kratos/v2@v2.7.1/transport/http/client.go
func NewClient(ctx context.Context, opts ...ClientOption) (*Client, error) {
  // ...
  // options {discovery: r (nacos registry above), block: false}
  // target {Scheme: "discovery", Endpoint: "srv1.http"}
  selector := selector.GlobalSelector().Build()
  var r *resolver
  if options.discovery != nil {
		if target.Scheme == "discovery" {
			if r, err = newResolver(ctx, options.discovery, target, selector, options.block, insecure, options.subsetSize); err != nil {
				return nil, fmt.Errorf("[http client] new resolver failed!err: %v", options.endpoint)
			}
		}
	}
  return &Client{
		opts:     options,
		target:   target,
		insecure: insecure,
		r:        r,
		cc: &http.Client{
			Timeout:   options.timeout,
			Transport: options.transport,
		},
		selector: selector,
	}, nil
}

// kratos/v2@v2.7.1/transport/http/resolver.go
func newResolver(ctx context.Context, discovery registry.Discovery, target *Target,
	rebalancer selector.Rebalancer, block, insecure bool, subsetSize int,
) (*resolver, error) {
	watcher, err := discovery.Watch(ctx, target.Endpoint) // target.Endpoint = srv1.http
	r := &resolver{
		target:      target,
		watcher:     watcher,
		rebalancer:  rebalancer, // assign directly: selector.GlobalSelector().Build()
		insecure:    insecure,
		selecterKey: uuid.New().String(),
		subsetSize:  subsetSize,
	}
	go func() {
		for {
			// Watcher.Next returns services in the following two cases:
			// 1.the first time to watch and the service instance list is not empty.
			// 2.any service instance changes found.
			// if the above two conditions are not met, it will block until context deadline exceeded or canceled
			// 这是 watcher.Next() 的官方说明，所以会阻塞在该行
			services, err := watcher.Next()
			if err != nil {
				if errors.Is(err, context.Canceled) {
					return
				}
				log.Errorf("http client watch service %v got unexpected error:=%v", target, err)
				time.Sleep(time.Second)
				continue
			}
			r.update(services) // 如果能拿到services就更新resolver
		}
	}()
	return r, nil
}

func (r *resolver) update(services []*registry.ServiceInstance) bool {
	// ServiceInstance{ID, Name, Version, Metadata: map[string]string, Endpoints: []string}
	// Rebalancer is nodes rebalancer.
	// Rebalancer.Apply is apply all nodes when any changes happen
	r.rebalancer.Apply(nodes)
	// 将 services 中的 Endpoints 转换成 url，接着转换成 nodes，进行应用，均衡器会选择某个node进行访问，
	// 但是在上面调用处已经阻塞，根本不会有nodes注册到 rebalancer 中。这导致在选择node访问时，其数组为空，
	// 导致报错。
	return true
}

// kratos/v2@v2.7.1/transport/http/http
// 调用栈
// - client.SayHello(context.Background(), &v1.HelloRequest{Name: "http yes!"})
// - err := c.cc.Invoke(ctx, "GET", path, nil, &out, opts...)
// - client.invoke(ctx, req, args, reply, c, opts...)
// - res, err := client.do(req.WithContext(ctx))
func (client *Client) do(req *http.Request) (*http.Response, error) {
	var done func(context.Context, selector.DoneInfo)
	if client.r != nil {
		var (
			err  error
			node selector.Node
		)
		// Selector is node pick balancer.
		// Selector.Select nodes. if err == nil, selected and done must not be empty.
		if node, done, err = client.selector.Select(req.Context(), selector.WithNodeFilter(client.opts.nodeFilters...)); err != nil {
			// 报错 reason = NODE_NOT_FOUND message = error: code = 503 reason = no_available_node message =  metadata = map[] cause = <nil> metadata = map[] cause = <nil>
			return nil, errors.ServiceUnavailable("NODE_NOT_FOUND", err.Error())
		}
	}
}
```

##### 2. `rpc error: code = DeadlineExceeded desc = context deadline exceeded` 问题解决

客户端每次运行前都需要将 cache/naming 中的内容删除，否则无法启动显示: rpc error: code = DeadlineExceeded desc = context deadline exceeded

##### 3. nacos 调研进度

注册中心

- [x] 理想情况下的服务注册与访问（grpc、http）
- [ ] 注册服务的访问权限控制
- [ ] 服务可用性监测

配置中心

- [x] 服务启动时读取 nacos 配置
- [ ] 服务运行中实时同步最新的 nacos 配置

### 访问令牌一期

> 时间: 2023-11-18

访问令牌一期的目标是提供新的登录方式，用该方式代替用户名密码登录方式，降低密码泄露的风险。具体使用上，只需要提供 appid 和 appsecret 即可登录，产生对应的用户 token 信息。appsecret 为随机生成的长度为 50 的字符串，该字符串保证全局唯一。

### 权限设计一期调研

> 时间: 2023-11-23

目前存在两个需求点：1. 资产单元的权限管理; 2. 微服务架构下服务的权限管理;

_资产单元的权限管理_

即 right 用户的 right 模型在 right 资产单元进行下单交易。那么就需要考虑如下几个问题：

1. 如何保证 right 用户? 即资产单元允许哪些用户进行访问（资产单元的权限管理）
2. 如何保证 right 模型? 模型是用户自己创建的，是否为正确的模型是由用户进行管理，平台可以提供一套机制协助进行管理。

casbin 提供了 RBAC 的权限设计方案，可以将用户作为 sub，资产单元作为 obj，资产单元访问方式作为 act；另外还需要有`角色/组`的概念，`角色/组` 与资产单元进行绑定，表示该`角色/组`可以访问哪些资产单元，而用户可以跟`角色/组`进行绑定，表示该用户可以访问该`角色/组`中所有资产单元。这样 casbin 就帮助我们解决第一个问题了（如何保证 right 用户?）；第二个问题由访问令牌进行解决，用户自行选择对可访问的资产单元并输入 appid 和 expires 后，生成对应的 appsecret，然后在模型登录时提供 appid 和 appsecret 进行鉴权即可。

- authcode{id, appid, appsecret, expires, aucodes, allow, userid}

整个流程如下

1. 管理员先配置好用户可访问的资产单元列表
1. 用户生成某个资产单元（au1）的访问令牌 appsecret，另外也可以生成能访问所有可访问资产单元（\*）的令牌 appsecret
1. 用户编写的模型要访问资产单元进行下单前，需要提供 appid, appsecret, aucode 进行鉴权，鉴权通过之后会生成 jwt 信息
1. 后续访问携带 jwt 信息访问

**实验测试**

管理员设置了资产单元的访问规则如下。为了更好的进行资产单元的管理，将资产单元按组为单位进行划分后，再将整个组分配给特定用户。

```text
==== policy.csv 内容 ====
# 产品 & 资产单元对应关系
p, PRODUCT_EAMLS1, AU_300016, *
p, PRODUCT_EAMLS1, AU_88853899_ww, r
p, PRODUCT_EAMLS1, AU_EAMLS1ZT_00, w
p, PRODUCT_EAMLS1, AU_EAMLS1ZTX_00, *
p, PRODUCT_DRW004, AU_121000, *

# 投资经理 & 资产单元对应关系
p, MANAGER_WW, AU_0148P1016_ww, *
p, MANAGER_WW, AU_88853899_ww, r
p, MANAGER_WW, AU_DRWZQ1ZT_03, w
# p, MANAGER_WSY, AU_DRW001ZTX_04, *

# 临时配置某个用户对某个资产单元的配置
p, USER_wsy, AU_DRW001ZTX_04, *

# 定义关联: 用户 - 可访问的资产单元(组)
g, USER_ww, MANAGER_WW
g, USER_xjw, PRODUCT_EAMLS1
g, USER_yrl, MANAGER_WW
# g, USER_wsy, MANAGER_WSY


==== policy.csv 解读 ====
用户ww可以访问 MANAGER_WW组中的资产单元（MANAGER_WW组 = {0148P1016_ww, 88853899_ww, DRWZQ1ZT_03}）
用户xjw可以访问 PRODUCT_EAMLS1组中的资产单元（PRODUCT_EAMLS1组 = {300016, 88853899_ww, EAMLS1ZT_00, EAMLS1ZTX_00}）
用户wsy可以访问资产单元DRW001ZTX_04
用户yrl可以访问 MANAGER_WW组中的资产单元
```

场景设计如下，样例中所说的成功/失败表示预期的鉴权结果（成功：鉴权通过；失败：鉴权不通过）

1. 用户 ww 生成*只能*访问资产单元`[0148P1016_ww]`的访问令牌，并访问资产单元`0148P1016_ww` —— 成功
2. 用户 ww 生成*只能*访问资产单元`[0148P1016_ww, 88853899_ww]`的访问令牌，并访问资产单元`0148P1016_ww` —— 成功
3. 用户 ww 生成*只能*访问资产单元`[0148P1016_ww]`的访问令牌，并访问资产单元`88853899_ww` —— 失败，该令牌没有访问`88853899_ww`的权限
4. 用户 ww 生成可访问*所有*资产单元（`MANAGER_WW组`）的访问令牌，并访问资产单元`88853899_ww` —— 成功
5. 用户 ww 生成可访问*所有*资产单元（`MANAGER_WW组`）的访问令牌，并访问资产单元`EAMLS1ZT_00` —— 失败，用户 ww 没有访问`EAMLS1ZT_00`的权限
6. 用户 ww 生成*不能*访问资产单元`[0148P1016_ww]`的访问令牌，并访问资产单元`0148P1016_ww` —— 失败
7. 用户 ww 生成*不能*访问资产单元`[0148P1016_ww]`的访问令牌，并访问资产单元`88853899_ww` —— 成功，`88853899_ww` 在 MANAGER_WW 组中，但不在不能访问的列表中
8. 用户 ww 生成*不能*访问资产单元`[0148P1016_ww]`的访问令牌，并访问资产单元`EAMLS1ZT_00` —— 失败，用户 ww 没有访问`EAMLS1ZT_00`的权限
9. 用户 wsy 生成*只能*访问资产单元`[DRW001ZTX_04]`的访问令牌，并访问资产单元`DRW001ZTX_04` —— 成功
10. 用户 xjw 生成*只能*访问资产单元`[EAMLS1ZT_00]`的访问令牌，并访问资产单元`EAMLS1ZT_00` —— 成功

代码实现上，设计了两张表来维护数据，分别为*访问令牌表*和*用户表*，表结构和上述实验的数据如下所示。

```text
用户表 User {Id, UserName, Mobile}
访问令牌表 AccessToken {Id, AppId, AppSecret, UserId, AuCodes, Allow, Expires}

== 实验数据 ==
用户表 User 数据:
[
	{Id: 15739, UserName: "ww", Mobile: "15308681364"},
	{Id: 15743, UserName: "xjw", Mobile: "13608681364"},
	{Id: 15747, UserName: "wsy", Mobile: "13708681364"},
]

访问令牌表 AccessToken 数据:
[
	// ww(id:15739) has generated authToken to access 0148P1016_ww
	{Id: 1, AppId: "asdj", AppSecret: "d54sdfejbd561sa", UserId: 15739, AuCodes: ["0148P1016_ww"], Allow: true, Expires: "Seven days later"},

	// xjw(id:15743) has generated authToken to access EAMLS1ZT_00
	{Id: 2, AppId: "kfuks", AppSecret: "4fd1ufklnksbry9", UserId: 15743, AuCodes: ["EAMLS1ZT_00"], Allow: true, Expires: "Seven days later"},

	// ww(id:15739) has generated authToken to access all au which ww can access.
	{Id: 3, AppId: "jkwsx", AppSecret: "luwxtuf5twprw5l", UserId: 15739, AuCodes: ["*"], Allow: true, Expires: "Seven days later"},

	// ww(id:15739) has generated authToken to access all au which ww can access except 0148P1016_ww.
	{Id: 4, AppId: "ggTks", AppSecret: "psuhl055bwaeTIjk", UserId: 15739, AuCodes: ["0148P1016_ww"], Allow: false, Expires: "Seven days later"},

	// ww(id:15739) has generated authToken to access [0148P1016_ww, 88853899_ww]
	{Id: 5, AppId: "xstt", AppSecret: "abeo5tgrt754arh57", UserId: 15739, AuCodes: ["0148P1016_ww", "88853899_ww"], Allow: true, Expires: "Seven days later"},

	// wsy(id:15747) has generated authToken to access DRW001ZTX_04
	{Id: 5, AppId: "ko8w", AppSecret: "8hw416ery9ah4foig", UserId: 15747, AuCodes: ["DRW001ZTX_04"], Allow: true, Expires: "Seven days later"},

	// xjw(id:15743) has generated authToken to access EAMLS1ZT_00
	{Id: 5, AppId: "eut2", AppSecret: "tyt1ra48is13awer6", UserId: 15743, AuCodes: ["EAMLS1ZT_00"], Allow: true, Expires: "Seven days later"},
]
```

_微服务架构下服务的权限管理_

用 nacos 作为服务注册&发现中心，各个`资产单元`和`用户中心`都会在 nacos 进行注册。当用户的某个模型需要在资产单元`Au1`中下单时，带上 appid 和 appsecret，`Au1`会去访问`用户中心`的接口进行鉴权，当鉴权通过之后，则将该对 appid 和 appsecret 保存在内存中，下次如果再遇到该对时就不用再去用户中心鉴权而直接放行。这样的设计会导致

### 权限设计一期

> 时间: 2024-01-12

在*权限设计一期调研*中，已经完成 casbin 进行鉴权的过程，并且设计了 GSF 进行鉴权的方式方法。但是该套方案在讨论之后发现存在延迟高和引用服务过多的问题，~~所以目前设计的方案为：用户中心管理并维护用户可访问资产单元列表的数据，并生成每个资产单元各自的*用户访问权限信息表*推送到 nacos；GSF 只需要读取并监听 nacos 中对应的资产单元的*用户访问权限信息表*即可。当有用户需要在 gsf 中下单时，首先 gsf 会从请求的 token 中获取到用户的信息，接着去*用户访问权限信息表*查看该用户是否有权限，即可完全鉴权。~~需要调研如下内容。

- [x] casbin 鉴权的单元测试改造。在*调研 v1.0*中单元测试写的比较分散并且可读性不高的问题。由于*调研 v1.0*中鉴权的起点是 appid 和 appsecret，而在该版本中起点是 token，所以原本的测试用例也需要调整。
- [x] 通过接口修改权限信息。casbin 在启动时会读取 policy.csv 文件的权限配置，在运行过程存在需要更新某些权限配置。
- [x] nacos 上传数据。用户中心将 casbin 保存的权限配置根据各个资产单元生成一份*用户访问权限信息表*上传到 nacos 中。
- [x] 设计上传到 nacos 中数据内容。gsf 读取 nacos 对应的权限信息后自行完成请求用户的权限校验。

在上述的设计方案中，前提条件是 gsf 服务已经知道用户的信息，而这个过程是通过解析 token 得到，虽然能够完成鉴权，但不足够。因为只要有 token 就可以进行访问了，但实际述求中需要用户可以自行配置哪些 key 能够访问某些 gsf 服务。所以其实权限控制分为两块，一块是*管理员*维护的用户能访问的服务列表；另一块是*用户*自己维护的 appsecret 能访问的服务列表。所以整体的工作流程是如下所示。

1. 管理员通过维护 casbin 的 policy.csv 来管理`用户&服务`之间的映射关系。
2. 用户通过页面维护 key pair `<appid, appsecret>` 与 服务的映射关系，即哪些 key pair 可以访问哪些服务。
3. 用户中心 uc 会提供对应的 api 给管理员和用户去进行 1 和 2 的维护，并且在 nacos 的配置中心中，按照服务生成一张可以访问该服务的 appsecret。（appsecret 需要保持全局唯一）
4. gsf 服务启动时会拉取 nacos 中对应的可访问 appsecret 表。用户发送的请求要求携带 appsecret 信息，此时 gsf 校验用户携带的 appsecret 信息是否在可访问的 appsecret 列表中即可判断是否合法。
5. 当管理员和用户修改了权限信息，uc 都会将最新的 appsecret 列表信息发送到 nacos 中，gsf 服务也会一直监听 nacos 中的变化，一直维护最新的 appsecret 列表。

用户权限维护使用 casbin 进行负责，数据存放在 mysql 的表（默认表名为 casbin_rule）中，字段有 p_type, v0, v1, v2, v3, v4, v5。在项目*启动时*会加载该表的配置，后续如果需要改动只能通过接口去改动。使用方式如下

- 例 1：给用户 ww（uid: 1523580757186973696）添加访问*资源* EAM101:v1:ip:test 的权限

```text
step1: 调用 POST /api/uc/v1/gsfsrv/policy/add, body中携带参数如下所示。需要注意的是用户uid前面需要加前缀 USER:。
{
	"tpe": "p",
	"sub": "USER:1523580757186973696",
	"obj": "EAM101:v1:ip:test"
}
如果返回如下结果，则表示添加权限成功；否则为失败。
{
    "code": 0,
    "msg": "",
    "data": {
        "@type": "type.googleapis.com/api.gsfsrv.v1.CreatePolicyReply",
        "ok": true
    }
}

step2: 调用查询接口可以查到上述插入的记录 POST /api/uc/v1/gsfsrv/policy/list, body中携带参数如下所示。
{
	"tpe": ["p"],
	"sub": ["USER:1523580757186973696"],
	"obj": ["EAM101:v1:ip:test"]
}
如果返回如下结果，则确认添加权限无误；否则服务有问题。
{
    "code": 0,
    "msg": "",
    "data": {
        "@type": "type.googleapis.com/api.gsfsrv.v1.ListPolicyReply",
        "result": [
            {
                "tpe": "p",
                "v0": "USER:1523580757186973696",
                "v1": "EAM101:v1:ip:test",
                "v2": "*"
            }
        ],
        "count": 1
    }
}
```

- 例 2：给用户 ww（uid: 1523580757186973696）添加访问*资源组* EAM 的权限

```text
step1: 调用 POST /api/uc/v1/gsfsrv/policy/add, body中携带参数如下所示。需要注意的是用户uid前面需要加前缀 USER:， 资源组需要携带前缀 SRCGROUP:。
{
	"tpe": "g",
	"sub": "USER:1523580757186973696",
	"obj": "SRCGROUP:EAM"
}
如果返回如下结果，则表示添加权限成功；否则为失败。
{
    "code": 0,
    "msg": "",
    "data": {
        "@type": "type.googleapis.com/api.gsfsrv.v1.CreatePolicyReply",
        "ok": true
    }
}

step2: 调用查询接口可以查到上述插入的记录 POST /api/uc/v1/gsfsrv/policy/list, body中携带参数如下所示。
{
	"tpe": ["g"],
	"sub": ["USER:1523580757186973696"],
	"obj": ["SRCGROUP:EAM"]
}
如果返回如下结果，则确认添加权限无误；否则服务有问题。
{
    "code": 0,
    "msg": "",
    "data": {
        "@type": "type.googleapis.com/api.gsfsrv.v1.ListPolicyReply",
        "result": [
            {
                "tpe": "g",
                "v0": "USER:1523580757186973696",
                "v1": "SRCGROUP:EAM",
                "v2": ""
            }
        ],
        "count": 1
    }
}
```

- 例 3：给用户组 quant 添加访问*资源* EAM101:v1:ip:test 的权限

```text
step1: 调用 POST /api/uc/v1/gsfsrv/policy/add, body中携带参数如下所示。需要注意的是用户组需要带上前缀 USERGROUP:。另外说明下，如果是授权可以访问所有的资源，只需要把资源名改成 * 即可。
{
	"tpe": "p",
	"sub": "USERGROUP:quant",
	"obj": "EAM101:v1:ip:test"
}
如果返回如下结果，则表示添加权限成功；否则为失败。
{
    "code": 0,
    "msg": "",
    "data": {
        "@type": "type.googleapis.com/api.gsfsrv.v1.CreatePolicyReply",
        "ok": true
    }
}

step2: 调用查询接口可以查到上述插入的记录 POST /api/uc/v1/gsfsrv/policy/list, body中携带参数如下所示。
{
	"tpe": ["p"],
	"sub": ["USERGROUP:quant"],
	"obj": ["EAM101:v1:ip:test"]
}
如果返回如下结果，则确认添加权限无误；否则服务有问题。
{
    "code": 0,
    "msg": "",
    "data": {
        "@type": "type.googleapis.com/api.gsfsrv.v1.ListPolicyReply",
        "result": [
            {
                "tpe": "p",
                "v0": "USERGROUP:quant",
                "v1": "EAM101:v1:ip:test",
                "v2": "*"
            }
        ],
        "count": 1
    }
}
```

- 例 4：将用户 ww （uid: 1523580757186973696）添加到 用户组 quant 中

```text
step1: 调用 POST /api/uc/v1/gsfsrv/policy/add, body中携带参数如下所示。需要注意的是用户uid前面需要加前缀 USER:，用户组需要携带前缀 USERGROUP:。
{
	"tpe": "g",
	"sub": "USER:1523580757186973696",
	"obj": "USERGROUP:quant"
}
如果返回如下结果，则表示添加权限成功；否则为失败。
{
    "code": 0,
    "msg": "",
    "data": {
        "@type": "type.googleapis.com/api.gsfsrv.v1.CreatePolicyReply",
        "ok": true
    }
}

step2: 调用查询接口可以查到上述插入的记录 POST /api/uc/v1/gsfsrv/policy/list, body中携带参数如下所示。
{
	"tpe": ["g"],
	"sub": ["USER:1523580757186973696"],
	"obj": ["USERGROUP:quant"]
}
如果返回如下结果，则确认添加权限无误；否则服务有问题。
{
    "code": 0,
    "msg": "",
    "data": {
        "@type": "type.googleapis.com/api.gsfsrv.v1.ListPolicyReply",
        "result": [
            {
                "tpe": "g",
                "v0": "USER:1523580757186973696",
                "v1": "USERGROUP:quant",
                "v2": ""
            }
        ],
        "count": 1
    }
}
```

特别说明：

- 如果是想给赋予所有的资源的访问权限，只需要添加一个资源为 `*` 的记录即可，例如：`{tpe: "p", sub: "xxx", obj: "*"}`
- 资源的命名是用户自行定义，用户、用户组、资源组的命名的规则是必须携带上对应的前缀，分别为`USER:`、`USERGROUP:`、`SRCGROUP:`

casbin 的 model.conf 内容如下

```conf
[request_definition]
r = sub, obj, act

[policy_definition]
p = sub, obj, act

[role_definition]
g = _, _

[policy_effect]
e = some(where (p.eft == allow))

[matchers]
m = g(r.sub, p.sub) && (r.obj == p.obj || p.obj == '*') && (r.act == p.act || p.act == '*')
```

由于 nacos 上会按照命名空间 namespaceId 和 groupId 进行保存每一个服务可以访问的 appsecret 的列表。比如服务 EAM001:v1.0.1:192.168.15.42:prod 在 nacos 中会有一个组（Group） EAM001:v1.0.1:192.168.15.42:prod，该组有 index 文件存放所有该服务的配置文件列表，而权限的配置文件（Data Id）叫做 authorized_keys.yml。内容结构如下所示。

```yaml
server: bards:1.2.0:192.168.15.33:dev
secrets:
  - appid: test
    appsecret: yFfl8GWRRGc5S3LKY3a6FsmSQJB9R47vR7rBmrwAYIEnlH297j
    uid: 1506439972247310336
    privilege: rw
  - appid: tttst
    appsecret: OzmeWJipI5TR2R41rWEdZjx6ejDAedWv3xu13UrtkIox4OJiss
    uid: 1547818295636267008
    privilege: rw
```

其中 `secrets` 是能够访问该服务的 secret 列表。能够导致列表变化的情况有两种，一是管理员调整*用户-用户组-资源组-资源*之间的关联关系，二是用户增删访问令牌时。往 nacos 推送 secret 列表数据发生在 uc 项目刚启动时 和 列表数据发送变化时 两种情况。

### 权限设计二期

> 时间: 2024-02-07

在*权限设计一期*中，利用 casbin 已经完成用户对服务的访问控制，以及用户令牌对服务的访问控制。因为该权限仅仅到服务这一层级，而服务下存在多个模块（如配置文件模块），需要对这些模块进行更细粒度的访问。并且，目前对于某个服务只能设置是否访问，也需要进一步细化是什么访问类型（如可读`r`、可写`w`、读写`*`等）。

目前的设计实体中有用户 USER、用户组 USERGROUP、资源组 SRCGROUP、资源 SRC，其中资源 SRC 对应是业务中的服务。这四个实体之间的关联关系为 `用户->用户组->资源组->资源`。

因为最新的需求中服务下需要细分模块，所以多加一个命名维度来表示模块。所以目前的策略规则命名规则为`sub, obj, mod, act`，所有情况如下。

- 用户 lvx（uid: 10068）可访问所有资源：`p, USER:10068, *, *, *`
- 用户 lvx（uid: 10068）可访问资源 srv1：`p, USER:10068, srv1, *, *`
- 用户 lvx（uid: 10068）可访问资源 srv1 的配置文件模块：`p, USER:10068, srv1, conf, *`
- 用户 lvx（uid: 10068）只可读资源 srv1 的配置文件模块：`p, USER:10068, srv1, conf, r`
- 用户 lvx（uid: 10068）只可写资源 srv1 的配置文件模块：`p, USER:10068, srv1, conf, w`
- 用户组 it 可访问资源 srv1：`p, USERGROUP:it, srv1, *, *`
- 资源组 test 可访问资源 srv1：`p, SRCGROUP:test, srv1, *, *`
- 用户 lvx（uid: 10068）归属于 用户组 it：`g, USER:10068, USERGROUP:it`
- 用户 lvx（uid: 10068）可访问资源组 test：`g, USER:10068, SRCGROUP:test`
- 用户组 it 可访问资源组 test：`g, USERGROUP:it, SRCGROUP:test`

`g`策略仅仅为了表示归属关系，情况较为简单；而`p`策略由于存在通配符`*`，在配置策略存在因为“显式”包含关系而冗余的情况。显式是指不需要通过链式（用户组/资源组）推导出来的策略关系，比如 `{sub: a, obj: xxx, mod: conf, act: r}` 与 `{sub: a, obj: xxx, mod: conf, act: *}` 之间就是存在“显式”包含关系（其中`sub: a`需要注意有三种可能的情况，分别是用户 USER、用户组 USERGROUP、资源组 SRCGROUP）。所以，在新增策略时，就需要检查该策略在 casbin 中是否存在*包含*和*被包含*的关系。对于*包含*的关系需要进行删除；而如果存在*被包含*的关系，则不能新增该策略，并返回报错，告知用户已经某某策略包含在该策略了。在处理顺序上应该先*被包含*再*包含*关系。而删除策略只允许单条精确删除。

新增策略的包含关系和被包含关系的情况如下所示，其中 sub、obj、mod、act 可以构成一条记录，mbsup 表示该记录存在的*被包含*关系，mbsub 表示该记录存在的*包含*关系。mbsub 其实比较好看出来，就是非`*`列相同就是了（简称`el` easy look）。

| id  | sub | obj | mod | act | mbsup | mbsub |
| --- | --- | --- | --- | --- | ----- | ----- |
| 1   | a1  | a2  | a3  | a4  | 2~8   | -     |
| 2   | a1  | a2  | a3  | `*` | 5,7,8 | el    |
| 3   | a1  | a2  | `*` | a4  | 6,7,8 | el    |
| 4   | a1  | `*` | a3  | a4  | 5,6,8 | el    |
| 5   | a1  | `*` | a3  | `*` | 8     | el    |
| 6   | a1  | `*` | `*` | a4  | 8     | el    |
| 7   | a1  | a2  | `*` | `*` | 8     | el    |
| 8   | a1  | `*` | `*` | `*` | -     | el    |

上述的策略配置是让管理员去管理用户的资源访问权限，用户自己也有一套自己的资源访问权限（访问令牌）。管理员可以通过配置 `mod: accessToken` 的策略去影响用户能访问的资源权限。并且需要注意的是管理员的`*`和用户自己的`*`含义并不一样。管理员的`*`是真正意义上的所有，而用户的`*`表示该用户*能访问*的所有。用户*能访问*的所有由于受到管理员配置的策略影响，所以它是一个动态变化的量。

在目前的实现上，用户生成的访问令牌会在表 authcode 会一个字段以 json 格式存放它能够访问的服务信息，如`[{"obj":"xx", "act": "*"}]`（表示用户生成的该访问令牌可以以任何该用户*可访问类型*去访问服务 xx）。所以用户生成访问令牌的服务权限有如下情况，注意：不允许存在包含关系，即必定只会匹配一条规则。另外，真正令牌鉴权是否能够访问哪些服务是由另外一个 casbin 服务进行维护。（实际上 json 对象中还有 mod 字段，不过 mod 必然为 accessToken ）

- `[{obj: a1, act: a2}, {...}]`：特定的资源 obj、特定的访问类型 act
- `[{obj: a1, act: *}, {...}]`：特定的资源、所有的有读写访问类型，这种情况下不会再出现其他有关该资源 obj 的规则。比如 `obj: a1` 就有只有 `{obj: a1, act: *}` 一条规则（足够了，不存在冗余的）
- ~~`[{obj: *, act: a1}, {...}]`：所有的资源、特定的访问类型，这种情况下只能出现其他特定的访问类型 act~~
- ~~`[{obj: *, act: *}]`：所有的资源，所有的访问类型。只有唯一一条。~~
- ~~前三种情况的组合关系。~~

> 在跟测试讨论整理之后，在访问令牌表中，不存在 obj 为 \* 的情况，即不存在动态扩大生成的令牌的资源访问范围。页面上提供的“\*(用户所有)”选项仅仅代表此刻的所有。另外，act 的取值只有 r、w、\*，所以直接将\*理解为 rw 。

由于管理员 admin 配置`mod: accessToken`的策略时会影响到用户 user 配置了含有`*`的资源访问的访问令牌的访问资源范围。不过这不影响访问令牌那张表。访问令牌真正能够访问哪些服务，是由 casbin 服务进行维护，其记录格式为`{p, appsecret, obj, mod, act}`（p 表示策略类型，appsecret 是令牌），所以当管理员删除某个用户访问某个资源的权限时，只需要将该 casbin 服务的表更新即可（`delete where appsecret=<userid's secret> and obj=? and act=?`）；当用户删除访问令牌，只需要再多做一步删除访问令牌表中对应记录。

总结起来，主要涉及的接口及流程如下。涉及三张表 1. authcode 存放访问令牌的配置信息 2. casbin_authcode 存放访问令牌的权限访问信息 3. casbin_rbac 存放用户的权限访问信息。下面描述中 `ANY` 表示任意取值。用户配置的访问令牌能访问哪些服务是 authcode 中进行保存，但是该访问服务只能视为“历史”的。casbin_authcode 中只有访问令牌的访问信息，访问令牌能够访问哪些资源仅仅看这张表中是否相关的记录，这个表中只有 p 类型的策略; 另外，casbin_authcode 表中不允许出现 `obj=*` 或 `mod=*` 的记录（casbin_authcode 中的`*`与 casbin_rbac 的`*`含义一致，都是管理员视角下的所有）。casbin_rbac 保存用户能够访问哪些资源的权限，其中存在四个实体，并且存在`*`的通配符，表示 gsfsrv 表中的所有资源。虽然访问令牌是用户下的，但在技术实现上，casbin_rbac 和 casbin_authcode 是彼此独立的，用户的资源访问和访问令牌的资源访问是两套 casbin 鉴权系统，管理员对用户的资源访问对访问令牌的资源访问的影响关系由后端进行维护。

- 【用户】创建访问令牌

  1. 参数合法性校验：基本字段 appid, expires, remark, name; 授权对象列表（对象字段 obj, mod=accessToken, act）
  2. 策略`(obj,act)`合法性校验：
     - `(a1,a2)`：去 casbin_rbac 中查询该用户 u 有无权限 `{sub:u, obj:a1, mod:accessToken, act:a2}`
     - `(a1,*)`：去 casbin_rbac 中查询该用户 u 有无权限 `{sub:u, obj:a1, mod:accessToken, act:ANY}`，这种情况页面不会展示，所以后端需要校验是否存在至少一条。
     - `(*,a2)`：去 casbin_rbac 中查询该用户 u 有无权限 `{sub:u, obj:ANY, mod:accessToken, act:a2}`，这种情况下暂时不存在也允许创建
     - `(*,*)`：去 casbin_rbac 中查询该用户 u 有无权限 `{sub:u, obj:ANY, mod:accessToken, act:ANY}`，这种情况下暂时不存在也允许创建
  3. 在 authcode 中保存该访问令牌的相关信息 appid, expires, remark, name, 授权对象列表
  4. 在 casbin_authcode 添加该新生成的访问令牌 secret 可访问的资源（步骤 2 中所命中的策略）的策略 `secret -> {obj, mod, act}[]`。casbin_authcode 表中的 mod 字段暂时不启用（所有都是 `*`）
     - `(a1,a2)`：需要添加一条特定的记录 `{sub:secret, obj:a1, mod:*, act:a2}`
     - `(a1,*)`：需要添加一批特定的记录 `{sub:secret, obj:a1, mod:*, act:ANY}`
     - `(*,a2)`：需要添加一批特定的记录 `{sub:secret, obj:*, mod:*, act:a2}`
     - `(*,*)`：需要添加一批特定的记录 `{sub:secret, obj:*, mod:*, act:*}`
  5. 更新 nacos 上对应服务的 authorized_keys.yml：根据 obj 进行全量刷新：在 casbin_authcode 表通过 obj 过滤出目标 secret 及访问方式写入 nacos
  6. 事务处理：当任何一步发生错误时都需要进行回滚，当进行 sql 操作时，形成对应的相反操作。保存成 Map，key 为表，value 是具体的 sql 操作参数

- 【用户】删除访问令牌：令牌删除是用唯一键 id 进行

  1. 存在性校验：是否存在该 id
  2. 删除 casbin_authcode 中所有该访问令牌的策略 `{sub:secret, obj:ANY, mod:*, act:ANY}`
  3. 将步骤 2 中所有涉及的 obj 都需要进行 nacos 全量更新
  4. 事务处理：需要考虑在 nacos 多个文件写入的期间发生异常时的事务回滚问题，大概率网络问题

- 【管理员】创建访问策略

  1. 合法性校验：
     - 参数合法性：tpe（p/g）, sub（`USER:`/`USERGROUP:`/`SRCGROUP:`）, obj（`*`/gsfsrv.id/`USERGROUP:`/`SRCGROUP:`）, mod（conf/accessToken）, act（`*`/r/w）。
     - 存在性：(tpe, sub, obj, mod, act)
  2. 如果存在包含和被包含的情况需要进行处理。
  3. p 类型策略，该类型策略就是赋具体资源访问权限的设置。mod 若不是 accessToken 则直接插入数据库 casbin_rbac 即可；否则除了在 casbin_rbace 中添加一条记录以外，还需要考虑用户已经配置了访问令牌的影响（需要区分访问令牌中的`*`是*用户能访问*的全部）。存在三种策略配置的情况：
     - 用户-资源`(obj,act)`，而访问令牌为`<obj,act>`（根据 authcode 的授权对象列表），存在的情况如下
       1. `(a1,a2)-<a1,*>`：用户有一个以任何方式访问服务 a1 的访问令牌，此时管理员添加一条允许该用户以 a2 方式访问服务 a1 的策略。该种情况需要在 casbin_authcode 中添加一条记录 `{sub:secret, obj:a1, mod:*, act:a2}`，并刷新 nacos 中服务 a1 的 authorized_keys.yml
       2. `(a1,a2)-<*,a2>`：用户有一个以 a2 方式访问所有服务的访问令牌，此时管理员添加一条允许用户以 a2 方式访问服务 a1 的策略。该种情况需要在 casbin_authcode 中添加一条记录 `{sub:secret, obj:a1, mod:*, act:a2}`，并刷新 nacos 中服务 a1 的 authorized_keys.yml
       3. `(a1,a2)-<a3,a4>`：用户有一个以 a4 方式访问服务 a3 的访问令牌，此时管理员添加一条允许用户以 a2 方式访问服务 a1 的策略。因为必然不存在`a3==a1&&a4==a2`的情况，所以此时不需要多余操作。
       4. `(a1,a2)-<*,*>`：用户有一个以任何方式访问任何服务的访问令牌，此时管理员添加一条允许用户以 a2 方式访问服务 a1 的策略。该种情况需要再 casbin_authcode 中添加一条记录 `{sub:secret, obj:a1, mod:*, act:a2}`，并刷新 nacos 中服务 a1 的 authorized_keys.yml
       5. `(a1,*)`：注意`()`中的`*`是管理员角度的所有。
       6. `(*,a2)`：
     - 用户组`USERGROUP:`-资源：与 用户-资源 的情况类似，区别在于需要观察该用户组下所有用户的情况
     - 资源组`SRCGROUP:`-资源：与 用户-资源 的情况类似，需要考虑两种情况，有该能访问该资源组的用户的情况
  4. g 类型策略，该类型策略就是设置归属关系从而获得权限（资源组、用户组）
     - 用户-用户组，如果用户有访问令牌`<obj,act>`，存在的情况如下：
       1. `<a1,*>`：用户有一个以任何方式访问服务 a1 的访问令牌，此时管理员将用户添加到某个用户组中。如果该用户组有以任何方式访问服务 a1 的策略，则需要在 casbin_authcode 中添加对应记录并刷新 nacos 的 authorized_keys.yml 文件。
       2. `<*,a2>`：用户有一个以 a2 方式访问任意服务的访问令牌，此时管理员将用户添加到某个用户组中。如果该用户组有以 a2 方式访问的策略，则都需要在 casbin_authcode 中添加对应记录并刷新 nacos 的 authorized_keys.yml 文件。
       3. `<*,*>`：用户有一个以任何方式访问任何服务的访问令牌，此时管理员将用户添加到某个用户组中。那么则需要在 casbin_authcode 添加上该用户组能够访问的策略并刷新 nacos 的 authorized_keys.yml 文件。
     - 用户-资源组：情况同上述类似
     - 用户组-资源组：情况同上述类似

- 【管理员】删除访问策略：要求完全匹配

  1. 参数合法性校验
  2. 将 casbin_rbac 中将该记录删除，另外，如果删除的记录 mod 为 accessToken，还需要对应删除掉 casbin_authcode 中相关的记录。关联关系为 `用户组/资源组 --> 用户->访问令牌`。具体情况可以分为以下情况
     - 用户-资源`(obj,act)`：获取该用户生成的有关于该资源`(obj,act)`的访问令牌，然后在 casbin_authcode 中删除
     - 用户组-资源：同上，查询多个用户的情况
     - 资源组-资源：类似情况
     - 用户-用户组、用户-资源组、用户组-资源组：情况都类似

**特殊场景说明**

场景 1：删除策略

casbin_rbac 中有如下配置，那么用户 0001 在访问令牌中的可访问服务中可以看到 bards:0.0.1:192.168.15.42:dev 下有 bards:0.0.1:192.168.15.42:dev-r 和 bards:0.0.1:192.168.15.42:dev-w 两项目。此时用户 0001 可以生成单独的某个权限的访问令牌，比如 `(bards:0.0.1:192.168.15.42:dev,accessToken,r)` 或者 `(bards:0.0.1:192.168.15.42:dev,accessToken,w)`。也可以生成以任何方式访问资源 bards:0.0.1:192.168.15.42:dev 的访问令牌 `(bards:0.0.1:192.168.15.42:dev,accessToken,*)`，在 casbin_authcode 中的内容如下。

```
# casbin_rbac 记录
== begin ==
p, USER:0001, bards:0.0.1:192.168.15.42:dev, accessToken, r
g, USER:0001, USERGROUP:it
p, USERGROUP:it, bards:0.0.1:192.168.15.42:dev, accessToken, w
== end ==

# casbin_authcode 记录
== begin ==
# 生成 (bards:0.0.1:192.168.15.42:dev,accessToken,r) 的访问令牌
p, appsecret, bards:0.0.1:192.168.15.42:dev, accessToken, r

# 生成 (bards:0.0.1:192.168.15.42:dev,accessToken,w) 的访问令牌
p, appsecret, bards:0.0.1:192.168.15.42:dev, accessToken, w

# 生成 (bards:0.0.1:192.168.15.42:dev,accessToken,*) 的访问令牌
p, appsecret, bards:0.0.1:192.168.15.42:dev, accessToken, *
== end ==
```

此时，如果删除了 `g,USER:0001,USERGROUP:it` 的记录，此时用户 0001 不再有资源 bards:0.0.1:192.168.15.42:dev 的 w 权限。若用户 0001 有一条可以以任何方式访问 bards:0.0.1:192.168.15.42:dev 的访问令牌 `(bards:0.0.1:192.168.15.42:dev,accessToken,*)`。在 casbin_authcode 中存在的记录`p, appsecret, bards:0.0.1:192.168.15.42:dev, accessToken, *`不应该直接删除，而是变为 `p, appsecret, bards:0.0.1:192.168.15.42:dev, accessToken, r`。

目前的实现方式设置为删除策略时，取所有受影响的用户的所有 authcodes，用用户级能访问的资源和 authcodes 设定的访问资源作交集进行全量更新（原先的在 casbin_authcode 表中进行删除）。

## 参考

- [SSL/TLS 协议运行机制的概述](https://www.ruanyifeng.com/blog/2014/02/ssl_tls.html)
- [grpc-auth-support.md(grpc-go Documentation)](https://github.com/grpc/grpc-go/blob/master/Documentation/grpc-auth-support.md)
- [gRPC authentication guide](https://grpc.io/docs/guides/auth/)
- [jwt 在线解密](https://www.box3.cn/tools/jwt.html)
- [Casbin 文档](https://casbin.org/zh/docs/overview)
- [Basic Role-Based HTTP Authorization in Go with Casbin](https://zupzup.org/casbin-http-role-auth/)
- [学习分布式不得不会的 ACP 理论](https://mp.weixin.qq.com/s?__biz=MzI3NzE0NjcwMg==&mid=2650121696&idx=1&sn=d8043efa332f3f76b96f7067754f2f01&chksm=f36bb8c1c41c31d7ca5f6bd02246bdb68ea49dd178c0b007fc0ff2f11d1a625113b5cccdc908&mpshare=1&scene=1&srcid=10046deszd2jz8vupmr0vSk6#rd)
- [Nacos 详细教程](https://blog.csdn.net/Top_L398/article/details/111352983)
- [go-kratos nacos 服务注册 demo](https://github.com/go-kratos/kratos/tree/main/contrib/registry/nacos)
