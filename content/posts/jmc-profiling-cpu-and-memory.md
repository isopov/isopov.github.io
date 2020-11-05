+++ 
draft = false
date = 2020-11-01T16:34:58+03:00
title = "Profiling Java CPU and RAM with Mission Control"
tags = ["Java"]
+++
Some time ago I was a lecturer on Java courses and wanted to craft a sample for profiling Java applications. Currently, we have plenty of profilers, and more than that - many of them are completely free to use. One of them is [Java Mission Control](https://openjdk.java.net/projects/jmc/). It was a commercial offering with a limited free version, but now it is completely free and open-sourced. It was required to buy it to use in production back in the days and this alone should give a hint, that this profiler is so lightweight that it can be used in production. Now you can download builds of JMC from different vendors. I downloaded one from [Azul](https://www.azul.com/products/zulu-mission-control/).

But let's dive into code. First of all, I wanted to provide a sample to profile the CPU-bounded algorithm. And I wrote this:
```
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RestController;

import java.math.BigInteger;

@RestController
public class CpuController {

    @GetMapping("/factorial/{num}")
    public Double burn(@PathVariable int num) {
        var result = BigInteger.ONE;
        for (int i = 1; i <= num; i++) {
            result = result.multiply(BigInteger.valueOf(i));
        }
        return result.doubleValue();
    }
}
```
What can be more CPU-intensive than crunching numbers? I started mission control flight recording with default settings and started updated the page `http://localhost:8080/factorial/100000`. Factorial for 100_000 is too large to be represented as double and I got `"Infinity"` all the time, but this computation lasted for several seconds every time, so there was more computation, then waiting for me requesting it one more time.

Right on the first page of the flight recording result, there is the following statement:
>The program allocated 95.2 % of the memory outside of TLABs.
>Allocating objects outside of Thread Local Allocation Buffers (TLABs) is more expensive than allocating inside TLABs. This may be acceptable if the individual allocations are intended to be larger than a reasonable TLAB. It may be possible to avoid this by decreasing the size of the individual allocations. There are some TLAB related JVM flags that you can experiment with, but it is usually better to let the JVM manage TLAB sizes automatically.

In the Memory section, I found that it was allocated more than 200GB of `int[]` arrays. That alone (and there were other allocations as well - 1.8GB of `BigInteger` objects among others) stand for more than 3 GB/s. And since these allocations were expensive outside TLABs (thread-local allocation buffers - allocating inside them is basically one internal memory increment most of the time) this is very close be the bottleneck of the algorithm. 

I wanted to make a CPU-bound program but unintentionally made a RAM-bound one. Profiling results were an unexpected but really good sample that our assumptions do not always hold.

Nevertheless, let's profile another algorithm, that was originally meant to be RAM-bound. Here it is:
```
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RestController;

import java.util.concurrent.ThreadLocalRandom;

@RestController
public class MemoryLoadController {

    @GetMapping("/allocate/{mbs}")
    public int allocate(@PathVariable int mbs) {
        final var random = ThreadLocalRandom.current();
        var dummyresult = 0;
        for (int i = 0; i < mbs; i++) {
            final byte[] mb = new byte[1024 * 1024];
            random.nextBytes(mb);
            dummyresult += sum(mb);
        }
        return dummyresult;
    }

    private static int sum(byte[] bytes) {
        var result = 0;
        for (byte b : bytes) {
            result += b;
        }
        return result;
    }
}
```
It should be memory bound since it is doing almost only allocations. However JVM has a good just-in-time compiler (JIT) with good dead code elimination (DCE) and we must be sure, that our allocations will actually happen. For that reason, I fill them with garbage. If we are lucky enough pre-zeroing of allocated arrays will be eliminated by the JIT. But to not get our garbage eliminated also let's make a checksum of it and return to the caller.

So I fired several requests to `http://localhost:8080/allocate/10000`. Moreover, to fully load the allocator I fired these requests concurrently. I have RAM with two channels after all. So during 1-minute flight recording even more memory should be allocated, than was with my "CPU-bound" algorithm. But! Actually, without firing up profiler it should be clear that something is not going as expected, since running only one request to allocate 10_000 MBs takes several long seconds and profiler gives as an overwhelming result that only 44 GBs of `byte[]` arrays were allocated during the profiling session. That is less than 1 GB/s! Sure, modern machines can do better. 

Let's look into the "Method Profiling" section. It has some unexpected results too. Most of the samples (7110) of this sampling profiler were caught in `java.util.Random.nextBytes(byte[])` method. That is my "filing with garbage" that ought to be very cheap and only serve the purpose of not eliminating allocation. The second method that caught most of samples is `MemoryLoadController.sum(byte[])` - 1696. The third is what I wanted to be the leader - `MemoryLoadController.allocate(int)` method itself with 8 samples, not counting any method calls inside it. That should be actual memory allocation.

It turns out that my second algorithm is not RAM-bound, but rather CPU-bound. That is another sample of our assumptions contradicting reality. But is it good, since in the end, I have two samples - one CPU-bound and one RAM-bound. Just that I have mistaken which one is which.
