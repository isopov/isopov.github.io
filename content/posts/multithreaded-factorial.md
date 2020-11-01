+++ 
draft = false
date = 2011-07-03T09:55:51+03:00
title = "Multithreaded factorial"
tags = ["Java"]
+++
**Update**: [the follow up post]({{< ref "/posts/multithreaded-factorial-with-threadpoolexecutor" >}}) gives another version, that lacks the drawback of unequal load on processor cores.

There is a very good blog post about [how to write the factorial](https://chaosinmotion.com/blog/?p=622) method in Java (really it is not about righting factorial). However, even in this master peace;-) there is no multi-threaded factorial. So here it is.
There are some problems with it. The most significant is that multiplying all the BigIntegers from 1 to 50000 is much less heavy job for the computer, than multiplying all BigIntegers from 50001 to 100000. So the the amount of work that is done by each worker-thread is not equal. But investigation on how this may be resolved in a common way is much harder problem, than just writing what I am publishing here. Maybe I'll do it in the other post.

```
package sample;

import java.math.BigDecimal;
import java.util.concurrent.TimeUnit;

import junit.framework.Assert;

import org.junit.Test;

public class FactorialTest {

    private static final int INPUT = 203457;

    @Test
    public void test() throws InterruptedException {
        BigDecimal fact1, fact2;
        {
            long start = System.nanoTime();
            fact1 = fact(INPUT);
            long end = System.nanoTime();
            System.out.println(TimeUnit.SECONDS.convert(end - start,
                    TimeUnit.NANOSECONDS) + " seconds for 1 thread");
        }
        {
            long start = System.nanoTime();
            fact2 = factMt(INPUT, 4);
            long end = System.nanoTime();
            System.out.println(TimeUnit.SECONDS.convert(end - start,
                    TimeUnit.NANOSECONDS) + " seconds for 4 threads");
        }
        Assert.assertEquals(fact1, fact2);

    }

    private static BigDecimal fact(int input) {
        BigDecimal result = BigDecimal.valueOf(1);
        for (int i = 1; i <= input; i++) {
            result = result.multiply(BigDecimal.valueOf(i));
        }
        return result;
    }

    private static BigDecimal factMt(int input, int numThreads)
            throws InterruptedException {
        BigDecimal result = BigDecimal.valueOf(1);
        Thread[] threads = new Thread[numThreads];
        FactComputer[] workers = new FactComputer[numThreads];
        for (int i = 1; i <= numThreads; i++) {
            int start = i == 1 ? 1 : (input / numThreads * (i - 1)) + 1;
            int end = i == numThreads ? input : input / numThreads * i;
            workers[i - 1] = new FactComputer(start, end);
            threads[i - 1] = new Thread(workers[i - 1]);
        }

        for (int i = 0; i < numThreads; i++) {
            threads[i].start();
        }

        for (int i = 0; i < numThreads; i++) {
            threads[i].join();
        }
        for (int i = 0; i < numThreads; i++) {
            result = result.multiply(workers[i].result);
        }
        return result;
    }

    private static class FactComputer implements Runnable {
        BigDecimal result;
        private final int from;
        private final int to;

        public FactComputer(int from, int to) {
            this.from = from;
            this.to = to;
        }

        public void run() {
            result = BigDecimal.valueOf(from);
            for (int i = from + 1; i <= to; i++) {
                result = result.multiply(BigDecimal.valueOf(i));
            }

        }
    }
}
```
Here is the result in `System.out`:
```
56 seconds for 1 thread
26 seconds for 4 threads
```
I have written it as the unit test to be sure that at least this simplistic distribution of work between threads has no bugs (I tested it with different values of INPUT). I think it is obvious from this code that I run it on the 4-core processor. Actually on the 2-core but with hyper-threading enabled.
