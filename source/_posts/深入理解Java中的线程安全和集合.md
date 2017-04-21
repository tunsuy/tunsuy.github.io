---
title: 深入理解Java中的线程安全和集合
date: 2016-10-09 19:20:00
tags: [java]
categories: java
keywords: [java,线程安全,集合]
description:
---

翻译于[http://www.codejava.net/java-core/collections/understanding-collections-and-thread-safety-in-java](http://www.codejava.net/java-core/collections/understanding-collections-and-thread-safety-in-java)

注：不是原文翻译，有些自己的理解改动

# 一、 简介
为什么大部分的collection类都是线程不安全的呢？  
正如你知道的，大部分的collection类如：`ArrayList、LinkedList、HashMap、HashSet、TreeMap、TreeSet` 等等都是线程不安全的。事实上，在 `java.util` 包中的所有collection类（除了 `Vector` 和 `HashTable` ）都是线程不安全的。为什么呢？

原因：同步的代价是很大的。

<!-- more -->

`Vector` 和` HashTable` 是java早期就存在的两个集合，他们一开始就被设计为线程安全的（如果你看源码，你会发现他们的所有方法都是 `synchronized` 的）。然而，很快在多线程程序中它们就被暴露出性能是很低下的，因为同步是需要加锁的，这很耗性能。

这也是为什么在后来新的collections( `List、Set、Map` 等)都提供了在单线程应用中非并发性的控制来获得最大的性能。

下面的程序比较了 `Vector` 和 `ArrayList` 的性能（ `Vector` 是线程安全的，而 `ArrayList` 不是）：

```java
import java.util.*;

/**
 * This test program compares performance of Vector versus ArrayList
 * @author www.codejava.net
 *
 */
public class CollectionsThreadSafeTest {

    public void testVector() {
        long startTime = System.currentTimeMillis();

        Vector<Integer> vector = new Vector<>();

        for (int i = 0; i < 10_000_000; i++) {
            vector.addElement(i);
        }

        long endTime = System.currentTimeMillis();

        long totalTime = endTime - startTime;

        System.out.println("Test Vector: " + totalTime + " ms");

    }

    public void testArrayList() {
        long startTime = System.currentTimeMillis();

        List<Integer> list = new ArrayList<>();

        for (int i = 0; i < 10_000_000; i++) {
            list.add(i);
        }

        long endTime = System.currentTimeMillis();

        long totalTime = endTime - startTime;

        System.out.println("Test ArrayList: " + totalTime + " ms");

    }

    public static void main(String[] args) {
        CollectionsThreadSafeTest tester = new CollectionsThreadSafeTest();

        tester.testVector();

        tester.testArrayList();

    }

}
```
该程序测试了分别给这两种类型增加1000000个元素所耗费的时间，结果如下：
```java
Test Vector: 9266 ms
Test ArrayList: 4588 ms
```
说明在大数据量下，ArrayList的性能比Vector的好两倍以上。

# 二、 快速失败
快速失败又叫做"`Fail-Fast`"策略。当使用集合时，你需要理解它们的迭代器的并发策略：`Fail-Fast Iterators`

看下面的代码片段：Strings列表的迭代
```java
List<String> listNames = Arrays.asList("Tom", "Joe", "Bill", "Dave", "John");

Iterator<String> iterator = listNames.iterator();

while (iterator.hasNext()) {
    String nextName = iterator.next();
    System.out.println(nextName);
}
```
这里我们使用了集合的迭代器来遍历集合元素。加入listNames是被两个线程所共享：正在迭代遍历的目前的线程和另外一个线程。当第一个线程正在迭代遍历的时候，第二个线程也在修改这个集合（增加或者减少元素）。那么结果会怎样呢？

第一个线程将抛出这个异常 `ConcurrentModificationException `并立刻失败。所以这就叫做 `fail-fast iterators`。

迭代器为什么会这么快的失败呢？这是因为当遍历一个正在修改的集合时是非常危险的：这个迭代器被生成之后，这个集合可能有更多、更少或者没有元素，以至于导致无法预期或者不一致的结果。这应该要尽可能早的避免，因此这个迭代器必现抛出一个异常来结束目前线程的执行。

下面的例子模拟了抛出 `ConcurrentModificationException` 异常的情况：
```java
import java.util.*;

/**
 * This test program illustrates how a collection's iterator fails fast
 * and throw ConcurrentModificationException
 * @author www.codejava.net
 *
 */
public class IteratorFailFastTest {

    private List<Integer> list = new ArrayList<>();

    public IteratorFailFastTest() {
        for (int i = 0; i < 10_000; i++) {
            list.add(i);
        }
    }

    public void runUpdateThread() {
        Thread thread1 = new Thread(new Runnable() {

            public void run() {
                for (int i = 10_000; i < 20_000; i++) {
                    list.add(i);
                }
            }
        });

        thread1.start();
    }


    public void runIteratorThread() {
        Thread thread2 = new Thread(new Runnable() {

            public void run() {
                ListIterator<Integer> iterator = list.listIterator();
                while (iterator.hasNext()) {
                    Integer number = iterator.next();
                    System.out.println(number);
                }
            }
        });

        thread2.start();
    }

    public static void main(String[] args) {
        IteratorFailFastTest tester = new IteratorFailFastTest();

        tester.runIteratorThread();
        tester.runUpdateThread();
    }
}
```
代码中，这个 `thread1` 正在迭代遍历 `list`，而 `thread2` 也在继续增加元素到这个集合，这就导致了 `ConcurrentModificationException ` 抛出。

注意：迭代器的 `fail-fast` 行为只是用来帮助更容易的找到或者诊断bug，我们不应该在程序中依赖并操作这个 `ConcurrentModificationException` 因为 `fail-fast` 不被保障的。这就是说当这个异常抛出后，我们的程序应该立即停止运行。

# 三、 同步装饰器
## 1、 简介
目前我们已经知道了这些基本的集合类的实现都是线程不安全的，是为了最大的提高在单线程程序中的性能。那么如果我们要用在多线程应用中呢？

当然我们不应该在并发的上下文中使用线程不安全的集合类，因为这将导致无法预期和不一致的结果。我们可以通过同步代码块来手动的控制我们的代码。然而使用线程安全的集合类总是比我们手动写同步代码块是更好的。

可能你已经知道的，Java的 `Collections` 库给我们提供了创建线程安全的集合的工厂方法。这些方法就是来自下面的形式：
```java
Collections.synchronizedXXX(collection)
```
这些工厂方法装饰指定的集合，然后返回线程安全的实现。这里的 `XXX` 可以是 `Collection, List, Map, Set, SortedMap` 和 `SortedSet`。下面就是一个例子：
```java
List<String> safeList = Collections.synchronizedList(new ArrayList<>());
```
如果我们已经有一个存在的线程不安全的集合对象，我们也可以使用这个来装饰它：
```java
Map<Integer, String> unsafeMap = new HashMap<>();
Map<Integer, String> safeMap = Collections.synchronizedMap(unsafeMap);
```

这些工厂方法都是以一个相同的接口实现了对指定的集合的线程安全实现的装饰，所以叫做 `'synchronized wrappers'`。实际上，就是这些线程不安全的集合将所有的工作都委托给了这个装饰器集合来处理。

## 2、 注意
当使用一个同步集合的迭代器的时候，我们需要使用同步块来保证迭代器的安全，因为这个迭代器不是线程安全的。看下面的例子：
```java
List<String> safeList = Collections.synchronizedList(new ArrayList<>());

// adds some elements to the list

Iterator<String> iterator = safeList.iterator();

while (iterator.hasNext()) {
    String next = iterator.next();
    System.out.println(next);
}
```
虽然 `safeList` 是线程安全的，但是他的迭代器不是，因此我们需要手动增加同步块：
```java
synchronized (safeList) {
    while (iterator.hasNext()) {
        String next = iterator.next();
        System.out.println(next);
    }
}
```
同样的需要注意：同步集合的迭代器也是 ` fail-fast` 的。

虽然同步装饰器可以安全的用在多线程应用中，但是有一些缺点需要说明，在下面将介绍

# 四、 并发集合
## 1、 简介
同步集合的一个缺点就是它们的同步机制是使用的它们自己作为锁对象。这就意味着一个线程正在遍历这个集合的时候，在这个集合其他方法块的其他线程就得等待，这就导致程序性能下降。

这就是为什么说Java5及以上介绍说 `concurrent collections`(并发集合)的性能比同步装饰器好。并发集合是在 `java.util.concurrent` 包中。它们根据它们的线程机制被分为了三个组。

## 2、 copy-on-write collections
第一个组叫 `copy-on-write collections` ：这种线程安全的集合是在不可变的数组中存储值的；这个集合的值的任何改变，结果都是返回一个新的数组用来反射到新的值；  这种集合适合于读操作远远多于写操作的情况；有两种实现：`CopyOnWriteArrayList` 和 `CopyOnWriteArraySet`。

注意：这种集合有一个 `iterators` 快照，不会抛出 `ConcurrentModificationException` 异常。因为这种集合是基于不可变数组的，所以一个线程能够读取集合中的数据而不用担心其他线程改变了它们。

## 3、 CAS collections
第二组叫 `Compare-And-Swap or CAS collections`：这组集合是基于 `Compare-And-Swap (CAS)` 算法实现的线程安全。CAS算法是这样的：

为了执行计算或者更新变量的值，它对变量在本地做了一份复制，计算的时候使用的是这个本地的值。当它想要更新这个变量的值时，它会用本地的值跟这个变量进行对比，如果它们是相同的，则用新的值更新这个变量。

如果它们是不相同的，那么说明这个变量被其他线程改变了，这种情况下，`CAS` 线程会尝试使用这个新值进行计算，或者放弃，或者继续。`CAS` 包括 `ConcurrentLinkedQueue` 和  `ConcurrentSkipListMap`。

注意：`CAS` 集合的迭代器是弱一致性的。就是指它可能只会反射这个集合中的一些而不是所有的变化。弱一致性不会抛出 `ConcurrentModificationException`。

## 4、 special lock object
第三组叫 `concurrent collections using a special lock object (java.util.concurrent.lock.Lock)`：  
这个机制是比经典的同步机制更加的灵活。

这个锁跟经典的同步有相同的行为，但是一个线程在以下几种情况也能够获取它：锁目前不被持有，超时或者线程没有被中断。

不同于同步代码，一个锁对象在代码块或者方法被执行的时候就被持有。而这个 `lock` 是在 `unlock()` 方法被调用时才持有。这个机制很有用的一些实现就是将集合分成部分，从而可以分别持有。为了提高并发，比如 `LinkedBlockingQueue` ，这个队列的头和尾被分开 `lock`，因此这个元素能够并行的增减。

一些集合使用包含 `ConcurrentHashMap` 的锁和 `BlockingQueue` 的大部分实现。

这个组中的集合也是弱一致性的迭代器，不会抛出 `ConcurrentModificationException`。
