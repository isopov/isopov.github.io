+++ 
draft = false
date = 2024-08-05T11:55:49+03:00
title = "Go concurrent sort of the same slice"
tags = ["Golang"]
+++ 
Recently I've stumbled upon a broken cache in my application written in Go. Some slices of structs got from database were
cached in memory. In the hindsight the problem is obvious - every consumer of this cache sorted these slices independently.
And since slices were cached in some cases they were sorting one and the same slice concurrently. Someone may suggest that
sorting should be done in the database with sort by, but even in the simplest case where it can be done on this level simply
adding `SORT BY` clause it may be not desirable. Usually application layer is scaled much easier than the storage layer.
Only some newest databases introduce their own scalable computational nodes separate from the storage nodes. Traditionally 
you may want to make your queries as lightweight as possible and move some CPU-bound postprocessing to application layer.
Also, you don't want to query your database for one and the same information over and over. So caches are introduced to the
application layer. And here we go with DAO -> Cache -> Provider/Manager. 

Here is the code that emulates this concurrent sorting of one and the same slice without database and cache layers.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"sort"
	"sync"
)

func main() {
	const size = 100
	s := make([]int, size)
	for i := 0; i < size; i++ {
		s[i] = i
	}
	rand.Shuffle(size, func(i, j int) {
		s[i], s[j] = s[j], s[i]
	})

	start := sync.WaitGroup{}
	start.Add(1)

	const goroutines = 10
	end := sync.WaitGroup{}
	end.Add(goroutines)
	for i := 0; i < goroutines; i++ {
		go func() {
			start.Wait()
			sort.Slice(s, func(i, j int) bool {
				return s[i] < s[j]
			})
			end.Done()
		}()
	}
	start.Done()
	end.Wait()

	incorrect := 0
	for i := 0; i < size; i++ {
		if s[i] != i {
			fmt.Printf("%d instead of %d\n", s[i], i)
			incorrect++
			if incorrect > 10 {
				return
			}
		}
	}
}
```
And the results:
```
1 instead of 0
5 instead of 1
5 instead of 2
5 instead of 3
6 instead of 4
6 instead of 5
6 instead of 7
7 instead of 8
13 instead of 11
14 instead of 12
14 instead of 13
```
As you can see not only some elements of the slice are in the wrong places after this sort, but they are duplicated. This
problem is easily solved moving the sort below the cache and returning already sorted slice from the cache. Not that easy
solution is making the cached value immutable - you cannot simply get slices from the cache in this case.