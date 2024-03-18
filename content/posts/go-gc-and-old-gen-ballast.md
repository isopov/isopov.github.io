+++
date = 2024-03-18T10:10:42+03:00
title = "Go GC, generational hypothesis and throughput"
tags = ["Golang"]
+++

In the [previous post]({{< ref "/posts/java-gc-and-generations" >}}) I investigated effect of old generation objects on Java garbage collectors. Here I want to run similar test for Go language. It is a bit less interesting since there is only one option, but nevertheless some useful insights can be taken out. Like how much RAM to give for GC breathing. Or maybe is it worth giving some extra GBs of RAM to squeeze out last tiny percents of performance. And when adding more RAM starts making worse (in theory it is easy to achieve this especially if you measure your success in terms of latency and not throughput). However, any complex test is better done with more realistic workload. Here I want to check just the simple assumptions and dependencies. 

First, I wrote this simple test to check how half the occupied heap affects garbage collection and overall performance.
Half heap ballast
```go 
package main

import (
	"runtime/debug"
	"testing"

	"golang.org/x/exp/rand"
)

var BBlackhole []byte

func BenchmarkBallast(b *testing.B) {
	debug.SetMemoryLimit(100 * 1024 * 1024) //100 MB

	//run everything 10 times to catch possible warmup effects
	for bench := 0; bench < 10; bench++ {
		b.Run("NoBallast", func(b *testing.B) {
			for i := 0; i < b.N; i++ {
				BBlackhole = make([]byte, 10*1024) //10KB
			}
		})

		//make ~50MB ballast in heap
		olds := make([][]byte, 5)
		for i := 0; i < 5; i++ {
			olds[i] = make([]byte, 10*1024*1024) //10 MB
		}

		b.Run("Ballast", func(b *testing.B) {
			for i := 0; i < b.N; i++ {
				BBlackhole = make([]byte, 10*1024) //10KB
			}
		})

		//"use" ballast so that it would no be cleared earlier
		BBlackhole = olds[rand.Intn(5)]
	}
}
```
The results (all the repetitions omitted for brevity - they are almost identical to each other) are quite disappointing.
```
BenchmarkBallast/NoBallast
BenchmarkBallast/NoBallast-10      	 1667966	       742.1 ns/op
BenchmarkBallast/Ballast
BenchmarkBallast/Ballast-10        	 3101397	       346.6 ns/op
```
With no old gen ballast performance it almost twice worse, than with it. But that is easily explained by the defaults of Go GC heuristics. Besides new and shiny option GOMEMLIMIT there is old, but still used GOGC option that by default trigger garbage collection after 200% heap is occupied compared to the state after the previous collection. With no old object in the heap that means GC triggering almost non-stop one right after another. Disabling this behaviour with this line (alternatively, just like with memory limit it can be done with env variable)

```go
	//to turn off automatic collections on nearly empty heap
	debug.SetGCPercent(-1)
```
We will receive the following results:
```
BenchmarkBallast/NoBallast
BenchmarkBallast/NoBallast-10      	 3771654	       306.6 ns/op
BenchmarkBallast/Ballast
BenchmarkBallast/Ballast-10        	 3588614	       348.4 ns/op
```
As expected allocation-bounded benchmark gets throughput performance boost from more free heap and less frequent garbage collections.

