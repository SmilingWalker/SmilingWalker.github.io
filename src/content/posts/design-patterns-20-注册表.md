---
title: 设计模式（20）：注册表——名字到实现的地图，与它的黑暗面
published: 2026-08-30
pinned: false
description: ""
tags: [design-patterns, go, registry, cloudshop]
category: 设计模式
draft: false
---

[第 4 篇工厂方法](/posts/design-patterns-4-工厂方法/)预告过："工厂视角看它'造对象'，注册表视角看它'管理映射'——同一结构，两个意图，第 20 篇从另一个视角重游。"伏笔兑现。

CloudShop 长到这个规模，"名字 → 实现"的映射已经遍地都是：支付渠道表、促销类型表、事件 handler 表、报表导出格式表……每个模块各自手写一份 `map + mutex + 重复检查`，其中两份忘了重复检查——大促热加载促销策略时，同名注册 panic，带崩了整个网关进程。

## v0：四份各自为政的样板

```go
// pay/registry.go —— 第一份
var (
	payMu       sync.RWMutex
	payChannels = map[string]Channel{}
)

func RegisterChannel(name string, ch Channel) { /* mu + 存 */ }
func GetChannel(name string) (Channel, error) { /* mu + 查 */ }

// promo/registry.go —— 第二份（忘了重复检查）
// report/registry.go —— 第三份
// event/handlers.go —— 第四份
```

同一个结构抄四遍，质量参差（那两份漏掉重复检查的就是抄的时候"觉得不会发生"）。**样板该被抽掉，这是 Go 泛型的正当场景之一**。

## v1：一个泛型注册表

```go
// pkg/registry/registry.go —— 全项目共用
type Registry[T any] struct {
	mu    sync.RWMutex
	items map[string]T
}

func New[T any]() *Registry[T] {
	return &Registry[T]{items: map[string]T{}}
}

func (r *Registry[T]) Register(name string, v T) {
	r.mu.Lock()
	defer r.mu.Unlock()
	if _, dup := r.items[name]; dup {
		panic(fmt.Sprintf("registry: %q registered twice", name)) // 重复注册大声失败（第 4 篇 sql.Register 的哲学）
	}
	r.items[name] = v
}

func (r *Registry[T]) Get(name string) (T, error) {
	r.mu.RLock()
	defer r.mu.RUnlock()
	v, ok := r.items[name]
	if !ok {
		var zero T
		return zero, fmt.Errorf("registry: %q not found", name)
	}
	return v, nil
}

func (r *Registry[T]) Names() []string { /* 遍历，运维接口/调试页用 */ }
```

四份样板归一。使用处反而更清楚：

```go
var Channels = registry.New[Channel]()      // pay 包
var Promotions = registry.New[Promotion]()  // promo 包（[第 1 篇](…)策略的注册）

Channels.Register("wechat", &WechatAdapter{})
```

## 命名时刻：注册表是什么，以及它和全局变量的距离

**注册表：维护"名字 → 实现"映射、支持运行期注册与查找的受管容器**。它回应的力：**组件需要在运行期被发现**——编译期不知道会有哪些实现（插件、渠道、热加载的策略）。

"受管"两个字是它与裸全局变量的全部距离，也是它的正当性所在：

| 裸全局 map | 注册表 |
|---|---|
| 谁都能写，写坏没人知道 | Register 强制重复检查，冲突即 panic |
| 无遍历接口 | Names() 支撑调试页、健康检查、运维清单 |
| 并发裸奔 | 读写锁保护 |
| 注册了什么，靠 grep import | 注册行为集中、可观测 |

**全局可达本身不是罪，不受管的全局才是**——这句话同时回答了 [第 3 篇单例](/posts/design-patterns-3-单例/)的争议：`database/sql` 的驱动表是全局单例，却因为"受管"（Register 的三重防御）成为标准库的地基而非污点。

### 与工厂的回环（第 4 篇承诺的"另一视角"）

第 4 篇的注册表式工厂，从工厂视角看是"造对象的手段"；本篇从注册表视角看，工厂的 `map` 只是它的一层皮。两个意图各自独立成模式：

- **工厂**关心"造"：拿到名字，产出新实例（每次 `Get` 返回新造的 handler）
- **注册表**关心"管"：映射的增删查、生命周期、冲突检测（存的是实例还是构造函数，看需要）

CloudShop 的用法两态并存：支付渠道注册**构造函数**（每次支付造新请求上下文——工厂+注册表），事件 handler 注册**实例**（单例 handler 常驻——纯注册表）。同一个容器，装什么由意图决定。

## 标准库里的落地

