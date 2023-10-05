+++ 
date = 2023-10-05T13:30:49+03:00
title = "Go map of maps of maps vs complex keys"
tags = ["Golang"]
+++ 

In other languages I've seen single map performing better than a map of maps or even map of maps of maps. I wanted to test which way is better in Go. So I wrote this test:

```go
package main

import (
	"fmt"
	"testing"
)

type structID struct {
	first  int
	second int
	third  int
}

var Blackholemapmapmap map[int]map[int]map[int]string
var Blackholemapstruct map[structID]string

func BenchmarkName(b *testing.B) {
	b.Run("mapmapmap def maps", func(b *testing.B) {
		b.ReportAllocs()
		for i := 0; i < b.N; i++ {
			m := map[int]map[int]map[int]string{}
			test(func(first, second, third int, value string) {
				seconds := m[first]
				if seconds == nil {
					seconds = map[int]map[int]string{}
					m[first] = seconds
				}
				thirds := seconds[second]
				if thirds == nil {
					thirds = map[int]string{}
					seconds[second] = thirds
				}
				thirds[third] = value
			})
			Blackholemapmapmap = m
		}
	})
	b.Run("mapstruct def map", func(b *testing.B) {
		b.ReportAllocs()
		for i := 0; i < b.N; i++ {
			m := map[structID]string{}
			test(func(first, second, third int, value string) {
				m[structID{
					first:  first,
					second: second,
					third:  third,
				}] = value
			})
			Blackholemapstruct = m
		}
	})
	b.Run("mapmapmap exact maps", func(b *testing.B) {
		b.ReportAllocs()
		for i := 0; i < b.N; i++ {
			m := make(map[int]map[int]map[int]string, firstsCount)
			test(func(first, second, third int, value string) {
				seconds := m[first]
				if seconds == nil {
					seconds = make(map[int]map[int]string, secondsCount)
					m[first] = seconds
				}
				thirds := seconds[second]
				if thirds == nil {
					thirds = make(map[int]string, thirdsCount)
					seconds[second] = thirds
				}
				thirds[third] = value
			})
			Blackholemapmapmap = m
		}
	})
	b.Run("mapstruct exact map", func(b *testing.B) {
		b.ReportAllocs()
		for i := 0; i < b.N; i++ {
			m := make(map[structID]string, firstsCount*secondsCount*thirdsCount)
			test(func(first, second, third int, value string) {
				m[structID{
					first:  first,
					second: second,
					third:  third,
				}] = value
			})
			Blackholemapstruct = m
		}
	})
}

const (
	firstsCount  = 100
	secondsCount = 10
	thirdsCount  = 10
)

func test(add func(first, second, third int, value string)) {
	for first := 0; first < firstsCount; first++ {
		for second := 0; second < secondsCount; second++ {
			for third := 0; third < thirdsCount; third++ {
				add(first, second, third, fmt.Sprintf("%d-%d-%d", first, second, third))
			}
		}
	}
}
```

Running it I got these results.

```
BenchmarkName
BenchmarkName/mapmapmap_def_maps
BenchmarkName/mapmapmap_def_maps-10     634	  1787864 ns/op	 812567 B/op	13332 allocs/op
BenchmarkName/mapstruct_def_map
BenchmarkName/mapstruct_def_map-10      686	  1723303 ns/op	1606390 B/op	10125 allocs/op
BenchmarkName/mapmapmap_exact_maps
BenchmarkName/mapmapmap_exact_maps-10   750	  1614700 ns/op	 584977 B/op	12225 allocs/op
BenchmarkName/mapstruct_exact_map
BenchmarkName/mapstruct_exact_map-10    892	  1327966 ns/op	 814922 B/op	10003 allocs/op
PASS
```

It seems to me that there is no clear advantage of on way of writing things over the other. It is clear that specifying the exact size of the map upfront is beneficial, but that is the case with any single map also. Regarding the topic in question - one way we receive slightly better performance and less memory allocations, the other way we receive less memory allocated. What is more important depends on the context. 