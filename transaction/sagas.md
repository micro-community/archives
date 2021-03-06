# 什么是saga

saga这个名词通常被用在CQRS的讨论中，它是指一段在限定上下文（bounded contexts ）和聚合（aggregates）之间起协作和路由（coordinates and routes ）消息作用的代码。然而，在这个指南中我们更喜欢用Process manager这个词语去表示saga。有两个原因：

之前已经有了一个广泛被熟知的名词saga，这个saga和CQRS中的saga有着不同的含义。
process manager 是一个更合适用来描述saga在这里扮演角色的名词。
虽然saga经常在CQRS模式中见到，然而它已经在这之前有了其自己的含义。在这个指南中我们使用process manager 以便于和saga之前的含义区分开来。

saga，与分布式相关，最早被定义在Hector Garcia-Molina和Kenneth Salem的论文"Sagas"中。这篇论文提出了一个saga机制来作为分布式事务的替代品以解决长时间运行的分布式事务（long-running process）的问题。这篇论文认为业务过程经常由很多步骤组成，每个步骤都涉及一个事务，如果将这些事务组成一个分布式事务，就可以实现总体一致（overall consistency ）。然而在长时间运行的分布式事务中，使用分布式事务会影响效率和系统的并发处理能力，因为在执行分布式事务的时候会有锁产生。

注意：

saga通过确保每一个业务过程都有修正事务（compensating transaction）来减少了系统对分布式事务的依赖。在这种方式下，如果业务过程遇到了错误的情况并且无法继续，它就可以执行修正事务来修正已经完成的步骤。这种在业务流程中去撤销已经完成的工作的方式保证了系统的一致性。

尽管我们已经选择使用process manager这个名词了，sagas仍然在实现CQRS系统中的限定上下文中扮演着一些角色。比如说，你可能会希望看到process manager在一个限定上下文中的聚合中路由消息，你也可能会希望看到saga管理一个在多个限定上下文中长时间运行的业务过程。

以下的几个部分描述了process manager，这个是我们在我们的CQRS之旅项目中对saga的定义。

注意：

在使用process manager之前，我们的团队曾经有一段时间使用coordinating workflow 这个名词。这种模式在Gregor Hohpe 和 Bobby Woolf的书"Enterprise Integration Patterns”中有所描述。

 

Process Manager
这一节概括了我们的process manager定义。在描述process manager之前有一段简短的关于CQRS使用消息（messages）在聚合和限定上下文中通讯的回顾。

消息和CQRS
当你实现CQRS模式的时候，你可能会思考两种类型的消息如何在你的系统中交换数据：command和事件。

command是一种请求，他们请求系统去执行一个任务或者动作。例如“预定两个X会议的座位”或者“把演讲者Y分配到Z室”。command通常只被一个接收者执行一次。

事件是一种通知，他们告诉系统中一些它们感兴趣的部分：有一些事情已经发生了。例如“支付被拒绝了”或者“产生了X类型座位”。注意他们使用的是过去式——事件已经被产生并且可能有许多订阅者。

通常来说，command被发送到同一个限定上下文中。事件的订阅者可能在它们发布的限定上下文中，或者在其他的限定上下文中。

引用指南中的"A CQRS and ES Deep Dive"章节详细地介绍了这两种不同的消息类型。

process manager是什么？
在一个复杂系统建模中，你可能已经使用了聚合和限定上下文，他们可能有一些包含了很多聚合的业务过程，或者在一个限定上下文中有很多的聚合。在这个业务过程中，许多不同类型的消息被交换。例如，在一个会议管理系统中，有一个预定座位的业务过程包含了order聚合，一个reservation聚合和一个payment的聚合。他们必须结合起来以保证一个客户能够完成预定交易。

图1表示了一个简化的消息场景，在这个场景中这些聚合互相交互来完成这个订单。数字表明了消息流转的顺序

Sagas长事务

