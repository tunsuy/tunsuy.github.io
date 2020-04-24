## 一、AOP基本概念

1、连接点(join point)  
程序运行中的一些时间点, 例如一个方法的执行, 或者是一个异常的处理.  
在 Spring AOP 中, join point 总是方法的执行点, 即只有方法连接点.

2、切点(point cut)  
匹配 join point 的谓词(a predicate that matches join points).  
Advice 是和特定的 point cut 关联的, 并且在 point cut 相匹配的 join point 中执行.   
在 Spring 中, 所有的方法都可以认为是 joinpoint, 但是我们并不希望在所有的方法上都添加 Advice, 而 pointcut 的作用就是提供一组规则(使用 AspectJ pointcut expression language 来描述) 来匹配joinpoint, 给满足规则的 joinpoint 添加 Advice.

关于join point 和 point cut 的区别  
在 Spring AOP 中, 所有的方法执行都是 join point. 而 point cut 是一个描述信息, 它修饰的是 join point, 通过 point cut, 我们就可以确定哪些 join point 可以被织入 Advice. 因此 join point 和 point cut 本质上就是两个不同纬度上的东西.  
advice 是在 join point 上执行的, 而 point cut 规定了哪些 join point 可以执行哪些 advice

声明 pointcut  
一个 pointcut 的声明由两部分组成:  
一个方法签名, 包括方法名和相关参数  
一个 pointcut 表达式, 用来指定哪些方法执行是我们感兴趣的(即因此可以织入 advice).

例如：  
`<aop:pointcut id="pointcut2“ expression="execution(* org.restlet.resource.ServerResource.handle(..))" />`

## 二、AOP的通知

1、前置通知：执行目标方法前拦截到的方法。没有特殊注意的地方，只需要一个连接点JoinPoint，即可获取拦截目标方法以及请求参数。

2、后置通知： 切面的后置通知，不管方法是否抛出异常，都会走这个方法。只需要一个连接点JoinPoint，即可获取当前结束的方法名称。

3、返回通知：  在方法正常执行通过之后执行的通知叫做返回通知。此时注意，不仅仅使用JoinPoint获取连接点信息，同时要在返回通知注解/配置里写入resut=“result”。在切面方法参数中加入Object result，用于接受返回通知的返回结果。如果目标方法是void返回类型则返回NULL

4、异常通知： 在执行目标方法过程中，如果方法抛出异常则会走此方法。和返回通知很相似，在注解/配置中加入throwing="ex"，在切面方法中加入Exection ex用于接受异常信息

5、环绕通知：环绕通知需要携带ProceedingJoinPoint 这个类型的参数，环绕通知类似于动态代理的全过程，ProceedingJoinPoint类型的参数可以决定是否执行目标函数，环绕通知必须有返回值。其实就是包含了所有通知的全过程

比如：  
`<aop:around pointcut-ref="pointcut2" method="around" />`

## 三、目标对象的参数传递

在Spring AOP中，除了execution和bean指示符不能传递参数给通知方法，其他指示符都可以将匹配的相应参数或对象自动传递给通知方法。

JoinPoint获取参数：  
Spring AOP提供使用org.aspectj.lang.JoinPoint类型获取连接点数据，任何通知方法的第一个参数都可以是JoinPoint(环绕通知是ProceedingJoinPoint，JoinPoint子类)。  
（1）JoinPoint：提供访问当前被通知方法的目标对象、代理对象、方法参数等数据  
（2）ProceedingJoinPoint：只用于环绕通知，使用proceed()方法来执行目标方法  
如参数类型是JoinPoint、ProceedingJoinPoint类型，可以从“argNames”属性省略掉该参数名（可选，写上也对），这些类型对象会自动传入的，但必须作为第一个参数。

## 四、匹配表达式

`execution(modifiers-pattern? ret-type-pattern declaring-type-pattern? name-pattern(param-pattern)throws-pattern?)`

modifier-pattern：表示方法的修饰符  
ret-type-pattern：表示方法的返回值  
declaring-type-pattern?：表示方法所在的类的路径  
name-pattern：表示方法名  
param-pattern：表示方法的参数  
throws-pattern：表示方法抛出的异常

注意事项  
其中后面跟着“?”的是可选项；  
在各个pattern中，可以使用“*”来表示匹配所有；  
在param-pattern中，可以指定具体的参数类型，多个参数间用“,”隔开，也可以用“*”来表示匹配任意类型的参数，如(String)表示匹配一个String参数的方法；(*,String)表示匹配有两个参数的方法，第一个参数可以是任意类型，而第二个参数是String类型；  
可以用(..)表示零个或多个任意的方法参数；  
使用&&符号表示与关系，使用||表示或关系、使用!表示非关系。在XML文件中使用and、or和not这三个符号。

## 五、Spring的AOP的三种使用方式：
原则是：基于注解的配置优先于基于Java的配置，基于Java的配置优先于基于XML的配置。但是，如果要声明切面，而有没法为通知类添加注解的时候，就只能使用XML配置了。

使用XML方式进行AOP的一般步骤  
1、编写Spring的xml文件  
在xml文件中定义aop的相关配置，定义增加类的bean

2、编写增强类  
可以基于切面中的方法，比如前置通知，后置通知，返回通知，异常通知，以及环绕通知写自己的业务逻辑，定义切点"execution(* com.huawei.service.*.*(..))"，即那些方法需要执行这些方法。如果想获取到方法的名字和参数，可以在方法中加入JoinPoint参数，可以获取到进入切面的方法细节。







