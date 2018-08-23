---
layout: post
title: "Golang Slice"
date: 2018-07-25
---

```go
package main

import "fmt"

type CollectionItem struct {
	ID             int64
}

type Collection []*CollectionItem

func doThing0(result *Collection) {
	temp := make(Collection, 1)
	temp[0] = &CollectionItem{ID: 1111}
        result = &temp
}

func doThing(result Collection) {
	temp := make(Collection, 1)
	temp[0] = &CollectionItem{ID: 1111}
        result = temp
}

func doThing2(result []*CollectionItem) {
	temp := make([]*CollectionItem, 1)
	temp[0] = &CollectionItem{ID: 1111}
        result = temp
}

func main() {
   	result0 := make(Collection, 0)
	doThing0(&result0)
	fmt.Println(&result0)

   	result := make(Collection, 0)
	doThing(result)
	fmt.Println(result)

   	result2 := make([]*CollectionItem, 0)
	doThing2(result2)
	fmt.Println(result2)
}
```
