+++ 
draft = false
date = 2021-03-30T12:25:42+03:00
title = "Moving work to another goroutine"
tags = ["Golang"]
+++
Concurency in Go is super easy. Still to write it inside my very large work project I first needed to get my hands with it on really small and simple example. So I wrote one and it seems that this blog is a perfect place to store it.
```
package main

import (
	"time"
)

func main() {
	worker := make(chan int, 10)
	done := make(chan bool)

	go func() {
		for data := range worker {
			time.Sleep(50 * time.Millisecond)
			println(data)
		}
		close(done)
	}()

	for i := 1; i <= 20; i++ {
		worker <- i
	}

	close(worker)
	println("done settings tasks")
	<-done
}
```
The only thing that is too simplistic here is that we have only one worker goroutine. So let's make some more - we need a separate bool channel to track finish.
```
package main

import (
	"time"
)

func main() {
	worker := make(chan int, 10)
	workers := 2
	done := make([]chan bool, workers)
	for i := 0; i < workers; i++ {
		done[i] = make(chan bool)
	}

	for i := 0; i < 2; i++ {
		doneChannel := done[i]
		go func() {
			for data := range worker {
				time.Sleep(100 * time.Millisecond)
				println(data)
			}
			close(doneChannel)
		}()
	}

	for i := 1; i <= 30; i++ {
		worker <- i
	}

	close(worker)
	println("done settings tasks")
	for i := 0; i < workers; i++ {
		<-done[i]
	}
}
```
