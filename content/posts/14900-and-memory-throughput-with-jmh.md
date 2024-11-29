---
title: "Benchmarking memory throughput of 14900K CPU"
date: 2024-11-29T11:57:39+03:00
draft: false
tags: ["Java"]
---
In [previous post]({{< ref "/posts/14900-with-jmh" >}}) I benchmark a heavily CPU-bound code. It may not seem so, however it [appears to be memory bound]({{< ref "/posts/jmc-profiling-cpu-and-memory" >}}).
I've read somewhere that 14900 CPU efficient cores scale worse on memory accesses, than on just cracking numbers. So I wrote the following:

```java
@State(Scope.Thread)
@BenchmarkMode(Mode.AverageTime)
@Fork(3)
@OutputTimeUnit(TimeUnit.MICROSECONDS)
@Warmup(iterations = 3, time = 1, timeUnit = TimeUnit.SECONDS)
@Measurement(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
public class ThreadsBenchmark {

    @Param("default")
    public String threads;


    @Benchmark
    public BigInteger bench() {
        var result = BigInteger.ONE;
        for (int i = 1; i <= 2048; i++) {
            result = result.multiply(BigInteger.valueOf(i));
        }
        return result;
    }

    public static void main(String[] args) throws RunnerException {
        var results = new ArrayList<RunResult>();
        for (int i = 1; i <= 36; i++) {
            Options opt = new OptionsBuilder()
                    .include(".*" + ThreadsBenchmark.class.getSimpleName() + ".*")
                    .addProfiler(GCProfiler.class)
                    .verbosity(VerboseMode.SILENT)
                    .param("threads", String.valueOf(i))
                    .threads(i)
                    .build();
            results.addAll(new Runner(opt).run());
            System.out.println("Finished " + i);
        }

        OutputFormatFactory.createFormatInstance(System.out, Defaults.VERBOSITY).endRun(results);
    }
}
```
And got the following results:
```
Benchmark             (threads)  Mode  Cnt        Score      Error   Units
ThreadsBenchmark.bench        1  avgt   15      381.051 ±    6.303   us/op
ThreadsBenchmark.bench        2  avgt   15      404.597 ±    4.457   us/op
ThreadsBenchmark.bench        3  avgt   15      429.157 ±    3.651   us/op
ThreadsBenchmark.bench        4  avgt   15      470.670 ±    2.058   us/op
ThreadsBenchmark.bench        5  avgt   15      526.845 ±    3.049   us/op
ThreadsBenchmark.bench        6  avgt   15      595.859 ±    4.137   us/op
ThreadsBenchmark.bench        7  avgt   15      653.487 ±    3.804   us/op
ThreadsBenchmark.bench        8  avgt   15      708.640 ±    6.378   us/op
ThreadsBenchmark.bench        9  avgt   15      794.764 ±    7.892   us/op
ThreadsBenchmark.bench       10  avgt   15      879.006 ±    7.382   us/op
ThreadsBenchmark.bench       11  avgt   15      971.415 ±    8.570   us/op
ThreadsBenchmark.bench       12  avgt   15     1101.470 ±   56.425   us/op
ThreadsBenchmark.bench       13  avgt   15     1115.850 ±    5.051   us/op
ThreadsBenchmark.bench       14  avgt   15     1202.304 ±    5.871   us/op
ThreadsBenchmark.bench       15  avgt   15     1299.112 ±   10.511   us/op
ThreadsBenchmark.bench       16  avgt   15     1395.623 ±   10.643   us/op
ThreadsBenchmark.bench       17  avgt   15     1487.989 ±   10.312   us/op
ThreadsBenchmark.bench       18  avgt   15     1589.793 ±   10.064   us/op
ThreadsBenchmark.bench       19  avgt   15     1684.144 ±   11.208   us/op
ThreadsBenchmark.bench       20  avgt   15     1774.493 ±   18.495   us/op
ThreadsBenchmark.bench       21  avgt   15     1875.067 ±   16.203   us/op
ThreadsBenchmark.bench       22  avgt   15     1973.474 ±   11.102   us/op
ThreadsBenchmark.bench       23  avgt   15     2072.661 ±   16.053   us/op
ThreadsBenchmark.bench       24  avgt   15     2171.723 ±   14.909   us/op
ThreadsBenchmark.bench       25  avgt   15     2249.195 ±   16.467   us/op
ThreadsBenchmark.bench       26  avgt   15     2327.629 ±   11.873   us/op
ThreadsBenchmark.bench       27  avgt   15     2417.717 ±   11.460   us/op
ThreadsBenchmark.bench       28  avgt   15     2507.770 ±    7.841   us/op
ThreadsBenchmark.bench       29  avgt   15     2593.257 ±   15.216   us/op
ThreadsBenchmark.bench       30  avgt   15     2668.016 ±   10.484   us/op
ThreadsBenchmark.bench       31  avgt   15     2755.329 ±   16.218   us/op
ThreadsBenchmark.bench       32  avgt   15     2824.268 ±    6.769   us/op
ThreadsBenchmark.bench       33  avgt   15     2923.594 ±   13.014   us/op
ThreadsBenchmark.bench       34  avgt   15     3021.506 ±   22.468   us/op
ThreadsBenchmark.bench       35  avgt   15     3106.939 ±   11.508   us/op
ThreadsBenchmark.bench       36  avgt   15     3193.335 ±   14.416   us/op
```
I've omitted GCProfilers output. Norm allocation rate is constant across benchmarks as it should be and is equal to 4173400 B/op. 
Total allocation rate is more interesting:
```
Benchmark                         (threads)  Mode  Cnt        Score      Error   Units
ThreadsBenchmark.bench:·gc.alloc.rate     1  avgt   15    10445.158 ±  171.097  MB/sec
ThreadsBenchmark.bench:·gc.alloc.rate     2  avgt   15    19668.996 ±  215.412  MB/sec
ThreadsBenchmark.bench:·gc.alloc.rate     3  avgt   15    27810.120 ±  234.417  MB/sec
ThreadsBenchmark.bench:·gc.alloc.rate     4  avgt   15    33800.293 ±  149.677  MB/sec
ThreadsBenchmark.bench:·gc.alloc.rate     5  avgt   15    37741.635 ±  222.418  MB/sec
ThreadsBenchmark.bench:·gc.alloc.rate     6  avgt   15    40045.123 ±  274.842  MB/sec
ThreadsBenchmark.bench:·gc.alloc.rate     7  avgt   15    42593.372 ±  239.159  MB/sec
ThreadsBenchmark.bench:·gc.alloc.rate     8  avgt   15    44896.284 ±  388.314  MB/sec
ThreadsBenchmark.bench:·gc.alloc.rate     9  avgt   15    45246.806 ±  425.496  MB/sec
ThreadsBenchmark.bench:·gc.alloc.rate    10  avgt   15    45570.187 ±  450.535  MB/sec
ThreadsBenchmark.bench:·gc.alloc.rate    11  avgt   15    45503.650 ±  375.426  MB/sec
ThreadsBenchmark.bench:·gc.alloc.rate    12  avgt   15    43901.752 ± 2187.294  MB/sec
ThreadsBenchmark.bench:·gc.alloc.rate    13  avgt   15    46962.621 ±  179.555  MB/sec
ThreadsBenchmark.bench:·gc.alloc.rate    14  avgt   15    46922.119 ±  183.105  MB/sec
ThreadsBenchmark.bench:·gc.alloc.rate    15  avgt   15    46607.365 ±  352.800  MB/sec
ThreadsBenchmark.bench:·gc.alloc.rate    16  avgt   15    46193.735 ±  291.412  MB/sec
ThreadsBenchmark.bench:·gc.alloc.rate    17  avgt   15    46032.232 ±  246.677  MB/sec
ThreadsBenchmark.bench:·gc.alloc.rate    18  avgt   15    45633.265 ±  184.391  MB/sec
ThreadsBenchmark.bench:·gc.alloc.rate    19  avgt   15    45489.668 ±  155.867  MB/sec
ThreadsBenchmark.bench:·gc.alloc.rate    20  avgt   15    45376.666 ±  297.628  MB/sec
ThreadsBenchmark.bench:·gc.alloc.rate    21  avgt   15    45100.441 ±  218.035  MB/sec
ThreadsBenchmark.bench:·gc.alloc.rate    22  avgt   15    45053.339 ±  279.386  MB/sec
ThreadsBenchmark.bench:·gc.alloc.rate    23  avgt   15    44801.914 ±  216.204  MB/sec
ThreadsBenchmark.bench:·gc.alloc.rate    24  avgt   15    44563.037 ±  249.170  MB/sec
ThreadsBenchmark.bench:·gc.alloc.rate    25  avgt   15    44691.730 ±  193.041  MB/sec
ThreadsBenchmark.bench:·gc.alloc.rate    26  avgt   15    44813.572 ±  155.758  MB/sec
ThreadsBenchmark.bench:·gc.alloc.rate    27  avgt   15    44807.594 ±  178.611  MB/sec
ThreadsBenchmark.bench:·gc.alloc.rate    28  avgt   15    44730.577 ±  136.466  MB/sec
ThreadsBenchmark.bench:·gc.alloc.rate    29  avgt   15    44797.707 ±  263.377  MB/sec
ThreadsBenchmark.bench:·gc.alloc.rate    30  avgt   15    45044.544 ±  165.125  MB/sec
ThreadsBenchmark.bench:·gc.alloc.rate    31  avgt   15    45031.436 ±  193.781  MB/sec
ThreadsBenchmark.bench:·gc.alloc.rate    32  avgt   15    45254.605 ±  107.844  MB/sec
ThreadsBenchmark.bench:·gc.alloc.rate    33  avgt   15    45205.982 ±  115.027  MB/sec
ThreadsBenchmark.bench:·gc.alloc.rate    34  avgt   15    45121.584 ±  304.400  MB/sec
ThreadsBenchmark.bench:·gc.alloc.rate    35  avgt   15    45162.079 ±  119.766  MB/sec
ThreadsBenchmark.bench:·gc.alloc.rate    36  avgt   15    45257.289 ±  166.231  MB/sec
```
It rises with using more threads up to approximately 8 (CPU has 8 physical performance cores) and stay aroung 45 thousand MB/sec after that.

