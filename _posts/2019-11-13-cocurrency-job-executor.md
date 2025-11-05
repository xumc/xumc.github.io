---
layout: post
title: "撸了一个并发执行job的util库"
date: 2019-11-13
tags: Golang
---

在日常开发过程中，为了提高性能，我们经常会并发执行多个任务，让总执行时间变短。一般来说，我们可以用channel来实现，或者使用标准库里面的errgroup。但是这些都没有对并发执行的结果进行处理，需要我们写代码进行处理。 处理返回结果一般采用一个channel或者加了mutex的map等来实现。对于返回结果是同一个类型的任务来说，这个还好，但是如果我们的job返回的是不同类型的结果，当然也可以采用channel或者mutex来处理。

一般来说， 在web开发中，聚合数据的需求还是很多的， 比如当前数据需要访问service1来获取data1， 访问service2来获取data2.然后把data1和data2糅合到一块去。我们希望有一个通用的解决方案，而且我们希望结果的顺序和发起调用的顺序是一致的。这种情况下，使用channel或者mutex的map来处理的时候，我们就需要人工处理这种顺序，显得特别麻烦。

我的方案就是解决这个问题的。
```go

package main

import (
	"context"
	"fmt"
	"testing"
	"time"
)

func slowQuery(id int) int {
	time.Sleep(time.Second)
	return id * 10
}

type Job func() (interface{}, error)

type JobResult struct {
	Result interface{}
	Error error
}

func ConcurrencyExecute(jobs []Job) []JobResult {
	resultChan := make(chan JobResult, 0)
	var previousChan chan struct{}
	var notifyNextChan chan struct{}

	for i := 0; i < len(jobs); i++ {
		if i == 0 {
			previousChan = nil
		} else {
			previousChan = notifyNextChan
		}

		if i == (len(jobs) - 1) {
			notifyNextChan = nil
		} else {
			notifyNextChan = make(chan struct{}, 0)
		}


		go func(fn Job, previous chan struct{}, next chan struct{}, results chan JobResult) {
			ret, err := fn()

			if previous != nil {
				<- previous
			}

			results <- JobResult{
				Result: ret,
				Error:  err,
			}

			if next == nil {
				close(results)
			} else {
				close(next)
			}

		}(jobs[i], previousChan, notifyNextChan, resultChan)
	}

	results := make([]JobResult, 0)

	for result := range resultChan {
		results = append(results, result)
	}

	return results
}

func Test_concurrencyGet(t *testing.T) {
	jobs := []Job {
		func() (interface{}, error) {
			return slowQuery(1), nil
		},
		func() (interface{}, error) {
			return slowQuery(2), nil
		},
		func() (interface{}, error) {
			return "other type func", nil
		},
	}

	results := ConcurrencyExecute(jobs)

	for i := range results {
		fmt.Println(results[i])
	}
}

func Test_concurrencyGetWithCancel(t *testing.T) {
	ctx, cancel := context.WithCancel(context.Background())

	jobs := []Job {
		func() (interface{}, error) {
			return slowQuery(1), nil
		},
		func() (interface{}, error) {
			return slowQuery(2), nil
		},
		func() (interface{}, error) {
			select {
			case <-time.After(500 * time.Millisecond):
				return "job 3 done", nil
			case <-ctx.Done():
				return "job 3 canceled", nil
			}

			return "other type func", nil
		},
		func() (interface{}, error) {
			cancel()
			return "triggle cancel", nil
		},
	}

	results := ConcurrencyExecute(jobs)

	for i := range results {
		fmt.Println(results[i])
	}
}

```

代码还算简单， 大致解决思路就是每个goroutine执行完成后，并不是立即返回结果，而是要看前一个job执行完了没有，也即previous channal close了么。 
