---
title: 深入详解Java迭代器
date: 2016-07-05 18:16:54
tags: java
categories: java
keywords: [java,迭代器,设计模式]
description:
---

java中的迭代器有三种

# 一、 Enumeration

它是用于获取早期遗留集合（Vector，Hashtable）元素的接口。`Enumeration` 是JDK 1.0中的第一个迭代器，其余的包含在JDK 1.2中，具有更多的功能。`Enumeration` 也用于指定 `SequenceInputStream` 的输入流。  
我们可以通过在任何 `Vector` 对象上调用 `elements()` 方法来创建 `Enumeration` 对象
```java
Enumeration e = v.elements();
```

<!-- more -->

`Enumeration`有两个方法：
```java
// Tests if this enumeration contains more elements
public boolean hasMoreElements();

// Returns the next element of this enumeration
// It throws NoSuchElementException
// if no more element present
public Object nextElement();
```

下面演示下该迭代器的使用
```java
import java.util.Enumeration;
import java.util.Vector;

public class Test
{
    public static void main(String[] args)
    {
        // Create a vector and print its contents
        Vector v = new Vector();
        for (int i = 0; i < 10; i++)
            v.addElement(i);
        System.out.println(v);

        // At beginning e(cursor) will point to
        // index just before the first element in v
        Enumeration e = v.elements();

        // Checking the next element availability
        while (e.hasMoreElements())
        {
            // moving cursor to next element
            int i = (Integer)e.nextElement();

            System.out.print(i + " ");
        }
    }
}
```
输出：
```java
[0, 1, 2, 3, 4, 5, 6, 7, 8, 9]  
0 1 2 3 4 5 6 7 8 9
```

`Enumeration` 的局限性：
* `Enumeration` 仅用于遗留类（Vector，Hashtable）, 因此，它不是一个通用的迭代器。
* 在遍历的时候不能执行删除操作。
* 只有向前方向迭代。

# 二、 Iterator
`Iterator` 它是一个通用的迭代器，因为我们可以将它应用到任何Collection对象。通过使用 `Iterator`，我们可以执行读取和删除操作。

当我们想遍历所有Collection框架中，实现了接口如Set，List，Queue，Deque以及Map接口的，所有实现类的元素时，都必须使用 `Iterator`。`Iterator` 是唯一可用于整个集合框架的游标。  
可以通过集合类的 `iterator()` 方法来创建：
```java
Iterator itr = c.iterator();
```

`Iterator` 接口定义了三个方法：
```java
// Returns true if the iteration has more elements
public boolean hasNext();

// Returns the next element in the iteration
// It throws NoSuchElementException if no more
// element present
public Object next();

// Remove the next element in the iteration
// This method can be called only once per call
// to next()
public void remove();
```
`remove()` 方法可以抛出两个异常：
* `UnsupportedOperationException` —— 此迭代器不支持删除操作
* `IllegalStateException` —— 如果 `next()` 方法尚未被调用，或者在最后一次调用 `next()` 方法之后，已经调用了 `remove()` 方法

下面演示下这个迭代器的应用：
```java
import java.util.ArrayList;
import java.util.Iterator;

public class Test
{
    public static void main(String[] args)
    {
        ArrayList al = new ArrayList();

        for (int i = 0; i < 10; i++)
            al.add(i);

        System.out.println(al);

        // at beginning itr(cursor) will point to
        // index just before the first element in al
        Iterator itr = al.iterator();

        // checking the next element availabilty
        while (itr.hasNext())
        {
            //  moving cursor to next element
            int i = (Integer)itr.next();

            // getting even elements one by one
            System.out.print(i + " ");

            // Removing odd elements
            if (i % 2 != 0)
               itr.remove();
        }
        System.out.println();
        System.out.println(al);
    }
}
```
输出：
```java
[0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
0 1 2 3 4 5 6 7 8 9
[0, 2, 4, 6, 8]
```

`Iterator` 的局限性：
* 在遍历的时候不支持替换和新增元素。
* 只有向前方向迭代。

# 三、 ListIterator
它仅适用于列表集合实现的类，如 `arraylist，linkedlist` 等。它提供双向迭代。

 当我们要遍历列表元素时，必须使用 `ListIterator`。它比 `Iterator` 有更多的功能（方法）。   
 `ListIterator` 对象可以通过调用列表接口中存在的 `listIterator()` 方法来创建。
