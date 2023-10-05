+++ 
date = 2023-10-05T16:10:49+03:00
title = "Java map of maps of maps vs record keys"
tags = ["Java"]
+++ 

In the [previous post]({{< ref "/posts/go-map-of-maps-of-maps" >}}) I investigated using single map with struct keys verses using map of maps of maps in Go language. Here I want to run the same test for Java.

```java
@State(Scope.Thread)
@BenchmarkMode(Mode.AverageTime)
@Fork(3)
@OutputTimeUnit(TimeUnit.MICROSECONDS)
@Warmup(iterations = 3, time = 1, timeUnit = TimeUnit.SECONDS)
@Measurement(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
public class MapMapMapBenchmark {

    @Benchmark
    public Map<Integer, Map<Integer, Map<Integer, String>>> mapMapMap() {
        var firsts = new HashMap<Integer, Map<Integer, Map<Integer, String>>>();
        test(((first, second, third, value) -> {
            var seconds = firsts.computeIfAbsent(first, unused -> new HashMap<>());
            var thirds = seconds.computeIfAbsent(second, unused -> new HashMap<>());
            thirds.put(third, value);
        }));
        return firsts;
    }

    @Benchmark
    public Map<Integer, Map<Integer, Map<Integer, String>>> mapMapMapExact() {
        var firsts = HashMap.<Integer, Map<Integer, Map<Integer, String>>>newHashMap(FIRSTS);
        test(((first, second, third, value) -> {
            var seconds = firsts.computeIfAbsent(first, unused -> HashMap.newHashMap(SECONDS));
            var thirds = seconds.computeIfAbsent(second, unused -> HashMap.newHashMap(THIRDS));
            thirds.put(third, value);
        }));
        return firsts;
    }

    record IntKey(int first, int second, int third) {
    }

    @Benchmark
    public Map<IntKey, String> singleMap() {
        var map = new HashMap<IntKey, String>();
        test((first, second, third, value) -> map.put(new IntKey(first, second, third), value));
        return map;
    }


    @Benchmark
    public Map<IntKey, String> singleMapExact() {
        var map = HashMap.<IntKey, String>newHashMap(FIRSTS * SECONDS * THIRDS);
        test((first, second, third, value) -> map.put(new IntKey(first, second, third), value));
        return map;
    }

    interface Tester {
        void accept(int first, int second, int third, String value);
    }

    //not static, not final to prevent constant folding (see JMHSample_10_ConstantFold)
    private int FIRSTS = 100;
    private int SECONDS = 10;
    private int THIRDS = 10;

    private void test(Tester tester) {
        for (int first = 0; first < FIRSTS; first++) {
            for (int second = 0; second < SECONDS; second++) {
                for (int third = 0; third < THIRDS; third++) {
                    tester.accept(first, second, third, first + "-" + second + "-" + third);
                }
            }
        }
    }

    public static void main(String[] args) throws RunnerException {
        Options opt = new OptionsBuilder()
                .include(".*" + MapMapMapBenchmark.class.getSimpleName() + ".*")
                .addProfiler(GCProfiler.class)
                .build();

        new Runner(opt).run();
    }
}
```

Running it I got these results.

```
Benchmark                                              Mode  Cnt        Score     Error   Units
MapMapMapBenchmark.mapMapMap                           avgt   15      379.447 ±  15.983   us/op
MapMapMapBenchmark.mapMapMap:·gc.alloc.rate            avgt   15     2461.668 ± 105.561  MB/sec
MapMapMapBenchmark.mapMapMap:·gc.alloc.rate.norm       avgt   15   978128.156 ±   0.007    B/op
MapMapMapBenchmark.mapMapMap:·gc.count                 avgt   15      124.000            counts
MapMapMapBenchmark.mapMapMap:·gc.time                  avgt   15       79.000                ms
MapMapMapBenchmark.mapMapMapExact                      avgt   15      292.492 ±  30.147   us/op
MapMapMapBenchmark.mapMapMapExact:·gc.alloc.rate       avgt   15     4263.455 ± 413.234  MB/sec
MapMapMapBenchmark.mapMapMapExact:·gc.alloc.rate.norm  avgt   15  1297112.120 ±   0.012    B/op
MapMapMapBenchmark.mapMapMapExact:·gc.count            avgt   15      202.000            counts
MapMapMapBenchmark.mapMapMapExact:·gc.time             avgt   15      146.000                ms
MapMapMapBenchmark.singleMap                           avgt   15      162.902 ±   0.913   us/op
MapMapMapBenchmark.singleMap:·gc.alloc.rate            avgt   15     6856.225 ±  38.460  MB/sec
MapMapMapBenchmark.singleMap:·gc.alloc.rate.norm       avgt   15  1171248.067 ±   0.002    B/op
MapMapMapBenchmark.singleMap:·gc.count                 avgt   15      222.000            counts
MapMapMapBenchmark.singleMap:·gc.time                  avgt   15      151.000                ms
MapMapMapBenchmark.singleMapExact                      avgt   15      129.910 ±   1.519   us/op
MapMapMapBenchmark.singleMapExact:·gc.alloc.rate       avgt   15     8116.308 ±  92.953  MB/sec
MapMapMapBenchmark.singleMapExact:·gc.alloc.rate.norm  avgt   15  1105616.054 ±   0.002    B/op
MapMapMapBenchmark.singleMapExact:·gc.count            avgt   15      260.000            counts
MapMapMapBenchmark.singleMapExact:·gc.time             avgt   15      190.000                ms
```

The map of maps of maps case without upfront map creation is obviously the slowest one. However its allocations per operations are the smallest too. But the performance difference is so huge, that it is pretty obvious that single map with record keys should be preferred. This may be retested with non-primitive record fields, however is some CPU-bound tasks the advantage of on way of dealing with maps over the other is obvious.