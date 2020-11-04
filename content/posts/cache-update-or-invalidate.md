+++ 
draft = false
date = 2020-11-04T10:51:48+03:00
title = "Updating value in cache or simply invalidate it instead"
tags = ["Java"]
+++
Recently I worked on a process of updating many entities in DB. And since these entities are read really often, there was a Redis-based cache in front of DB. After updating the entity in DB it was updated in the cache. And to be safe the whole process was under Redsic-based lock. So:
 - Lock for the entity is taken in Redis
 - Entity is updated in DB
 - Entity is updated in Redis
 - Lost is released in Redis

Everything is good and data is consistent everywhere, but this process is too long. Lock acquire and release are the longest steps. But if we simply remove the locks data may become inconsistent between the DB and Redis. To craft such a sample is a too big task for a blog post, but a very similar concept can be recreated with any technology and at any level. In Java, there is [Jcstress](https://github.com/openjdk/jcstress) - an excellent tool to dive into concurrency problems. Let's create a sample with it:
```
import org.openjdk.jcstress.annotations.*;
import org.openjdk.jcstress.infra.results.Z_Result;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

@JCStressTest
@Outcome(id = "true", expect = Expect.ACCEPTABLE, desc = "Cached value is equal to actual")
@Outcome(expect = Expect.FORBIDDEN, desc = "Value in cache differs from actual")
@State
public class CacheUpdateTest {
    private static final Object KEY = new Object();
    private volatile Object value;
    private final Map<Object, Object> cache = new ConcurrentHashMap<>();

    @Actor
    public void set1() {
        value = new Object();
        cache.put(KEY, value);
    }

    @Actor
    public void set2() {
        value = new Object();
        cache.put(KEY, value);
    }

    @Arbiter
    public void arbiter(Z_Result r) {
        r.r1 = value == cache.get(KEY);
    }
}
```
No DB, no Redis, but both have their counterparts in this sample. Simple volatile variable to hold a single value. And ConcurrentHashMap to serve as a cache. Running this sample produces the following:
```
*** FAILED tests
  Strong asserts were violated. Correct implementations should have no assert failures here.

  4 matching test results. 
  [FAILED] com.sopovs.moradanen.jcstress.CacheUpdateTest
    (JVM args: [-XX:-TieredCompilation, -XX:+StressLCM, -XX:+StressGCM])
  Observed state   Occurrences   Expectation  Interpretation                                              
           false       391,352     FORBIDDEN  Value in cache differs from actual                          
            true    23,479,779    ACCEPTABLE  Cached value is equal to actual                             

  [FAILED] com.sopovs.moradanen.jcstress.CacheUpdateTest
    (JVM args: [-XX:-TieredCompilation])
  Observed state   Occurrences   Expectation  Interpretation                                              
           false       404,027     FORBIDDEN  Value in cache differs from actual                          
            true    26,152,354    ACCEPTABLE  Cached value is equal to actual                             

  [FAILED] com.sopovs.moradanen.jcstress.CacheUpdateTest
    (JVM args: [-XX:TieredStopAtLevel=1])
  Observed state   Occurrences   Expectation  Interpretation                                              
           false       379,522     FORBIDDEN  Value in cache differs from actual                          
            true    21,681,059    ACCEPTABLE  Cached value is equal to actual                             

  [FAILED] com.sopovs.moradanen.jcstress.CacheUpdateTest
    (JVM args: [-Xint])
  Observed state   Occurrences   Expectation  Interpretation                                              
           false        53,317     FORBIDDEN  Value in cache differs from actual                          
            true     2,087,904    ACCEPTABLE  Cached value is equal to actual 
```
After updating the value and cache concurrently with two different values we may trap into inconsistency between the cache and final value. Is the lock the only way to solve this problem? Actually, not. Updating the value in cache is not a usual pattern, to begin with. Usually, we request values from the cache and some underlying machinery emits a request to DB if there is no cached value. And we can make sure that the next such query will get the right answer after updating the value in DB - simply remove the value from the cache. It has another benefit - if no one will ever request the value from cache - it will not be put there and will not waste resources.
Let's write this down:
```
import org.openjdk.jcstress.annotations.*;
import org.openjdk.jcstress.infra.results.Z_Result;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

@JCStressTest
@Outcome(id = "true", expect = Expect.ACCEPTABLE, desc = "Cached value is equal to actual")
@Outcome(expect = Expect.FORBIDDEN, desc = "Value in cache differs from actual")
@State
public class CacheInvalidateTest {
    private static final Object KEY = new Object();
    private volatile Object value;
    private final Map<Object, Object> cache = new ConcurrentHashMap<>();

    private Object getValue() {
        return cache.computeIfAbsent(KEY, o -> value);
    }

    @Actor
    public void set1() {
        value = new Object();
        cache.remove(KEY);
    }

    @Actor
    public void set2() {
        value = new Object();
        cache.remove(KEY);
    }

    @Arbiter
    public void arbiter(Z_Result r) {
        r.r1 = value == getValue();
    }
}
```
This code passes all the tests. However, without removals, it will pass too. The only cache retrieval is happening in the check phase. To test a bit more we can add more actors like this:
```
    @Actor
    public void getValue1() {
        getValue();
    }
```
With them, tests pass too.

