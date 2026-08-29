---
title: 设计模式（10）：代理——大字段懒加载、鉴权包装，和它跟装饰器的官司
published: 2026-08-30
pinned: false
description: ""
tags: [design-patterns, go, proxy, cloudshop]
category: 设计模式
draft: false
---

[第 5 篇](/posts/design-patterns-5-装饰器/)埋了根刺：装饰器和代理**结构一模一样**，怎么分？这篇正面处理它，顺便解决两个高频业务疼：重资源的延迟加载、敏感操作的访问控制。

CloudShop 商品实体的现实：标题价格是小字段，但详情富文本 200KB、视频预览 5MB。列表页一次取 50 个商品——`SELECT *` 把 250MB 的详情拖回来，列表页 P99 直接爆表。

## v0：全量加载

```go
// catalog/product.go —— CloudShop v0
type Product struct {
	ID       int64
	Title    string
	Price    int64
	Detail   string // 200KB 富文本
	VideoURL string
	// ...
}

func GetProduct(ctx context.Context, id int64) (*Product, error) {
	// SELECT * FROM product WHERE id = ? —— 详情视频全量回来
}
```

列表页只需要前三个字段，却为详情付了全部传输和反序列化成本。而详情页才真的需要那 200KB——**加载时机和消费时机错位**。

## v1：虚拟代理——首次访问才真加载

```go
// catalog/lazy.go
type ProductRef struct {
	id     int64
	once   sync.Once  // 第 3 篇的积木
	loaded *Product
	loader func(context.Context, int64) (*Product, error)
}

func (r *ProductRef) get(ctx context.Context) (*Product, error) {
	var err error
	r.once.Do(func() {
		r.loaded, err = r.loader(ctx, r.id)
	})
	return r.loaded, err
}

// 对外：和 *Product 同一个接口（这里用接口保形）
type ProductView interface {
	Title(ctx context.Context) string
	Price(ctx context.Context) int64
	Detail(ctx context.Context) (string, error) // 只有它触发真实加载
}

type LazyProduct struct{ ref *ProductRef }

func (p *LazyProduct) Title(ctx context.Context) string {
	pr, _ := p.ref.get(ctx) // 小字段也被顺带加载了？见下文拆分
	return pr.Title
}

func (p *LazyProduct) Detail(ctx context.Context) (string, error) {
	pr, err := p.ref.get(ctx) // 首次访问详情 → 才查大字段表
	return pr.Detail, err
}
```

配套的表拆分（真实做法）：`product` 主表放小字段，`product_detail` 表放大字段，`LazyProduct` 的 loader 只在 `Detail()` 时才查详情表。**调用方的代码一行没改**——它拿到的还是 `ProductView`，接口形状完全没变，但贵的成本被推迟到了真正消费它的那一刻。这就是代理的第一要义：**形状不变，控制访问的时机与条件**。

## v2：同一个结构的另外两种代理

代理的经典三连，结构相同、被控制的"东西"不同：

```go
// 保护代理：控制"谁"能访问
type MemberPrice struct {
	Inner ProductView
}

func (m *MemberPrice) Price(ctx context.Context) int64 {
	if !memberOf(ctx) { // 非会员拿到划线价
		return m.Inner.Price(ctx) * 100 / 95
	}
	return m.Inner.Price(ctx)
}

// 缓存代理：控制"何时"真的穿透到源
type CachedPrice struct {
	Inner  ProductView
	cache  *ttlcache.Cache
}

func (c *CachedPrice) Price(ctx context.Context) int64 {
	if v, ok := c.cache.Get(ctxKey(ctx)); ok {
		return v.(int64) // 命中，不打 DB
	}
	v := c.Inner.Price(ctx)
	c.cache.Set(ctxKey(ctx), v, time.Minute) // 60s 内价格可陈旧
	return v
}
```

| 代理类型 | 控制什么 | CloudShop 现场 |
|---|---|---|
| 虚拟代理 | 何时加载（延迟重成本） | `LazyProduct` 的 200KB 详情 |
| 保护代理 | 谁能访问 | 会员价的脱敏 |
| 缓存代理 | 何时穿透 | 价格 60 秒缓存 |
| 远程代理 | 在哪里（隔网络） | 下文的 `httputil.ReverseProxy` |

## 命名时刻

**代理模式：为对象提供一个替身，控制对它的访问**。关键词是**控制**——代理不增加业务行为，它守门：推迟、缓存、鉴权、转发。这也是与装饰器的分界线，单独开一节说透。

### 与装饰器的官司，正式判决

[第 5 篇](/posts/design-patterns-5-装饰器/)说过两者结构相同（实现同一接口 + 包着同类），判断标准就一句话：

> **包住之后"拦了什么"是代理，"加了什么"是装饰器。**

