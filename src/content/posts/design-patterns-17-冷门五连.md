---
title: 设计模式（17）：冷门五连——中介者、备忘录、迭代器、解释器、访问者
published: 2026-08-30
pinned: false
description: ""
tags: [design-patterns, go, mediator, memento, iterator, interpreter, visitor, cloudshop]
category: 设计模式
draft: false
---

行为级收尾，五个冷门模式合辑。它们冷门不是因为没用，是因为**命中场景窄**——命中即刚需，不命中硬上就是灾难。每个仍走压缩版流程。

## 中介者：六个服务打成死结的时候

### 痛点

CloudShop 微服务化中期：订单、库存、积分、通知、风控、优惠券六服务，两两互调——改价要通知订单，退款要碰库存+券+积分+支付……依赖图连成了蛛网，任何一个服务发版都要全联调。

### 浮现与命名

**中介者模式：对象之间不直接通信，通过一个中介协调**。蛛网的解法是把 N×N 的两两依赖收敛成 N×1 的星型——所有协调逻辑集中到一处。

```
改造前：6 服务 15 条依赖边（两两互调）
改造后：6 服务各连 1 条边到交易编排中心（事件总线的协调形态）
```

CloudShop 的中介者不是新造的上帝类，而是[第 12 篇](/posts/design-patterns-12-观察者/)事件总线的**协调升级版**：退款流程由一个编排者（saga）订阅 `refund.requested`，按剧本依次驱动库存→券→积分→支付并处理失败补偿。**事件总线广播（谁都可以听），中介者裁决（按剧本推进流程）**——前者是知情，后者是指挥。

### 与观察者的分界

观察者单向广播，源不认识听众；中介者双向协调，各同事通过它互相对话（库存扣完要告诉中介者，中介者再决定下一步）。广播解决"通知"，中介解决"协作"。

### 何时不用

三四个对象、依赖边五六条：直接互相调用，比引入中介少一层政治局。中介者是给"改一处、动全身"的蛛网准备的——蛛网没成形就上中介者，等于提前把灵活性的死刑判决签了。

## 备忘录：价格快照，审计的弹药

### 痛点

促销期间连续改价五次，客诉"昨天明明是 199"——当前价只有一份，历史无凭。金融侧更狠：成交时刻的费率、汇率、规则必须可回溯，监管要求。

### 浮现与命名

**备忘录模式：在不破坏封装的前提下捕获对象的内部状态，以便之后恢复**。Go 的实现直白得没有仪式感——快照就是**不可变的值拷贝**：

```go
type PriceSnapshot struct {
	ProductID int64
	Price     int64
	At        time.Time
	By        string  // 操作人
	Reason    string  // 改价原因
}

func (s *SnapshotRepo) Save(ctx context.Context, snap PriceSnapshot) error // append-only 表
```

[第 13 篇](/posts/design-patterns-13-命令/)的伏笔在这里兑现：Undo 的第二档语义（状态快照）由备忘录供货——命令负责"何时存、何时恢复"，备忘录负责"存什么、怎么存"。`OrderItem.PriceSnapshot` 则是设定集里埋的另一处：**下单时刻锁定价格，之后改价不影响已成交订单**——备忘录作为业务规则（而非撤销机制）的用法。

### 标准库里的落地

**Kubernetes 的 `rollout undo`**：每次 Deployment 发布存一个 Revision（快照），回滚 = 把旧 Revision 换回来。**git stash**：工作区状态的暂存快照。两者都是"快照表 + 指回旧版"的同构。

### 何时不用

快照大、变更频繁（每秒改价）：全量快照撑爆存储，上增量或事件溯源。对象状态简单（两个字段）：直接存这两个字段，别抽象"Memento"。

## 迭代器：Go 把它做进了语言，然后又给了个函数

### 痛点（曾经有）

遍历订单分页：`for offset := 0; ; offset += 1000 { orders := fetch(offset); if len(orders)==0 {break} ... }`——游标、终止、翻页逻辑每个调用方写一遍。

### Go 的两段历史

**第一段：range 吃掉了一切。** 切片、map、channel 的遍历是语言内建——绝大多数 GoF 迭代器场景（自定义集合类）在 Go 里根本不会出现，因为 Go 的集合就是语言原生的切片和 map，**迭代器模式被语言吸收**。这是"模式被语言吃掉"的极限案例。

