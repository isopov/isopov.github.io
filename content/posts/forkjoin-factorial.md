+++ 
draft = false
date = 2011-10-09T10:05:54+03:00
title = "ForkJoin factorial"
tags = ["Java"]
+++
Several months ago I wrote a [multithreaded factorial]({{< ref "/posts/multithreaded-factorial" >}}) method.  It was very simple from the point of view of the underlying technology, but not so trivial from the point of view of synchronizing threads. It used simple start() and join() methods that are available since the Java 1.0 And than I thought that with all the power of java I can improve it. So i used the ThreadPoolExecutor - a piece of technology from the Java 5 and really improved - here is my post about [multithreaded factorial using TreadPoolExecutor]({{< ref "/posts/multithreaded-factorial-with-threadpoolexecutor" >}}).

But Java 5 is a bit old now. And this year the Java 7 has been released! So here is the new version - using the new ForkJoin Framework available in it. Actually when I started writing this simple piece of code is didn'nt think that the result can be like that. I thought that all the power of my 4-core processor was already utilized by the variant with the ThreadPoolExecutor and considered this new method only as an exercise on the new API. I previewed that new version may be simpler as this is one of the stated goals of ForkJoin Framework and it is. But the actual performance increase was absolutely unforeseen by me.

So no more jabber, here are the results:
```
21 seconds for ForkJoin
27 seconds for ThreadPoolExecutor
```
Here is the code:

```
    private static BigInteger factFjPool(int input, int numThreads)
            throws InterruptedException, ExecutionException {
        ForkJoinPool forkJoinPool = new ForkJoinPool(numThreads);
        ForkJoinTask<BigInteger> future = forkJoinPool.submit(new FactorialRecursiveTask(1, input + 1));
        return future.get();
    }

    private static class FactorialRecursiveTask extends RecursiveTask<BigInteger> {
        private static final long serialVersionUID = 1L;

        private static final int THRESHOLD = 1000;

        private final int lo, hi;

        public FactorialRecursiveTask(int lo, int hi) {
            this.lo = lo;
            this.hi = hi;
        }

        @Override
        protected BigInteger compute() {
            if (hi - lo < THRESHOLD) {
                BigInteger result = BigInteger.valueOf(lo);
                for (int i = lo + 1; i < hi; i++) {
                    result = result.multiply(BigInteger.valueOf(i));
                }
                return result;
            } else {
                int mid = (lo + hi) >>> 1;

                FactorialRecursiveTask f1 = new FactorialRecursiveTask(lo, mid);
                f1.fork();
                FactorialRecursiveTask f2 = new FactorialRecursiveTask(mid, hi);
                return f2.compute().multiply(f1.join());
            }
        }
    }
```

I've tried to improve the previous version using other amount of threads, since this version obviously uses division into mush smaller sub-tasks, but to no result. Maybe I used wrong `BlockingQueue<Runnable>`, but this may be regarded as the simplicity of using this API. And definitely `ForkJoinFramework` is superior here, since not only decision about which Queue to use is not needed but also the division into sub-tasks is much simpler.

As usual the full code for this example may be found on [github](https://gist.github.com/1272876).
