+++ 
draft = false
date = 2011-07-22T10:00:17+03:00
title = "Multithreaded factorial with ThreadPoolExecutor"
tags = ["Java"]
+++
**Update**: [the follow up post]({{< ref "/posts/forkjoin-factorial" >}}) gives another version using newly introduced API.

In the [previous post]({{< ref "/posts/multithreaded-factorial" >}}) I have written a multithreaded factorial. It was not really good, because last one of the worker threads worked significantly longer than the first one. Fortunately we are living in 2011 and Java 1.5 was released so long time ago. The solution is simple - do not create 4 threads of the 4 virtual cores processors. Create really many threads and let the standard concurrent library execute them all. So ok, here is the code for this:

```
    private static BigInteger factMtExecutor(int input, int numThreads)
            throws InterruptedException, ExecutionException {
        FactCallable[] workers = new FactCallable[100];
        for (int i = 1; i <= 100; i++) {
            int start = i == 1 ? 1 : (input / 100 * (i - 1)) + 1;
            int end = i == 100 ? input : input / 100 * i;
            workers[i - 1] = new FactCallable(start, end);
        }

        ThreadPoolExecutor executor = new ThreadPoolExecutor(numThreads,
                numThreads, 0, TimeUnit.SECONDS,
                new LinkedBlockingQueue<Runnable>());

        List<Future<BigInteger>> futures = executor.invokeAll(Arrays
                .asList(workers));
        BigInteger result = BigInteger.valueOf(1L);
        for (Future<BigInteger> future : futures) {
            result = result.multiply(future.get());
        }

        return result;
    }

    private static class FactCallable implements Callable<BigInteger> {
        private final int from;
        private final int to;

        public FactCallable(int from, int to) {
            this.from = from;
            this.to = to;
        }

        @Override
        public BigInteger call() throws Exception {
            BigInteger result = BigInteger.valueOf(from);
            for (int i = from + 1; i <= to; i++) {
                result = result.multiply(BigInteger.valueOf(i));
            }
            return result;
        }
    }
```

And the result comparing to the previous version:
```
35 seconds for simple start/join
25 seconds for ThreadPoolExecutor
``

And the second time, swapping the order of execution:
```
29 seconds for ThreadPoolExecutor
38 seconds for simple start/join
```
Full code for comparison may be found on [github](https://gist.github.com/1098118).