**第二段：数据库游标等不进 range。** 千万级订单的遍历不能全载内存，需要惰性、分批、可中断——Go 1.23 的 `iter` 包给出了官方抽象：

```go
// iter.Seq —— 迭代器的函数形态
type Seq[V any] func(yield func(V) bool)

func Orders(ctx context.Context, db *DB) iter.Seq[*Order] {
	return func(yield func(*Order) bool) {
		cursor := ""
		for {
			batch, next := db.FetchBatch(ctx, cursor, 1000)
			for _, o := range batch {
				if !yield(o) { // 调用方 break → 停止拉取
					return
				}
			}
			if next == "" {
				return
			}
			cursor = next
		}
	}
}

// 用起来和 range 融为一体
for o := range Orders(ctx, db) {
	if o.Amount > 1_000_000 {
		break // 中断：底层查询立即停
	}
}
```

分页游标、批量拉取、`yield` 返回 false 即中断——标准库 `iter.Seq` 是迭代器在 Go 的现代形态，`range over func` 让它与语言无缝焊接。

**遍历时增删：Go 官方承认的未决行为。** Go 语言规范对 map 遍历中增删写着：删除尚未到达的条目，对应的迭代值**不会**产出；遍历中新建的条目，**可能产出也可能跳过**。这不是文档疏漏，是和 Java 走了不同的路——Java 用 modCount 计数做 fail-fast（遍历中改动直接抛 ConcurrentModificationException，把未决变成确定报错），Go 的规范**直接承认未决**，把责任交给写代码的人。于是 Go 的惯用法是防御性的：要边遍历边删，先拷贝再改——

```go
for k := range m { keys = append(keys, k) } // 先快照
for _, k := range keys { delete(m, k) }      // 再修改
```

王争讲快照迭代器时用双时间戳（addTimestamp/delTimestamp，标记删除不真删）实现了"不拷贝容器的快照"，思路接近数据库 MVCC；Go 侧的对应物其实在第 3 篇出现过——促销策略表热更新"构建新对象再原子换引用"，就是整个容器的快照切换。切片的语义又不同：`range` 的长度在进入循环时定格，之后 append 的元素不一定被看到——同样是规范承认的未决。**结论：在 Go 里"遍历时改集合"没有侥幸，要么快照、要么换引用，规范不兜底。**

### 何时不用

绝大多数时候：切片 + range 就是答案。`iter.Seq` 留给惰性大数据集和自定义流。

## 解释器：促销 DSL 的甜蜜与深渊

### 痛点

运营提需求："满 300 减 40 且 PLUS 会员再 95 折，或者周二全场 98 折"——配置表的表达力到头了，运营开始手写类似规则的表达式。

### 浮现与命名

**解释器模式：为一种语言定义文法和解释器**。迷你实现四件套：词法（tokenize）→ 语法（parse 成 AST——[第 9 篇](/posts/design-patterns-9-组合/)的组合模式树！）→ 求值（对 AST 递归 eval——每类节点一个小解释器）→ 注册函数（`满()`、`会员()`、`折()`）。

```
"满(300).减(40).且(会员(PLUS).折(95))" 
  → tokenize → AST：AndNode(FullNode(300,40), DiscountNode(PLUS,95))
  → eval(order) → 价格引擎调用
```

### 甜蜜与深渊（深度核心）

解释器是**表达力换复杂度**的交易。交易曲线分三档：

| 档 | 形态 | 成本 |
|---|---|---|
| 配置表 | 结构化字段（满减阈值/折扣率） | 低，表达力有限——CloudShop 现状 |
| 表达式 DSL | 有限文法（本篇） | 中：文法设计、测试矩阵、运营文档 |
| 图灵完备 | 嵌入脚本（Lua/expr-lang） | 高：安全沙箱、性能、调试、运营培训 |

