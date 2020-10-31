---
title: "Benchmarking an interview question"
date: 2015-08-15T23:22:39+03:00
draft: false
---
*Author benchmarks a super-simple method with a dumb implementation from his recent job interview. Results of this benchmark contradict with assumptions. The process of benchmarking produces some public benefit.*


So, on a recent job interview it was suggested to write implementation for a really simple method

```
public int sum(int x, int y);
```

The main question for implementation is surely dealing with integer overflow (we were speaking about Java programming language). As it was left for me to decide what to do in case of overflow I suggested a standard way - throwing exception. In Java8 there is a standard method that does exactly this - Math.addExact. However I needed to re-implement it myself in order to use on older versions. To ease possible migration I used the same ArithmeticException that is used in the 8th version of standard library.

Another difficulty that is natural for a job interview is that I could look some nice bit-shifting trick on the Internet, but needed to write that implementation myself. Since we are dealing with ints it is really easy to write correct implementation just casting arguments to longs inside the method:

```
static int sum(int x, int y) {
        long xl = x;
        long yl = y;
        long result = xl + yl;

        if (result < Integer.MIN_VALUE || result > Integer.MAX_VALUE) {
            throw new ArithmeticException("integer overflow");
        }

        return (int) result;
    }
```

This method is correct - I wrote some tests to prove it. But my assumption was that it will be much slower than the standard one, or some using some bit-shifting operations. 

After the interview I have benchmarked it together with the standard one and was very surprised to see that this dumb implementation is even marginally better than the standard one on my laptop (Java8 on the x64 CPU and Linux OS). I decided to dive a bit deeper with the perf profiler and support for it in the JMH, but discovered that this support has been broken for my version of linux kernel and perf tool. I [reported this regression](http://mail.openjdk.java.net/pipermail/jmh-dev/2015-August/001996.html) and [it was fixed the same day](http://mail.openjdk.java.net/pipermail/jmh-dev/2015-August/002006.html) and [released on the next day](http://mail.openjdk.java.net/pipermail/jmh-dev/2015-August/002016.html). So even such a very simple interview question may produce some public benefit.

Back to this method, here are the result of benchmarking with the perfnorm profiler (some results omitted for brevity):

```
Benchmark                                          Mode  Cnt   Score    Error  Units
AddExactBenchmark.addWithLongs                     avgt   15   4.288 ±  0.043  ns/op
AddExactBenchmark.addWithLongs:·CPI                avgt    3   0.313 ±  0.014   #/op
AddExactBenchmark.addWithLongs:·branches           avgt    3   7.921 ±  2.076   #/op
AddExactBenchmark.addWithLongs:·cycles             avgt    3  12.057 ±  2.867   #/op
AddExactBenchmark.addWithLongs:·instructions       avgt    3  38.571 ±  8.450   #/op
AddExactBenchmark.mathAddExact                     avgt   15   4.306 ±  0.100  ns/op
AddExactBenchmark.mathAddExact:·CPI                avgt    3   0.356 ±  0.072   #/op
AddExactBenchmark.mathAddExact:·branches           avgt    3   6.782 ±  0.589   #/op
AddExactBenchmark.mathAddExact:·cycles             avgt    3  12.102 ±  3.079   #/op
AddExactBenchmark.mathAddExact:·instructions       avgt    3  33.972 ±  2.560   #/op
```

Let me comment these results a bit. The difference between methods is really small - it varies from run to run (maybe I should try more forks), but my dumb implementation is always better than the standard one. My next assumption (the very first one - that standard method is faster in all cases - was not true) is that this will not be the case on x86 and other 32-bit hardware and JVMs. This dumb implementation has more instructions than the standard one (instructions result), but exact instructions seems cheaper since CPU executes more of them per cycle (CPI - cycles per instruction - result that is the inverse of what I'm talking about) and as a sequence - a bit less cycles are used for execution. 

Another interesting result (probably it would be better to observe it with something like perfasm, but I have no such experience) is that this dumb implementation has more branches, than the standard one.

The source for this benchmark may be [found on Github](https://github.com/isopov/isopov-jmh/blob/master/src/main/java/com/sopovs/moradanen/jmh/AddExactBenchmark.java).
