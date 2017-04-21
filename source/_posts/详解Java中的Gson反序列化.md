---
title: 深入详解Java中的Gson反序列化
date: 2017-03-25 13:13:12
tags: [java,gson]
categories: java
keywords: [java,gson,反序列化,序列化]
description:
---

译自 [GSON DESERIALISER EXAMPLE](http://www.javacreed.com/gson-deserialiser-example/)

在这篇文章中，我们将看到如何将复杂的JSON实体反序列化为已有的Java对象。我们将看到怎么使用GSON的`deserialiser `，以控制JSON实体映射到Java对象。

# 一、 前言

请注意，在这篇文章中我们将使用术语解析或反序列化。

下面列出的所有代码均放在：[ https://java-creed-examples.googlecode.com/svn/gson/Gson Deserialiser Example]( https://java-creed-examples.googlecode.com/svn/gson/Gson Deserialiser Example)。

<!-- more -->

# 二、 简单例子
比方说，我们有以下的JSON实体
```json
{
  'title':    'Java Puzzlers: Traps, Pitfalls, and Corner Cases',
  'isbn-10':  '032133678X',
  'isbn-13':  '978-0321336781',
  'authors':  ['Joshua Bloch', 'Neal Gafter']
}
```
上述JSON包括四个字段，其中之一是一个数组。这些字段代表我们的书。默认情况下，GSON希望在Java中的变量名跟JSON中是一样的。因此，我们应该创建具有如下字段名的类：title，isbn-10，isbn-13和authors。但在Java中，变量名不能包含减号。

那怎么解决上面的问题呢？  

一种方法就是使用注解：注释提供了较少的控制，但更简单的使用和理解。也就是说，注释其局限性太大，不能满足这里描述的所有问题。

另一种方法就是使用 `JsonDeserializer`，使用它我们可以完全控制Json实体怎么被解析。

看如下的Java类
```java
package com.javacreed.examples.gson.part1;

public class Book {

  private String[] authors;
  private String isbn10;
  private String isbn13;
  private String title;

  // Methods removed for brevity
}
```
此Java对象将被用来保存在前面显示的JSON实体列出的书。这两个对象（Java和JSON）的结构是在本例中相同，但是这不是必需的。

为了能够解析JSON到Java，我们需要创建实现 `JsonDeserializer` 接口的类。下面的例子显示了我们实现的 `JsonDeserializer` 类。
```java
package com.javacreed.examples.gson.part1;

import java.lang.reflect.Type;

import com.google.gson.JsonArray;
import com.google.gson.JsonDeserializationContext;
import com.google.gson.JsonDeserializer;
import com.google.gson.JsonElement;
import com.google.gson.JsonObject;
import com.google.gson.JsonParseException;

public class BookDeserializer implements JsonDeserializer<Book> {

  @Override
  public Book deserialize(final JsonElement json, final Type typeOfT, final JsonDeserializationContext context)
      throws JsonParseException {

    //The deserialisation code is missing

    final Book book = new Book();
    book.setTitle(title);
    book.setIsbn10(isbn10);
    book.setIsbn13(isbn13);
    book.setAuthors(authors);
    return book;
  }
}
```
上面的代码是不完整的，我们还需要增加其他的处理，先让我们来解释下这些代码。

接口 `JsonDeserializer` 是一个泛型，接收一个类型，就是我们的目的解析对象的类型。在这个例子中就是Book。该接口的方法 `deserialize()` 方法也必须返回同样类型的对象。

GSON将JSON实体解析为 `JsonElement` 类型的Java对象。`JsonElement` 实例可以是下列之一：  
* JsonPrimitive - 如字符串或整数
* JsonObject - `JsonElement` 对象的集合，索引就是它们的名称（类型String）。类似于 `Map<String, JsonElement>`
* JsonArray - `JsonElement` 对象的集合。该集合中的元素可以是任意的 `JsonElement` 类型对象。
* JsonNull - 一个null值

{% asset_img Types-of-JsonElement.png %}

上图为 `JsonElement` 的类型图

{% asset_img Json-Object-Hierarchy.png %}

上图为 `JsonObject` 的嵌套结构图

{% asset_img Book-Json-Object-Hierarchy.png %}

上图为 Book例子中的 `JsonObject` 结构图

如果我们要反序列化此JSON实体，我们首先需要将JSON实体转换成 `JsonObject` ，如下所示。
```java
// The variable 'json' is passed as a parameter to the deserialize() method
final JsonObject jsonObject = json.getAsJsonObject();
```
一个JSON实体能够转换为上述中的任何一种类型。

可以通过JSON实体中的字段名称检索出该字段值
```java
// The variable 'json' is passed as a parameter to the deserialize() method
final JsonObject jsonObject = json.getAsJsonObject();
JsonElement titleElement = jsonObject.get("title")
```
该方法返回的是 `JsonElement` 类型对象，可以通过如下方法将其转换为 `String` 对象
```java
// The variable 'json' is passed as a parameter to the deserialize() method
final JsonObject jsonObject = json.getAsJsonObject();
JsonElement titleElement = jsonObject.get("title")
final String title = jsonTitle.getAsString();
```

下面的代码完整的演示了自定义反序列化类 `BookDeserializer`
```java
package com.javacreed.examples.gson.part1;

import java.lang.reflect.Type;

import com.google.gson.JsonArray;
import com.google.gson.JsonDeserializationContext;
import com.google.gson.JsonDeserializer;
import com.google.gson.JsonElement;
import com.google.gson.JsonObject;
import com.google.gson.JsonParseException;

public class BookDeserializer implements JsonDeserializer<Book> {

  @Override
  public Book deserialize(final JsonElement json, final Type typeOfT, final JsonDeserializationContext context)
      throws JsonParseException {
    final JsonObject jsonObject = json.getAsJsonObject();

    final JsonElement jsonTitle = jsonObject.get("title");
    final String title = jsonTitle.getAsString();

    final String isbn10 = jsonObject.get("isbn-10").getAsString();
    final String isbn13 = jsonObject.get("isbn-13").getAsString();

    final JsonArray jsonAuthorsArray = jsonObject.get("authors").getAsJsonArray();
    final String[] authors = new String[jsonAuthorsArray.size()];
    for (int i = 0; i < authors.length; i++) {
      final JsonElement jsonAuthor = jsonAuthorsArray.get(i);
      authors[i] = jsonAuthor.getAsString();
    }

    final Book book = new Book();
    book.setTitle(title);
    book.setIsbn10(isbn10);
    book.setIsbn13(isbn13);
    book.setAuthors(authors);
    return book;
  }
}
```

为了自定义JSON实体的解析，还需要先指定（注册）目的解析对象的自定义反序列化类，如下所示。
```java
package com.javacreed.examples.gson.part1;

import java.io.InputStreamReader;
import com.google.gson.Gson;
import com.google.gson.GsonBuilder;

public class Main {
  public static void main(String[] args) throws Exception {
    // Configure Gson
    GsonBuilder gsonBuilder = new GsonBuilder();
    gsonBuilder.registerTypeAdapter(Book.class, new BookDeserializer());
    Gson gson = gsonBuilder.create();

    // The JSON data
    try(Reader reader = new InputStreamReader(Main.class.getResourceAsStream("/part1/sample.json"), "UTF-8")){

      // Parse JSON to Java
      Book book = gson.fromJson(reader, Book.class);
      System.out.println(book);
    }
  }
}
```

自定义解析JSON实体的内部原理一般为如下：  
1、将输入的JSON实体解析为 `JsonElement` 类型的Java对象  
2、找到目的解析对象的反序列化类，这个例子中就是 `BookDeserializer`  
3、调用 `deserialize()` 方法，并返回目的解析对象  
4、将`deserialize()` 方法的返回结果作为 `fromJson()` 方法的返回结果

上述例子的结果输出为：
```java
Java Puzzlers: Traps, Pitfalls, and Corner Cases
  [ISBN-10: 032133678X] [ISBN-13: 978-0321336781]
Written by:
  >> Joshua Bloch
  >> Neal Gafter
```

# 三、 嵌套对象
现在让我们扩展上面的例子，将JSON实体嵌套如下：
```json
{
  'title': 'Java Puzzlers: Traps, Pitfalls, and Corner Cases',
  'isbn': '032133678X',
  'authors':[
    {
      'id': 1,
      'name': 'Joshua Bloch'
    },
    {
      'id': 2,
      'name': 'Neal Gafter'
    }
  ]
}
```
该JSON实体对应的 `JsonObject` 结构图如下：

{% asset_img Book-and-Authors-Json-Objects-Hierarchy.png %}

因为Book嵌套author这样一个JSON实体，所以我们需要新增一个author类来表示该author实体，并且Book类需要关联该author类。  

问题就来了：我们应该怎么反序列化出author类。  
有如下几个解决方案：  
1、直接在 `BookDeserializer` 增加author的反序列化代码。但是这对于程序的扩展很不好，因此不推荐。  
2、我们可以使用默认的GSON实现，因为该例中author类的实例变量和author实体字段具有相同的名称，因此可以使用默认的GSON解析。  
3、我们可以再写一个 `AuthorDeserializer` 类，将author的反序列化独立出来。

第二种方案的实现

先说下GSON中 `JsonDeserializer` 上下文  
在 `JsonDeserializer` 接口的 `deserialize()` 方法中，提供了参数 `JsonDeserializationContext`。我们可以将对象的反序列化委托给 `JsonDeserializationContext` 实例，如下所示。
```java
Author author = context.deserialize(jsonElement, Author.class);
```

完整的代码如下所示。
```java
package com.javacreed.examples.gson.part2;

import java.lang.reflect.Type;

import com.google.gson.JsonArray;
import com.google.gson.JsonDeserializationContext;
import com.google.gson.JsonDeserializer;
import com.google.gson.JsonElement;
import com.google.gson.JsonObject;
import com.google.gson.JsonParseException;

public class BookDeserializer implements JsonDeserializer<Book> {

  @Override
  public Book deserialize(final JsonElement json, final Type typeOfT, final JsonDeserializationContext context)
      throws JsonParseException {
   final JsonObject jsonObject = json.getAsJsonObject();

    final String title = jsonObject.get("title").getAsString();
    final String isbn10 = jsonObject.get("isbn-10").getAsString();
    final String isbn13 = jsonObject.get("isbn-13").getAsString();

    // Delegate the deserialization to the context
    Author[] authors = context.deserialize(jsonObject.get("authors"), Author[].class);

    final Book book = new Book();
    book.setTitle(title);
    book.setIsbn10(isbn10);
    book.setIsbn13(isbn13);
    book.setAuthors(authors);
    return book;
  }
}
```

第三种方案的实现

自定义author实体的反序列化类 `AuthorDeserializer` 如下所示：
```java
package com.javacreed.examples.gson.part2;

import java.lang.reflect.Type;

import com.google.gson.JsonDeserializationContext;
import com.google.gson.JsonDeserializer;
import com.google.gson.JsonElement;
import com.google.gson.JsonObject;
import com.google.gson.JsonParseException;

public class AuthorDeserializer implements JsonDeserializer {

  @Override
  public Author deserialize(final JsonElement json, final Type typeOfT, final JsonDeserializationContext context)
      throws JsonParseException {
    final JsonObject jsonObject = json.getAsJsonObject();

    final Author author = new Author();
    author.setId(jsonObject.get("id").getAsInt());
    author.setName(jsonObject.get("name").getAsString());
    return author;
  }
}
```

为了使用 `ArthurDeserialiser`，我们也需要像之前Book一样进行注册，如下所示
```java
package com.javacreed.examples.gson.part2;

import java.io.IOException;
import java.io.InputStreamReader;
import java.io.Reader;

import com.google.gson.Gson;
import com.google.gson.GsonBuilder;

public class Main {

  public static void main(final String[] args) throws IOException {
    // Configure GSON
    final GsonBuilder gsonBuilder = new GsonBuilder();
    gsonBuilder.registerTypeAdapter(Book.class, new BookDeserializer());
    gsonBuilder.registerTypeAdapter(Author.class, new AuthorDeserializer());
    final Gson gson = gsonBuilder.create();

    // Read the JSON data
    try (Reader reader = new InputStreamReader(Main.class.getResourceAsStream("/part2/sample.json"), "UTF-8")) {

      // Parse JSON to Java
      final Book book = gson.fromJson(reader, Book.class);
      System.out.println(book);
    }
  }
}
```

该例子输出如下所示：
```java
Java Puzzlers: Traps, Pitfalls, and Corner Cases [032133678X]
Written by:
  >> [1] Joshua Bloch
  >> [2] Neal Gafter
```

# 四、 对象引用
请看下面的JSON。
```json
{
  'authors': [
    {
      'id': 1,
      'name': 'Joshua Bloch'
    },
    {
      'id': 2,
      'name': 'Neal Gafter'
    }
  ],
  'books': [
    {
      'title': 'Java Puzzlers: Traps, Pitfalls, and Corner Cases',
      'isbn': '032133678X',
      'authors':[1, 2]
    },
    {
      'title': 'Effective Java (2nd Edition)',
      'isbn': '0321356683',
      'authors':[1]
    }
  ]
}
```
上面所示的JSON实体包括两个author和两本book。其中book通过author的id对author进行引用。这是一个相当常见的情况，因为这种方法会降低JSON实体的大小。下图为该JSON实体的 `JsonObject` 结构图

{% asset_img New-JSON-Object-Hierarchy.png  %}

这种新的JSON实体结构中引入了新的挑战：当反序列化book时，我们需要持有从JSON实体层次的其他分支反序列化出的author。

有很多种方案可以解决这个问题，下面只介绍一种  

具体方案为：`AuthorDeserialiser` 作为反序列出的作者的缓存，返回下一次过来的id请求。这种方法的优势在于它利用了 `JsonDeserializationContext`，并使关系变得透明。不幸的是，它因为需要处理缓存而增加了复杂性。

下面介绍下具体实现

由于该JSON实体包含两个数组，故我们需要定义一个新的类来映射该JSON实体
```java
package com.javacreed.examples.gson.part3;

public class Data {

  private Author[] authors;
  private Book[] books;

  // Methods removed for brevity
}
```
字段顺序决定了谁先被反序列化，但是这在我们这个例子中是没有关系的。

`AuthorDeserialiser` 需要被修改，使得它缓存反序列化出来的author。
```java
package com.javacreed.examples.gson.part3;

import java.lang.reflect.Type;
import java.util.HashMap;
import java.util.Map;

import com.google.gson.JsonDeserializationContext;
import com.google.gson.JsonDeserializer;
import com.google.gson.JsonElement;
import com.google.gson.JsonObject;
import com.google.gson.JsonParseException;
import com.google.gson.JsonPrimitive;

public class AuthorDeserializer implements JsonDeserializer<Author> {

  private final ThreadLocal<Map<Integer, Author>> cache = new ThreadLocal<Map<Integer, Author>>() {
    @Override
    protected Map<Integer, Author> initialValue() {
      return new HashMap<>();
    }
  };

  @Override
  public Author deserialize(final JsonElement json, final Type typeOfT, final JsonDeserializationContext context)
      throws JsonParseException {

    // Only the ID is available
    if (json.isJsonPrimitive()) {
      final JsonPrimitive primitive = json.getAsJsonPrimitive();
      return getOrCreate(primitive.getAsInt());
    }

    // The whole object is available
    if (json.isJsonObject()) {
      final JsonObject jsonObject = json.getAsJsonObject();

      final Author author = getOrCreate(jsonObject.get("id").getAsInt());
      author.setName(jsonObject.get("name").getAsString());
      return author;
    }

    throw new JsonParseException("Unexpected JSON type: " + json.getClass().getSimpleName());
  }

  private Author getOrCreate(final int id) {
    Author author = cache.get().get(id);
    if (author == null) {
      author = new Author();
      author.setId(id);
      cache.get().put(id, author);
    }
    return author;
  }
}
```

下面让我们来一个个讲解这个类的变更  
1、author被缓存在如下的对象中
```java
private final ThreadLocal<Map<Integer, Author>> cache = new ThreadLocal<Map<Integer, Author>>() {
    @Override
    protected Map<Integer, Author> initialValue() {
      return new HashMap<>();
    }
  };
```
它使用 `Map<String, Object>` 作为缓存机制。该map被保存一个 `ThreadLocal` 实例中。因此该类允许多个线程使用相同的变量，而不受其他线程的干扰。

2、author总是使用如下的方式进行检索
```java
private Author getOrCreate(final int id) {
    Author author = cache.get().get(id);
    if (author == null) {
      author = new Author();
      cache.get().put(id, author);
    }
    return author;
  }
```

3、`deserialize()`方法也将做如下的修改
`descerialiser` 可以接收 `JsonPrimitive` 或 `JsonObject` 参数。当 `BookDeserialiser` 执行以下代码时，传递给 `AuthorDeserialiser` 的 `JsonElement` 将是 `JsonPrimitive` 的一个实例。
```java
// This is executed within the BookDeserialiser
    Author[] authors = context.deserialize(jsonObject.get("authors"), Author[].class);
```
逻辑图如下：

{% asset_img Delegating-Deserialisation-to-Context.png  %}

`BookDeserialiser` 使用 `context` 代理author的反序列化，并提供了一个整形数组。对于该数组的每一个元素，该 `context` 都将调用下 `AuthorDeserialiser` 的 `deserialize()` 方法，并将该整形元素作为 `JsonPrimitive` 传递给方法。

另一个方面，当author被反序列化时，我们将得到一个包含author元素的 `JsonObject`。因此，在我们转换给定的 `JsonElement` 之前，我们需要确定它是否是正确的类型。
```java
// Only the ID is available
    if (json.isJsonPrimitive()) {
      final JsonPrimitive primitive = json.getAsJsonPrimitive();
      final Author author = getOrCreate(primitive.getAsInt());
      return author;
    }
```
在上面的例子中，只有id可用。 `JsonElement` 被转换为 `JsonPrimitive`，然后转换为int。该int用于从 `getOrCreate()` 方法中检索作者。

JsonElement可以是JsonObject类型，如下所示。
```java
// The whole object is available
    if (json.isJsonObject()) {
      final JsonObject jsonObject = json.getAsJsonObject();

      final Author author = getOrCreate(jsonObject.get("id").getAsInt());
      author.setName(jsonObject.get("name").getAsString());
      return author;
    }
```
在这种情况下，在返回作者之前，该名称将添加到由 `getOrCreate()` 方法返回的作者实例中。

最后，如果给定的 `JsonElement` 实例既不是 `JsonPrimitive` 也不是 `JsonObject`，将抛出异常。
```java
throw new JsonParseException("Unexpected JSON type: " + json.getClass().getSimpleName());
```

