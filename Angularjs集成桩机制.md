## 一、桩数据框架Canjs

Canjs库下载地址：
https://v2.canjs.com/download.html

文档地址：
https://v2.canjs.com/docs/can.fixture.html


主要引入两个文件：  
Can.js、fixture.js

## 二、Fixture简介

1、can.fixture( url, toUrl )

2、can.fixture( url, handler(request, response, requestHeaders) )

3、can.fixture(fixtures)  
fixtures {Object<url,requestHandler(request, response, requestHeaders) | String>}

注：Request请求必须是can.ajax, jQuery.ajax,or jQuery.get类型的请求

4、关闭桩的几种方式  
I、`can.fixture("GET todos.json", null)`

II、`can.fixture.on = false`

III、```js
can.ajax({
    type: 'POST',
    url: '/foo',
    fixture: false
});```

## 三、Angularjs和requirejs的区别

Angularjs是一个js框架，用于具体业务实现，AngularJS确保了当你的应用在运行时，所有的代码已经加载完毕，并且可用。AngularJS主要通过依赖注入完成这项工作。

requirejs是模块加载器，用于代码组织，构建。

比如：  
Tinyui中就集成了angularjs  
Vbs就使用了requirejs来进行模块的加载  
```html
<script type="text/javascript" src="./lib/require.js"></script>
<script type="text/javascript" src="lib/tiny/tiny.min.js"></script>
<script type="text/javascript" src="main.js"></script>
```

## 四、桩的作用

1、性能测试  
构造大量的桩数据response

2、延迟测试  
在桩function中设置延迟：
```js
can.fixture( "/foobar.json", function(request, response){
  setTimeout(function() {
    response({ foo: "bar" });
  }, 1000);
})```

3.。。。

## 五、Require.js中的define介绍：
定义一个模块，依赖注入。。。，加载完成之后，执行定义function，  
在需要桩数据的地方就要在这个模块中注入fixture

比如：  
```js
define(["app/business/common/services/timeService",
    "app/business/common/services/validatorService",
    "fixtures/bks/bksBackupFixture"
], function (timeService){。。。}
```

## 六、项目增加桩机制的步骤：
1、下载canjs库

2、将can.js、fixture.js文件添加进项目中

3、导入can.js、fixture.js

4、编写fixture文件

5、在需要请求的文件模块中注入该fixture

