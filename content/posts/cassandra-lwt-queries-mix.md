+++ 
date = 2023-10-05T15:10:49+03:00
title = "Cassandra lightweight transactions mixed with ordinary queries"
tags = ["Cassandra", "Java"]
+++ 

Recently I've discovered that adding a simple "if exists" clause to query in Cassandra turns it into something completely different. Lightweight transaction is started that is not really compatible with running some data modification queries without such clause. I debugged this problem to the simple independent reproducer, so I'd like to share it. 

```java
@Testcontainers
class UpdateTest {

    @Container
    private static final CassandraContainer cassandra
            = (CassandraContainer) new CassandraContainer("cassandra:4.1.3").withExposedPorts(9042);

    private static CqlSession session;

    @BeforeAll
    static void setup() {
        session = CqlSession.builder()
                .addContactPoint(cassandra.getContactPoint())
                .withLocalDatacenter("datacenter1")
                .build();

        session.execute("drop keyspace if exists updatetests");
        session.execute("create keyspace updatetests with replication = {'class' : 'SimpleStrategy', 'replication_factor' : 1}");
        session.execute("use updatetests");
        session.execute("drop table if exists test");
        session.execute("create table test (a text, b int, primary key(a))");
    }

    @AfterAll
    static void tearDown() {
        session.close();
    }

    @Test
    void testLwtInsertAndLwtUpdate() {
        session.execute("insert into test(a,b) values(?,?) if not exists", "x", 1);
        session.execute("update test set b=? where a=? if exists", 2, "x");
        assertUpdate("x");
    }

    @Test
    void testInsertAndLwtUpdate() {
        session.execute("insert into test(a,b) values(?,?)", "y", 1);
        session.execute("update test set b=? where a=? if exists", 2, "y");
        assertUpdate("y");
    }

    @Test
    void testLwtInsertAndUpdate() {
        session.execute("insert into test(a,b) values(?,?) if not exists", "z", 1);
        session.execute("update test set b=? where a=?", 2, "z");
        assertUpdate("z");
    }
    private static void assertUpdate(String id) {
        var res = session.execute("select * from test where a=?", id);
        res.forEach(row -> assertEquals(2, row.getInt("b")));
    }
}
```

Running this sample only 2 out of 3 tests pass. `testInsertAndLwtUpdate` fails. What is even more interesting is that in my tests it reports lightweight transaction applied.

You may find full self-contained example at [my github](https://github.com/isopov/cassandra-lwt-test).