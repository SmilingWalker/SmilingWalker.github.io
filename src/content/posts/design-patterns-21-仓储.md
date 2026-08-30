---
title: 设计模式（21）：仓储 Repository——领域逻辑和 SQL 之间的那道墙
published: 2026-08-30
pinned: false
description: ""
tags: [design-patterns, go, repository, ddd, cloudshop]
category: 设计模式
draft: false
---

[第 18 篇依赖注入](/posts/design-patterns-18-依赖注入/)里被注入的最典型依赖，就是本篇的主角。它回应的疼每个写过业务的人都有：**SQL 字符串散落在业务代码里**。

CloudShop 订单模块的某次统计：`db.Where("user_id = ? AND status = ?", ...)` 这样的查询条件，在 23 处业务代码里以 17 种写法出现（有人查 `status IN (2,3)`，有人两次单查合并）。要给订单表分库分表，这 23 处全部要动；更早之前，"已完成的订单"的语义变了（要包含"部分退款后完成"），改了 6 处漏了 2 处。

## v0：业务代码直接写查询

```go
// order/service.go —— CloudShop v0
func (s *Service) UserCompletedOrders(ctx context.Context, uid int64) ([]*Order, error) {
	var orders []*Order
	// 查询语义长在业务代码里——语义变了要满项目找
	err := s.db.WithContext(ctx).
		Where("user_id = ? AND status IN (?, ?)", uid, StatusCompleted, StatusPartRefunded).
		Find(&orders).Error
	return orders, err
}
```

三个病：查询语义（"什么算已完成"）没有唯一住所；存储技术（GORM/SQL）渗进业务层，换存储=满项目手术；测试必须起真库（[第 18 篇](/posts/design-patterns-18-依赖注入/)的疼在这儿的具象）。

## v1：把查询语义收进接口

```go
// order/repository.go —— 领域层定义接口
type Repository interface {
	FindByID(ctx context.Context, id int64) (*Order, error)
	FindCompletedByUser(ctx context.Context, uid int64) ([]*Order, error) // "已完成"的语义住在这
	Save(ctx context.Context, o *Order) error
}

// order/repo_mysql.go —— 基础设施层实现
type mysqlRepo struct{ db *gorm.DB }

func (r *mysqlRepo) FindCompletedByUser(ctx context.Context, uid int64) ([]*Order, error) {
	var orders []*Order
	err := r.db.WithContext(ctx).
		Where("user_id = ? AND status IN (?, ?)", uid, StatusCompleted, StatusPartRefunded).
		Find(&orders).Error
	return orders, err
}
```

业务代码从此只写 `s.repo.FindCompletedByUser(ctx, uid)`。语义变更（"已完成"加一种状态）改一个方法；存储更换（MySQL→分库分表）换一个实现文件；测试用内存实现（下文）。

## 命名时刻

**仓储模式：像操作内存中的对象集合一样操作领域对象的存取**（Eric Evans，《领域驱动设计》）。它回应的力：**领域逻辑与持久化技术的解耦**——领域层说领域语言（"查已完成订单"），基础设施层说技术语言（SQL/GORM）。

四个设计要点，每个都值回票价：

**接口定义在领域包，实现在基础设施包。** 方向不是"存储层暴露接口给业务"，而是"业务定义自己需要什么，存储去满足"——这正是 [第 0 篇](/posts/design-patterns-0-开篇/) DIP 的"抽象属于高层"的落地。`order.Repository` 接口在 `order` 包里，`mysqlRepo` 在 `order/storage` 里，依赖箭头永远指向领域。

**按聚合根建仓，不按表。** `OrderRepository` 管 Order 和它的 OrderItem（一起存取、一起保证聚合内一致），**没有** `OrderItemRepository`——子实体没有独立生命周期，不配拥有自己的仓。这一条是 Repository 和 DAO 的分水岭（下文详辨），也是 DDD "聚合根"概念对数据访问的塑造。

**方法名说领域语义，不说表结构。** `FindCompletedByUser`（业务词汇）而不是 `SelectByStatusAndUserId`（字段组合）。语义漂移时改一个方法名，调用方 IDE 一键跟随——命名即防腐层。

**内存实现是合法实现。** 测试的 `memRepo` 用 map 存——它不只是测试工具，更是**接口语义的活文档**（"已完成"在内存版里的过滤逻辑就是这个概念的权威定义），还免费支撑了本地开发和演示环境。

### Repository vs DAO：一词之差的两个世界

Java 世界传过来的 DAO（Data Access Object）常被当成 Repository 的同义词，实际是两个层次的东西：

| | DAO | Repository |
|---|---|---|
| 单位 | 表（OrderDAO 对应 order 表） | 聚合根（OrderRepository 对应 Order 聚合） |
| 语言 | 数据库语言（SelectByUserId） | 领域语言（FindCompletedByUser） |
| 职责 | 封装 SQL 细节 | 维护对象集合的表象 |
| 典型形状 | 一表一 DAO，CRUD 四件套 | 一聚合一仓，方法跟着业务问题走 |

