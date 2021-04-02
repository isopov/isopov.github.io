+++ 
draft = false
date = 2021-04-02T18:30:49+03:00
title = "Go scheduler fairness"
tags = ["Golang"]
+++ 
Recently in Jaeger I stumbled at an interesting trace - the work was done for 1 second, than 20 there was a gap for 20 seconds and after that work continued. My best guess currently is that goroutine slept for 20 seconds. To reproduce it I wrote the following:
```
package main

import (
	"context"
	"fmt"
	"runtime"
	"sync/atomic"
	"time"

	"github.com/couchbaselabs/ghistogram"
)

func main() {
	runtime.GOMAXPROCS(1)

	ctx, cancel := context.WithCancel(context.Background())
	workers := 300
	done := make([]chan bool, workers)
	for i := 0; i < workers; i++ {
		done[i] = make(chan bool)
	}

	times := ghistogram.NewHistogram(10, 5000, 0)
	minCount := int64(0)
	maxCount := int64(0)

	for i := 0; i < workers; i++ {
		doneChannel := done[i]
		count := int64(0)
		go func(ctx context.Context) {
			prev := time.Now()
			for {
				select {
				case <-ctx.Done():
					if count == 0 {
						println("Boom!")
					}
					for {
						savedMinCount := minCount
						if count < savedMinCount || savedMinCount == 0 {
							if atomic.CompareAndSwapInt64(&minCount, savedMinCount, count) {
								break
							}
						} else {
							break
						}
					}

					for {
						savedMaxCount := maxCount
						if count > savedMaxCount {
							if atomic.CompareAndSwapInt64(&maxCount, savedMaxCount, count) {
								break
							}
						} else {
							break
						}
					}
					close(doneChannel)
					return
				default:
					next := time.Now()
					dif := next.Sub(prev)
					times.Add(uint64(dif), 1)

					prev = next
					count++
				}
			}
		}(ctx)
	}

	time.Sleep(30 * time.Second)
	cancel()
	for i := 0; i < workers; i++ {
		<-done[i]
	}

	println("Times")
	for i, r := range times.Ranges {
		fmt.Printf("%s - %d\n", time.Duration(r), times.Counts[i])
	}
	fmt.Printf("min - %s\n", time.Duration(times.MinDataPoint))
	fmt.Printf("max - %s\n", time.Duration(times.MaxDataPoint))

	println("Counts")
	fmt.Printf("min - %d\n", int(minCount))
	fmt.Printf("max - %d\n", int(maxCount))
}
```
In this code I start 300 goroutines in 1 thread and try to iterate without any sleeps inside each goroutine. The only thing that is done in those loops - recording time of the iteration and storing it in histogram.
Output is:
```
Times
0s - 366750248
5µs - 634
10µs - 3142
15µs - 18721
20µs - 11191
25µs - 2397
30µs - 120
35µs - 2
40µs - 2
45µs - 1215
min - 58ns
max - 6.097832638s
Counts
min - 917729
max - 1502439
```
CPU time is distributed pretty good, but there are some outliers with largest more than 6 seconds out of only 30 seconds of all the working time. So my assumption about scheduler seems plausible.
