---
title: Java：ArrayList和LinkedList的对比
date: 2016-05-13 10:00:53
tags: [java]
categories: java
keywords: [java,ArrayList,LinkedList]
description:
---

`ArrayList` 和 `LinkedList` 都实现了 `list` 接口，它们的方法和返回结果都几乎是一样的。然而它们还是有一些差异，不同的场合它们各自有不同的优势。

# 一、 不同点
## 1、 Search搜索
`ArrayList` 的搜索操作比 `LinkedList` 快很多。`get(int index)` 在 `ArrayList` 的时间复杂度为 `O(1)`，而在 `LinkedList` 中是 `O(n)`.

原因：`ArrayList` 是基于索引的元素，因为它内部是使用的array数据结构。而 `LinkedList` 是基于双向链表的实现，需要遍历元素来找到需要的元素。

<!-- more -->

## 2、 Deletion删除
`LinkedList` 的删除操作的时间复杂度为 `O(1)`，而 `ArrayList` 在最坏的情况下(删除首元素)为 `O(n)`，最好的情况下(删除最后一个元素)为 `O(1)`。

结论：`LinkedList` 元素删除操作是快于 `ArrayList` 的。

原因：`LinkedList` 每个元素都持有两个引用指向相邻的元素。因此删除元素只需要改变所删除元素相邻节点的引用即可。然而 `ArrayList` 需要移动所有的元素来填充被删除元素的位置。

## 3、 插入性能
`LinkedList` 的插入操作的时间复杂度为 `O(1)`，而 `ArrayList` 的时间复杂度为 `O(n)`，原因跟删除操作一样。

## 4、 内存占用
`ArrayList` 持有索引和元素数据，而 `LinkedList` 持有元素数据和指向两个相邻节点的引用。因此 `LinkedList` 内存占用相对来说更多。

# 二、 相同点
1、两个都实现了 `List` 接口  
2、两个都保持元素插入顺序，意味着元素显示顺序跟插入的顺序是一致的。  
3、两个都是非同步的(线程不安全的)，能够使用 [`Collections.synchronizedList`](http://docs.oracle.com/javase/6/docs/api/java/util/Collections.html#synchronizedList(java.util.List)) 进行同步。  
4、两个返回的 `iterator` 和 `listIterator` 都是 `fail-fast` (如果列表在迭代器创建之后的任何时间被修改，除非通过迭代器自己的remove或add方法，否则迭代器将抛出 [`ConcurrentModificationException`](http://docs.oracle.com/javase/6/docs/api/java/util/ConcurrentModificationException.html))

# 三、 使用场景
1、如上所述，与 `ArrayList(O(n))` 相比，插入和移除操作在  `LinkedList` 中有更好的性能 `(O(1))`。因此，如果需要在应用程序中频繁添加和删除，那么 `LinkedList` 是一个最好的选择。  
2、搜索(get方法)操作在 `Arraylist(O(1))` 中很快，但在 `LinkedList(O(n))` 中更慢，所以如果有较少的添加和删除操作和更多的搜索操作要求，`ArrayList` 将是你最好的选择。