- `CachedPrice` 拦截了"打到 DB 的访问"——本来要发生的查询被它挡下了，这是**控制访问**
- [第 5 篇](/posts/design-patterns-5-装饰器/)的 `AuditedRisk` 没拦任何东西，风控照常执行，它**附加**了审计行为，这是**加行为**

实践里的灰区是诚实的：`CachedPrice` 也可以说"加了缓存行为"。争议模式（连 GoF 都承认两者同构）的正确打开方式不是背判词，而是**用意图给读代码的人发信号**：命名叫 `XxxProxy` 就在说"我在守门"，叫 `XxxDecorator`/`XxxMiddleware` 就在说"我在加料"。同一结构，命名即文档。

## 标准库里的落地

**`database/sql.DB`——最著名的虚拟代理。** `sql.Open` 从不连接数据库——它只造了一个"数据库的替身"。第一次 `Query` 才真正建连，之后由连接池代理每次请求的取还。**Open 的返回值本身就是代理**：调用方以为拿着数据库，实际拿着一个懒加载的门卫。这个设计让"检查配置文件里 50 个 DSN"的启动逻辑不会真的连 50 个库。

**`httputil.ReverseProxy`——官方远程代理。** 它站在用户和后端之间：转发请求、改写 Host、处理 hop-by-hop 头、失败换目标。生产里"入口域名 → 内部服务"的每一次转发都是它。源码本身是学习代理的教材（`net/http/httputil/reverseproxy.go`，也是 Go 标准库里少见的直接以 Pattern 命名的大件）。

## 业务实战

CloudShop 商品读链路的最终形态——**代理叠代理**，从外到内：

```
请求 → CachedPrice（缓存代理，60s）
         → MemberPrice（保护代理，会员脱敏）
            → LazyProduct（虚拟代理，大字段懒加载）
               → 真实查询
```

三层都是 `ProductView`，装配顺序在依赖注入处一眼看清（[第 18 篇](/posts/design-patterns-18-依赖注入/)的现场预告）。真实世界对照：各电商商品详情页的"多级缓存"（本地缓存 → Redis → DB）本质就是代理链，每层控制一级穿透条件。

缓存代理的代价要单独记账：**60 秒内的价格陈旧是产品决策不是技术决策**。改价场景（秒杀开抢瞬间）必须绕开缓存直读或主动失效——代理模式管控制，管不了业务对一致性的要求，这条线要写进装配处的注释。

## 好处与代价

| 好处 | 代价 |
|---|---|
| 重成本推迟到真实消费，列表/详情各取所需 | 首次访问的延迟尖刺（冷启动；可预热） |
| 鉴权、缓存、懒加载互不纠缠，各自一层 | 调用方对真实成本无感（以为便宜，实际穿透三层） |
| 接口保形，调用方零改动 | 调试多两层跳转 |
| 失败换目标（远程代理）天然支持 | 缓存陈旧窗口 = 产品决策（技术挡不了锅） |

## 什么时候不要用

- **字段都便宜**：几十字节的实体懒加载是自找间接层。
- **没有访问控制的真实需求**：为了"以后可能要鉴权"包一层空代理，是提前支付永不到账的税。
- **一致性要求高**：库存这种强一致数据套缓存代理，读到的旧值会直接造成超卖——超卖事故里缓存代理常是共犯。
- **直连更简单**：内网服务调用套远程代理转发，多一跳网络没有换来任何控制。

## 易混淆

**代理 vs [装饰器](/posts/design-patterns-5-装饰器/)**：本文主菜——拦 vs 加；结构同，意图异，命名即文档。

**代理 vs [适配器](/posts/design-patterns-6-适配器/)（第 6 篇）**：代理**保形**（同一接口），适配器**变形**（接口转换）。`LazyProduct` 和真品是同一个 `ProductView`；`WechatAdapter` 两边是两套接口。

**代理 vs [外观](/posts/design-patterns-7-外观/)（第 7 篇）**：代理一对一（替身换本尊），外观一对多（一个门进多个子系统）。

## 自测

1. `sql.Open` 不连库这个设计，替调用方避免了什么具体场景的坑？第一次 `Query` 时连接池代理做了哪三件事？
2. 用"拦了什么/加了什么"标准，裁决 `CachedPrice`、`MemberPrice`、`AuditedRisk`（第 5 篇）各是代理还是装饰器，说明理由和灰区。
3. 缓存代理引入的 60 秒陈旧窗口，为什么说"产品决策挡不住技术执行"？秒杀改价场景怎么处理这层代理？
4. 三层代理的装配顺序（Cache→Member→Lazy）如果调成 Lazy→Cache→Member，行为有什么变化？哪种装配更合理？

---

**参考来源**

- GoF, *Design Patterns* — 代理三型（虚拟/保护/远程），与装饰器同构的说明
- `database/sql` 连接池文档、`net/http/httputil/reverseproxy.go` 源码 — 标准库代理双雄
- Refactoring.guru, [Proxy](https://refactoring.guru/design-patterns/proxy)
