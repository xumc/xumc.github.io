---
layout: post
title: "简析微服务中的幂等性设计"
date: 2018-12-07
---

在微服务大行其道的今天，微服务之间的API接口调用变得越来越多，也就产生了系统数据一致性的问题。怎么解决数据一致性的问题呢。业界也产生了很多的解决方案。比如XA, TCC 等分布式事务解决方案，再比如使用mq来达到最终一致性。本文介绍使用API幂等性设计来解决一致性的问题。

<div class="mermaid">
graph LR
A[Service A] -->B[Service B]
</div>

考察上图的场景。有ServiceA调用ServiceB。其中两个service均为两个分布式系统，后面有多个应用实例。

1. 如果是查询请求，一般来说是HTTP GET请求。这种情况下，基本上都是幂等的，不用考虑一致性问题。
2. 创建资源的请求， 一般来说是HTTP POST。这种情况下，不是幂等的。这种情况下，我们应该如何将其转换为幂等的请求呢？一种简单的方法是在ServiceB的数据库中设计唯一性索引。假设ServiceA调用ServiceB的API是根据手机号注册用户。那么我们可以在ServiceB的用户表中将手机号字段设置上唯一性索引。 如果第二次用同一个手机号注册，那么我们不论我们是先查询再插入用户表或者直接插入用户表，数据库都会返给我们唯一性索引错误。ServiceB只需要catch住这个异常，并且返回已有的的用户信息就OK了。
另一种做法是生成全局唯一标示，如下图：

<div class="mermaid">
sequenceDiagram
　　ServiceA->>ServiceB: 1.给我一个新增用户的一个唯一标识符
　　ServiceB->>ServiceA: 2.好的，给你
　　ServiceA->>ServiceB: 3.我要用你给我的唯一标识符新增一个用户信息。
</div>

第三步， ServiceB在已经生成的历史标识符中寻找，如果找到标识符并且此标识符还没有被处理，则进行正常处理处理；如果找到了标识符但是已经被处理过了，则返回标识符绑定的用户信息；如果没有找到标识符，则ServiceA内部肯定存在问题，返给ServiceA标识符错误。实际上标识符和手机号这两种做法都是类似的，都是根据显性或者隐形的唯一标识符来达到幂目的。这里有3个问题，
1. 在步骤3后，如果ServiceB新增了用户信息，但是此时还没有来的及返回结果给ServiceA，此时ServiceA和ServiceB之间的网络断了，并且如果ServiceA在调用ServiceB来新增用户信息之前或者之后有一个本地事务，需要处理，在遇到网络断掉或者ServiceB崩溃，应该应该怎么办？
2. 在步骤一中，产生的唯一标识符长时间没有被用掉，应该怎么办？
3. 标识符存储在哪里？
4. 如果新增用户信息有validation的需求，并且ServiceA请求ServiceB的新增用户接口有可能触发validation 不通过的情况，这种情况下怎么办？
5. 可以看到在ServiceA和ServiceB交互过程中，ServiceA至少要发起两次http请求。如果放到ServiceA的本地事务中执行，必然会导致本地事务执行效率的下降，我们能不能有办法把两次http请求放到本地事务之外呢？

对于问题1：这个问题分成两种情况：
1. 对于不需要立刻返回执行结果的代码，可以无限次retry，直至retry成功。
2. 但是如果ServiceA提供的是http服务，那么就不能无限制的retry，在达到retry最大次数后，如果ServiceA返回给客户端（浏览器或者app）返回500错误并且回滚本地事务的话，我们看看此时的数据情况，ServiceB已经增加新用户信息，但是ServiceA的本地事务却回滚了，造成数据不一致。所以这么做事不可以的，一种解决这个问题的办法是在ServiceA本地事务操作的业务表中添加一个bool column，标示这个唯一标识对应的新增用户信息的操作是否完成，默认为false，在步骤3完成之后改为true。同时ServiceA后台要启动一个定时运行的job，扫描这个bool 字段，对于未完成的进行后台retry。需要注意的是如果根据ServiceA的数据库不能恢复步骤3所需的信息的情况，还需要存储下额外的信息。这样就能够达到最终一致性的需求。

对于问题2： 长时间没有被用掉的标识符，可以考虑删掉。当然具体多少才算作长时间，这要根据业务需求去定。

对于问题3：标识符可以存储在ServiceB的数据库中，在标识符被使用后， 可以存储到业务实体有关联的地方，比如在本例中，在用户表中添加标识符这一列。当然也可以存储到redis等kv数据库中。

对于问题4： 对于validation 有失败的情况，这种情况下，可以采用Service B调用Service A的对应的接口，使得Service A的本地事务的结果revert。但是这种操作也要注意Service A的接口应该也需要是幂等的，并且要同样引入retry机制。另外，最重要的一点要保证ServiceA的接口是没有validation的，或者保证validation一定会通过。否则就会产生相互validation不通过的情况，这种情况下，就真的无解了。

对于问题5： 可以的，在问题1中有讲到本地事务会包含http请求是否成功标志位，我们可以在本地事务完成后或者本地事务开始前，执行http请求，此时，http的请求时间并不算在本地事务时间内。