As in the previous post these results may be valuable to investigate potential degradations of serving many concurrent requests on the server, but since total amount of work is different in different benchmarks, they are hard to compare. So I've written also the following code:

```java
@BenchmarkMode(Mode.AverageTime)
@Fork(3)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
@Warmup(iterations = 3, time = 1, timeUnit = TimeUnit.SECONDS)
@Measurement(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
public class Threads2Benchmark {

    @Param({"1", "2", "3", "4", "5", "6", "7", "8", "9", "10", "11", "12", "13", "14", "15", "16",
            "17", "18", "19", "20", "21", "22", "23", "24", "25", "26", "27", "28", "29", "30", "31", "32",
            "33", "34", "35", "36",
    })
    public String threads;

    private static final int FACTORIALS = 128;

    @Benchmark
    public List<BigInteger> bench() throws ExecutionException, InterruptedException {
        try (var executor = Executors.newFixedThreadPool(Integer.parseInt(threads))) {
            var futures = new ArrayList<Future<BigInteger>>(FACTORIALS);
            for (int i = 0; i < FACTORIALS; i++) {
                futures.add(executor.submit(() -> {
                    var result = BigInteger.valueOf(1);
                    for (int j = 1; j <= 2048; j++) {
                        result = result.multiply(BigInteger.valueOf(j));
                    }
                    return result;
                }));
            }
            var result = new ArrayList<BigInteger>(FACTORIALS);
            for (int i = 0; i < FACTORIALS; i++) {
                result.add(futures.get(i).get());
            }
            return result;
        }

    }

    public static void main(String[] args) throws RunnerException {
        Options opt = new OptionsBuilder()
                .include(".*" + Threads2Benchmark.class.getSimpleName() + ".*")
                //From the comment to GCProfiler internals:
                //The problem is threads can come and go while performing the benchmark,
                //thus we would miss allocations made in a thread that was created and died between the snapshots.
                //So this profiler reports zeros for pur benchmark
                //.addProfiler(GCProfiler.class)
                .build();
        new Runner(opt).run();
    }
}
```
GCProfiler cannot help with it, however from the results it is clear that allocations were not removed by DCE. This benchmark produces the following result:

