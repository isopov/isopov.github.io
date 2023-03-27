---
title: "Go channel(s) contention"
date: 2023-03-27T20:22:39+03:00
draft: false
tags: ["Golang"]
---
In the [previous post]({{< ref "/posts/sync-pool-contention" >}}) I've tried to measure overhead from highly contended access to sync.Pool on go. There was some measurable overhead, but it was mild. However in order for another goroutine to recieve some work to be done or/and to publish work results you may need go channel also. In this post I'll try to measure the overhead of contended access to go channel. 

So I've written the following code
```go
package main

import (
	"runtime"
	"strconv"
	"sync/atomic"
	"testing"
)

const workSize = 1000

func workWithChan(ch chan int) {
	for i := 0; i < workSize; i++ {
		select {
		case ch <- i:
		default:
			panic("chan is full")
		}
	}
	for i := 0; i < workSize; i++ {
		select {
		case <-ch:
		default:
			panic("chan is empty")
		}
	}
}

func Benchmark_Chan(b *testing.B) {
	for _, parallelism := range []int{1, 2, 3, 4, 5, 10, 20, 50, 100} {
		b.Run("Parallelism "+strconv.Itoa(parallelism), func(b *testing.B) {
			b.Run("one chan", func(b *testing.B) {
				b.SetParallelism(parallelism)
				ch := make(chan int, workSize*parallelism*runtime.GOMAXPROCS(0))
				b.RunParallel(func(pb *testing.PB) {
					for pb.Next() {
						workWithChan(ch)
					}
				})
			})
			b.Run("chan per core", func(b *testing.B) {
				b.SetParallelism(parallelism)
				numChans := runtime.GOMAXPROCS(0)
				chans := make([]chan int, numChans)
				for i := 0; i < numChans; i++ {
					chans[i] = make(chan int, workSize*parallelism)
				}

				procs := uint32(0)
				b.RunParallel(func(pb *testing.PB) {
					chanidx := atomic.LoadUint32(&procs)
					for {
						if atomic.CompareAndSwapUint32(&procs, chanidx, chanidx+1) {
							break
						}
						chanidx = atomic.LoadUint32(&procs)
					}
					chanidx = chanidx / uint32(parallelism)
					for pb.Next() {
						workWithChan(chans[chanidx])
					}
				})
			})
			b.Run("chan per goroutine", func(b *testing.B) {
				b.SetParallelism(parallelism)
				numChans := runtime.GOMAXPROCS(0) * parallelism
				chans := make([]chan int, numChans)
				for i := 0; i < numChans; i++ {
					chans[i] = make(chan int, workSize)
				}

				procs := uint32(0)
				b.RunParallel(func(pb *testing.PB) {
					chanidx := atomic.LoadUint32(&procs)
					for {
						if atomic.CompareAndSwapUint32(&procs, chanidx, chanidx+1) {
							break
						}
						chanidx = atomic.LoadUint32(&procs)
					}
					for pb.Next() {
						workWithChan(chans[chanidx])
					}
				})
			})
		})
	}
}
```
Here we have 3 variants - one pool shared between all the goroutines, one pool per each goroutine and one pool per each CPU core while additional parameter parallelism is equal to number of goroutines per CPU core.

Here are the results with some omissions and reformatting to simplify reading.

```
Benchmark_Chan/Parallelism_1/one_chan                  60868 ns/op
Benchmark_Chan/Parallelism_1/chan_per_core              3594 ns/op
Benchmark_Chan/Parallelism_1/chan_per_goroutine         3584 ns/op
Benchmark_Chan/Parallelism_2/one_chan                  61708 ns/op
Benchmark_Chan/Parallelism_2/chan_per_core              4758 ns/op
Benchmark_Chan/Parallelism_2/chan_per_goroutine         3328 ns/op
Benchmark_Chan/Parallelism_3/one_chan                  62023 ns/op
Benchmark_Chan/Parallelism_3/chan_per_core              5119 ns/op
Benchmark_Chan/Parallelism_3/chan_per_goroutine         3243 ns/op
Benchmark_Chan/Parallelism_4/one_chan                  63873 ns/op
Benchmark_Chan/Parallelism_4/chan_per_core              5343 ns/op
Benchmark_Chan/Parallelism_4/chan_per_goroutine         3375 ns/op
Benchmark_Chan/Parallelism_5/one_chan                  63416 ns/op
Benchmark_Chan/Parallelism_5/chan_per_core              5749 ns/op
Benchmark_Chan/Parallelism_5/chan_per_goroutine         3319 ns/op
Benchmark_Chan/Parallelism_10/one_chan                 63559 ns/op
Benchmark_Chan/Parallelism_10/chan_per_core             6549 ns/op
Benchmark_Chan/Parallelism_10/chan_per_goroutine        3243 ns/op
Benchmark_Chan/Parallelism_20/one_chan                 64031 ns/op
Benchmark_Chan/Parallelism_20/chan_per_core             7948 ns/op
Benchmark_Chan/Parallelism_20/chan_per_goroutine        3230 ns/op
Benchmark_Chan/Parallelism_50/one_chan                 64367 ns/op
Benchmark_Chan/Parallelism_50/chan_per_core            14881 ns/op
Benchmark_Chan/Parallelism_50/chan_per_goroutine        3285 ns/op
Benchmark_Chan/Parallelism_100/one_chan                64066 ns/op
Benchmark_Chan/Parallelism_100/chan_per_core           30397 ns/op
Benchmark_Chan/Parallelism_100/chan_per_goroutine       3273 ns/op
```
So with single channel for all goroutines we immediately recieve enourmous contention that is nearly equal with any number of goroutines per CPU core.

With separate channel per each goroutine there is no contention obviously and no overhead from it.

With a channel per CPU core we start seeing the larger overhead the more goroutines we launch per CPU core.

Together with the previous post I can make an assumption that contention on some single sync.Pool may not become a bottleneck in a real world application, while contention on a single channel may cause some degradation.