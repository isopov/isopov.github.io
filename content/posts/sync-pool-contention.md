---
title: "Go sync.Pool contention"
date: 2023-03-24T16:22:39+03:00
draft: false
tags: ["Golang"]
---
Recently my colleague suggested that contention on a single sync.Pool in Go may hit performance. I decided to reproduce this performance hit in a benchmark. This is what I was capable of writing.

```go
import (
	"runtime"
	"strconv"
	"sync"
	"sync/atomic"
	"testing"
)

func workWithPool(pool *sync.Pool) {
	const size = 1000
	values := [size]int{}
	for i := 0; i < size; i++ {
		values[i] = pool.Get().(int)
	}
	for i := 0; i < size; i++ {
		pool.Put(values[i])
	}
}

func Benchmark_Pool(b *testing.B) {
	for _, parallelism := range []int{1, 2, 3, 4, 5, 10, 20, 50, 100} {
		b.Run("Parallelism "+strconv.Itoa(parallelism), func(b *testing.B) {

			b.Run("one pool", func(b *testing.B) {
				b.SetParallelism(parallelism)
				pool := &sync.Pool{
					New: func() any {
						return 42
					},
				}
				b.RunParallel(func(pb *testing.PB) {
					for pb.Next() {
						workWithPool(pool)
					}
				})
			})
			b.Run("many pools", func(b *testing.B) {
				b.SetParallelism(parallelism)
				numPools := runtime.GOMAXPROCS(0) * parallelism
				pools := make([]*sync.Pool, numPools)
				for i := 0; i < numPools; i++ {
					pools[i] = &sync.Pool{
						New: func() any {
							return 42
						},
					}
				}

				procs := uint32(0)
				b.RunParallel(func(pb *testing.PB) {
					poolidx := atomic.LoadUint32(&procs)
					for {
						if atomic.CompareAndSwapUint32(&procs, poolidx, poolidx+1) {
							break
						}
						poolidx = atomic.LoadUint32(&procs)
					}
					for pb.Next() {
						workWithPool(pools[poolidx])
					}
				})
			})
		})
	}
}
```
It seems that I need to comment a bit on the parallelism parameter. It stands for number of goroutines per each thread, while number of threads by default is equal to number of CPU cores. In my case I have 10 cores and thus 10 threads.
Here are the results with some omissions and reformatting to simplify reading.

```
Benchmark_Pool/Parallelism_1/one_pool         	  	      4513 ns/op
Benchmark_Pool/Parallelism_1/many_pools       	  	      2364 ns/op

Benchmark_Pool/Parallelism_2/one_pool         	  	      3078 ns/op
Benchmark_Pool/Parallelism_2/many_pools       	  	      2273 ns/op

Benchmark_Pool/Parallelism_3/one_pool         	  	      3144 ns/op
Benchmark_Pool/Parallelism_3/many_pools       	  	      2272 ns/op

Benchmark_Pool/Parallelism_4/one_pool         	  	      2979 ns/op
Benchmark_Pool/Parallelism_4/many_pools       	  	      2315 ns/op

Benchmark_Pool/Parallelism_5/one_pool         	  	      3355 ns/op
Benchmark_Pool/Parallelism_5/many_pools       	  	      2314 ns/op

Benchmark_Pool/Parallelism_10/one_pool        	  	      4948 ns/op
Benchmark_Pool/Parallelism_10/many_pools      	  	      2269 ns/op

Benchmark_Pool/Parallelism_20/one_pool        	  	      6979 ns/op
Benchmark_Pool/Parallelism_20/many_pools      	  	      2288 ns/op

Benchmark_Pool/Parallelism_50/one_pool        	  	      8324 ns/op
Benchmark_Pool/Parallelism_50/many_pools      	  	      2371 ns/op

Benchmark_Pool/Parallelism_100/one_pool       	  	     10044 ns/op
Benchmark_Pool/Parallelism_100/many_pools     	  	      2295 ns/op
```
It seems that with only 1 goroutine per CPU core this code for some reason suffers from contention more than with 2-5 goroutines. However in order to observe noticeable penalty we need 10+ goroutines per core. Keep in mind also that every operation here invokes Get on sync.Pool 1000 times and Put 1000 times also.

After writing and running this benchmark I conclude that there is no real danger in contention on sync.Pool and it may be used for really common low-level things.