```
Benchmark               (threads)  Mode  Cnt   Score   Error  Units
ThreadsBenchmark.bench          1  avgt   15  51.556 ± 0.803  ms/op
ThreadsBenchmark.bench          2  avgt   15  30.049 ± 1.001  ms/op
ThreadsBenchmark.bench          3  avgt   15  22.340 ± 0.407  ms/op
ThreadsBenchmark.bench          4  avgt   15  18.544 ± 0.286  ms/op
ThreadsBenchmark.bench          5  avgt   15  16.336 ± 0.184  ms/op
ThreadsBenchmark.bench          6  avgt   15  15.114 ± 0.184  ms/op
ThreadsBenchmark.bench          7  avgt   15  14.417 ± 0.144  ms/op
ThreadsBenchmark.bench          8  avgt   15  13.758 ± 0.098  ms/op
ThreadsBenchmark.bench          9  avgt   15  12.940 ± 0.072  ms/op
ThreadsBenchmark.bench         10  avgt   15  12.500 ± 0.053  ms/op
ThreadsBenchmark.bench         11  avgt   15  12.260 ± 0.062  ms/op
ThreadsBenchmark.bench         12  avgt   15  12.122 ± 0.052  ms/op
ThreadsBenchmark.bench         13  avgt   15  11.931 ± 0.040  ms/op
ThreadsBenchmark.bench         14  avgt   15  11.929 ± 0.037  ms/op
ThreadsBenchmark.bench         15  avgt   15  11.812 ± 0.064  ms/op
ThreadsBenchmark.bench         16  avgt   15  11.836 ± 0.043  ms/op
ThreadsBenchmark.bench         17  avgt   15  11.846 ± 0.038  ms/op
ThreadsBenchmark.bench         18  avgt   15  11.801 ± 0.033  ms/op
ThreadsBenchmark.bench         19  avgt   15  11.831 ± 0.027  ms/op
ThreadsBenchmark.bench         20  avgt   15  11.831 ± 0.026  ms/op
ThreadsBenchmark.bench         21  avgt   15  11.770 ± 0.067  ms/op
ThreadsBenchmark.bench         22  avgt   15  11.813 ± 0.022  ms/op
ThreadsBenchmark.bench         23  avgt   15  11.770 ± 0.040  ms/op
ThreadsBenchmark.bench         24  avgt   15  11.724 ± 0.037  ms/op
ThreadsBenchmark.bench         25  avgt   15  11.730 ± 0.015  ms/op
ThreadsBenchmark.bench         26  avgt   15  11.711 ± 0.036  ms/op
ThreadsBenchmark.bench         27  avgt   15  11.686 ± 0.024  ms/op
ThreadsBenchmark.bench         28  avgt   15  11.655 ± 0.029  ms/op
ThreadsBenchmark.bench         29  avgt   15  11.664 ± 0.014  ms/op
ThreadsBenchmark.bench         30  avgt   15  11.672 ± 0.081  ms/op
ThreadsBenchmark.bench         31  avgt   15  11.650 ± 0.018  ms/op
ThreadsBenchmark.bench         32  avgt   15  11.674 ± 0.012  ms/op
ThreadsBenchmark.bench         33  avgt   15  11.680 ± 0.034  ms/op
ThreadsBenchmark.bench         34  avgt   15  11.697 ± 0.040  ms/op
ThreadsBenchmark.bench         35  avgt   15  11.731 ± 0.020  ms/op
ThreadsBenchmark.bench         36  avgt   15  11.721 ± 0.038  ms/op
```
The most gain is got from using the second thread of heavily memory allocation bound work. Probably two channels of RAM have something to deal with this result. However, adding even more threads gives even more speed increase.

General conclusion that I can make is that we should not worry about efficient cores - even if they are not helping, they do not make things worse. And in some cases they can help. So usual rule of thumb stays as it was - use all the cores/threads of the CPU and try not to only in some really specific cases. Just like previously in some specific cases we turned off hyper-threading.  