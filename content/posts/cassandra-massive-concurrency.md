+++
draft = true
date = 2023-11-05T21:30:49+03:00
title = "Cassandra queried concurrently from a single client"
tags = ["Cassandra", "Java", "Golang"]
+++ 

It is a common pattern working with cassandra to query it concurrently with plenty of identifiers for every request to 
hit only one partition.  Recently I have rewritten a plenty of heavy queries to this pattern and wondered how many 
concurrent queries cassandra (or the driver) can handle. So I've written the following test in Java with official 
cassandra driver and in Go with the dominantly used driver. 

First, I run local cassandra in docker with `docker run --rm --name=cassandra -p 9042:9042 cassandra:5.0`

For java I used most recent Java 21 with virtual threads. Running this sample with `-Djdk.tracePinnedThreads` java 
system property reveals that at least some support for virtual threads is present in the official java driver, since it 
does not pin platform threads in my program.

```java

public class Main {

    public static final int WORKERS = 50_000;
    public static final int QUERIES = 200;

    public static void main(String[] args) {
        DriverConfigLoader loader =
                DriverConfigLoader.programmaticBuilder()
                        .withClass(DefaultDriverOption.REQUEST_THROTTLER_CLASS, ConcurrencyLimitingRequestThrottler.class)
                        .withInt(DefaultDriverOption.REQUEST_THROTTLER_MAX_CONCURRENT_REQUESTS, 600)
                        .withInt(DefaultDriverOption.REQUEST_THROTTLER_MAX_QUEUE_SIZE, 100_000)
                        .build();
        var counter = new AtomicInteger();
        try (var session = CqlSession.builder()
                .addContactPoint(new InetSocketAddress(9042))
                .withConfigLoader(loader)
                .withLocalDatacenter("datacenter1")
                .build()) {

            session.execute("drop keyspace if exists pargettest");
            session.execute("create keyspace pargettest with replication = {'class' : 'SimpleStrategy', 'replication_factor' : 1}");
            session.execute("use pargettest");
            session.execute("drop table if exists test");
            session.execute("create table test (a text, b int, primary key(a))");
            session.execute("insert into test (a, b) values ( 'a', 1)");

            var start = System.nanoTime();
            try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
                for (int i = 0; i < WORKERS; i++) {
                    executor.submit(() -> {
                        try {
                            for (int j = 0; j < QUERIES; j++) {
                                var res = session.execute("select * from test where a=?", "a");
                                if (res.all().size() != 1) {
                                    System.out.println("WAT!");
                                }
                                int totalQueries = counter.incrementAndGet();
                                if (totalQueries % 100_000 == 0) {
                                    System.out.println("total queries " + totalQueries);
                                }
                            }
                        } catch (Exception e) {
                            e.printStackTrace();
                            throw e;
                        }
                    });
                }
            }
            var duration = Duration.ofNanos(System.nanoTime() - start);

            System.out.println("All is done in " +
                    String.format("%sm%ss%sms",
                            duration.toMinutesPart(),
                            duration.toSecondsPart(),
                            duration.toMillisPart()
                    )
            );
        }
    }
}
```
The main idea is to have 50 thousand concurrent workers to query local cassandra 200 times. On client side it is a 
beautiful opportunity for virtual threads. However, doing it in straightforward manner fails with errors since the level
of concurrency is too big. Experimentally I came to configuring out of the box throttler to limit concurrent requests 
to 600.
- 100 gave me errors with breaking default 2 seconds timeout for a query
- 200 gave me `4m0s400ms` 
- 400 gave me `2m14s162ms`
- 600 gave me in 3 runs `1m46s425ms`, `1m42s384ms` and `1m36s647ms`
- 700 gave me errors similar to absence of throttler

However, also I wrote very similar test with go 
```go
package main

import (
	"fmt"
	"strconv"
	"sync"
	"sync/atomic"
	"time"

	"github.com/gocql/gocql"
)

const workers = 50_000
const queries = 200

func main() {
	cluster := gocql.NewCluster("localhost:9042")
	session, err := cluster.CreateSession()
	if err != nil {
		panic(err)
	}

	execRelease(session.Query("drop keyspace if exists pargettest"))
	execRelease(session.Query("create keyspace pargettest with replication = {'class' : 'SimpleStrategy', 'replication_factor' : 1}"))
	execRelease(session.Query("drop table if exists pargettest.test"))
	execRelease(session.Query("create table pargettest.test (a text, b int, primary key(a))"))
	execRelease(session.Query("insert into pargettest.test (a, b) values ( 'a', 1)"))

	var wg sync.WaitGroup
	var counter atomic.Int64
	start := time.Now()

	for i := 1; i <= workers; i++ {
		wg.Add(1)

		go func() {
			defer wg.Done()
			for j := 0; j < queries; j++ {
				iterRelease(session.Query("select * from pargettest.test where a='a'"))
				count := counter.Add(1)
				if count%100_000 == 0 {
					println("total queries " + strconv.FormatInt(count, 10))
				}

			}
		}()
	}

	wg.Wait()
	fmt.Printf("All is done in %v\n", time.Now().Sub(start))
}

func iterRelease(query *gocql.Query) {
	_, err := query.Iter().SliceMap()
	if err != nil {
		println(err.Error())
		panic(err)
	}
	query.Release()
}

func execRelease(query *gocql.Query) {
	if err := query.Exec(); err != nil {
		println(err.Error())
		panic(err)
	}
	query.Release()
}
```

Running it 5 times I recieved `1m29.209706666s`, `55.384113667s`, `56.5917855s`, `55.201668208s` and `1m11.538293375s`. 
Variance is unpleasant but numbers a better than with java version. The downside is that there is no builtin throttler, 
so giving this program 100_000 workers results in errors. While with 
