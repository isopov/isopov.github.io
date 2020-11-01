+++ 
draft = false
date = 2011-04-30T10:23:54+03:00
title = "Not symmetric equals after override"
tags = ["Java"]
+++
There is really cool tool [FindBugs](http://findbugs.sourceforge.net/) and recently I found that it can discover (and discovered such a situation in my experience) [broken symmetry for equals](http://findbugs.sourceforge.net/bugDescriptions.html#EQ_OVERRIDING_EQUALS_NOT_SYMMETRIC) method when overriding it. So this is just a little example of this bug. (HashCode method is generated and I assume it to be all right).

```
public class BrokenEqualsTest {

  public static void main(String[] args) {
    Test1 test1 = new Test1();
    test1.field = "test";

    Test2 test2 = new Test2();
    ((Test1) test2).field = "test";
    test2.field2 = "test";

    System.out.println("test1 equals test2 - " + test1.equals(test2));
    System.out.println("test2 equals test1 - " + test2.equals(test1));
  }

  private static class Test1 {
    private String field;

    @Override
    public int hashCode() {
      final int prime = 31;
      int result = 1;
      result = prime * result + ((field == null) ? 0 : field.hashCode());
      return result;
    }

    @Override
    public boolean equals(Object obj) {
      if (obj instanceof Test1) {
        return field.equals(((Test1) obj).field);
      }
      return false;
    }
  }

  private static class Test2 extends Test1 {
    private String field2;

    @Override
    public int hashCode() {
      final int prime = 31;
      int result = super.hashCode();
      result = prime * result + ((field2 == null) ? 0 : field2.hashCode());
      return result;
    }

    @Override
    public boolean equals(Object obj) {
      if (obj instanceof Test2) { //here is the error reported by FindBugs
        return field2.equals(((Test2) obj).field2);
      }
      return false;
    }
  }
}
```
The output of running main of this class is:
```
test1 equals test2 - true
test2 equals test1 - false
```
