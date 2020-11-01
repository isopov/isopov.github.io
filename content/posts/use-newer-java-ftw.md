+++ 
draft = false
date = 2013-04-05T09:44:46+03:00
title = "Use newer Java FTW!"
tags = ["Java"]
+++

One of my recent tasks was trying to convince client, that we should target the 7th version of the JVM, and not the 5th. I failed. Actually I don't think that such task is possible unless there are no any real reasons in using older Java. Nevertheless here is the text, that I wrote.

What Java implementation are we going to use? The most used one is Oracle.
Oracle Java5 reached its official end of life in Oct 2009
Oracle Java6 reached its official end of life in Feb 2013 (there will be no public updates anymore, nor any security updates)

Reasons to use the newest java possible:
- This year Spring 4 will be released with minimum Java version 6. Most libraries and frameworks are in process of dropping the Java5 support. If Application is going to be supported more than for one year it is reasonable to ease that future support as much as possible.
- Recent Eclipse versions are targeted for Java6 and Java7 and are tested only on these versions. Developing new project with the old versions of tools and libraries should be really justified and reasoning is well-understood by everyone in the team.
- ~~Most recent Eclipse WTP (tooling for developing web-applications - we are going to build the web-application, right) supports only Java6+ out-of-the-box.~~ **UPDATE:** to get rid of warning that Java5 is not supported by WTP you just need to change version="3.0" in your web.xml to version="2.5" or below since servlet 3.0 requires Java6 to run. Not very intuitive, but after that support is good.
- Many small aditions to the platform are made with every release and while it is definitely possible to live without them, but performance, readability and maintainability of the source will suffer. To name just a few that I really needed, but the target Java version of the project was too low are:
  - (Added in Java 6) @Override annotation on interface method implementations. Hard minutes and sometimes even hours of debugging...
  - (Added in Java 6) NavigableSet and NavigableMap that are so easy to use for any non-trivial logic on ordered data.
  - (Added in Java 7) Try-with-resource. In any reasonably large application that I've seen there are hundreds of places without proper closing of resources - some of them are yet to be discovered bugs and some of them are just a junk that hinder the real problems. Any tooling that helps avoiding these problems is greatly appreciated and especially such a good tooling built right into the language.
  - (Added in Java 7) ForkJoin framework - hardly ever used in daily development, but if there is a proper place it is hardly possible to think of anything better. It could be used on Java 6 with a lot of trouble involved, but its benefits are so high that in some cases it is reasonable. Of course if there is no option of moving to Java 7.
  - (Added in Java 7) Diamonds instead of generics. Makes the code so much more readable and thus maintainable.

This is absolutely not a complete list of benefits, but only the list of those that I have felt myself.
