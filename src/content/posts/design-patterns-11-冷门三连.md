---
title: 设计模式（11）：冷门三连——享元、抽象工厂、原型
published: 2026-08-30
pinned: false
description: ""
tags: [design-patterns, go, flyweight, abstract-factory, prototype, cloudshop]
category: 设计模式
draft: false
---

第二部分收尾。三个模式使用频率低但各有"命中即刚需"的场景：享元管内存、抽象工厂管产品族、原型管复制。合辑处理，每个走一遍压缩版流程：痛点 → 浮现 → 命名 → 落地 → 何时不用。

## 享元：十万个 SKU 的"红色 L 码"

### 痛点

SKU 的属性键值对：`{"颜色":"红","尺码":"L"}`。10 万个 SKU 里，"红"这个字符串和它的元数据（色值、多语言名）被分配了三万次。

```go
// v0：每个 SKU 各自持有完整属性
type SKU struct {
	Attrs map[string]Attr // Attr{Name:"红", Color:"#f00", I18n:...} 重复分配
}
```

内存profile 显示属性对象占了 SKU 结构的三分之一，其中九成是重复值。这不是算法问题，是**共享**问题。

### 浮现与命名

**享元模式：共享细粒度对象，减少实例数量**。关键词是**不可变**——"红"的元数据不会因为哪个 SKU 引用它而改变，不可变才有资格被共享。

```go
// catalog/attrpool.go
type Attr struct {
	Name  string
	Color string
	I18n  map[string]string
}

type AttrPool struct {
	mu    sync.RWMutex
	pool  map[string]*Attr // key: "颜色|红"
}

func (p *AttrPool) Get(kind, value string) *Attr { // 驻留（intern）
	p.mu.RLock()
	if a, ok := p.pool[kind+"|"+value]; ok {
		p.mu.RUnlock()
		return a
	}
	p.mu.RUnlock()
	// 首见：构建并入驻
	p.mu.Lock()
	defer p.mu.Unlock()
	if a, ok := p.pool[kind+"|"+value]; ok { // 双检（第 3 篇的教训）
		return a
	}
	a := buildAttr(kind, value)
	p.pool[kind+"|"+value] = a
	return a
}
```

三万个"红"变成一个 `*Attr`，10 万 SKU 各持一个 8 字节指针。

### 标准库里的落地

Go 编译器对字符串常量做驻留（同包内相同的字符串字面量共享一份底层内存）；`sync.Pool` 是享元的近亲但语义不同——**享元共享"不可变"的值（永远共享），`sync.Pool` 复用"可变"的临时对象（用完归还，随时蒸发）**。JSON 高频场景给 `bytes.Buffer` 配 `sync.Pool`，是后者；给属性值配 `AttrPool`，是前者。混用这两者是把临时对象当享元共享——数据竞争的种子。

### 何时不用

属性种类本来就少（几十个），直接建对象，内存差异不值得一层池。享元命中的标志是 profile 说话，不是直觉说话。

## 抽象工厂：多币种账务的一族对象

### 痛点

[第 8 篇](/posts/design-patterns-8-桥接/)埋的账务场景展开：CNY 和 USD 各有一套**互相配套**的对象——记账规则、精度处理、对账格式是一族，抽单个接口会破坏族内配套性（拿 CNY 的记账规则配 USD 的对账格式，账就错了）。

### 浮现与命名

**抽象工厂模式：创建**一族**相关对象，无需指定具体类**。与[第 4 篇](/posts/design-patterns-4-工厂方法/)简单工厂的分界：工厂方法造**一种**对象（按订单类型造 handler），抽象工厂造**一族配套**对象（CNY 套装或 USD 套装）。

```go
// ledger/factory.go
type LedgerFactory interface {
	Account() Account       // 账户：分账规则
	Journal() Journal       // 流水：精度与格式
	FX() RateConverter      // 汇兑：折算口径
}

type CNYLedger struct{}
func (CNYLedger) Account() Account  { return &cnyAccount{precision: 2} }
func (CNYLedger) Journal() Journal  { return &cnyJournal{thousandSep: ","} }
func (CNYLedger) FX() RateConverter { return &cnyFX{base: "CNY"} }

type USDLedger struct{ /* 一整套 USD 配套 */ }
```

