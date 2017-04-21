---
title: Protobuf协议代码生成器之wire
date: 2017-03-15 18:09:10
tags: [java,android,protobuf]
categories: java
keywords: [java,android,protobuf,wire]
description:
---

# 一、 前言
首先要搞清楚 `Protocol Buffers` 和代码生成器。

`Protocol Buffers` 只是一种数据传输协议格式，是Google定义的，它是与语言和平台均无关的，用于描述和传输数据的语言。开发人员可以在不同的环境中使用相同的模式开发。`Protocol Buffers`中的 `.proto` 源文件是人类可读的，可以包含注释。`Protocol Buffers`定义了一种紧凑的二进制格式，允许项目结构在不破坏现有客户端的情况下发展。

而代码生成器则是根据这个协议文档自动生成相应语言代码，任何组织机构都可以开发该工具。也就是说，使用代码生成器来读取 `.proto` 文件，并以你选择的语言生成源代码。这种方法有助于加快开发速度。。下文中将Google自身的工具称为标准工具，即 `protoc`

<!-- more -->

`Protocol Buffers` 标准使用：[Protocol Buffer Basics: Java](https://developers.google.com/protocol-buffers/docs/javatutorial#compiling-your-protocol-buffers)

`Protocol Buffers` 标准工具 `protoc` 会为你的 `Message` 中的每个可选或必需字段生成至少九种方法，以及至少十八种重复字段的方法！在非限制环境中拥有所有这些灵活性是非常好的。但是在Android环境中，Dalvik字节码格式在单个应用程序中强加了64K的方法数限制。所以这将大大限制业务代码方法数。也是就有了一些第三方组织自研的代码生成器。

# 二、 wire简介
Wire是基于Google的 [Protocol Buffers](https://github.com/google/protobuf) 的新的开源实现。它适用于Android设备，但可用于运行Java语言代码的任何地方。

项目地址： [https://github.com/square/wire](https://github.com/square/wire)

对于Android应用开发中的数据传输协议，我们希望其应该具有如下几个特性：  
1、消息应包含最少数量的生成方法  
2、消息应该是纯净的且是开发友好的数据对象  
3、他们应该高度可读的  
4、他们应该是不可改变的  
5、他们应该有诸如 `equals`，`hashCode` 和 `toString` 等这些有用的方法  
6、他们应该支持链式Builder模式  
7、他们应该从.proto源文件继承文档  
8、`Protocol Buffers`中的 `enums` 类型应该映射到Java中的 `enums`  
9、理想情况下，所有应用程序都可以使用基于Java的工具进行构建

wire正是在这样的需求之上实现的，它是构建在 [ProtoParser](https://github.com/square/protoparser/) 和 [JavaWriter](https://github.com/square/javapoet) 之上的。

google也针对Android这样的移动设备推出了自己的协议 [nano](https://github.com/android/platform_external_protobuf/tree/master/java/src/main/java/com/google/protobuf/nano)，这个协议有更少的方法产生，但是它不具有上述的所有需求。

# 三、 wire应用
看下面的protobuf文件定义：
```protobuf
message Person {
  // The customer's full name.
  required string name = 1;
  // The customer's ID number.
  required int32 id = 2;
  // Email address for the customer.
  optional string email = 3;
  enum PhoneType {
    MOBILE = 0;
    HOME = 1;
    WORK = 2;
  }
  message PhoneNumber {
    // The user's phone number.
    required string number = 1;
    // The type of phone stored here.
    optional PhoneType type = 2 [default = HOME];
  }
  // A list of the user's phone numbers.
  repeated PhoneNumber phone = 4;
}
```

通过wire生成的java部分代码如下所示：
```java
public final class Person extends Message {
  /** The customer's full name. */
  @ProtoField(tag = 1, type = STRING, label = REQUIRED)
  public final String name;
  /** The customer's ID number. */
  @ProtoField(tag = 2, type = INT32, label = REQUIRED)
  public final Integer id;
  /** Email address for the customer. */
  @ProtoField(tag = 3, type = STRING)
  public final String email;
  /**  A list of the user's phone numbers. */
  @ProtoField(tag = 4, label = REPEATED)
  public final List<PhoneNumber> phone;
  private Person(Builder builder) {
    super(builder);
    this.name = builder.name;
    this.id = builder.id;
    this.email = builder.email;
    this.phone = immutableCopyOf(builder.phone);
  }
  @Override public boolean equals(Object other) {
    if (!(other instanceof Person)) return false;
    Person o = (Person) other;
    return equals(name, o.name)
        && equals(id, o.id)
        && equals(email, o.email)
        && equals(phone, o.phone);
  }
  @Override public int hashCode() {
    int result = hashCode;
    if (result == 0) {
      result = name != null ? name.hashCode() : 0;
      result = result * 37 + (id != null ? id.hashCode() : 0);
      result = result * 37 + (email != null ? email.hashCode() : 0);
      result = result * 37 + (phone != null ? phone.hashCode() : 0);
      hashCode = result;
    }
    return result;
  }
  public static final class Builder extends Message.Builder<Person> {
    // not shown
  }
}
```

`Message` 类的实例只能由相应的嵌套 `Builder` 类创建。 Wire在每个构建器中为每个字段生成单个方法，以支持链接：
```java
Person person = new Person.Builder()
    .name("Omar")
    .id(1234)
    .email("omar@wire.com")
    .phone(Arrays.asList(new PhoneNumber.Builder()
        .number("410-555-0909")
        .type(PhoneType.MOBILE)
        .build()))
    .build();
```

Wire通过为每个 `Message` 字段使用 `public final` 修饰来减少生成的方法的数量。数组被包装，所以 `Message` 实例是不可改变的。每个字段都用 `@ProtoField` 来注解，以便提供Wire执行序列化和反序列化所需要的元数据，比如：
```java
@ProtoField(tag = 1, type = STRING, label = REQUIRED)
public final String name;
```

可以直接使用这些字段来访问你的数据，比如：
```java
if (person.phone != null) {
  for (PhoneNumber phone : person.phone)
    if (phone.type == PhoneType.MOBILE) {
      sendSms(person.name, phone.number, message);
      break;
    }
  }
}
```

我们将上面创建的 `person` 实例进行序列化和反序列化，代码如下所示：
```java
// 序列化
byte[] data = person.toByteArray();

//反序列化
Wire wire = new Wire();
Person newPerson = wire.parseFrom(data, Person.class);
```

wire使用了反射机制来实现一些功能，如序列化，反序列化和 `toString` 方法。并且缓存有关每个消息类的反射信息，以获得更好的性能。

在标准 `Protocol Buffers`(protoc)生成的代码中，你可以调用 `person.hasEmail()` 来查看是否已经设置了电子邮件地址。但是使用wire，你只需检查 `person.email == null`。对于诸如 `phone` 等重复字段，Wire还需要你的应用程序一次性获取或设置 `PhoneNumber` 实例列表，从而节省了大量方法。

Wire支持附加功能，如扩展名和未知字段。目前，它缺乏对一些高级功能的支持，包括自定义选项，服务和运行时自省等。也不支持目前已经过时的 `groups` 功能。

