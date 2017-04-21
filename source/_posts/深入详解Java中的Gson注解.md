---
title: 深入详解Java中的Gson注解
date: 2017-03-20 15:14:37
tags: [java,gson]
categories: java
keywords: [java,gson,注解]
description:
---

译自 [GSON ANNOTATIONS EXAMPLE](http://www.javacreed.com/gson-annotations-example/)

Gson提供了一组注解来简化序列化和反序列化过程。在本文中，我们将看到我们如何使用这些注释，以及如何简化Gson在Java对象和JSON对象之间的转换。

以下列出的所有代码在该链接可以找到：[Gson Annotations Example]( https://java-creed-examples.googlecode.com/svn/gson/Gson Annotations Example)。

Gson提供了四个注解，如Java文档中所述。这些注解可以分为三类。下面每个类别将分别讨论。

<!-- more -->

# 一、 自定义字段名称
看下面的类：
```java
package com.javacreed.examples.gson.part1;

import com.google.gson.annotations.SerializedName;

public class Box {

  @SerializedName("w")
  private int width;

  @SerializedName("h")
  private int height;

  @SerializedName("d")
  private int depth;

  // Methods removed for brevity
}
```
这个类有三个字段: 表示一个box的width，height和depth。这些字段用 `@SerializedName` 注解。此注解的参数（值）是序列化和反序列化对象时要使用的名称。例如，Java字段depth在JSON中表示为d。

Gson支持此注解，而不需要任何配置，如下所示。
```java
package com.javacreed.examples.gson.part1;

import com.google.gson.Gson;
import com.google.gson.GsonBuilder;

public class Main {
  public static void main(final String[] args) {
    final GsonBuilder builder = new GsonBuilder();
    final Gson gson = builder.create();

    final Box box = new Box();
    box.setWidth(10);
    box.setHeight(20);
    box.setDepth(30);

    final String json = gson.toJson(box);
    System.out.printf("Serialised: %s%n", json);

    final Box otherBox = gson.fromJson(json, Box.class);
    System.out.printf("Same box: %s%n", box.equals(otherBox));
  }
}
```
输出如下所示：
```java
Serialised: {"w":10,"h":20,"d":30}
Same box: true
```
第一行是序列化的输出。请注意，当序列化Box Java对象时，Gson如何使用新名称作为JSON字段名称。同样适用于反序列化。 Gson将JSON字段重新映射到适当的Java字段名称。这由 `equals()` 方法验证。第二行打印为true，这意味着反序列化出来的对象与最初创建的对象相同。

# 二、 字段控制
Gson提供了两种类型的转换：序列化（从Java到JSON）和反序列化（从JSON到Java）。标记为 `transient` 的Java字段不会被序列化和反序列化。因此，不应该序列化的敏感信息可以被标记为 `transient`，而Gson不会将其序列化为JSON。

Gson还提供更精细的序列化和反序列化控制和过滤。使用Gson，我们可以使用注解来独立地控制什么是序列化和反序列化。或者，我们可以使用自定义 `JsonDeserializer<T>` 和自定义 `JsonSerializer<T>`。虽然这些接口提供了完全的控制和灵活性，但本文中描述的注解方法更简单，因为它不需要额外的类，我们将在下面的示例中看到。

看下面的类：
```java
package com.javacreed.examples.gson.part2;

import com.google.gson.annotations.Expose;

public class Account {

  @Expose(deserialize = false)
  private String accountNumber;

  @Expose
  private String iban;

  @Expose(serialize = false)
  private String owner;

  @Expose(serialize = false, deserialize = false)
  private String address;

  private String pin;
}
```
这里我们使用了 `@Expose` 注解，它具有两个可选元素：`deserialize` 和 `serialize`。通过这两个元素，我们可以独立地控制序列化和反序列化，如下表所示。元素的默认值都为true。

为了能够使用 `@Expose` 此注解，我们需要使用正确的配置。与 `@SerializedName` 注解不同，我们需要将Gson配置为仅expose注释的字段，并忽略其余的字段，如以下代码片段所示。
```java
final GsonBuilder builder = new GsonBuilder();
    builder.excludeFieldsWithoutExposeAnnotation();
    final Gson gson = builder.create();
```
没有这个配置，这个注释将被忽略，所有符合条件的字段被序列化和反序列化，包括字段引脚。以下是完整的例子。
```java
package com.javacreed.examples.gson.part2;

import java.io.InputStreamReader;
import java.io.Reader;

import com.google.gson.Gson;
import com.google.gson.GsonBuilder;

public class Main {
  public static void main(final String[] args) throws Exception {
    final GsonBuilder builder = new GsonBuilder();
    builder.excludeFieldsWithoutExposeAnnotation();
    final Gson gson = builder.create();

    final Account account = new Account();
    account.setAccountNumber("A123 45678 90");
    account.setIban("IBAN 11 22 33 44");
    account.setOwner("Albert Attard");
    account.setPin("1234");
    account.setAddress("Somewhere, Far Far Away");

    final String json = gson.toJson(account);
    System.out.printf("Serialised%n  %s%n", json);

    try (final Reader data = new InputStreamReader(Main.class.getResourceAsStream("account.json"), "UTF-8")) {

      final Account otherAccount = gson.fromJson(data, Account.class);
      System.out.println("Deserialised");
      System.out.printf("  Account Number: %s%n", otherAccount.getAccountNumber());
      System.out.printf("  IBAN:           %s%n", otherAccount.getIban());
      System.out.printf("  Owner:          %s%n", otherAccount.getOwner());
      System.out.printf("  Pin:            %s%n", otherAccount.getPin());
      System.out.printf("  Address:        %s%n", otherAccount.getAddress());
    }
  }
}
```
输出：
```java
Serialised
  {"accountNumber":"A123 45678 90","iban":"IBAN 11 22 33 44"}
Deserialised
  Account Number: null
  IBAN:           IBAN 11 22 33 44
  Owner:          Albert Attard
  Pin:            null
  Address:        null
```
请注意，只有 `accountNumber` 和 `iban` 被序列化，如上表所示。此外，当反序列化以下JSON时，同样的只处理了 `owner` 和 `iban`。其他字段被忽略。
```json
{
  "accountNumber": "A123 45678 90",
  "iban":          "IBAN 11 22 33 44",
  "owner":         "Albert Attard",
  "address":       "Somewhere, Far Far Away",
  "pin":           "1234"
}
```

使用 `@Expose` 注解，我们可以获得对串行化和反序列化的细粒度控制，而无需编写处理此逻辑的代码。

# 三、 版本控制
Gson提供了两个注释，称为 `@Since` 和 `@Until`，可以用于控制在使用Gson在Java对象和JSON对象之间进行转换时，进行版本的标注。`@Since` 表示增加，`@Until` 表示删除。

考虑下面这样一个类，有四个字段。其中一个字段在版本1.2中被添加，另一个字段在版本0.9中被删除。
```java
package com.javacreed.examples.gson.part3;

import com.google.gson.annotations.Since;
import com.google.gson.annotations.Until;

public class SoccerPlayer {

  private String name;

  @Since(1.2)
  private int shirtNumber;

  @Until(0.9)
  private String country;

  private String teamName;

  // Methods removed for brevity
}
```

与之前描述的@Expose注释类似，为了使此注释生效，我们需要进行正确的配置，如下所示。
```java
package com.javacreed.examples.gson.part3;

import java.io.InputStreamReader;
import java.io.Reader;

import com.google.gson.Gson;
import com.google.gson.GsonBuilder;

public class Main {
  public static void main(final String[] args) throws Exception {

    final GsonBuilder builder = new GsonBuilder();
    builder.setVersion(1.0);

    final Gson gson = builder.create();

    final SoccerPlayer account = new SoccerPlayer();
    account.setName("Albert Attard");
    account.setShirtNumber(10); // Since version 1.2
    account.setTeamName("Zejtun Corinthians");
    account.setCountry("Malta"); // Until version 0.9

    final String json = gson.toJson(account);
    System.out.printf("Serialised (version 1.0)%n  %s%n", json);

    try (final Reader data = new InputStreamReader(Main.class.getResourceAsStream("player.json"), "UTF-8")) {
      // Parse JSON to Java
      final SoccerPlayer otherPlayer = gson.fromJson(data, SoccerPlayer.class);
      System.out.println("Deserialised (version 1.0)");
      System.out.printf("  Name:          %s%n", otherPlayer.getName());
      System.out.printf("  Shirt Number:  %s (since version 1.2)%n", otherPlayer.getShirtNumber());
      System.out.printf("  Team:          %s%n", otherPlayer.getTeamName());
      System.out.printf("  Country:       %s (until version 0.9)%n", otherPlayer.getCountry());
    }
  }
}
```
在这个例子中，我们将版本定义为1.0，这意味着只有 `name` 和 `teamName` 会被处理。版本1.2中添加的 `shirtNumber`，因此不符合条件，0.9版本后删除了 `country`。得到如下输出：
```java
Serialised (version 1.0)
  {"name":"Albert Attard","teamName":"Zejtun Corinthians"}
Deserialised (version 1.0)
  Name:          Albert Attard
  Shirt Number:  0 (since version 1.2)
  Team:          Zejtun Corinthians
  Country:       null (until version 0.9)
```

