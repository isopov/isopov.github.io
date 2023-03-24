---
title: "Growing goroutine stacks"
date: 2023-03-24T20:22:39+03:00
draft: false
tags: ["Golang"]
---
Go allows to run very lightweight goroutines and throw them away (for garbage collector) after they are no longer needed. You can execute any code inside them and this caode can allocate as much as needed. Both on stack and on heap. If goroutine stack is too small it grows. This grow is not free. Suppose you have a widely used function that needs to allocate a bit of memory on stack. Depending on whether there is enough of memory on stack or not it will take different amount of time, since in some cases it will include time to grow the stack. It happens that a really common function time.Now() is allocating. So this effect can be demoed with this function. Consider the following program.

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	for depth := 0; depth < 1000; depth++ {
		var res time.Duration
		for i := 0; i < 10_000; i++ {
			done := make(chan struct{})
			go func() {
				res += twoNowsDiffInDepth(depth)
				close(done)
			}()
			<-done
		}
		fmt.Printf("%03d - %d\n", depth, res)
	}
}

func twoNowsDiffInDepth(depth int) time.Duration {
	if depth == 0 {
		return nowDiff(time.Now())
	}
	return twoNowsDiffInDepth(depth - 1)
}

func nowDiff(t time.Time) time.Duration {
	return time.Now().Sub(t)
}
```

The output has 1000 lines, obviously. Each line has total number of nanoseconds to call time.Now() on stacks depth from 1 to 1000 in a fresh goroutine 10 thousand times. Usually it takes 400000-600000 nanoseconds. However on some stack depth the result is different. Here is some lines from the output.

```
000 - 838890
001 - 796577
002 - 691937
003 - 578079
004 - 531705
005 - 499433
...
017 - 405944
018 - 9895890
019 - 9742951
020 - 594866
...
060 - 458445
061 - 20673514
062 - 20978750
063 - 617336
...
145 - 470577
146 - 42773890
147 - 42822470
148 - 462516
...
316 - 480823
317 - 87866537
318 - 89459648
319 - 438552
...
657 - 504671
658 - 177226079
659 - 177820189
660 - 484756
...
997 - 564084
998 - 556363
999 - 524782
```

It is clear that at some stack depths we run out of stack memory and need to create new stack and copy old stack to prefix of the new one. This operation takes time and the larger the stack the larger the cost. I found curious that without growing the stack the cost of calling time.Now() on different stack depths is constant. Sure there is price for unwinding stack if you want to return the value. Consider the following benchmark.

```go
package main

import (
	"strconv"
	"testing"
	"time"
)

var Blackhole time.Time

func BenchmarkNow(b *testing.B) {
	for depth := 0; depth < 100; depth++ {
		b.Run("Depth"+strconv.Itoa(depth), func(b *testing.B) {
			for i := 0; i < b.N; i++ {
				Blackhole = nowInDepth(depth)
			}
		})
	}
}

func nowInDepth(depth int) time.Time {
	if depth == 0 {
		return time.Now()
	}
	return nowInDepth(depth - 1)
}
```
Here are the results with boring ones ommitted.
```
BenchmarkNow/Depth0         		        41.12 ns/op
BenchmarkNow/Depth1         		        38.26 ns/op
BenchmarkNow/Depth2         		        38.82 ns/op
BenchmarkNow/Depth3         		        39.13 ns/op
...
BenchmarkNow/Depth15        		        40.29 ns/op
BenchmarkNow/Depth16        		        40.55 ns/op
BenchmarkNow/Depth17        		        43.13 ns/op
BenchmarkNow/Depth18        		       107.70 ns/op
BenchmarkNow/Depth19        		       103.90 ns/op
BenchmarkNow/Depth20        		       111.50 ns/op
...
BenchmarkNow/Depth28        		        96.53 ns/op
BenchmarkNow/Depth29        		        94.81 ns/op
BenchmarkNow/Depth30        		        92.30 ns/op
BenchmarkNow/Depth31        	 	       246.70 ns/op
BenchmarkNow/Depth32        	 	       255.40 ns/op
BenchmarkNow/Depth33        	 	       269.50 ns/op
BenchmarkNow/Depth36        	 	       267.80 ns/op
BenchmarkNow/Depth37        	 	       276.70 ns/op
BenchmarkNow/Depth38        	 	       310.40 ns/op
BenchmarkNow/Depth39        	 	       307.20 ns/op
...
BenchmarkNow/Depth78        	 	       368.00 ns/op
BenchmarkNow/Depth79        	 	       368.30 ns/op
BenchmarkNow/Depth80        	 	       364.30 ns/op
BenchmarkNow/Depth81        	 	       551.50 ns/op
BenchmarkNow/Depth82        	 	       565.40 ns/op
BenchmarkNow/Depth83        	 	       573.00 ns/op
...
BenchmarkNow/Depth97        	 	       624.90 ns/op
BenchmarkNow/Depth98        	 	       640.00 ns/op
BenchmarkNow/Depth99        	 	       649.90 ns/op
```
The cost of unwinding stack grows not linear. At some stack depths there are leaps. I believe they are caused by stack memory growing beyond different levels of memory caches.  