调用方拿到 `LedgerFactory` 就拿到了配套的一族——**不可能混搭出错**，族约束由类型系统背书。

### 标准库里的落地

`database/sql/driver` 是标准库里的抽象工厂：每个驱动造的不是单个对象，而是一族配套的 `Conn`/`Stmt`/`Tx`/`Rows`。mysql 的 Conn 配 mysql 的 Stmt，类型签名保证了不会拿到 mysql 连接配 pg 语句。`sql.DB` 通过 driver 工厂拿到整族，桥接（第 8 篇）拿着用——**工厂造族，桥接用族**，两个模式在 `database/sql` 里正好各就各位。

### 何时不用

只有一族（或族内单件）时就是普通工厂；族成员会独立变化时（CNY 突然要配 USD 对账格式）族约束反而碍事——那是桥接的场景。**先问"族约束是真实业务规则吗"，再决定抽象工厂。**

## 原型："再来一单"的深拷贝陷阱

### 痛点

"再来一单"：复制三个月前的旧订单（含 8 个订单项、地址、发票信息）。v0 的直觉代码埋着一个 Go 最经典的坑：

```go
func CloneOrder(o *Order) *Order {
	n := *o          // 结构体浅拷贝
	n.Items = o.Items // 切片头拷贝——底层数组还是共享的！
	return &n
}

// 后果：改新单第一项数量，旧订单跟着变
newOrder.Items[0].Qty = 3
oldOrder.Items[0].Qty // 也是 3 —— 同一个底层数组
```

### 浮现与命名

**原型模式：通过复制现有对象创建新对象**。Go 没有内建深拷贝，这个模式在 Go 里就是一门**拷贝语义课**：

```go
func CloneOrder(o *Order) *Order {
	n := *o
	n.Items = make([]OrderItem, len(o.Items)) // 显式新底层数组
	copy(n.Items, o.Items)                     // 逐元素值拷贝
	n.Address = *o.Address.Clone()             // 指针字段递归克隆
	n.Coupon = nil                             // 引用型字段按业务重置：券不能复用
	return &n
}
```

三条铁律：**切片/map 拷贝的是头，必须重建**；**指针字段递归克隆**；**引用语义字段（券、支付流水）按业务置空**——"复制"从来不是纯技术动作，每类字段都要过一遍业务审查（券被复制 = 一张券当两张用）。

### 标准库里的落地

protobuf 的 `proto.Clone` 是工业级实现：按 schema 递归深拷贝，指针、切片、map 全部处理。另一个常用兜底是 JSON 序列化往返（`json.Unmarshal(json.Marshal(x))`）——省事但有暗坑：未导出字段丢失、时间格式被字符串化、性能三倍于手写克隆。低频场景（"复制上一个配置"）用它，高频路径（每单）手写。

### 何时不用

字段就三五个的简单结构直接字段赋值；包含数据库实体 ID 的对象复制后要全量重审 ID 语义——有时"按模板新建"比"复制"更诚实地表达业务。

## 自测

1. 享元和 `sync.Pool` 都是"共享"，语义差异是什么？把 `bytes.Buffer` 放进享元池会发生什么？
2. 抽象工厂和工厂方法各回应什么力？`database/sql/driver` 里两者如何分工？
3. `n := *o` 之后 `n.Items` 和 `o.Items` 是什么关系？画一下切片头的内存图。
4. "再来一单"里 Coupon 置空是技术决定还是业务决定？还有哪些字段需要同等待遇？

---

**参考来源**

- GoF, *Design Patterns* — 享元/抽象工厂/原型原始定义
- `database/sql/driver` 接口族 — 抽象工厂的标准库形态
- protobuf `proto.Clone`、`encoding/json` — 深拷贝的两种工程解
- Refactoring.guru 对应三章
