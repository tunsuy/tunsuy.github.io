---
title: Java：重温数据结构-链表
date: 2016-09-13 09:59:43
tags: [java,数据结构]
categories: java
keywords: [java,数据结构,链表]
description:
---

翻译于[https://www.cs.cmu.edu/~adamchik/15-121/lectures/Linked%20Lists/linked%20lists.html](https://www.cs.cmu.edu/~adamchik/15-121/lectures/Linked%20Lists/linked%20lists.html)

注：不是原文翻译，有些自己的理解改动

# 一、 引言
一个链表就是一个线性的数据结构，如下图所示：

{% asset_img linkedlist.bmp  %}

<!-- more -->

一个链表中的每个元素（我们叫它节点）包含两个部分-数据和指向下一个节点的引用，最后一个节点的引用部分则指向null。指向链表的一个对象（实体）叫做这个链表的head（头部），这个head不是链表中一个独立的节点，但它是指向链表的第一个节点。如果链表为空，head则表示一个空的引用。

链表是一个动态的数据结构，链表中节点的数量不是固定的，能够根据需要动态的增减。如果一个应用需要处理的对象（实体）数是不确定的，则需要使用链表来表示。

链表相对于数组来说的一个缺点就是不允许直接访问其中的元素。如果你想要访问一个指定的元素，你必须从链表的head开始，依次找到它的下一个引用，直到你得到这个想要的元素。

链表的另外一个缺点就是比数组占用更多的内存空间。因为需要使用额外的空间保存下一个节点引用（next）。

# 二、 链表的类型
1、单链表，就是上面描述的  
2、双向链表，就是指一个链表有两个引用，一个指向下一个节点，另一个指向前一个节点，如图

{% asset_img doubly.bmp  %}

链表的另一个重要的类型叫循环链表，就是指链表的最后一个节点指向链表的第一个节点（或者head）。

# 三、 节点类
java语言中，你可以定义一个类（B）被包含于另一个类（A）。类A叫做外部类，类B叫做内部类。内部类的作用仅仅就是用于内部帮助类。下面就是链表内部节点类的定义
```java
private static class Node<AnyType>
{
   private AnyType data;
   private Node<AnyType> next;

   public Node(AnyType data, Node<AnyType> next)
   {
      this.data = data;
      this.next = next;
   }
}
```
一个内部类就是外部类的一个成员，能够访问外部类的其他成员（包括私有成员）。反之亦然，外部类也能够访问内部类的所有成员。一个内部类能够被private、public、protected或者package访问权限修饰。有两种内部类：静态和非静态的。

这里我们使用两个内部类来实现链表：静态 `Node` 类和非静态 `LinkedListIterator` 。完整的实现请看 [LinkedList.java](https://www.cs.cmu.edu/~adamchik/15-121/lectures/Linked%20Lists/code/LinkedList.java)

# 四、 例子
让我们分段的来跟踪每一步的影响，在每一步执行之前，链表存储它的初始状态。

```java
head = head.next;
```

{% asset_img linkedlist2.bmp  %}

```java
head.next = head.next.next;
```

{% asset_img linkedlist3.bmp  %}

```java
head.next.next.next.next = head;
```

{% asset_img linkedlist4.bmp  %}

# 五、 链表操作
## 1、 addFirst
这个方法创建一个节点，并放在链表的开始位置，如图：

{% asset_img prepend.bmp  %}

```java
public void addFirst(AnyType item)
{
   head = new Node<AnyType>(item, head);
}
```

## 2、 Traversing
从head开始，访问每一个节点，直到null，不改变head的引用。如图：

{% asset_img traverse.bmp  %}

```java
Node tmp = head;

while(tmp != null) tmp = tmp.next;
```

## 3、 addLast
这个方法增加一个节点到链表的最后，这需要 `Traversing` ，但确保在最后一个节点的时候停止。如图：

{% asset_img append.bmp  %}

```java
public void addLast(AnyType item)
{
   if(head == null) addFirst(item);
   else
   {
      Node<AnyType> tmp = head;
      while(tmp.next != null) tmp = tmp.next;

      tmp.next = new Node<AnyType>(item, null);
   }
}
```

## 4、 Inserting "after"
找到指定key的节点，然后在它之后插入一个节点。如下图所示，我们在"E"之后插入一个新的节点。

{% asset_img after.bmp  %}

```java
public void insertAfter(AnyType key, AnyType toInsert)
{
   Node<AnyType> tmp = head;
   while(tmp != null && !tmp.data.equals(key)) tmp = tmp.next;

   if(tmp != null)
      tmp.next = new Node<AnyType>(toInsert, tmp.next);
}
```

## 5、 Inserting "before"
找到指定key的节点，然后在它之前插入一个节点。如下图所示，我们在"A"之前插入一个新的节点。

{% asset_img before.bmp  %}

为了方便起见，我们增加两个引用 `pre` 和 `cur`，保证 `pre` 在 `cur` 之前。同时平移这两个引用，直到 `cur` 到达我们想要插入的节点之前。如果 `cur` 到达null，则不插入，否则我们插入一个新的节点在 `pre` 和 `cur` 之间。
```java
public void insertBefore(AnyType key, AnyType toInsert)
{
   if(head == null) return null;
   if(head.data.equals(key))
   {
      addFirst(toInsert);
      return;
   }

   Node<AnyType> prev = null;
   Node<AnyType> cur = head;

   while(cur != null && !cur.data.equals(key))
   {
      prev = cur;
      cur = cur.next;
   }
   //insert between cur and prev
   if(cur != null) prev.next = new Node<AnyType>(toInsert, cur);
}
```

## 6、 Deletion
找到指定key的节点并删除它。如下图所示删除包含"A"的节点。

{% asset_img delete.bmp  %}

这个方法的算法和前一个很相似。为了方便起见，我们增加两个引用 `pre` 和 `cur`，保证 `pre` 在 `cur` 之前。同时平移这两个引用，直到 `cur` 到达我们想要删除的节点。这里有三种情况需要注意：  
1、链表为空  
2、删除的是head节点  
3、需要删除的节点不再链表中
```java
public void remove(AnyType key)
{
   if(head == null) throw new RuntimeException("cannot delete");

   if( head.data.equals(key) )
   {
      head = head.next;
      return;
   }

   Node<AnyType> cur  = head;
   Node<AnyType> prev = null;

   while(cur != null && !cur.data.equals(key) )
   {
      prev = cur;
      cur = cur.next;
   }

   if(cur == null) throw new RuntimeException("cannot delete");

   //delete cur node
   prev.next = cur.next;
}
```

## 7、 Iterator
迭代器提供了对集合数据的访问方式，隐藏集合数据的内部表示形式。在java中迭代器是一个对象，因此需要创建一个类来实现，并且这个类需要实现 `Iterator` 接口。通常这样的类是作为一个内部类来实现的。`iterator` 接口包含下面三个方法：  
1、AnyType next() - 返回容器中的下一个元素  
2、boolean hasNext() - 检查是否还有下一个元素  
3、void remove() - (可选的操作).移除通过 `next()` 方法返回的元素

下面讲讲 `LinkedList` 类中的 `iterator` 实现。首先我在 `LinkedList` 中增加一个方法：
```java
public Iterator<AnyType> iterator()
{
   return new LinkedListIterator();
}
```
下面 `LinkedListIterator ` 是 `LinkedList` 的一个私有内部类
```java
private class LinkedListIterator implements Iterator<AnyType>
{
   private Node<AnyType> nextNode;

   public LinkedListIterator()
   {
      nextNode = head;
   }
   ...
}
```
`LinkedListIterator` 类 必须实现 `next()` 和  `hasNext()` 方法.
```java
public AnyType next()
{
   if(!hasNext()) throw new NoSuchElementException();
   AnyType res = nextNode.data;
   nextNode = nextNode.next;
   return res;
}
```

# 六、 Cloning克隆
就像任何其他一个对象一样，我们需要学习怎样clone一个链表。如果我们简单的使用 `Object` 类的 `clone()` 方法，那么就是"浅复制"，如下图所示：

{% asset_img shallowclone.bmp  %}

这个Object的 `clone()` 方法只会对第一个节点进行一份内容复制，其他节点都是引用复制，内容共享。这显然不是我们想要的效果，那么看看下面的"深复制"

{% asset_img deepclone.bmp  %}

有几种方法可以"深复制"链表，一种简单的方法就是遍历每一个节点，然后使用 `addFirst()` 方法。最后你将得到一个倒序的新的链表，然后我们不得不反转这个链表。
```java
public Object copy()
{
   LinkedList<AnyType> twin = new LinkedList<AnyType>();
   Node<AnyType> tmp = head;
   while(tmp != null)
   {
      twin.addFirst( tmp.data );
      tmp = tmp.next;
   }

   return twin.reverse();
}
```
一个更好的方法就是对这个新链表使用尾引用，在最后的节点增加每一个新节点。
```java
public LinkedList<AnyType> copy3()
{
   if(head==null) return null;
   LinkedList<AnyType> twin = new LinkedList<AnyType>();
   Node tmp = head;
   twin.head = new Node<AnyType>(head.data, null);
   Node tmpTwin = twin.head;

   while(tmp.next != null)
   {
      tmp = tmp.next;
      tmpTwin.next = new Node<AnyType>(tmp.data, null);
      tmpTwin = tmpTwin.next;
   }

   return twin;
}
```