在Sagas事务模型中，一个长事务是由一个预先定义好执行顺序的子事务集合和他们对应的补偿子事务集合组成的。典型的一个完整的交易由T1、T2、……、Tn等多个业务活动组成，每个业务活动可以是本地操作、或者是远程操作，所有的业务活动在Sagas事务下要么全部成功，要么全部回滚，不存在中间状态。

![sagas](http://mmbiz.qpic.cn/mmbiz_jpg/LaW7jDBKBg20ygGehFf5yZsSibbeicKAhQcoicGCrgCzhFelbrrpxtialWR09hn6ibUm0JcrxCWl3jpdnuWricm02Cibw/640?wx_fmt=jpeg)

微服务中使用工作流方式Sagas事务来保证数据完整

Sagas事务模型的实现机制：

每个业务活动都是一个原子操作；

每个业务活动均提供正反操作；

任何一个业务活动发生错误，按照执行的反顺序，实时执行反操作，进行事务回滚；

回滚失败情况下，需要记录待冲正事务日志，通过重试策略进行重试；

冲正重试依然失败的场景，提供定时冲正服务器，对回滚失败的业务进行定时冲正；

定时冲正依然失败的业务，等待人工干预；

Sagas长事务模型支持对数据一致性要求比较高的场景比较适用，由于采用了补偿的机制，每个原子操作都是先执行任务，避免了长时间的资源锁定，能做到实时释放资源，性能相对有保障。

Sagas长事务方式如果由业务去实现，复杂度与难度并存。在我们实际使用过程中，开发了一套支持Sagas事务模型的框架来支撑业务快速交付。



开发人员：业务只需要进行交易编排，每个原子操作提供正反交易；

配置人员：可以针对异常类型设定事务回滚策略（哪些异常纳入事务管理、哪些异常不纳入事务管理）；每个原子操作的流水是否持久化（为了不同性能可以支持缓存、DB、以及扩展其它持久化方式）；以及冲正选项配置（重试次数、超时时间、是否实时冲正、定时冲正等）；


框架：提供事务保障机制，负责原子操作的流水落地，原子操作的执行顺序，提供实时冲正、定时冲正、事务拦截器等基础能力；

Sagas框架的核心是IBusinessActivity、IAtomicAction。IBusinessActivity完成原子活动的enlist()、delist()、prepare()、commit()、rollback()等操作；IAtomicAction主要完成对状态上下文、正反操作执行。



微服务中使用工作流方式Sagas事务来保证数据完整

![sagas2](http://mmbiz.qpic.cn/mmbiz_jpg/LaW7jDBKBg20ygGehFf5yZsSibbeicKAhQYOe68ic8RN9tfujZTXj2kMwvkIWWOsH5C2ibXR8YgHfIoRK7gDgk4hGQ/640?wx_fmt=jpeg)

限于文章篇幅，本文不对具体实现做详述；后面找时间可以详细介绍基于Sagas长事务模型具体的实现框架。Sagas长事务需要交易提供反操作，支持事务的强一致性，由于没有在整个事务周期内锁定资源，对性能影响较小，适合对数据要求比较高的场景中使用。

补偿模式

Sagas长事务模型本质上是补偿机制的复杂实现，如果实际业务场景上不需要复杂的Sagas事务框架支撑，可以在业务中实现简单的补偿模式。补偿过程往往也同样需要实现最终一致性，需要保证取消服务至少被调用一次和取消服务必须实现幂等性。



微服务中使用工作流方式Sagas事务来保证数据完整

![sagas3](http://mmbiz.qpic.cn/mmbiz_jpg/LaW7jDBKBg20ygGehFf5yZsSibbeicKAhQFjdM7geV3r48vwUKXgoib54RUvU2L2MdjyePfs8oovfwjPsA8y7ka2A/640?wx_fmt=jpeg)

补偿机制不推荐在复杂场景（需要多个交易的编排）下使用，优点是非常容易提供回滚，而且依赖的服务也非常少，与Sagas长事务比较来看，使用起来更简便；缺点是会造成代码量庞大，耦合性高，对应无法提供反操作的交易不适合。


## 引用

https://mp.weixin.qq.com/s/AWdgVcPGjGgBM9T9D8VVmw
