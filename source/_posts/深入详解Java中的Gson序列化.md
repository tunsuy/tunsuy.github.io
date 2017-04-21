---
title: 深入详解Java中的Gson序列化
date: 2017-03-22 17:46:38
tags: [java,gson]
categories: java
keywords: [java,gson,序列化]
description:
---

译自 [GSON SERIALISER EXAMPLE](http://www.javacreed.com/gson-serialiser-example/
  )

Java对象可以使用Gson API（Homepage）转换成JSON字符串。在本文中，我们将看到我们如何使用默认的Gson实现和自定义实现将Java对象转换为JSON字符串。

下面列出的所有代码均放在：[http://java-creed-examples.googlecode.com/svn/gson/Gson Serialiser Example/](http://java-creed-examples.googlecode.com/svn/gson/Gson Serialiser Example/)

<!-- more -->

# 一、 简单例子
看下面的一个类：
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
这是一个代表一本书的简单Java类。现在我们需要将这个类序列化成以下的JSON对象。
```json
{
  "title": "Java Puzzlers: Traps, Pitfalls, and Corner Cases",
  "isbn-10": "032133678X",
  "isbn-13": "978-0321336781",
  "authors": [
    "Joshua Bloch",
    "Neal Gafter"
  ]
}
```

Gson可以在没有任何特殊配置的情况下序列化Book类，只需使用默认配置即可。 Gson使用Java字段名作为JSON名称，并相应地对其值进行序列化。

但是我们仔细查看上面显示的JSON示例，我们将看到ISBN字段包含负号：isbn-10和isbn-13。不幸的是，我们无法使用默认的Gson配置获取这些字段名称。解决此问题的一种方法是使用注释。使用注释，我们可以定义JSON字段名称，而Gson将使用另外的字段名。另一种方法是使用 `JsonSerialiser`，如以下示例所示。
```java
package com.javacreed.examples.gson.part1;

import java.lang.reflect.Type;

import com.google.gson.JsonArray;
import com.google.gson.JsonElement;
import com.google.gson.JsonObject;
import com.google.gson.JsonPrimitive;
import com.google.gson.JsonSerializationContext;
import com.google.gson.JsonSerializer;

public class BookSerialiser implements JsonSerializer {

    @Override
    public JsonElement serialize(final Book book, final Type typeOfSrc, final JsonSerializationContext context) {
        final JsonObject jsonObject = new JsonObject();
        jsonObject.addProperty("title", book.getTitle());
        jsonObject.addProperty("isbn-10", book.getIsbn10());
        jsonObject.addProperty("isbn-13", book.getIsbn13());

        final JsonArray jsonAuthorsArray = new JsonArray();
        for (final String author : book.getAuthors()) {
            final JsonPrimitive jsonAuthor = new JsonPrimitive(author);
            jsonAuthorsArray.add(jsonAuthor);
        }
        jsonObject.add("authors", jsonAuthorsArray);

        return jsonObject;
    }
}
```

让我们一步步来解释下上面的过程

如果我们要序列化这个Java对象，我们首先需要创建 `JsonElement` 的正确实例。在我们的例子中，我们返回一个 `JsonObject` 的实例，因为我们的Book是一个对象，如下所示。
```java
final JsonObject jsonObject = new JsonObject();
```
该对象将使用我们喜欢的名称填充我们的字段，如下所示
```java
// The variable 'book' is passed as a parameter to the serialize() method
    final JsonObject jsonObject = new JsonObject();
    jsonObject.addProperty("title", book.getTitle());
    jsonObject.addProperty("isbn-10", book.getIsbn10());
    jsonObject.addProperty("isbn-13", book.getIsbn13());
```
使用 `addProperty()` 方法，我们可以添加任何的java基本类型(String和Number)。请注意，属性名称必须是唯一的，否则将替换以前的值。这可以看作是一个 `Map`，它将所有的值保存在以他们的名字作为的索引上。

更复杂的对象（如Java对象和数组）无法使用上述方法添加。 `JsonObject` 有另一个方法叫做 `add`，可以如下所示使用。
```java
// The variable 'book' is passed as a parameter to the serialize() method
    jsonObject.addProperty("title", book.getTitle());
    jsonObject.addProperty("isbn-10", book.getIsbn10());
    jsonObject.addProperty("isbn-13", book.getIsbn13());

    final JsonArray jsonAuthorsArray = new JsonArray();
    for (final String author : book.getAuthors()) {
      final JsonPrimitive jsonAuthor = new JsonPrimitive(author);
      jsonAuthorsArray.add(jsonAuthor);
    }
    jsonObject.add("authors", jsonAuthorsArray);
```
我们首先创建 `JsonArray` 并将所有author添加到里面。与Java不同，我们不需要在初始化时定义数组的大小。事实上，我们可以将JsonArray类作为一个列表而不是数组。最后，将 `jsonAuthorsArray` 变量添加到json对象。请注意，我们可以在添加其元素之前将 `jsonAuthorsArray` 添加到 `jsonObject`。

在我们可以使用我们自定义序列化类之前，我们需要用Gson注册，如下所示。
```java
package com.javacreed.examples.gson.part1;

import java.io.IOException;

import com.google.gson.Gson;
import com.google.gson.GsonBuilder;

public class Main {

  public static void main(final String[] args) throws IOException {
    // Configure GSON
    final GsonBuilder gsonBuilder = new GsonBuilder();
    gsonBuilder.registerTypeAdapter(Book.class, new BookSerialiser());
    gsonBuilder.setPrettyPrinting();
    final Gson gson = gsonBuilder.create();

    final Book javaPuzzlers = new Book();
    javaPuzzlers.setTitle("Java Puzzlers: Traps, Pitfalls, and Corner Cases");
    javaPuzzlers.setIsbn10("032133678X");
    javaPuzzlers.setIsbn13("978-0321336781");
    javaPuzzlers.setAuthors(new String[] { "Joshua Bloch", "Neal Gafter" });

    // Format to JSON
    final String json = gson.toJson(javaPuzzlers);
    System.out.println(json);
  }
}
```
通过注册自定义的序列化类，每当类型为Book的对象被序列化时，Gson都将使用此序列化类。

在上面的例子中，我们还指示Gson格式化JSON对象，通过调用下面语言格式化打印。
```java
 gsonBuilder.setPrettyPrinting();
```

虽然这对于教程和调试是有用的，但请避免在生产环境中进行格式化打印，因为格式化可能会产生较大的JSON对象（在文本大小方面）。此外，还有轻微的性能成本，因为Gson必须格式化JSON对象并进行相应的缩进。

# 二、 嵌套对象
在本例中，我们将介绍如何对嵌套对象进行序列化，即对象中的对象。这里我们将介绍一个新的实体,如下所示：
```json
{
  "title": "Java Puzzlers: Traps, Pitfalls, and Corner Cases",
  "isbn": "032133678X",
  "authors": [
    {
      "id": 1,
      "name": "Joshua Bloch"
    },
    {
      "id": 2,
      "name": "Neal Gafter"
    }
  ]
}
````

请注意，在上一个示例中，作者只是一个字符串数组，如下所示
```json
"authors": [
    "Joshua Bloch",
    "Neal Gafter"
  ]
```
在这个例子中，作者是JSON对象，而不仅仅是原始图形，如下所示。
```java
{
      "id": 1,
      "name": "Joshua Bloch"
    }
```

JSON author有一个id和一个name。因此Author类如下所示。
```java
package com.javacreed.examples.gson.part2;

public class Author {

  private int id;
  private String name;

  // Methods removed for brevity

}
```
这个类很简单。它包括两个字段，两个都是 `JsonPrimitive` 类型。 Book类需要被修改为包含Author类对象的数组，如下所示。
```java
package com.javacreed.examples.gson.part2;

public class Book {

  private Author[] authors;
  private String isbn;
  private String title;

  // Methods removed for brevity

}
```
authors字段从String数组更改为Author数组。为了适应这种变化，必须修改 `BookSerialiser` 类，如下所示。
```java
package com.javacreed.examples.gson.part2;

import java.lang.reflect.Type;

import com.google.gson.JsonElement;
import com.google.gson.JsonObject;
import com.google.gson.JsonSerializationContext;
import com.google.gson.JsonSerializer;

public class BookSerialiser implements JsonSerializer<Book> {

  @Override
  public JsonElement serialize(final Book book, final Type typeOfSrc, final JsonSerializationContext context) {
    final JsonObject jsonObject = new JsonObject();
    jsonObject.addProperty("title", book.getTitle());
    jsonObject.addProperty("isbn", book.getIsbn());

    final JsonElement jsonAuthros = context.serialize(book.getAuthors());
    jsonObject.add("authors", jsonAuthros);

    return jsonObject;
  }
}
```
author的序列化委托给上下文(`JsonSerializationContext`的一个实例)，其作为参数传递给 `serialize()` 方法。上下文将序列化给定的对象并返回一个 `JsonElement` 类型对象。上下文将尝试找到可以序列化给定对象的序列化类，如果没有找到，则使用默认机制。这里我们暂时不会为Author类编写自定义的序列化类，而是使用默认的。
```java
package com.javacreed.examples.gson.part2;

import java.io.IOException;

import com.google.gson.Gson;
import com.google.gson.GsonBuilder;

public class Main {

  public static void main(final String[] args) throws IOException {
    // Configure GSON
    final GsonBuilder gsonBuilder = new GsonBuilder();
    gsonBuilder.registerTypeAdapter(Book.class, new BookSerialiser());
    gsonBuilder.setPrettyPrinting();
    final Gson gson = gsonBuilder.create();

    final Author joshuaBloch = new Author();
    joshuaBloch.setId(1);
    joshuaBloch.setName("Joshua Bloch");

    final Author nealGafter = new Author();
    nealGafter.setId(2);
    nealGafter.setName("Neal Gafter");

    final Book javaPuzzlers = new Book();
    javaPuzzlers.setTitle("Java Puzzlers: Traps, Pitfalls, and Corner Cases");
    javaPuzzlers.setIsbn("032133678X");
    javaPuzzlers.setAuthors(new Author[] { joshuaBloch, nealGafter });

    final String json = gson.toJson(javaPuzzlers);
    System.out.println(json);
  }
}
```

下面对Author类编写自定义的序列化类：
```java
package com.javacreed.examples.gson.part2;

import java.lang.reflect.Type;

import com.google.gson.JsonElement;
import com.google.gson.JsonObject;
import com.google.gson.JsonSerializationContext;
import com.google.gson.JsonSerializer;

public class AuthorSerialiser implements JsonSerializer<Author> {

  @Override
  public JsonElement serialize(final Author author, final Type typeOfSrc, final JsonSerializationContext context) {
    final JsonObject jsonObject = new JsonObject();
    jsonObject.addProperty("id", author.getId());
    jsonObject.addProperty("name", author.getName());

    return jsonObject;
  }
}
```
同样的，为了使用这个新的序列化类，我们需要注册它。

# 三、 对象引用
看下面的例子：
```java
package com.javacreed.examples.gson.part3;

import java.io.IOException;

import com.google.gson.Gson;
import com.google.gson.GsonBuilder;

public class Example1 {

  public static void main(final String[] args) throws IOException {
    // Configure GSON
    final GsonBuilder gsonBuilder = new GsonBuilder();
    gsonBuilder.setPrettyPrinting();
    final Gson gson = gsonBuilder.create();

    final Author joshuaBloch = new Author();
    joshuaBloch.setId(1);
    joshuaBloch.setName("Joshua Bloch");

    final Author nealGafter = new Author();
    nealGafter.setId(2);
    nealGafter.setName("Neal Gafter");

    final Book javaPuzzlers = new Book();
    javaPuzzlers.setTitle("Java Puzzlers: Traps, Pitfalls, and Corner Cases");
    javaPuzzlers.setIsbn("032133678X");
    javaPuzzlers.setAuthors(new Author[] { joshuaBloch, nealGafter });

    final Book effectiveJava = new Book();
    effectiveJava.setTitle("Effective Java (2nd Edition)");
    effectiveJava.setIsbn("0321356683");
    effectiveJava.setAuthors(new Author[] { joshuaBloch });

    final Book[] books = new Book[] { javaPuzzlers, effectiveJava };

    final String json = gson.toJson(books);
    System.out.println(json);
  }
}
```
这里我们有两个作者和两本书。其中一个作者在两本书之间共享。最后注意到，我们目前没有使用任何定制的序列化类。执行上述代码时会产生以下内容。
```json
[
  {
    "authors": [
      {
        "id": 1,
        "name": "Joshua Bloch"
      },
      {
        "id": 2,
        "name": "Neal Gafter"
      }
    ],
    "isbn": "032133678X",
    "title": "Java Puzzlers: Traps, Pitfalls, and Corner Cases"
  },
  {
    "authors": [
      {
        "id": 1,
        "name": "Joshua Bloch"
      }
    ],
    "isbn": "0321356683",
    "title": "Effective Java (2nd Edition)"
  }
]
```
我们有两个JSON book和三个JSON author。author：Joshua Bloch，如下所示，是重复的。
```json
{
   "id": 1,
   "name": "Joshua Bloch"
 }
```

这增加了JSON对象的大小，特别是对于更复杂的对象。理想情况下，JSON对象只包含作者的ID，而不是整个对象。这类似于关系数据库，其中该书具有作者表的外键。

Book类被修改为增加一个方法，该方法仅返回作者ID，如下所示。
```java
package com.javacreed.examples.gson.part3;

public class Book {

  private Author[] authors;
  private String isbn;
  private String title;

  public Author[] getAuthors() {
    return authors;
  }

  public int[] getAuthorsIds() {
    final int[] ids = new int[authors.length];
    for (int i = 0; i < ids.length; i++) {
      ids[i] = authors[i].getId();
    }
    return ids;
  }

  // Other methods removed for brevity

}
```

当序列化书籍时，我们将使用作者的ID而不是作者对象，如下所示。
```java
package com.javacreed.examples.gson.part3;

import java.lang.reflect.Type;

import com.google.gson.JsonElement;
import com.google.gson.JsonObject;
import com.google.gson.JsonSerializationContext;
import com.google.gson.JsonSerializer;

public class BookSerialiser implements JsonSerializer {

  @Override
  public JsonElement serialize(final Book book, final Type typeOfSrc, final JsonSerializationContext context) {
    final JsonObject jsonObject = new JsonObject();
    jsonObject.addProperty("title", book.getTitle());
    jsonObject.addProperty("isbn", book.getIsbn());

    final JsonElement jsonAuthros = context.serialize(book.getAuthorsIds());
    jsonObject.add("authors", jsonAuthros);

    return jsonObject;
  }
}
```

使用该自定义序列化类，将得到如下输出内容：
```java
package com.javacreed.examples.gson.part3;

import java.lang.reflect.Type;

import com.google.gson.JsonElement;
import com.google.gson.JsonObject;
import com.google.gson.JsonSerializationContext;
import com.google.gson.JsonSerializer;

public class BookSerialiser implements JsonSerializer {

  @Override
  public JsonElement serialize(final Book book, final Type typeOfSrc, final JsonSerializationContext context) {
    final JsonObject jsonObject = new JsonObject();
    jsonObject.addProperty("title", book.getTitle());
    jsonObject.addProperty("isbn", book.getIsbn());

    final JsonElement jsonAuthros = context.serialize(book.getAuthorsIds());
    jsonObject.add("authors", jsonAuthros);

    return jsonObject;
  }
}
```

