+++ 
draft = false
date = 2022-02-25T13:30:49+03:00
title = "Go sync.Pool"
tags = ["Golang"]
+++ 
Recently I've seen sync.Pools in various Go libraries. Sometimes I wondered wether it is worth using at all. So I wrote some tests. Here are they:

```go
package main

import (
	"crypto/sha256"
	"hash"
	"strings"
	"sync"
	"testing"
)

var shaPool = &sync.Pool{
	New: func() interface{} {
		return sha256.New()
	},
}

var HashTrap hash.Hash
var Source = []byte(strings.ToLower("asdfavzxcvsdfqawesfdz"))
var ResultTrap []byte

func Benchmark(b *testing.B) {
	b.Run("Pool", func(b *testing.B) {
		b.ReportAllocs()
		for i := 0; i < b.N; i++ {
			HashTrap = shaPool.Get().(hash.Hash)
			ResultTrap = HashTrap.Sum(Source)
			HashTrap.Reset()
			shaPool.Put(HashTrap)
		}
	})
	b.Run("NoPool", func(b *testing.B) {
		b.ReportAllocs()
		for i := 0; i < b.N; i++ {
			HashTrap = sha256.New()
			ResultTrap = HashTrap.Sum(Source)
		}
	})
	b.Run("Local", func(b *testing.B) {
		b.ReportAllocs()
		for i := 0; i < b.N; i++ {
			h := sha256.New()
			ResultTrap = h.Sum(Source)
		}
	})
}
```
And the results:
```
Benchmark
Benchmark/Pool
Benchmark/Pool-8         	 4182459	       255.4 ns/op	      64 B/op	       1 allocs/op
Benchmark/NoPool
Benchmark/NoPool-8       	 4534873	       274.9 ns/op	     192 B/op	       2 allocs/op
Benchmark/Local
Benchmark/Local-8        	 5095738	       233.6 ns/op	      64 B/op	       1 allocs/op
```
If in your case sha256 hash struct allocation is not eliminated by go compiler, than pooling it may help a bit. But if it can be eliminated pooling will make only a bit worse.
And similar for my own struct instead of stsdlib sha256:
```
package main

import (
	"sync"
	"testing"
)

type Heavy struct {
	v1 uint64
	v2 uint64
	v3 uint64
	v4 uint64
	v5 uint64
	v6 uint64
	v7 uint64
	v8 uint64
}

var heavyPool = &sync.Pool{
	New: func() interface{} {
		return new(Heavy)
	},
}

func (q *Heavy) Release() {
	*q = Heavy{}
	heavyPool.Put(q)
}

func (q *Heavy) Init() {
	q.v1 = Counter
	Counter++
	q.v2 = Counter
	Counter++
	q.v3 = Counter
	Counter++
	q.v4 = Counter
	Counter++
	q.v5 = Counter
	Counter++
	q.v6 = Counter
	Counter++
	q.v7 = Counter
	Counter++
	q.v8 = Counter
	Counter++
}

func (q *Heavy) Count() {
	Result = q.v1 + q.v2*q.v3/q.v4 - q.v5 + q.v6*q.v7/q.v8
}

var Counter uint64 = 1
var Result uint64 = 0
var HeavyTrap *Heavy

func BenchmarkPool(b *testing.B) {
	b.Run("Pool", func(b *testing.B) {
		b.ReportAllocs()
		for i := 0; i < b.N; i++ {
			h := heavyPool.Get().(*Heavy)
			h.Init()
			h.Count()
			h.Release()
		}
	})
	b.Run("NoPool", func(b *testing.B) {
		b.ReportAllocs()
		for i := 0; i < b.N; i++ {
			HeavyTrap = &Heavy{}
			HeavyTrap.Init()
			HeavyTrap.Count()
		}
	})
	b.Run("Local", func(b *testing.B) {
		b.ReportAllocs()
		for i := 0; i < b.N; i++ {
			h := &Heavy{}
			h.Init()
			h.Count()
		}
	})
}
```
Once again, pooling may be handy, but if struct allocation is eliminated altogether it will only spoil things a bit.
```
BenchmarkPool
BenchmarkPool/Pool
BenchmarkPool/Pool-8         	36132384	        30.31 ns/op	       0 B/op	       0 allocs/op
BenchmarkPool/NoPool
BenchmarkPool/NoPool-8       	29203462	        42.57 ns/op	      64 B/op	       1 allocs/op
BenchmarkPool/Local
BenchmarkPool/Local-8        	73544418	        16.01 ns/op	       0 B/op	       0 allocs/op
```

