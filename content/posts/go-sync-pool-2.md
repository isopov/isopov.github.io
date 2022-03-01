+++ 
draft = false
date = 2022-03-01T13:10:49+03:00
title = "Go sync.Pool and gc"
tags = ["Golang"]
+++ 
I've posted my brief experiments with sync.Pool in go. However, reading about it guides that it may be hard to use tool. And sometimes it can degrade for third-party reasons.
It is easy to trap into old information regarding sync.Pool and gc interrogation. [Since go 1.13 sync.Pool is not cleared completely on every gc](https://github.com/golang/go/commit/2dcbf8b3691e72d1b04e9376488cef3b6f93b286). However, it is still affected by gc. Suppose the following code:


```go
type Obj []int

func (o *Obj) Fill() {
	for i := 0; i < cap(*o); i++ {
		*o = append(*o, i)
	}
}

func (o Obj) Count() int {
	result := 0
	for i := 0; i < len(o); i++ {
		result += o[i]
	}
	return result
}

func NewObj() *Obj {
	o := make(Obj, 0, 100*1024)
	return &o
}

var objPool = &sync.Pool{
	New: func() interface{} {
		return NewObj()
	},
}

var ResultTrap int

func WorkWithPool() {
	o := objPool.Get().(*Obj)
	o.Fill()
	ResultTrap = o.Count()
	released := (*o)[:0]
	objPool.Put(&released)
}
```
And let's use it this way:
```go
func BenchmarkPool(b *testing.B) {
	b.Run("OnlyGC", func(b *testing.B) {
		b.ReportAllocs()
		for i := 0; i < b.N; i++ {
			runtime.GC()
		}
	})
	b.Run("NoGC", func(b *testing.B) {
		b.ReportAllocs()
		for i := 0; i < b.N; i++ {
			WorkWithPool()
		}
	})
	b.Run("OneGC", func(b *testing.B) {
		b.ReportAllocs()
		for i := 0; i < b.N; i++ {
			WorkWithPool()
			runtime.GC()
		}
	})
	b.Run("TwoGC", func(b *testing.B) {
		b.ReportAllocs()
		for i := 0; i < b.N; i++ {
			WorkWithPool()
			runtime.GC()
			runtime.GC()
		}
	})
	b.Run("ThreeGC", func(b *testing.B) {
		b.ReportAllocs()
		for i := 0; i < b.N; i++ {
			WorkWithPool()
			runtime.GC()
			runtime.GC()
			runtime.GC()
		}
	})
	b.Run("FourGC", func(b *testing.B) {
		b.ReportAllocs()
		for i := 0; i < b.N; i++ {
			WorkWithPool()
			runtime.GC()
			runtime.GC()
			runtime.GC()
			runtime.GC()
		}
	})
}
```
Here are the results:
```
BenchmarkPool
BenchmarkPool/OnlyGC
BenchmarkPool/OnlyGC-8         	   69956	    149475 ns/op	      13 B/op	       0 allocs/op
BenchmarkPool/NoGC
BenchmarkPool/NoGC-8           	   50950	    225802 ns/op	      40 B/op	       1 allocs/op
BenchmarkPool/OneGC
BenchmarkPool/OneGC-8          	   29601	    405608 ns/op	  495570 B/op	       4 allocs/op
BenchmarkPool/TwoGC
BenchmarkPool/TwoGC-8          	   19710	    612537 ns/op	  820298 B/op	       5 allocs/op
BenchmarkPool/ThreeGC
BenchmarkPool/ThreeGC-8        	   15132	    806346 ns/op	  820312 B/op	       5 allocs/op
BenchmarkPool/FourGC
BenchmarkPool/FourGC-8         	   12048	   1012356 ns/op	  820327 B/op	       5 allocs/op
```
Even one garbage collection hugely affects allocations of this test. Two garbage collections affect it even more, while any additionals have no effect. Suppose you have some near zero-allocating application that uses sync.Pools heavily in various places. Any change leading to memory allocation big enough for garbage collection happening may lead to 
cascading failure of protection from allocations and hiding the original cause of the allocation spike.


