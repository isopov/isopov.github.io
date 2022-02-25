+++ 
draft = true
date = 2022-02-025T13:30:49+03:00
title = "Go sync.Pool"
tags = ["Golang"]
+++ 
Recently I've seen sync.Pools in various Go libraries. Sometimes I wondered wether it is worth using at all. So I wrote some tests. Here are they:

```
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