CloudShop v0 那个 `OrderService` 直接持有 `*gorm.DB`，如果只是包一层 `OrderDAO{Get,Insert,Update,Delete}`——存储技术是藏起来了，查询语义还在业务层漂着（service 里继续拼 `Where`）。**Repository 的关键动作不是"藏 SQL"，是"收编查询语义"**。

## 标准库与生态里的落地

标准库不管业务分层，但 `database/sql` 本身就是同一思想在数据库维度的极致：用户侧 `Query/Exec` 说"数据库通用语"，`driver` 实现各家方言——**存储抽象 + 多实现 + 注册选择**（[第 20 篇](/posts/design-patterns-20-注册表/)注册表选驱动），Repository 是这个三件套在领域层的镜像。

真实工程样本：**Kubernetes client-go**。`clientset.AppsV1().Deployments(ns).Get(ctx, name, opts)`——按"资源类型"（聚合根的 K8s 版）组织、领域语义方法（Get/List/Watch/UpdateStatus）、底下是 REST 还是缓存（lister）对调用方透明。这是 Repository 形态在超大规模项目里的样子，Watch（订阅变更）这个方法还接通了[事件](/posts/design-patterns-23-事件总线/)的世界。

## 业务实战

CloudShop 落地后的两个高光时刻。第一次：订单表按 user_id 分库（16 分片），`mysqlRepo` 重写为路由版（先算分片再查）——**业务层零改动**，这在 v0 是 23 处手术。第二次：新人的 PR 里出现 `repo.RawSQL(...)`，review 直接打回——接口里没有的方法就是领域语言还没有的词，先问"这个查询的业务名字是什么"，答案或者是新方法（语义收编），或者是这个查询根本不该存在。

一个持续的设计张力值得记录：接口方法数量的膨胀。每个新查询需求都想往 `Repository` 加方法，一年后它会长成 40 个方法的巨型接口（[第 0 篇](/posts/design-patterns-0-开篇/)接口隔离的老病）。CloudShop 的缓解：查询多、写入少且明确的场景，把"读"拆成 `OrderReader` 独立小接口（CQRS 的轻量版思路），写侧保持精简。这没有完美答案，只有"接口跟着领域问题走、不跟着查询便利走"的纪律。

## 好处与代价

| 好处 | 代价 |
|---|---|
| 查询语义唯一住所，语义变更一处改 | 每个聚合一套接口+实现，样板可观 |
| 存储可换、测试可内存化（接通第 18 篇） | 多一层，简单 CRUD 场景是空转 |
| 领域语言防腐：SQL/ORM 不渗进业务 | 接口方法膨胀需要持续治理 |
| 内存实现兼做语义文档和演示环境 | 复杂查询（多表 join/报表）放仓里很别扭 |

## 什么时候不要用

- **纯 CRUD 没有领域逻辑**：管理后台的配置表，直接 sqlc/gorm 生成代码就够——Repository 是给"查询语义会演化"的领域准备的。
- **只有一种存储且永远不换、也永不起测试替身**：抽象层没有第二个消费者（这条要诚实评估——测试替身往往是够格的"第二个实现"，很多人低估了它）。
- **复杂报表/分析查询**：它们天生跨聚合、面向数据列而非对象——那是 CQRS 读模型或直接 SQL 的领域，硬塞进 Repository 得到的是一个说不了领域语言又藏不住 SQL 的四不像。

## 易混淆

**Repository vs DAO**：上文主辨析——按聚合说领域语 vs 按表说数据库语。

**Repository vs [DI](/posts/design-patterns-18-依赖注入/)**：Repository 是被注入的头号依赖；DI 负责把 `mysqlRepo` 或 `memRepo` 接到 service 上。18 篇讲流动机制，本篇讲流动的东西长什么样。

**Repository vs [注册表](/posts/design-patterns-20-注册表/)**：注册表管"名字→实现"的运行期映射；仓储管"领域对象↔存储"的翻译。一个横向发现组件，一个纵向隔离存储。

## 自测

1. v0 的 17 种"查已完成"写法，本质是什么没有唯一住所？`FindCompletedByUser` 这个方法名治理了其中的什么？
2. "没有 OrderItemRepository"——这条规则背后的 DDD 概念是什么？如果允许子实体有自己的仓，会破坏什么一致性？
3. DAO 封装了 SQL 却仍被本篇判为不够，判词是哪一句？`SelectByStatusAndUserId` 和 `FindCompletedByUser` 的差别怎么影响语义演化？
4. "测试替身往往是够格的第二个实现"——用这个标准重新审视自己项目里"只有一种实现"的仓储抽象，哪些其实应该抽、哪些确实不该？

---

**参考来源**

- Eric Evans, *Domain-Driven Design* — Repository 与聚合根的原始定义
- Kubernetes client-go（clientset/lister）— 超大规模项目的仓储形态
- `database/sql` + driver — 存储抽象三件套的标准库样本
