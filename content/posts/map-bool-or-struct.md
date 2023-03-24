+++ 
draft = false
date = 2020-11-01T13:35:52+03:00
title = "Using map[]struct{} or map[]bool in Golang"
tags = ["Golang"]
+++
In Golang you do not have set data structure. So you can use map instead. When you do not need values there are two options widely used - using empty `struct{}` or `bool` as value. `struct{}` should be more performant, while `bool` is more convinient. I decided to check the first promise myslef. Golang have benchamrks support in its standard library. SO I wrote this one:
```go
package mapstruct

import (
	"math/rand"
	"testing"
)

func input() []int {
	slice := make([]int, 10_000)
	for i := 0; i < 10_000; i++ {
		slice[i] = rand.Intn(1_000)
	}
	return slice
}

func uniqueBool(input []int) []int {
	vals := map[int]bool{}
	for _, v := range input {
		vals[v] = true
	}
	result := make([]int, 0, len(vals))
	for v := range vals {
		result = append(result, v)
	}
	return result
}

func uniqueStruct(input []int) []int {
	vals := map[int]struct{}{}
	for _, v := range input {
		vals[v] = struct{}{}
	}
	result := make([]int, 0, len(vals))
	for v := range vals {
		result = append(result, v)
	}
	return result
}

func BenchmarkBool(b *testing.B) {
	for i := 0; i < b.N; i++ {
		uniqueBool(input())
	}
}

func BenchmarkStruct(b *testing.B) {
	for i := 0; i < b.N; i++ {
		uniqueStruct(input())
	}
}
```
And the result are quite variadic (Should developers write their own tooling to run benchamrks multiple times and compute averages, variances, etc?) Here are two results from different runs:
```
BenchmarkBool
BenchmarkBool-8     	    2378	    494532 ns/op
BenchmarkStruct
BenchmarkStruct-8   	    2210	    512692 ns/op
PASS

BenchmarkBool
BenchmarkBool-8     	    2272	    501090 ns/op
BenchmarkStruct
BenchmarkStruct-8   	    2313	    493226 ns/op
PASS
```
It seems the promise of more performant `map[]struct{}` is not true and I don't see any other reasons to use it over `map[]bool`.
