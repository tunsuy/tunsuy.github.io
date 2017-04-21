---
title: 使用Spring实现Java中的缓存
date: 2017-03-10 12:11:14
tags: [java,spring]
categories: java
keywords: [java,spring,缓存]
description:
---

译自 [CACHING MADE EASY WITH SPRING](http://www.javacreed.com/caching-made-easy-with-spring/)

Spring3.1引入了一种新的简单的方法来缓存结果。在本文中，我们将看到如何在我们的项目中使用新的Spring缓存。本文的读者应该有一些关于Spring和依赖注入的基本知识（或者也称为：控制反转IOC）。

以下列出的所有代码均可在 [http://code.google.com/p/java-creed-examples/source/checkout](http://code.google.com/p/java-creed-examples/source/checkout) 中找到。

# 一、 简单缓存
看看如下的类：

<!-- more -->

```java
package com.javacreed.examples.sc.part1;

import org.springframework.stereotype.Component;

@Component
public class Worker {

  public String longTask(final long id) {
    System.out.printf("Running long task for id: %d...%n", id);
    return "Long task for id " + id + " is done";
  }

  public String shortTask(final long id) {
    System.out.printf("Running short task for id: %d...%n", id);
    return "Short task for id " + id + " is done";
  }
}
```
这里我们有一个简单的Spring组件类，它有两个方法。一个名为 `longTask()` 方法：表示一个虚拟的耗时任务，而名为 `shortTask()` 的第二种方法可快速运行。两种方法的输出仅由这些方法的输入决定。因此，对于相同的输入，我们将始终得到相同的输出。这是非常重要的，否则我们不能应用缓存。

再看下对这个类的实际使用情况：
```java
package com.javacreed.examples.sc.part1;

import org.springframework.context.support.ClassPathXmlApplicationContext;

public class Main {

  public static void main(final String[] args) {
    final String xmlFile = "META-INF/spring/app-context.xml";
    try (ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext(xmlFile)) {

      final Worker worker = context.getBean(Worker.class);
      worker.longTask(1);
      worker.longTask(1);
      worker.longTask(1);
      worker.longTask(2);
      worker.longTask(2);
    }
  }
}
```
在这里，我们创建了Spring环境并从Spring检索了一个Worker的实例。然后我们调用 `longTask()` 方法五次。这将产生以下输出:
```java
Running long task for id: 1...
Running long task for id: 1...
Running long task for id: 1...
Running long task for id: 2...
Running long task for id: 2...
```
请注意，此方法只接收两个不同的输入。参数值为1的输入调用了3次，参数值为2的输入调用了2次。由于该方法的输出仅由其输入决定，因此可以使用缓存，在下一个请求该输入值时直接返回该输出值，而不是重新运行该方法。

为了使用缓存，我们需要做如下的三件事

1、标记输出将被缓存的方法（或类）。  
Spring 3.1添加了新的注释，启用方法缓存，如下所示。
```java
package com.javacreed.examples.sc.part1;

import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Component;

@Component
public class Worker {

  @Cacheable("task")
  public String longTask(final long id) {
    System.out.printf("Running long task for id: %d...%n", id);
    return "Long task for id " + id + " is done";
  }

  public String shortTask(final long id) {
    System.out.printf("Running short task for id: %d...%n", id);
    return "Short task for id " + id + " is done";
  }

}
```
通过简单地将 `@Cacheable` 注释添加到方法签名中，具有相同参数值的此方法的重复请求将简单地返回缓存的值。 Spring允许我们缓存值，而无需编写处理这种情况的样板代码。请注意，此注释还将获取一个值，该值是缓存存储库的名称。

请注意，`@Cacheable` 的注释可以应用于一个类，这意味着该类的所有方法都被缓存。

2、启用Spring缓存  
在Spring开始缓存我们的值之前，我们需要添加以下声明
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xmlns:context="http://www.springframework.org/schema/context"
  xmlns:cache="http://www.springframework.org/schema/cache"
  xmlns:p="http://www.springframework.org/schema/p"
  xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-3.1.xsd
    http://www.springframework.org/schema/cache http://www.springframework.org/schema/cache/spring-cache-3.2.xsd
    http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-3.1.xsd">

  <context:annotation-config />
  <context:component-scan base-package="com.javacreed.examples.sc" />

  <!-- Enables the caching through annotations -->
  <cache:annotation-driven />

</beans>
```
有了这个声明，Spring会寻找任何被标记为可缓存的类或方法，并且将采取所有必要的操作来提供缓存。

3、配置要使用的缓存存储库。  
在此代码可以工作之前，我们需要定义缓存存储库，如下所示。
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xmlns:context="http://www.springframework.org/schema/context"
  xmlns:cache="http://www.springframework.org/schema/cache"
  xmlns:p="http://www.springframework.org/schema/p"
  xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-3.1.xsd
    http://www.springframework.org/schema/cache http://www.springframework.org/schema/cache/spring-cache-3.2.xsd
    http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-3.1.xsd">

  <context:annotation-config />
  <context:component-scan base-package="com.javacreed.examples.sc" />

  <!-- Enables the caching through annotations -->
  <cache:annotation-driven />

  <!-- Generic cache manager based on the JDK ConcurrentMap -->
  <bean id="cacheManager" class="org.springframework.cache.support.SimpleCacheManager">
    <property name="caches">
      <set>
        <bean class="org.springframework.cache.concurrent.ConcurrentMapCacheFactoryBean" p:name="task" />
      </set>
    </property>
  </bean>
</beans>
```
缓存存储库是保存实际对象的地方。 Spring支持两种类型的存储库：一种基于 `JDK ConcurrentMap`，另一种在 `ehcache` 流行库中。这里我们使用 `JDK ConcurrentMap` 作为缓存存储库。存储库对代码的影响很小（如果有的话），并且存储库之间的切换很容易。在这个例子中，我们添加了一个名为 `task` 的缓存存储库。我们可以有多个存储库。请注意，此存储库的名称必须与之前的注释中显示的名称相同。

请注意，上述声明中所示的 `JDK ConcurrentMap` 类因Spring 3.1和3.2版本而异。这里我们使用的是Spring 3.2。在版本3.1中，类名如下所示。
```java
org.springframework.cache.concurrent.ConcurrentCacheFactoryBean
```

那么该缓存到底是怎么工作的呢？

在配置为使用缓存时，Spring将标记为要缓存的对象包装到代理中。调用者将不会使用我们的对象，而是使用代理，如下图所示。

{% asset_img Spring-Cache-Example-1.png %}

如果我们打印worker类的Spring环境返回的对象的规范名称，我们将看到如下。
```java
Worker class: com.javacreed.examples.sc.part1.Worker$$EnhancerByCGLIB$$4fa6f80b
```
请注意，这不是我们创建的Worker类（规范名称：com.javacreed.examples.sc.part1.Worker），而是其他类。实际上，这个类是由Spring使用代码生成技术生成的，这里没有讨论。当我们从Worker类调用任何方法时，我们正在调用生成的代理中的方法。此代理持有我们的Worker类的实例。它会将任何请求转发给我们的对象并返回其响应，如下图所示。如果方法被标记为可缓存，那么代理将绕过请求，并返回缓存的值。如果代理没有给定输入的缓存值，它将发出请求并保存响应以备将来使用。

{% asset_img Spring-Cache-Example-2.png %}

运行该例子，得到如下输出：
```java
Running long task for id: 1...
Running long task for id: 2...
```
这里耗时方法（`longTask()`）实际上被调用两次。代理在其他时间返回缓存的结果。我们的第一节关于Spring缓存。我们看到，这是很容易启用。我们所需要做的就是按照上面列出的三个步骤进行，我们有缓存。在下一节中，我们将看到我们如何使用递归应用缓存。

# 二、 缓存递归方法
看下面的一个类：
```java
package com.javacreed.examples.sc.part2_1;

import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Component;

@Component("fibonacci")
public class Fibonacci {

  private int executions = 0;

  public int getExecutions() {
    return executions;
  }

  public void resetExecutions() {
    this.executions = 0;
  }

  @Cacheable("fibonacci")
  public long valueAt(final long index) {
    executions++;
    if (index < 2) {
      return 1;
    }

    return valueAt(index - 1) + valueAt(index - 2);
  }

}
```
该类实现斐波那契序列，并用给定的索引返回斐波那契数。斐波那契数是使用以下函数递归计算的：`fib（n）= fib（n-1）+ fib（n-2）`。此递归函数的基本情况是前两个斐波那契数为1。

请注意，此类还会跟踪调用 `valueAt()` 方法的次数。我们可以通过getter方法获得这个值。斐波那契类还启用了这个值的重置，使得计数器从0开始再次启动。

现在我们执行下这个例子：
```java
package com.javacreed.examples.sc.part2_1;

import org.springframework.context.support.ClassPathXmlApplicationContext;

public class Main {

  public static void main(final String[] args) {
    final String xmlFile = "META-INF/spring/app-context.xml";
    try (ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext(xmlFile)) {

      final long start = System.nanoTime();
      final Fibonacci sequence = context.getBean("fibonacci", Fibonacci.class);
      final long fibNumber = sequence.valueAt(5);
      final int executions = sequence.getExecutions();
      final long timeTaken = System.nanoTime() - start;
      System.out.printf("The 5th Fibonacci number is: %d (%,d executions in %,d NS)%n", fibNumber, executions,
          timeTaken);
    }
  }
}
```
输出如下：
```java
The 5th Fibonacci number is: 8 (15 executions in 17,762,022 NS)
```
从输出可以看出：可缓存方法 `valueAt()` 被调用了15次。这看起来不正确。 `valueAt()` 方法应该只执行6次而不是15次。其他9次，应该直接返回缓存值。

问题出在哪里呢？

在 `main()` 方法中，我们通过Spring获得了一个 `Fibonacci` 类的实例。反过来，Spring将我们的对象包装成代理。因此在 `main()` 方法中，我们只能访问代理。但是在 `Fibonacci` 类中的 `valueAt()` 方法，调用自身（递归）。这不是通过代理调用 `valueAt()` 方法，而是直接从 `Fibonacci` 类调用。因此代理被绕过。这就是为什么没有使用缓存的原因

（注：如果我们再次调用 `sequence.valueAt（5）`，那么此时将直接返回缓存值，因为变量 `sequence` 是代理的斐波纳契的一个实例。）

怎么解决上面那个问题呢？

我们需要修改 `Fibonacci` 类并传递我们代理的引用，如下所示。
```java
package com.javacreed.examples.sc.part2_2;

import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Component;

@Component("fibonacci2")
public class Fibonacci {

  private int executions = 0;

  public int getExecutions() {
    return executions;
  }

  public void resetExecutions() {
    this.executions = 0;
  }

  @Cacheable("fibonacci")
  public long valueAt(final long index, final Fibonacci callback) {
    executions++;
    if (index < 2) {
      return 1;
    }

    return callback.valueAt(index - 1, callback) + callback.valueAt(index - 2, callback);
  }

}
```
请注意，现在我们的 `valueAt()` 方法有两个参数，而不是一个。它需要Fibonacci类的一个实例，称为回调。此外，它不是调用 `valueAt()` 本身，而是调用回调的 `valueAt()`。同样的 `main()` 方法，我们还需要传递代理的 `Fibonacci` 类的一个实例，如下例所示。
```java
package com.javacreed.examples.sc.part2_2;

import org.springframework.context.support.ClassPathXmlApplicationContext;

public class Main {

  public static void main(final String[] args) {
    final String xmlFile = "META-INF/spring/app-context.xml";
    try (ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext(xmlFile)) {

      final long start = System.nanoTime();
      final Fibonacci sequence = context.getBean("fibonacci2", Fibonacci.class);
      final long fibNumber = sequence.valueAt(5, sequence);
      final int executions = sequence.getExecutions();
      final long timeTaken = System.nanoTime() - start;
      System.out.printf("The 5th Fibonacci number is: %d (%,d executions in %,d NS)%n", fibNumber, executions,
          timeTaken);
    }
  }
}
```
输出如下：
```java
The 5th Fibonacci number is: 8 (6 executions in 18,320,003 NS)
```

# 三、 缓存实践
看下面一个类：
```java
package com.javacreed.examples.sc.part3;

public class Member {

  private final int memberId;
  private final String memberName;

  public Member(final int memberId, final String memberName) {
    this.memberId = memberId;
    this.memberName = memberName;
  }

  // Getters removed for brevity

  @Override
  public String toString() {
    return String.format("[%d] %s", memberId, memberName);
  }
}
```
这是一个简单的类，一个成员只有一个id(唯一标识一个成员)和一个名字。为了简单起见，我们将成员保存在如下格式的文本文件中。
```java
1,Albert Attard
2,Mary Borg
3,Tony White
4,Jane Black
```

现在有如下一个服务接口：
```java
package com.javacreed.examples.sc.part3;

public interface MembersService {

  Member getMemberWithId(int id);

  void saveMember(Member member);
}
```
该接口暴露了两种方法，一个用于检索具有给定ID的成员的方法，另一个用于持久化对文件的任何修改。在main方法中，向 `MembersService` 的实现类发起多个请求。
```java
package com.javacreed.examples.sc.part3;

import org.springframework.context.support.ClassPathXmlApplicationContext;

public class Main {

  public static void main(final String[] args) {
    final String xmlFile = "META-INF/spring/app-context.xml";
    try (ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext(xmlFile)) {

      final MembersService service = context.getBean(MembersService.class);

      // Load member with id 1
      Member member = service.getMemberWithId(1);
      System.out.println(member);

      // Load member with id 1 again
      member = service.getMemberWithId(1);
      System.out.println(member);

      // Edit member with id 1
      member = new Member(1, "Joe Vella");
      service.saveMember(member);

      // Load member with id 1 after it was modified
      member = service.getMemberWithId(1);
      System.out.println(member);
    }
  }
}
```
得到如下的输出：
```java
Retrieving the member with id: [1] from file: C:\javacreed\spring-cache\members.txt
[1] Albert Attard
[1] Albert Attard
Retrieving the member with id: [1] from file: C:\javacreed\spring-cache\members.txt
[1] Joe Vella
```
这里我们发起了两个请求来检索ID为1的成员，但该方法实际上被调用了一次。在第二个请求中，返回缓存的值。然后我们用相同的id修改了成员。由于成员被修改，缓存无效。因此，当检索到具有相同id的成员时，我们再次调用了实际的方法，并从文件加载它。此值将被缓存，直到再次失效。

现在让我们看看这是如何实现的。 `getMemberWithId()` 类似于我们已经看到的其他方法。它用 `@Cacheable` 注释。
```java
@Override
  @Cacheable("members")
  public Member getMemberWithId(final int id) {
    System.out.printf("Retrieving the member with id: [%d] from file: %s%n", id, dataFile.getAbsolutePath());
   // code removed for brevity
  }
```

`saveMember()` 需要使缓存无效。为了实现这一点，Spring提供了另一个名为：`@CacheEvict` 的注解，如下所示。
```java
@Override
  @CacheEvict(value = "members", allEntries = true)
  public void saveMember(final Member member) {
   // code removed for brevity
  }
```
无论何时调用此方法，名为 `members` 的高速缓存存储库将从所有成员中清除（按照以下命令执行：`allEntries = true annotation optional` 参数）。因此，下一次调用 `getMemberWithId()` 方法时，将不得不从文件加载成员，从而读取新的更改。如果没有这个，`getMemberWithId()` 方法仍然会返回id为1的成员的旧版本。

