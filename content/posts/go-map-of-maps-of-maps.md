+++ 
draft = true
date = 2023-04-13T13:30:49+03:00
title = "Go map of maps of maps vs complex keys"
tags = ["Golang"]
+++ 


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
And the results:
```
TODO
```