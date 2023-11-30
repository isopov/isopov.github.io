+++
date = 2023-11-30T15:10:49+03:00
title = "Java garbarge collectors, generational hypothesis and throughput"
tags = ["Java"]
+++

In recent years Java and Hotspot JMV in particular received several new garbage collectors. Most recently generational
Z collector [was introduced](https://inside.java/2023/11/28/gen-zgc-explainer/).

It is common knowledge that it is possible to achieve almost any results in artificial benchmarks, but nonetheless it may
be interesting to create them anyway.

```java
@State(Scope.Thread)
@Fork(value = 1, jvmArgs = {"-Xmx100m"})
@Warmup(iterations = 3, time = 1)
@Measurement(iterations = 5, time = 1)
public class GenerationsBenchmark {

    public static final int OBJECTS_1MB = 1024 * 1024 / 16;
    private ArrayList<Object> olds;

    @Param({"0", "20", "40", "60"})
    public int oldObjects;

    @Setup
    public void setup() {
        olds = new ArrayList<>(oldObjects * OBJECTS_1MB);
        for (int i = 0; i < oldObjects * OBJECTS_1MB; i++) {
            olds.add(new Object());
        }
    }

    @Benchmark
    @Fork(jvmArgsPrepend = {"-XX:+UseG1GC"})
    public void g1(Blackhole bh) {
        bench(bh);
    }

    private static void bench(Blackhole bh) {
        for (int i = 0; i < OBJECTS_1MB; i++) {
            bh.consume(new Object());
        }
    }

    public static void main(String[] args) throws RunnerException {
        Options opt = new OptionsBuilder().include(".*" + GenerationsBenchmark.class.getSimpleName() + ".*").build();

        new Runner(opt).run();
    }
}
```
First of all this benchmark is flawed that it is run with 1 fork of each configuration only. But results are quite 
stable and difference between configurations is so large, that we may save the time. Also, 100 MB of heap is not a
common size for modern application, so this bench is truly artificial and micro. Some garbage collectors are not
suited for such a small heap, while the others are optimized for this extreme too. So the comparison is not fair at all. 
Any judgments based on it should be applicable only in a similar very unusual situation.

Besides small heap of 100MB various amounts of it are occupied by always referenced objects. In different runs the 
total amount of such objects varies between 0 and 60MB (or even slightly more acounting for ArrayList referencing them). 

But having said all of the above here are the results.
```
Benchmark                       (oldObjects)   Mode  Cnt      Score     Error  Units
GenerationsBenchmark.g1                    0  thrpt    5  12434.452 ±  80.775  ops/s
GenerationsBenchmark.g1                   20  thrpt    5  12415.055 ± 167.382  ops/s
GenerationsBenchmark.g1                   40  thrpt    5  11219.861 ± 165.378  ops/s
GenerationsBenchmark.g1                   60  thrpt    5   9816.738 ± 962.618  ops/s
```
G1 has been the default garbage collector for many years by now. We may consider its results as reference. But since our
test is measuring throughput lets see the results of throughput optimized collector - Parallel. Here is the test for it.
```java 
    @Benchmark
    @Fork(jvmArgsPrepend = {"-XX:+UseParallelGC"})
    public void parallelDefault(Blackhole bh) {
        bench(bh);
    }
```
With the results.
```
GenerationsBenchmark.parallelDefault       0  thrpt    5  12712.576 ± 144.766  ops/s
GenerationsBenchmark.parallelDefault      20  thrpt    5  10652.029 ± 210.238  ops/s
GenerationsBenchmark.parallelDefault      40  thrpt    5    411.664 ±  30.261  ops/s
GenerationsBenchmark.parallelDefault      60  thrpt    5    180.962 ±  12.367  ops/s
```
They are a bit disappointing since it cannot deal with large enough old heap region... But the total heap is too small
for Parallel collector - lets try Serial that should be basically the same, but without multiple threads overhead. 
```java
    @Benchmark
    @Fork(jvmArgsPrepend = {"-XX:+UseSerialGC"})
    public void serialDefault(Blackhole bh) {
        bench(bh);
    }
```
Results are:
```
GenerationsBenchmark.serialDefault         0  thrpt    5  13048.856 ± 131.335  ops/s
GenerationsBenchmark.serialDefault        20  thrpt    5  13023.243 ± 266.766  ops/s
GenerationsBenchmark.serialDefault        40  thrpt    5  13051.316 ±  91.953  ops/s
GenerationsBenchmark.serialDefault        60  thrpt    5    234.666 ±   8.511  ops/s
```
It seems that young and old region sizes are different for Serial and Parallel collectors. What about trying to tune them
ourselves?
```java
    @Benchmark
    @Fork(jvmArgsPrepend = {"-XX:+UseSerialGC"}, jvmArgsAppend = {"-XX:NewSize=10m"})
    public void serial10MBNew(Blackhole bh) {
        bench(bh);
    }
```
And
```
GenerationsBenchmark.serial10MBNew         0  thrpt    5  12547.985 ±  67.592  ops/s
GenerationsBenchmark.serial10MBNew        20  thrpt    5  12490.781 ± 202.933  ops/s
GenerationsBenchmark.serial10MBNew        40  thrpt    5  12273.471 ± 109.860  ops/s
GenerationsBenchmark.serial10MBNew        60  thrpt    5  12186.136 ± 382.122  ops/s
```
Throughput for the no or small old heap region decreased a bit, but we managed to get rid of tremendous regression in
the large old region case. I suppose G1 collector was capable of dealing in all these cases without manual tuning due to
its heap regions dynamically changing to belong to young or old generation. However, lets try to tune Parallel collector 
in a similar way.
```java
    @Benchmark
    @Fork(jvmArgsPrepend = {"-XX:+UseParallelGC"}, jvmArgsAppend = {"-XX:NewSize=10m"})
    public void parallel10MBNew(Blackhole bh) {
        bench(bh);
    }
```
And
```
GenerationsBenchmark.parallel10MBNew       0  thrpt    5  11286.414 ± 741.980  ops/s
GenerationsBenchmark.parallel10MBNew      20  thrpt    5   6177.843 ± 663.973  ops/s
GenerationsBenchmark.parallel10MBNew      40  thrpt    5   4595.050 ± 227.420  ops/s
GenerationsBenchmark.parallel10MBNew      60  thrpt    5   3869.422 ± 138.329  ops/s
```
The results are much better than without manual tuning, but still degradation is tremendous. We should not use Parallel
gc with so small heap.

After these tests lets look at modern non-generational collectors.
```java
    @Benchmark
    @Fork(jvmArgsPrepend = {"-XX:+UseShenandoahGC"})
    public void shenandoah(Blackhole bh) {
        bench(bh);
    }
```
And
```
GenerationsBenchmark.shenandoah            0  thrpt    5    288.230 ± 135.077  ops/s
GenerationsBenchmark.shenandoah           20  thrpt    5    531.991 ±  85.333  ops/s
GenerationsBenchmark.shenandoah           40  thrpt    5    394.158 ±  44.923  ops/s
GenerationsBenchmark.shenandoah           60  thrpt    5    197.308 ±  23.372  ops/s
```
As we can see Shenandoah is not suited for throughput oriented workloads with so small heaps. 

How about Z garbage collector?
```java
    @Benchmark
    @Fork(jvmArgsPrepend = {"-XX:+UseZGC"})
    public void zgc(Blackhole bh) {
        bench(bh);
    }
```
And
```
GenerationsBenchmark.zgc                   0  thrpt    5   7753.977 ± 275.895  ops/s
GenerationsBenchmark.zgc                  20  thrpt    5   1906.405 ±  56.030  ops/s
GenerationsBenchmark.zgc                  40  thrpt    5    606.713 ±   4.973  ops/s
GenerationsBenchmark.zgc                  60  thrpt    5     27.959 ±   0.774  ops/s
```
Results are much better, though old objects in heap cause degradation. However, I have started my post with a link
to generational Z collector introduction, so lets see how it will deal with the task.

```java
    @Benchmark
    @Fork(jvmArgsPrepend = {"-XX:+UseZGC", "-XX:+ZGenerational"})
    public void zgcGen(Blackhole bh) {
        bench(bh);
    }
```
And
```
GenerationsBenchmark.zgcGen                0  thrpt    5  10271.648 ± 308.686  ops/s
GenerationsBenchmark.zgcGen               20  thrpt    5   7874.979 ± 622.759  ops/s
GenerationsBenchmark.zgcGen               40  thrpt    5   4300.210 ± 367.747  ops/s
GenerationsBenchmark.zgcGen               60  thrpt    5    481.354 ±  94.315  ops/s
```
Definitely better and in degraded extreme case better than its direct competitor Shenandoah.

It seems that we can try manual tuning with generational Z collector as well.

```java
    @Benchmark
    @Fork(jvmArgsPrepend = {"-XX:+UseZGC", "-XX:+ZGenerational"}, jvmArgsAppend = {"-XX:NewSize=10m"})
    public void zgcGen10MBNew(Blackhole bh) {
        bench(bh);
    }
```
And
```
GenerationsBenchmark.zgcGen10MBNew         0  thrpt    5  10362.045 ± 484.911  ops/s
GenerationsBenchmark.zgcGen10MBNew        20  thrpt    5   7856.814 ± 488.436  ops/s
GenerationsBenchmark.zgcGen10MBNew        40  thrpt    5   4400.990 ± 292.039  ops/s
GenerationsBenchmark.zgcGen10MBNew        60  thrpt    5    508.789 ±  28.856  ops/s
```
Absolutely no difference. It seems that NewSize option specifies only start size of new generation heap region and it
will change after some iterations. Just like in G1 collector, but with not that level of success yet.

To sum up - in this non-realistic artificial case it is seen that default collector is the best since it does not need
any tuning options. And in my experience tuning options for GC are obsolete and do not reflect the modern needs of the
application. So I prefer to not tune anything and always question not only permgen size specified. (Note: in modern 
Hotspot JVM there is no permgen so this configuration presence is a clear indicator that GC options need revisiting).

Though all the details and numbers should not be relied on, I must say on more thing - about my testing environment. I
run all the tests on M1Pro Macbook with these versions:
```
# JMH version: 1.36
# VM version: JDK 21.0.1, OpenJDK 64-Bit Server VM, 21.0.1+12-LTS
```