```go
package main

import (
	"runtime/debug"
	"strconv"
	"testing"

	"golang.org/x/exp/rand"
)

var Blackhole []byte

func BenchmarkAllocaWithOlds(b *testing.B) {
	debug.SetMemoryLimit(100 * 1024 * 1024) //100 MB
	//to turn off automatic collections on nearly empty heap
	debug.SetGCPercent(-1)
	for bench := 15; bench >= 0; bench-- {
		var olds [][]byte
		if bench != 0 {
			olds = make([][]byte, bench)
			for i := 0; i < bench; i++ {
				olds[i] = make([]byte, 10*1024*1024) //10 MB
			}
		}
		b.Run(strconv.Itoa(bench), func(b *testing.B) {
			for i := 0; i < b.N; i++ {
				Blackhole = make([]byte, 10*1024) //10 KB
			}
			if bench != 0 {
				Blackhole = olds[rand.Intn(bench)]
			}
		})
		olds = nil
	}
}
```
The results are:
```
BenchmarkAllocaWithOlds
BenchmarkAllocaWithOlds/15
BenchmarkAllocaWithOlds/15-10 	  611986	      2169 ns/op
BenchmarkAllocaWithOlds/14
BenchmarkAllocaWithOlds/14-10 	 1249198	      1013 ns/op
BenchmarkAllocaWithOlds/13
BenchmarkAllocaWithOlds/13-10 	 1164472	      1093 ns/op
BenchmarkAllocaWithOlds/12
BenchmarkAllocaWithOlds/12-10 	 1000000	      1095 ns/op
BenchmarkAllocaWithOlds/11
BenchmarkAllocaWithOlds/11-10 	 1000000	      1131 ns/op
BenchmarkAllocaWithOlds/10
BenchmarkAllocaWithOlds/10-10 	 1116847	      1071 ns/op
BenchmarkAllocaWithOlds/9
BenchmarkAllocaWithOlds/9-10  	 1073702	      1185 ns/op
BenchmarkAllocaWithOlds/8
BenchmarkAllocaWithOlds/8-10  	 1000000	      1080 ns/op
BenchmarkAllocaWithOlds/7
BenchmarkAllocaWithOlds/7-10  	 2256046	       505.5 ns/op
BenchmarkAllocaWithOlds/6
BenchmarkAllocaWithOlds/6-10  	 3219080	       384.3 ns/op
BenchmarkAllocaWithOlds/5
BenchmarkAllocaWithOlds/5-10  	 3429478	       346.5 ns/op
BenchmarkAllocaWithOlds/4
BenchmarkAllocaWithOlds/4-10  	 3476504	       335.5 ns/op
BenchmarkAllocaWithOlds/3
BenchmarkAllocaWithOlds/3-10  	 3768894	       334.3 ns/op
BenchmarkAllocaWithOlds/2
BenchmarkAllocaWithOlds/2-10  	 3570604	       338.5 ns/op
BenchmarkAllocaWithOlds/1
BenchmarkAllocaWithOlds/1-10  	 3689316	       320.5 ns/op
BenchmarkAllocaWithOlds/0
BenchmarkAllocaWithOlds/0-10  	 3913862	       311.2 ns/op
PASS
```
A bit interesting are the results with >100MB of old objects in a heap with 100MB soft limit. No errors are produced (unlike Java), but essentially garbage collections happens all the time trying to reclaim the RAM. It affects performance. With less than 100MB (or 80MB to be more specific) of old objects in the heap GC overhead decreases.

As a last experiment let's try to occupy heap not with 5 heavy old slices, but with 50 thousand.
```go
			olds = make([][]byte, bench*1024)
			for i := 0; i < bench*1024; i++ {
				olds[i] = make([]byte, 10*1024) //10 KB
			}
```
This is much more realistic (unless we are talking about usual before GOMEMLIMIT introduction memory ballast) and affects performace more.
```
BenchmarkAllocaWithOlds
BenchmarkAllocaWithOlds/15
BenchmarkAllocaWithOlds/15-10 	  196195	      6209 ns/op
BenchmarkAllocaWithOlds/14
BenchmarkAllocaWithOlds/14-10 	  260227	      4549 ns/op
BenchmarkAllocaWithOlds/13
BenchmarkAllocaWithOlds/13-10 	  319176	      3852 ns/op
BenchmarkAllocaWithOlds/12
BenchmarkAllocaWithOlds/12-10 	  233354	      5015 ns/op
BenchmarkAllocaWithOlds/11
BenchmarkAllocaWithOlds/11-10 	  258919	      6355 ns/op
BenchmarkAllocaWithOlds/10
BenchmarkAllocaWithOlds/10-10 	  200463	      5774 ns/op
BenchmarkAllocaWithOlds/9
BenchmarkAllocaWithOlds/9-10  	  206736	      5483 ns/op
BenchmarkAllocaWithOlds/8
BenchmarkAllocaWithOlds/8-10  	 1000000	      1112 ns/op
BenchmarkAllocaWithOlds/7
BenchmarkAllocaWithOlds/7-10  	 1908793	       641.3 ns/op
BenchmarkAllocaWithOlds/6
BenchmarkAllocaWithOlds/6-10  	 2411346	       494.2 ns/op
BenchmarkAllocaWithOlds/5
BenchmarkAllocaWithOlds/5-10  	 2926117	       442.5 ns/op
BenchmarkAllocaWithOlds/4
BenchmarkAllocaWithOlds/4-10  	 3119446	       423.4 ns/op
BenchmarkAllocaWithOlds/3
BenchmarkAllocaWithOlds/3-10  	 3158749	       354.8 ns/op
BenchmarkAllocaWithOlds/2
BenchmarkAllocaWithOlds/2-10  	 3334771	       347.9 ns/op
BenchmarkAllocaWithOlds/1
BenchmarkAllocaWithOlds/1-10  	 3824101	       318.6 ns/op
BenchmarkAllocaWithOlds/0
BenchmarkAllocaWithOlds/0-10  	 3775035	       320.7 ns/op
PASS
```
Some conclusions can be deducted from these simplistic benchmarks. However, only basic ones and I suggest to alter your configs or code of production applications only after load testing and profiling them under real load. 