## sqlalchemy中的session
session 是 query 对象的入口，query 对象从数据库获取的对象都会以 identity map 的方式存储在 session 内部。session 负责维护和跟踪这些 objects 的状态，直到调用 session 的 commit 把具有 pending 状态的数据持久化到数据库，或者 issue new query 重新获得新的数据。

1、session其实就可以看做一个事务

2、构建session时可以选择是否autoflush、autocommit
autoflush默认是true、autocommit默认是false

3、flush表示提交sql到数据库执行(预提交，等于提交到数据库内存，还未写入数据库文件)，此时同一个事务是可以读到最新的数据的
commit表示事务提交(会先自动执行flush),就是把内存里面的东西直接写入，可以提供查询了，此时其他事务也可以读取到最新的数据
注：如果不commit，只flush，那么其他事务能读取到什么数据还要看数据库的隔离级别

4、session的生命周期
session 的生命周期管理随使用的场景不同而不同。一般来说，session 是从一个伴随着一个逻辑事务的开始而开始，结束而结束。此时就需要开发者明确知道应用程序中事务的开始和结束。

5、session不是线程安全的

https://segmentfault.com/a/1190000018081790