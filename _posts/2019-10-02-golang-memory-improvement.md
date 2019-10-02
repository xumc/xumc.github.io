---
layout: post
title: "golang内存优化实践"
date: 2019-10-02
---

最近线上的一个后台job类型的应用内存报警， 显示峰值内存使用率已经到了90%以上。这个应用是几年前从其team接手的一个，原作者早已”逃之夭夭“。回顾我接手这几年，这个应用基本没有做过业务逻辑的变化， 我对它的修改也仅仅是改改小bug。除此之外就是多次的内存优化了。这几年主要优化了下面几次。
1.前年春节期间内存占用过高，由于在春节期间，为了及时解决问题，找运维加的内存。从32G加到了64G.以为加了一倍的内存，可以免受打扰了吧。

2.然后随着业务数据的增长，今年年初这个应用又一次抽风，通过减少不必要的数据加载减少了峰值内存约10G。心想着这个应用快被替换了，能撑一段时间就行了吧，10G够顶一年半载了。

3.半个月前，内存又一次报警，才半年而已，这次已经没有办法减少不必要的数据加载了， 没有办法，只能多多啃啃项目代码，查看有没有优化的空间。最后没有更改任何业务逻辑，而是通过改变一些简单粗暴的写法，让这个应用峰值内存又少了10G，又续了一段时间命。主要做法如下：
这个应用每15分钟运行一次，每次运行都会先从mysql load数据到内存，然后根据load到内存的数据进行计算，计算完后更新mysql，收集数据统计信息。因为数据非常多，所以内存使用量也比较多。load数据的代码大约是这样子的。

```
type Element struct {
    id int64
    xxx_id int64
}
func main() {
    datamap := make(map[int64][int64])
    
    go func(dm) {
        dataslice, err := load_xxx()
        if err != nil {
            panic("error")
        }
        
        for _, e := range dataslice {
            dm[e.id] = e.xxx_id
        }
    }(datamap)
}

func load_xxx() (dataslice []*Element, err error) {
    query := fmt.Sprintf("select id, xxx_id from xxx_table")
    
    rows, err := db.Query(query)
    if err != nil {
        return nil, err
    }
    
    for rows.Next() {
        var id, xxx_id int64
        rows.Scan(id, xxx_id)
        dataslice = append(dataslice, &Element{id: id, xxx_id: xxx_id)
    }
    
    return dataslice
}
```

这种写法看似很普遍， 但是如果在大数据量的情况下，在load_xxx中的dataslice局部变量会占用大量的内存，而且这个变量是中间变量，不是我们最终要使用的数据。虽说是局部变量，但是这个变量会在golang的heap上分配内存，也就意味着GC压力会比较大。这些巨型的局部变量最终会导致内存使用率曲线图变成桂林山水一样的突然暴涨，突然暴跌。也就很容易导致峰值内存使用量的增加。为了消除这个巨无霸变量，我使用了buffered channel，load_xxx返回<- chan Element类型的buffered channel，在main函数里面使用range独缺这个channel的值，这样就使得巨无霸变量消失了。 减少了这个应用中的10多处这种巨无霸变量后，内存使用率暴涨暴跌的情况有些好转。

除此之外，还优化了一些struct的字段的顺序，这样也减少了一些内存的使用，在这里推荐github上的一个工具。https://github.com/leeychee/mlayout, 还是很有用的。


经过这三轮优化后，这个应用目测还能再撑半年。应该可以挨到这个应用逻辑的重构了。但是我总是觉得这个应用可以优化的地方还有很多，目前发现一处比较奇怪的地方，还没有弄明白其中的原因。这几天研究研究。
![avatar](../img/golang_memory_improvement.png)
