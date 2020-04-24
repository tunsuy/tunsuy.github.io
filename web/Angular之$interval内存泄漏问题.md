## 一、问题背景
测试发现后台在不停的接收到请求，但是没人在不断的刷新console，只是在做一些常规的操作，
于是打开chrome的F12查看，发现network在不断的发送请求。为啥会出现这种情况呢？
首先想到的是我们的定时刷新器有问题：之前做了这样一个功能，在界面上有进行中的任务时，则定时刷新更新状态。
但是现在的问题是，为啥界面上已经没有了进行中的任务，界面还在不停的刷新呢。

于是开始了下面的定位旅程

## 二、定位过程
### 1、检查是否取消timer
在scope的destroy时，我们一般都是需要将定时器等相关操作给取消掉的，检查destroy如下：
```js
$scope.$on('$destroy', function () {
	if (timer) {
		$interval.cancel(timer);
	}
});
```
发现是取消了的，那为啥timer还存在呢

### 2、打断点调试
通过chrome里面断点调试，发现复现不了

### 3、打印日志
通过在关键地方console.log()打印日志发现，创建timer是比较耗时，在tab界面切换出去再切换回来的时候，之前的timer才创建成功，
但是已经没有变量引用它了，导致出现内存泄漏。之前创建timer的代码如下：
```js
if (timer === undefined) {
	timer = $interval($scope.backup_table.autoRefresh, autoRefreshInterval, -1);
}
```
通过这个代码我们可以发现，确实会存在内存泄漏的问题。

既然基本确定了根因，那么就来复现一次
### 4、问题复现
打开console，创建几个进行中的任务，不断的切换tab，观察到请求确实在不停的刷新（没有按照我们设定的定时时间刷新）以及日志显示timer的创建和取消没有一一对应。

## 三、问题修改
问题根因已经确定，那么怎么修改该问题呢？
既然我们无法预知timer创建完成的时机，那么我们就得使用一个全局数组变量来接收所有的timer，在destroy或者所有任务完成的情况下取消所有的timer，并且时刻监听该数组的变化，如果数组中的timer数大于1，则取消掉之前所有的timer，保证不管怎么频繁切换tab都只会有一个timer在执行。伪代码如下：
```js
if ($rootScope.backupListTimers.length === 0) {
	$rootScope.backupListTimers.push(
		$interval($scope.backup_table.autoRefresh, autoRefreshInterval, -1)
	);
}

...

$scope.$on('$destroy', function () {
	if ($rootScope.backupListTimers.length > 0) {
		_.each($rootScope.backupListTimers, function(timer) {
			$interval.cancel(timer);
		});
		$rootScope.backupListTimers = [];
	}
});

...

$scope.$watch("backupListTimers.length", function (newValue) {
	if (newValue <= 1) {
		return;
	}
	$rootScope.backupListTimers = _.sortBy($rootScope.backupListTimers, function(timer) {
		return -timer.$$intervalId;
	});
	_.each($rootScope.backupListTimers, function(timer, index) {
		if (index === 0) {
			return;
		}
		$interval.cancel(timer);
	})
})
```

