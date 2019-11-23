---
layout: post
title: "高性能counter服务设计"
date: 2019-11-23
---

在很多系统开发中，counter都是系统中必不可少的组成部分。 比如在淘宝中，我们需要存储库存量， 在广告系统中，需要存储广告主订单里面已经投放的广告impression/currency量。 在博客系统中，需要统计浏览量。不同的系统，对counter的要求也是不一样的。 在淘宝中， 库存量必须要准确，否则出现商品超卖的情况就麻烦了。 在广告系统中， 是允许有一定误差的，当然误差不能太大，而且广告系统需要counter计数时延要求低。在博客系统中，针对某一篇博客进行浏览量统计，是允许有一定误差，而且对时延要求并不是很高。总之，不同类型的系统的计数组件的设计也是不同的，需要case by case的对待。

在广告系统中，我们counter组件要求是：
1. 允许出现少量误差，允许是在极端情况（机器宕机）下，才出现误差。不允许出现大量误差的情况。 一般在和广告主的合同里面，都会有误差范围的要求。而且误差最终会被消除，也就是counter需要是最终一致性的。
2. 时延不能很大，比如我们希望的是counter能反映当时的impression/currency的情况。最好是时延是确定的，不能存在时延不确定的情况。
3. 因为广告系统的请求量很大， 高并发情况下的数据正确定是must have的需求。

之所以有以上要求，是因为ad server在广告投放逻辑中需要判断候选的的订单是不是已经完成广告投放，判断广告是否完成投放根据的就是counter的数值是不是已经达到订单规定的数值。 如果counter组件存在时延很大的情况，那么势必出现完成的广告投放量超过订单要求的量的情况。

从技术角度上将， 大致有两种思路， 一种是同步更新counter数值， 一种是异步更新counter数值。同步更新是指更新完counter数值后，再返回给ad server。 另一种是指示counter组件进行更新counter数值，但是不等counter更新后再返回，而是立即返回。

第一种方案，同步更新counter的数值，这种设计满足了系统的需求，但是会产生另外一个问题，在存在热点数据的情况下，表现不能满足需求。比如突发的某一个热点视频，会导致热点视频上的广告请求量很大。如果仅仅存在一个global counter的情况下， 我们需要保证counter的计数准确性，就势必造成热点视频上的广告response返回时间拉长，不能满足广告投放中低延迟的需求。


<div class="mermaid">
sequenceDiagram
Ad投放->>counter: 把订单111的已投放impression量加1
counter->>Ad投放: 加完了，你继续处理吧
</div>


第二种方案，异步更新counter的数值， 这种设计不能够很难满足对counter的时延的需求。在突发大流量的情况下，counter的延迟也在增加，那么势必出现完成的广告投放量大大超过订单要求的量的情况。


<div class="mermaid">
sequenceDiagram
Ad投放->>counter: 把订单111的已投放impression量加1
counter->>Ad投放: 好，我稍后给你加，你继续处理吧
</div>


另外，存在热点数据的情况下，不能仅仅存在一个global counter，如果仅仅存在一个global counter，成千上万的ad request并发去更新同一个key的counter的值，counter肯定很难在规定的时间内完成所有的更新counter的需求。 势必造成ad response不能及时返回。因此我们需要将counter拆分成多个counter，多个counter里面都有部分counter的数量， 所有的counter的counter值加起来就是global counter的值。热点ad request导致的大并发更新counter的需求也就能分散到多个counter里面。因此我们将counter分成两种， local counter和global counter。如下图所示。


<div class="mermaid">
sequenceDiagram
Ad投放->>local counter: 把订单111的已投放impression量加1
local counter->>Ad投放: 加完了，你继续处理吧
local counter->>global counter: 到时间了，我要把我的值更新到global counter中去。
</div>

这样子，我们只需要控制local counter更新数据到global counter的时间，我们就能够控制counter数据的时延了。 


另外， local counter可以和ad投放放同一个进程内，这样子就没有了网络请求，local counter定时将local counter值更新到global counter。 global counter可以采用redis持久化存储来持久化数据。local counter也需要定期把global counter的值刷新到local，ad投放会读取local counter的值来进行逻辑判断。

技术细节方面， 
1. local counter和global counter应该是分namespace的， 用以存储不同类型的counter。
2. 在local counter中，同一个namespace中的数据应该存储在一个golang map里面，这个map会有大量的并发读写的需求， 所以build-in map和sync.Map肯定是不能用了， 需要采用分段式锁实现的第三方map。
3. local counter也需要周期性的持久化，以防止ad投放机器宕机或者网络故障的情况下counter的数据的丢失。或者周期性的local counter的数据push到kafka中，使用kafka作临时存储。
4. global counter中需要定期淘汰过时的key，以便减少数据量，并且需要sync到local counter。



