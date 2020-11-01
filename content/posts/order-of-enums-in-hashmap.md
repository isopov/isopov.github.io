+++
title = "Order of Enums in Hashmap"
date = 2012-02-10T09:50:33+03:00
draft = false
tags = ["Java"]
+++
Recently I have found myself writing a complex sql-builder for two weeks already. Of cause I used [TDD](http://en.wikipedia.org/wiki/Test-driven_development) technique and this was much less like a pain comparing to the times when I didn't wrote any tests. In some moment I decided that just operating with strings in some thousand or more lines of code is not very cool and started to write some [OOD](http://en.wikipedia.org/wiki/Test-driven_development) for the problem. In the resulting type hierarchy there were some enums. And I used these enums as keys in HashMap. Maybe some curious reader have already guessed what this post is about ;-) If not or if you just want some detailed illustration, read the rest:

What is the order of elements in the HashSet, or the order of entries in the HashMap? It is actually undefined. But should it be consistent across launches of the one and the same JVM on one and the same machine with no changes to the code? I think it is rather safe to assume this for testing purposes. So, OK, what impacts the order of elements in the HashSet? It is the hash code of these elements.

And back to enums. What it the hash code of the enum instance? It has the default hashcode() implementation inherited from Object, so it is different every time the JVM is restarted. So this code:

```
package sample;

import java.util.Arrays;
import java.util.HashSet;

public class EnumHashTest {
    
    public static void main(String... args){
        System.out.println(new HashSet<FooBar>(Arrays.asList(FooBar.values())));
    }
    
    private enum FooBar {
        FIRST, SECOND, THIRD;
    }
}
```

will produce results similar to these on the subsequent invocations (with JVM restarting):
```
[FIRST, THIRD, SECOND]
[SECOND, THIRD, FIRST]
```

Interestingly enough, while I'm currently trying - the order is very consistent, but different across usual running and debugging. This may, or may not, be affected by the fact that I'm using JRockit and Idea for this.
My first attempt to fix this problem was not successful. I used LinkedHashMap. The order of elements in this map is persistent and consistent across JVM. So, why not?
I said that I was writing not just some sql-builder, but "complex" sql-builder. And there were many methods in this code. And several methods to produce the resulting map with enums as keys. And some of this methods return map themselves with their content being added to the result map with the putall(). Here were come. In the intermediate map the order elements is not persistent across JVM launches and thus it is not persistent in the result map that is dependent on the order of insert of the elements in the map.
The following code:

```
package sample;

import java.util.*;

public class EnumHashTest {
    
    public static void main(String... args){
        Set<FooBar> orderedSet = new LinkedHashSet<FooBar>();
        orderedSet.addAll(new HashSet<FooBar>(Arrays.asList(FooBar.values())));
        System.out.println(orderedSet);
    }
    
    private enum FooBar {
        FIRST, SECOND, THIRD;
    }
}
```

produces the following output being run several times:
```
[FIRST, SECOND, THIRD]
[THIRD, SECOND, FIRST]
```

That's it.
So what is the final solution? Enum's values are always limited in size. And enums always implement Comparable interface consistenly. So I just used TreeMap in the very end of this computation and got consistent ordering of elements across JVM launches.