**`net/http.DefaultServeMux`——每天在用的注册表。** `http.HandleFunc("/pay", h)` 就是 `Register("/pay", h)`：pattern 到 handler 的映射，`ServeMux` 查表分发。`http.DefaultServeMux` 那个全局变量让所有 handler 不经组装就能挂上——方便的代价是[第 3 篇](/posts/design-patterns-3-单例/)讲过的全局状态问题，所以框架（gin/chi）都改成显式持有 Router 实例。**同一个结构，标准库选了便捷，框架选了显式——两站路标都值得记**。

**`net/http/pprof`——注册表 + blank import 的组合拳。** `import _ "net/http/pprof"` 之后，`/debug/pprof/*` 端点凭空出现——魔法其实是 pprof 包的 `init()` 往 `DefaultServeMux` 注册了一串 handler。第 4 篇说过 blank import 是安静的工厂调用，这里补全：**注册的终点经常就是 DefaultServeMux 这张表**。

**`image.RegisterFormat` / `sql.Register`**（第 4 篇详析过源码）：从本篇视角重看一遍，注意它们与 ServeMux 的共同骨架——init 注册、锁、panic 防御、按名查找。四个标准库实现，一个模式。

## 业务实战

CloudShop 的注册表版图（v1 泛型版统一）：`pay.Channels`（构造函数态）、`promo.Promotions`（[第 1 篇](/posts/design-patterns-1-策略模式/)的策略，运营配置渲染成注册项）、`report.Exporters`（csv/excel/pdf 导出）、`event.Handlers`（[第 12 篇](/posts/design-patterns-12-观察者/)总线的订阅表，其实就是个多值注册表）。

上生产后兑现的两个红利：`Names()` 撑起了运维页（渠道列表、已注册促销类型一目了然，以前靠 grep）；重复注册 panic 在大促热载那次**当场抓住**冲突（以前是静默覆盖、深夜排查为什么改的策略不生效）。

黑暗面也要记账：注册表用多了，"谁注册了什么"重新变成暗知识——和 [DI](/posts/design-patterns-18-依赖注入/) 的显式连线是两种世界观。CloudShop 的边界：**框架层（渠道/策略/插件）用注册表**——实现动态、开放给外部；**业务层（订单服务依赖库存服务）用 DI**——依赖静态、团队可控。越界的信号是：业务代码里出现 `registry.Get("orderService")` 这种按名取**内部服务**的调用——那是把显式依赖偷偷换成了字符串魔法。

## 好处与代价

| 好处 | 代价 |
|---|---|
| 运行期扩展（插件/热载）唯一正解 | 按名查找 = 字符串魔法，编译器不查错 |
| 重复注册即时暴露 | "谁注册了什么"成为暗知识（要 Names() 页面 + 文档对冲） |
| 一个泛型件替代 N 份样板 | init 注册顺序敏感（依赖别的注册项时要小心） |
| 运维可观测（清单/健康检查） | 容易沦为"什么都往里塞"的垃圾抽屉 |

## 什么时候不要用

- **类型编译期已知且少**：switch 完胜——编译期查漏、IDE 可跳转。
- **注册表查的是内部服务**：业务依赖走 [DI](/posts/design-patterns-18-依赖注入/)，按名查找内部服务是暗依赖回潮。
- **为了"解耦"而注册**：两个永远同生共死的模块之间放一张注册表，只是把 import 依赖换成了字符串依赖——耦合没消失，还失去了编译器。

## 易混淆

**注册表 vs 工厂**（回环终审）：工厂是"造"的意图（名字→新实例），注册表是"管"的结构（映射的受管容器）。注册表常常是工厂的存储层，但也能单干（存实例）。

**注册表 vs [单例](/posts/design-patterns-3-单例/)**：单例是"唯一实例"，注册表是"名字到实例的集合"——注册表自己通常以单例形态存在（一张全局表），是[第 3 篇](/posts/design-patterns-3-单例/)"受管全局"的正面代表。

**注册表 vs [DI](/posts/design-patterns-18-依赖注入/)**：DI 显式连线、编译期可见；注册表运行期按名发现、字符串连接。框架层注册表 + 业务层 DI 的分层边界，见上文黑暗面一段。

## 自测

1. v0 的四份样板里两份漏了重复检查——为什么说这不是粗心而是抄写的必然？泛型注册表消除的是哪一类错误？
2. `DefaultServeMux` 和 `sql.Register` 的驱动表都是全局注册表，前者的全局性被框架社区诟病、后者成为标准库地基——差别在"受管"的哪些具体设计？
3. 注册表存实例和存构造函数，分别对应什么意图？CloudShop 里支付渠道和事件 handler 为什么一个存函数一个存实例？
4. 业务代码里出现 `registry.Get("orderService")`，为什么是越界信号？正确的形态是什么？

---

**参考来源**

- `net/http`（ServeMux/pprof）、`database/sql.Register`、`image.RegisterFormat` 源码 — 标准库四大注册表
- Go 1.18 泛型 — 通用注册表的前提
