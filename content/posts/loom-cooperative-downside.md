+++ 
draft = false
date = 2022-06-05T13:30:49+03:00
title = "Java Loom Thread Fairness with Noisy Neighbour"
tags = ["Java"]
+++ 
Recently Gunnar Morling [posted](https://www.morling.dev/blog/loom-and-thread-fairness/) about Loom thread fairness. However despite the fact that difference between Loom and platform threads was clearly visible, Loom was even better. Total execution time was equal (or should be even better for Loom). But also there was  some visible gradual progress.

However it is possible to get into situation where Loom virtual threads will behave worse than the platform threads. This situation is a pretty common pattern and it is called [noisy neighbor problem](https://en.wikipedia.org/wiki/Cloud_computing_issues#Performance_interference_and_noisy_neighbors). Nowadays we usually meet it microservices level, but the general idea is universal. So here is how it can be met inside a single JVM:


```java
import java.math.BigInteger;
import java.util.ArrayList;
import java.util.concurrent.*;

public class LoomTest {

    public static void main(String[] args) throws ExecutionException, InterruptedException {
        long platform = 0;
        long loom = 0;
        for (int i = 0; i < 5; i++) {
            platform += report(Thread.ofPlatform().factory());
            loom += report(Thread.ofVirtual().factory());
        }
        System.out.println("loom     " + TimeUnit.NANOSECONDS.toMillis(loom));
        System.out.println("platform " + TimeUnit.NANOSECONDS.toMillis(platform));
    }

    private static final int TASKS = 1_000;

    private static long report(ThreadFactory threadFactory) throws ExecutionException, InterruptedException {
        var threads = new ArrayList<Thread>(2 * TASKS);
        var lights = new ArrayList<LightWorker>(TASKS);
        for (int i = 0; i < TASKS; i++) {
            Thread thread = threadFactory.newThread(new HeavyWorker());
            threads.add(thread);
            thread.start();
        }
        for (int i = 0; i < TASKS; i++) {
            LightWorker light = new LightWorker();
            lights.add(light);

            Thread thread = threadFactory.newThread(light);
            threads.add(thread);

            thread.start();
        }
        for (var thread : threads) {
            thread.join();
        }
        long result = 0;
        for (var light : lights) {
            result += light.time;
        }
        return result;

    }

    private static class LightWorker implements Runnable {
        private final long start;
        private long time;

        private LightWorker() {
            this.start = System.nanoTime();
        }

        @Override
        public void run() {
            time = System.nanoTime() - start;
        }
    }

    public static long blackHole;

    private static class HeavyWorker implements Runnable {

        public void run() {
            BigInteger res = BigInteger.ZERO;
            for (int j = 0; j < 1_000; j++) {
                res = res.add(BigInteger.valueOf(1L));
            }
            blackHole = res.longValue();
        }
    }
}
```

The idea is that we have two kinds of tasks - heavy and light. We do not care about performance of heavy tasks, so we even do not measure it. But we do care about light tasks performance. For this extreme case they do nothing except measuring their own performance. Also we start all the heavy threads first - this is a major factor playing against Loom virtual threads scheduler.

And the result is:

```
loom     21686
platform 313
```
With platform threads our light tasks go executed 70 times faster. This may be a problem. Should fairness be introduced into Loom scheduler? Maybe yes and maybe no. There are other solutions. In modern microservices-oriented world the very first that comes to mind is moving heavy work to separate JVM, container, host or wherever else.
