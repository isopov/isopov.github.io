---
title: "Benchmarking 14900K CPU"
date: 2024-11-25T22:42:39+03:00
draft: false
---

So, I got bored with waiting for tests of my laptop and built myself a new desktop for the first time in 10 years. I've built it around 14900K CPU and no separate GPU at all. For the OS I use Ubuntu 24.04. And I wanted to look closer to the differences between cores in this CPU. It has 8 performance cores and 16 efficient cores. So I wrote this benchmark to test them.

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
    public byte[] bench() {
        var random = RandomGenerator.getDefault();
        var bytes = new byte[1024];
        for (int i = 0; i < 1024; i++) {
            random.nextBytes(bytes);
        }
        return bytes;
    }

    public static void main(String[] args) throws RunnerException {
        var results = new ArrayList<RunResult>();
        for (int i = 1; i <= 36; i++) {
            Options opt = new OptionsBuilder()
                    .include(".*" + ThreadsBenchmark.class.getSimpleName() + ".*")
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
I've run it with 
```
$ java -version
openjdk version "21.0.5" 2024-10-15
OpenJDK Runtime Environment (build 21.0.5+11-Ubuntu-1ubuntu124.04)
OpenJDK 64-Bit Server VM (build 21.0.5+11-Ubuntu-1ubuntu124.04, mixed mode, sharing)
```
And got the following results:
```
Benchmark               (threads)  Mode  Cnt     Score    Error  Units
ThreadsBenchmark.bench          1  avgt   15   456.827 ±  1.539  us/op
ThreadsBenchmark.bench          2  avgt   15   478.378 ±  0.669  us/op
ThreadsBenchmark.bench          3  avgt   15   478.158 ±  2.052  us/op
ThreadsBenchmark.bench          4  avgt   15   478.451 ±  2.894  us/op
ThreadsBenchmark.bench          5  avgt   15   479.349 ±  2.073  us/op
ThreadsBenchmark.bench          6  avgt   15   477.210 ±  2.160  us/op
ThreadsBenchmark.bench          7  avgt   15   476.744 ±  1.122  us/op
ThreadsBenchmark.bench          8  avgt   15   467.190 ± 21.820  us/op
ThreadsBenchmark.bench          9  avgt   15   510.866 ± 26.721  us/op
ThreadsBenchmark.bench         10  avgt   15   527.802 ± 31.445  us/op
ThreadsBenchmark.bench         11  avgt   15   558.562 ± 33.271  us/op
ThreadsBenchmark.bench         12  avgt   15   583.522 ± 38.983  us/op
ThreadsBenchmark.bench         13  avgt   15   586.813 ± 38.401  us/op
ThreadsBenchmark.bench         14  avgt   15   601.645 ± 40.331  us/op
ThreadsBenchmark.bench         15  avgt   15   616.489 ± 37.435  us/op
ThreadsBenchmark.bench         16  avgt   15   626.235 ± 39.271  us/op
ThreadsBenchmark.bench         17  avgt   15   640.925 ± 45.910  us/op
ThreadsBenchmark.bench         18  avgt   15   653.792 ± 45.098  us/op
ThreadsBenchmark.bench         19  avgt   15   649.266 ± 36.483  us/op
ThreadsBenchmark.bench         20  avgt   15   661.931 ± 40.578  us/op
ThreadsBenchmark.bench         21  avgt   15   675.848 ± 38.651  us/op
ThreadsBenchmark.bench         22  avgt   15   689.341 ± 35.174  us/op
ThreadsBenchmark.bench         23  avgt   15   705.485 ± 31.321  us/op
ThreadsBenchmark.bench         24  avgt   15   720.715 ± 30.871  us/op
ThreadsBenchmark.bench         25  avgt   15   745.423 ± 35.487  us/op
ThreadsBenchmark.bench         26  avgt   15   772.899 ± 30.168  us/op
ThreadsBenchmark.bench         27  avgt   15   796.469 ± 34.395  us/op
ThreadsBenchmark.bench         28  avgt   15   822.673 ± 33.470  us/op
ThreadsBenchmark.bench         29  avgt   15   839.802 ± 34.424  us/op
ThreadsBenchmark.bench         30  avgt   15   863.312 ± 34.641  us/op
ThreadsBenchmark.bench         31  avgt   15   876.696 ± 38.417  us/op
ThreadsBenchmark.bench         32  avgt   15   897.369 ± 35.127  us/op
ThreadsBenchmark.bench         33  avgt   15   925.832 ± 39.338  us/op
ThreadsBenchmark.bench         34  avgt   15   961.630 ± 37.463  us/op
ThreadsBenchmark.bench         35  avgt   15   989.904 ± 39.702  us/op
ThreadsBenchmark.bench         36  avgt   15  1031.029 ± 44.007  us/op
```
I expected to get some serious shifts between 8 and 9 threads since I have 8 performance cores. And basically get that. 

Next shift I expected between 16 and 17 threads, since I have 8 performance cores with hyper-threading. However, got nothing like that. Performance on each thread slowly dropped with each new thread added to the bench.

This performance drop on each thread continued after number of threads exceeded the number of CPU threads. 

From what I can see it seems that there is no need to profit to limit concurrency on this CPU to the number of performance threads at least in some cases demonstrated here.