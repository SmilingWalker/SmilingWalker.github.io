---
title: 设计模式（13）：命令——可撤销的操作，是一等公民的动作
published: 2026-08-30
pinned: false
description: ""
tags: [design-patterns, go, command, cloudshop]
category: 设计模式
draft: false
---

[第 12 篇](/posts/design-patterns-12-观察者/)末尾立了桩：事件广播不关心结果，命令**必须**关心。这篇从命令这头把线接上。

CloudShop 后台的商品编辑一直是"改完即生效"，跑了半年没人觉得有问题——直到那个周五下午。运营把一个头部商品的价格从 199 改成 19（少打了一个 9），四分钟后自己发现，立刻改回 199。但中间那四分钟，页面上的低价引来了两百多单。客诉处理了一个周末。周一早上，运营总监的需求单放在产品桌上，只有一句话："编辑要能撤销。"同一周，购物车团队也提了自己的版本：用户误删了购物车里的东西，想要撤销，最好还能重做。两个需求撞在同一个本质里：**操作不能只"发生完就完了"，它得被保管**——可以被重新执行、被逆转、被审计。

## v0：操作即函数调用，转瞬即逝

```go
// admin/product.go —— CloudShop v0
func (s *AdminService) UpdatePrice(ctx context.Context, id int64, price int64) error {
	return s.repo.UpdatePrice(ctx, id, price) // 调用结束，"发生过什么"只剩日志
}

func (s *AdminService) UpdateTitle(ctx context.Context, id int64, title string) error {
	return s.repo.UpdateTitle(ctx, id, title)
}
```

函数调用是最快的遗忘：执行完，"谁、在什么参数下、做了什么"就蒸发了。撤销无从谈起——撤销需要**之前的状态**或**逆操作**，两者都没被保管。

## v1：把动作打包成对象

撤销的第一要件是**逆操作**。把"执行"和"逆转"封进同一个对象：

```go
// admin/command.go
type Command interface {
	Execute(ctx context.Context) error
	Undo(ctx context.Context) error
	Desc() string // 审计描述，后文用
}

type PriceCmd struct {
	Repo     ProductRepo
	ID       int64
	Old, New int64 // 保管"之前的状态"——命令自己带着撤销所需的一切
}

func (c *PriceCmd) Execute(ctx context.Context) error {
	return c.Repo.UpdatePrice(ctx, c.ID, c.New)
}
func (c *PriceCmd) Undo(ctx context.Context) error {
	return c.Repo.UpdatePrice(ctx, c.ID, c.Old) // 写回旧值
}
func (c *PriceCmd) Desc() string {
	return fmt.Sprintf("改价 %d: %d → %d", c.ID, c.Old, c.New)
}
```

执行前先把 `Old` 抓进命令——**命令对象是"动作 + 撤销所需上下文"的完整包裹**。再加两个栈，撤销/重做就是弹栈：

```go
type History struct {
	mu         sync.Mutex
	undo, redo []Command
}

func (h *History) Do(ctx context.Context, cmd Command) error {
	if err := cmd.Execute(ctx); err != nil {
		return err
	}
	h.mu.Lock()
	h.undo = append(h.undo, cmd)
	h.redo = nil // 新操作打断重做分支（所有编辑器的共同语义）
	h.mu.Unlock()
	return nil
}

func (h *History) Undo(ctx context.Context) error {
	h.mu.Lock()
	if len(h.undo) == 0 {
		h.mu.Unlock()
		return nil
	}
	cmd := h.undo[len(h.undo)-1]
	h.undo = h.undo[:len(h.undo)-1]
	h.mu.Unlock()
	if err := cmd.Undo(ctx); err != nil {
		return err
	}
	h.mu.Lock()
	h.redo = append(h.redo, cmd)
	h.mu.Unlock()
	return nil
}
```

购物车同构：`AddItemCmd` 的逆是 `RemoveItemCmd`，`AddToCart(item)` 执行时顺手构造好逆命令入栈。

```mermaid
flowchart LR
    U[undo 栈<br/>PriceCmd(title=旧价)] -->|Undo 弹出执行| DB[(商品表)]
    DB -->|执行成功压入| R[redo 栈]
    R -->|Redo 弹出执行| DB
    N[新命令] -->|Do| U
    N -.打断.-> R
```

## 命名时刻

**命令模式：将请求封装为对象，从而可以参数化、排队、撤销、记日志**。它回应的力：**操作本身需要被当作数据对待**——被存储、传递、重放、逆转。函数调用做不到这些，因为调用是过程，不是对象。

命令模式一个被低估的副产品是**审计**：`Desc()` 那行代码意味着每条命令天然是一条审计记录——"谁在 14:32 把商品 8817 从 199 改到 149"。CloudShop 的后台操作流水就是命令历史本身，不需要另一套日志系统：**History 栈即审计日志**。金融场景（退款冲正、交易回溯）对不可篡改有额外要求时，把命令序列落到 append-only 存储——[第 11 篇](/posts/design-patterns-11-冷门三连/)备忘录的价格快照在这里合流：审计要的往往不是"操作"而是"操作前后的状态对"。

### Undo 的三档语义（深度核心）

实现 Undo 有三条路，成本与能力递增：

