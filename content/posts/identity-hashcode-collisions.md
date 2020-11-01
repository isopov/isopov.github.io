+++ 
draft = false
date = 2011-03-26T10:19:24+03:00
title = "Collisions of hashcodes - much more real than an md5 collision"
tags = ["Java"]
+++
So okay, hashcode is int and so there are only 4294967296 distinct hashcodes. It is not a little number, but in modern systems there could be definitely much more objects.

In a [very good book](http://www.javapuzzlers.com/) by Joshua Bloch I have read that there is a probability that two consequently created objects will have an equal system identity hashcode. Actually from the time of publishing that book major version of Java was released and the situation has improved dramatically. But still 232 is a very little number for objects. And still two consequently created objects may have an equal identity hashcode:
```
  private static final int SIZE = 500000000;

  public static void main(String[] args) {
    int count = 0;
    for (int i = 0; i < SIZE; i++) {
      if (new Object().hashCode() == new Object().hashCode()) {
        count++;
      }
    }
    System.out.println(count + " of " + SIZE + " consequently created pairs of objects had equal hashcodes");
  }
```

On my machine that produced the following output:
```
16 of 500000000 consequently created objects had equal hashcodes
```

But creating objects and leaving them for garbage collection is not the usual way to deal with. At least for those objects for that we are worried about hashcodes and their quality. So here is the small test on how often the hashcodes will be collisional in a reasonably large pool of objects (The pool of 5000000 millions was chosen as it near to the size that can be checked on 32 bit JVM - actually running this program involves setting not standard size for the Java heap).
```
  private static final int SIZE = 50000000;

  public static void main(String[] args) {
    //number of objects with the same hashcode for each collisionall hashcode
    Map<Integer, Integer> hashcodesMap = new HashMap<Integer, Integer>(SIZE);

    for (int i = 0; i < SIZE; i++) {
      int hashCode = (new Object()).hashCode();
      Integer integer = hashcodesMap.get(hashCode);
      if (integer == null) {
        hashcodesMap.put(hashCode, 1);
      } else {
        hashcodesMap.put(hashCode, integer + 1);
      }
    }
    Map<Integer, Integer> collisionsFrequency = new HashMap<Integer, Integer>();

    for (Integer value : hashcodesMap.values()) {
      if (value > 1) {
        Integer integer = collisionsFrequency.get(value);
        if (integer != null) {
          collisionsFrequency.put(value, integer + 1);
        } else {
          collisionsFrequency.put(value, 1);
        }
      }
    }
    int totalCollisionObjects = 0;
    for (Entry<Integer, Integer> entry : collisionsFrequency.entrySet()) {
      totalCollisionObjects += entry.getValue() * entry.getKey();
      System.out.println(entry.getValue() + " hashcodes with " + entry.getKey() + " objects per each");
    }
    System.out.println("");
    System.out.println("Only " + (SIZE - totalCollisionObjects) + " among " + SIZE + " have unique hashcode");
  }
```

This code produced the following output on my machine:
```
8509145 hashcodes with 2 objects per each
4195283 hashcodes with 3 objects per each
1523953 hashcodes with 4 objects per each
435035 hashcodes with 5 objects per each
102849 hashcodes with 6 objects per each
20149 hashcodes with 7 objects per each
3455 hashcodes with 8 objects per each
506 hashcodes with 9 objects per each
72 hashcodes with 10 objects per each
9 hashcodes with 11 objects per each
1 hashcodes with 12 objects per each

Only 11333712 among 50000000 have unique hashcode
```

So the major portion of used identity hashcodes are collisional. And with that less than a percent of possible unique values of int was used. With less objects in out pool the result is better:
```
317898 hashcodes with 2 objects per each
15295 hashcodes with 3 objects per each
524 hashcodes with 4 objects per each
13 hashcodes with 5 objects per each

Only 4316158 among 5000000 have unique hashcode
```

Actually all this is not very impressive and leads to the one major consequence:
Writing:
```
  @Override
  public boolean equals(Object obj) {
    return System.identityHashCode(this) == System.identityHashCode(obj);
  }
```
is a really bad idea. And yes, I saw this (not exactly this, there were at least null-check in that method...).