```java
ListIterator ltr = l.listIterator();
```
`ListIterator` 集成自 `Iterator`，所有具有 `Iterator` 所有的接口，并额外增加了6个接口
```java
// Forward direction

// Returns true if the iteration has more elements
public boolean hasNext();

// same as next() method of Iterator
public Object next();

// Returns the next element index
// or list size if the list iterator
// is at the end of the list
public int nextIndex();

// Backward direction

// Returns true if the iteration has more elements
// while traversing backward
public boolean hasPrevious();

// Returns the previous element in the iteration
// and can throws NoSuchElementException
// if no more element present
public Object previous();

// Returns the previous element index
//  or -1 if the list iterator is at the
// beginning of the list
public int previousIndex();

// Other Methods

// same as remove() method of Iterator
public void remove();

// Replaces the last element returned by
// next() or previous() with the specified element
public void set(Object obj);

// Inserts the specified element into the list at
// position before the element that would be returned
// by next(),
public void add(Object obj);
```
`ListIterator` 没有当前元素，它的光标位置总是位于通过调用 `previous()` 函数返回的元素和通过调用 `next()`返回的元素之间。

`set()` 可以抛出4个异常：
* `UnsupportedOperationException` —— 不支持该设置操作
* `ClassCastException ` —— 指定元素的类阻止将其添加到此列表中
* `IllegalArgumentException` —— 指定元素的某些方面阻止将其添加到此列表中
* `IllegalStateException` —— 没有调用 `next()` 或 `previous()`，或者在最后一次调用 `next()` 或 `previous()` 之后调用 `remove()` 或 `add()`


`add()` 可以抛出3个异常：
* `UnsupportedOperationException` —— 不支持该设置操作
* `ClassCastException ` —— 指定元素的类阻止将其添加到此列表中
* `IllegalArgumentException` —— 指定元素的某些方面阻止将其添加到此列表中

下面演示下这个迭代器的应用：
```java
import java.util.ArrayList;
import java.util.ListIterator;

public class Test
{
    public static void main(String[] args)
    {
        ArrayList al = new ArrayList();
        for (int i = 0; i < 10; i++)
            al.add(i);

        System.out.println(al);

        // at beginning ltr(cursor) will point to
        // index just before the first element in al
        ListIterator ltr = al.listIterator();

        // checking the next element availabilty
        while (ltr.hasNext())
        {
            //  moving cursor to next element
            int i = (Integer)ltr.next();

            // getting even elements one by one
            System.out.print(i + " ");

            // Changing even numbers to odd and
            // adding modified number again in
            // iterator
            if (i%2==0)
            {
                i++;  // Change to odd
                ltr.set(i);  // set method to change value
                ltr.add(i);  // to add
            }
        }
        System.out.println();
        System.out.println(al);
    }
}
```
输出：
```java
[0, 1, 2, 3, 4, 5, 6, 7, 8, 9]  
0 1 2 3 4 5 6 7 8 9
[1, 1, 1, 3, 3, 3, 5, 5, 5, 7, 7, 7, 9, 9, 9]
```
`ListIterator` 的局限性：  
它是最强大的迭代器，但它仅适用于List实现的类，因此它不是一个通用的迭代器。

# 四、 共同点
1、请注意，最初任何迭代器引用将指向一个集合中第一个元素的索引之前的索引。  
2、我们不能创建 `Enumeration`，`Iterator`，`ListIterator` 的对象，因为它们是接口。我们使用像 `elements()`，`iterator()`，`listIterator()`这样的方法来创建对象。这些方法具有匿名内部类，它们扩展了相应的接口并返回此类对象。如下代码所示：
```java
import java.util.Enumeration;
import java.util.Iterator;
import java.util.ListIterator;
import java.util.Vector;

public class Test
{
    public static void main(String[] args)
    {
        Vector v = new Vector();

        // Create three iterators
        Enumeration e = v.elements();
        Iterator  itr = v.iterator();
        ListIterator ltr = v.listIterator();

        // Print class names of iterators
        System.out.println(e.getClass().getName());
        System.out.println(itr.getClass().getName());
        System.out.println(ltr.getClass().getName());
    }
}
```
输出：
```java
java.util.Vector$1
java.util.Vector$Itr
java.util.Vector$ListItr
```
引用类名称中的$符号是使用内部类的概念并创建这些类对象的证明。