| 档位 | 机制 | 适用 | CloudShop 现场 |
|---|---|---|---|
| 互逆命令 | 每命令配逆操作 | 逆操作天然存在且无副作用 | 加购/移出购物车 |
| 状态快照 | Undo 前存档，恢复即回滚 | 逆操作难写（富文本编辑） | 商品多字段编辑（配[备忘录](/posts/design-patterns-17-冷门五连/)） |
| 补偿事务 | 失败后执行反向冲正 | 跨系统、已产生外部副作用 | 退款失败→原路退回；发错短信→发更正（真金融用 Saga） |

第三档最诚实也最重要：**短信发出去就收不回来，"撤销"的本质是补偿**。做"可撤销"产品功能之前，先对每个操作问一句"它的逆是什么"——答不出来的操作（发短信、扣款、发货）只能补偿，而补偿是业务决策（发更正短信？送优惠券？），不是技术能单方面兜底的。这一问决定了产品文案里"可撤销"三个字的边界。

## 标准库与真实工程里的落地

**数据库事务——最普及的命令形态。** `BEGIN` 造命令上下文，`COMMIT` 是 Execute，`ROLLBACK` 是 Undo。事务的原子性承诺本质上就是"这个命令可整体逆转，直到提交那一刻"。

**crush 的中断续传——命令重排队的真实样本。** 上下文压缩系列调研过的 [crush](https://github.com/charmbracelet/crush)（`agent.go` 的会话恢复逻辑）：agent 因上下文过长被压缩打断时，**未完成的调用被打包改写后重新塞回消息队列**——"要做的事"因为被封装成了可保管的对象，才能在进程级的中断后原样恢复。这就是命令模式"排队、延迟、重放"三连在真实代码里的样子：函数调用蒸发了，命令对象还活着。

**工作队列的任务体。** 每个投递到 MQ 的任务（JSON 反序列化成结构体，带着全部参数）是一条序列化命令：消费者拿到的是"要执行的动作"本身，而不是"去问生产者要做什么"的引用。分布式系统里命令模式不是可选风格，是任务能在机器之间搬来搬去的唯一原因。

## 业务实战

CloudShop 后台的最终形态：所有写操作走 `Command`，`History` 持久化到 `admin_op_log` 表（append-only），撤销 = 弹栈执行 Undo + 记一条"撤销了什么"的新命令（撤销本身也被审计）。审批撤回是同一套：提交审批是命令，撤回是它的互逆命令，都进历史。

第二个落点接住第 12 篇的伏笔：**命令总线**。后台的"批量上架"是宏命令（一组命令的容器，Execute 顺序执行、Undo 逆序回滚——宏本身也是命令，可嵌套）；"保存草稿"是命令入队不执行、"提交"是批量 Execute——命令的**排队与延迟执行**让"草稿箱"这种产品形态变成免费。

## 好处与代价

| 好处 | 代价 |
|---|---|
| 撤销/重做、审计、队列、宏操作全部就位 | 每操作两个方法 + 参数快照，样板翻倍 |
| 操作可序列化，跨进程/跨机器搬运 | 命令对象携带状态，长栈占内存 |
| 执行时机与构造时机解耦（草稿箱） | 互逆命令不好写（补偿是业务问题） |
| History 即审计日志，一套数据两用 | 并发下的栈一致性与持久化复杂度 |

## 什么时候不要用

- **操作不需要撤销、审计、排队**：普通函数调用是 Go 的第一公民，直接用。
- **撤销语义写不出来**（外部副作用无逆操作且无补偿方案）：硬做"可撤销"是骗产品——技术债伪装成功能承诺。
- **高频简单路径**：每秒千次的库存扣减不适合每个请求打包命令对象，性能与复杂度双输。

## 易混淆

**命令 vs [策略](/posts/design-patterns-1-策略模式/)（第 1 篇）**：结构像（接口 + 多实现 + 持有者），意图反。策略是**可替换的算法**（用哪个都行，用完即弃）；命令是**被保管的动作**（这一个动作的执行历史有延续意义）。问"这个对象用完还重要吗"：策略用完不重要，命令用完才是它生命的开始。

**命令 vs 事件**（第 12 篇）：命令有**唯一执行者**且必须关心结果；事件是**广播**且不关心谁听。同一次交互里两者并存：支付模块收到"执行退款"命令，完成后广播 `refund.succeeded` 事件。

**命令 vs [备忘录](/posts/design-patterns-17-冷门五连/)（第 17 篇）**：命令保管"动作及其逆"，备忘录保管"状态快照"。Undo 的第二档语义里两者配合——命令触发、快照回滚。

## 自测

1. v0 的函数调用为什么"无法撤销"？要从"调用"升级到"命令"，补上的本质要件是什么？
2. Undo 三档语义分别对应 CloudShop 的什么操作？"发短信"落在哪一档，它的 Undo 长什么样？
3. 宏命令（批量上架）的 Undo 为什么必须逆序？如果其中第三条命令 Undo 失败，整个宏处于什么状态，该怎么办？
4. crush 把被打断的调用重新入队，用到了命令模式的哪三个性质？"函数调用蒸发了，命令对象还活着"怎么理解？

---

**参考来源**

- GoF, *Design Patterns* — 命令原始定义（参数化/排队/撤销/日志四大用途）
- 数据库事务语义 — 命令形态的普及样本
- crush（charmbracelet/crush）会话恢复逻辑 — 命令重排队的一手源码
