## 一、angular原始指令
1、条件显示  
1.1、ng-show: 为true显示，但是即使不显示，也会加载渲染dom  
1.2、ng-hide: 如上  
1.3、ng-if：如ng-show，不显示的时候不会加载或者销毁掉  
1.4、ng-switch：控制哪一部分显示  

## 二、angular中的scope
1、ng-app是一个应用启动AngularJS的入口点，在这里也会创建一个root scope，在controller里可以通过$rootScope调到，每个应用只能有一个root scope（当然了～root嘛～），但它会有多个child scope。  
2、子scope是可以访问父scope上绑定的所有model和function的。

3、以下指令会创建新的 scope, 并且会在原型上继承 父scope (即$scope.$parent, 下文两个词互为同义词):  
ng-repeat  
ng-switch  
ng-view  
ng-controller  
带有 scope: true 的指令  
带有 transclude: true 的指令  

4、以下指令创建新的指令, 且在原型上 不继承 父scope:  
带有 scope: { ... } 的指令, 这会创建一个 独立的scope (isolate scope)  
注意: 默认指令并不会创建 scope, 默认是 scope: false, 通常称之为 共享scope.

## 三、angular.module(‘com.ngbook.demo’, [])
1、关于module函数可以传递3个参数，它们分别为：  
name：模块定义的名称，它应该是一个唯一的必选参数，它会在后边被其他模块注入或者是在ngAPP指令中声明应用程序主模块；  
requires：模块的依赖，它是声明本模块需要依赖的其他模块的参数。特别注意：如果在这里没有声明模块的依赖，则我们是无法在模块中使用依赖模块的任何组件的；它是个可选参数。  
configFn： 模块的启动配置函数，在angular config阶段会调用该函数，对模块中的组件进行实例化对象实例之前的特定配置，如我们常见的对$routeProvider配置应用程序的路由信息。它等同于”module.config“函数，建议用”module.config“函数替换它。这也是个可选参数。  

2、使用angular.module方法  
我们常用的方式有有种，分别为angular.module(‘xx’, [可选依赖])和angular.module(‘xx’)，请注意它是完全不同的方式：  
angular.module(‘xx’, [可选依赖]) —— 是声明创建module，  
angular.module(‘xx’)是获取已经声明了的module。  
在应用程序中，对module的声明应该有且只有一次；对于获取module，则可以有多次。  
推荐将angular组件独立分离在不同的文件中，module文件中声明module，其他组件则引入module  
需要注意的是在打包或者script方式引入的时候，我们需要首先加载module声明文件，然后才能加载其他组件模块。

## 四、require中的define
1、简单的键值对  
该模块没有任何依赖且不需要定义相关逻辑，只是一些属性的定义，则这样定义：  
```js
define({
    color: "black",
    size: "unisize"
});
2、没有依赖，但是需要有相关的逻辑处理
define(function () {
    //Do setup work here

    return {
        color: "black",
        size: "unisize"
    }
});
```
3、有依赖也有相关的逻辑处理

## 五、数据绑定
1、单向数据绑定  
ng-bind 单向数据绑定（$scope -> view），用于数据显示，简写形式是 {{}}。  
两者的区别在于页面没有加载完毕 {{val}} 会直接显示到页面，直到 Angular 渲染该绑定数据（这种行为有可能将 {{val}} 让用户看到）；而 ng-bind 则是在 Angular 渲染完毕后将数据显示。  
2、双向数据绑定  
ng-model 是双向数据绑定（$scope -> view and view -> $scope），用于绑定值会变化的表单元素等。

## 六、监听
1、$watch  
$watch是一个scope函数，用于监听模型变化，当你的模型部分发生变化时它会通知你。  
$watch(watchExpression, listener, objectEquality)，每个参数的说明如下：  
watchExpression：监听的对象，它可以是一个angular表达式如‘name‘,或函数如function(){return $scope.name}。  
listener: 当watchExpression变化时会被调用的函数或者表达式,它接收3个参数：newValue(新值), oldValue(旧值), scope(作用域的引用)
objectEquality：是否深度监听，如果设置为true,它告诉Angular检查所监控的对象中每一个属性的变化. 如果你希望监控数组的个别元素或者对象的属性而不是一个普通的值, 那么你应该使用它  
return：一个函数，如果想要停止监听，则可以直接调用该函数。

2、$watchGroup(watchExpressions, listener);  
使用$watchGroup可同时监听多个变量，如果任一一个变量发生变化就会触发listener。

## 七、注意
ng-if是是对控件的销毁和创建，会影响到数据的保持。

