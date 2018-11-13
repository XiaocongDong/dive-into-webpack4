# Tapable
Webpack实现插件机制的主心骨是[Tapable](!https://github.com/webpack/tapable)库，在Tapable库里面主要的一个类是Hook类，中文名可以译作钩子。不同的插件（实际上是各个回调函数）可以tap在钩子上面，当这个钩子被执行的时候（hook.call(xxx)），所有被tapped的回调函数都会被执行。

一个Hook上面的回调函数有两种执行方式，一种是同步执行（Sync），另外一种是异步执行（Async）。无论何种类型的钩子实际上都是继承了钩子基类（Hook）。所有钩子在初始化时接收的参数都是一样的，它们只接受一个可选的options数组，这个数组是钩子接收的所有参数名字符串。

钩子会编译出运行所有回调函数的实际执行函数，生成最终的执行函数由以下几个方面的内容决定的：
* 注册的plugin的数目
* 注册的插件的类型
* 调用钩子的方法（sync, async, 和 promise）
* 传参的个数
* 是否使用了interception

## Sync
对于同步执行挂载的回调函数的钩子又有以下几种类型
### SyncHook
串行同步执行Tapped的回调函数

### SyncBailHook
同步串行执行挂载的回调函数，和SyncHook不一样的是，如果任何回调函数有返回值，则证明回调函数执行的过程中出了问题，剩下的回调函数都不会被执行了。

### SyncWaterfallHook
串行执行Tapped的回调函数，和SyncBailHook的区别是它将上一个回调函数执行的结果作为参数传给下一个回调函数。

### SyncLoopHook
Loop Hook还没有被使用

## Async
对于异步执行的钩子有以下几种类型
### AsyncParallelHook
Async-parallel 的钩子可以挂载同步方法，回调函数的方法（callback-based），promise-based方法，可以使用以下方法来进行回调函数的挂载`myTap.tap()`，`myTap.tapAsync()`，`myTap.tapPromise()`。对于异步方法的执行顺序是并行执行。
### AsyncParallelBailHook
### AsyncSeriesHook
串行执行
### AsyncSeriesBailHook
### AsyncSeriesWaterfallHook
串行执行，上一个回调函数的结果传给下一个。

## Interception
所有的Hook都支持额外的截击方法（interception method）：
```javascript
myCar.hooks.calculateRoutes.intercept({
  call: (source, target, routesList) => {
    console.log('Starting to calculate routes')
  },
  register: (tapInfo) => {
    // tapInfo = {type: "promise", name: "GoogleMapsPlugin", fn:...}
    console.log(`${tapInfo.name} is doing it's job`)
    return tapInfo
  }
})
```