真实促销系统大多停在第一档——[促销系统的产品分析](https://www.woshipm.com/pd/4877172.html)里的"规则"基本是结构化配置，不是自由语法。**深渊在第二档向第三档滑坡**：运营今天要"或"条件，明天要变量，后天要循环——文法每扩一点，测试矩阵翻倍，最终得到一门没有编译器团队却要长期维护的语言。**解释器模式的第一判断不是"怎么实现"，是"这门语言配不配存在"**。

### 何时不用

表达力没到天花板（配置表还能表达）；规则数量少（十几条规则用 [策略](/posts/design-patterns-1-策略模式/)+配置足够）；没有专人养护文法。

## 访问者：一笔账的三个视图

### 痛点

财务要账务分录视图，运营要毛利视图，客服要订单追踪视图——**同一棵账单树（[第 9 篇](/posts/design-patterns-9-组合/)的 `CatalogNode`/账单树），三种遍历操作**。操作直接写成节点方法的话：每加一种视图，全部节点类型加一个方法——结构类被操作的变化绑架。

### 浮现与命名

**访问者模式：把对结构的操作从结构里搬出来，每个操作一个访问者**。节点只保留一个 `Accept(v Visitor)` 钩子，操作种类（视图）可以无限增加而不动结构：

```go
type BillNode interface {
	Accept(v BillVisitor)
}

type BillVisitor interface {
	VisitOrderItem(*OrderItem)
	VisitRefund(*Refund)
	VisitFee(*Fee)
}

// 财务视图：一个访问者 = 一套完整遍历逻辑
type LedgerView struct{ entries []Entry }

func (v *LedgerView) VisitOrderItem(it *OrderItem) {
	v.entries = append(v.entries, debit(it.Amount, "应收账款"), credit(it.Amount, "销售收入"))
}
func (v *LedgerView) VisitRefund(r *Refund) { /* 红字分录 */ }
func (v *LedgerView) VisitFee(f *Fee)       { /* 手续费分录 */ }
```

结构（账单树）与操作（三视图）**双轴独立演化**——加第四个"税务视图" = 加一个访问者，树类型零改动。这本质上是把[第 8 篇](/posts/design-patterns/8-桥接/)的维度分离思想应用到"结构 × 操作"这对维度上。

Go 的实现要用**类型断言替代双分派**（Go 没有方法重载）：常见简化是 `Accept` 里 `switch n := n.(type)` 分发——效果相同，少一层接口仪式。**结构稳定、操作常变**是访问者的命门：反过来（结构常变、操作稳定），每加一个节点类型全部访问者都要改——致命弱点，这个方向请用普通方法。

### 标准库里的落地

**`go/ast.Inspect`**——标准库最大的访问者：遍历 Go 语法树，`func(n ast.Node) bool` 对每类节点做判断处理。所有 linter 工具（`staticcheck` 等）建立在它之上——AST 结构由语言团队锁死，检查规则（操作）由全世界任意增加，正是访问者的理想土壤。**kubectl 的资源遍历**也用过同样的结构（其源码里有名实相符的 `Visitor` 接口族，社区早有专文分析），读源码时留意 `Visitor` 后缀。

### 何时不用

操作就一两种：直接写成节点方法。结构不稳定：访问者的修改范围会爆炸。结构不需要统一遍历：那可能连[组合](/posts/design-patterns-9-组合/)都不需要。

## 自测

1. 中介者和事件总线的分界（"知情 vs 指挥"）在退款流程里怎么体现？saga 编排者算哪个？
2. `OrderItem.PriceSnapshot`（下单锁价）和后台改价的历史快照，同为备忘录，各自的业务动机差在哪？
3. `iter.Seq` 的 `yield` 返回 false 为什么是设计的点睛之笔？没有它，大数据集遍历会怎样？
4. 解释器三档交易曲线里，"深渊在第二档向第三档滑坡"的机制是什么？
5. 访问者的命门"结构稳定、操作常变"反过来会怎样？`go/ast` 为什么恰好满足正向要求？

---

**参考来源**

- GoF, *Design Patterns* — 五个模式的原始定义
- `iter` 包（Go 1.23）、`go/ast.Inspect` — 迭代器与访问者的标准库形态
- Kubernetes rollout undo / git stash — 备忘录的工程样本
- 《6000字看懂促销系统的底层逻辑》— 真实促销规则以结构化配置为主的事实
- 王争《设计模式之美》（迭代器三篇）— 未决行为、Java modCount fail-fast 源码、快照双时间戳方案
