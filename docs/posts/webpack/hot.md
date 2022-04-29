---
sidebar: false
date: "2022-4-27"
tag: webpack
category: 
 - frontEnd  
title: 给公司项目加上热更新功能
---

### 背景

最近入职了新公司，项目用的内部自研脚手架(react+ts+webpack+gulp+koa)，其中除了前端webpack的devServer，还用koa实现了一套MVC风格的中间层服务，让我顿时来了兴趣，赶紧 `npm start` 起来，找到默认打开页面的页面，随便改几个字，控制台显示开始重新编译，可编译成功后浏览器却没动静，按理webpack的热更新会给浏览器推送最新编译后的文件信息，浏览器更新最新文件后是会自动刷新的，接着又改了几个字，编译之后浏览还是器没反应，问了其他同事，他们表示从来没用过什么热更新，是的，你没有看错，<font color='#dd0000'>公元2022年的react项目没有热更新</font>，在这个很多人都用上 `vite` 来实现瞬间热更新的年代，怎么还有手动刷新的道理，这开发体验，简直惨无人道啊(说到这里不得不佩服同事们的忍耐力)，可这样技术倒退的开发体验实在是感觉很糟糕，不能忍！

> koa服务作用:
> 1、给跨域请求当代理，
> 2、用来规范和约束接口字段ts类型
> 3、对于后端返回的数据可以做二次逻辑处理

#### 源码版本

> os：macOS
> webpack：4.43.0
> webpack-dev-server：3.11.2

### 热更新失效的原因

既然大家用了这么久都没啥意见，那就自己动手吧，常规套路，先打开控制台瞅瞅

![start](./img/fixHotUpdate-websocket-connect-fail.jpg "图片地址")

![start](./img/fixHotUpdate-WebsocketClient.jpg "图片地址")

不看不知道，一看就是问题，问题也很清晰， `WebSocket connection to 'ws://localhost:3000/sockjs-node' failed:` , 意思就是websocket在连接这个地址的失败了，原来这问题一直都在眼皮子底下啊... 从 `WebsocketClient.js` 中也可以看出来在webpackSocket实例化的时候就报错了，那就是 `ws://localhost:3000/sockjs-node` 这个地址有问题了，地址中 `ws` 是协议，，大概率是 `localhost` 后面的端口和地址有问题，这时，偷瞄了一眼项目运行的地址，

![start](./img/fixHotUpdate-project-url.png "图片地址")

好家伙! 端口不对啊，项目启动端口是 `3001` ，这边socket连接的端口是 `3000` ，都是是用的 `webpack-dev-server` (后面简称wds) 提供服务，端口不一致怎么玩，为了验证我的想法，那就现场以验证一下3001吧

![start](./img/fixHotUpdate-socket-conect-test.jpg "图片地址")

不出所料，果然是端口问题, 那就找下车祸现场吧

#### 车祸现场

因为项目脚手架是自定义的，就从参数配置的部分忘里面看，在看到调用webpack和wds的时候，发现有些不对劲了，在开启wds监听事件的时候端口加了+1, 因为项目还有启了个koa服务用的是3000端口，区分端口以防占用，这里可以理解，不过在这个地方临时+1的话，和其他webpack的配置就不统一的，看到这问题已经浮现了

![start](./img/fixHotUpdate-wpConfig.png "图片地址")

找到公共配置webpack的地方+1应该就行了，不过现在又有个问题了，socket地址的地址是怎么生成的呢？那就继续挖一挖吧

![start](./img/fixHotUpdate-wpCommonConfig.png "图片地址")

#### socket地址是如何生成的呢？

从配置开始看，有个很重要的点，在webpack的配置参数里面 `entry` , 对于启用热更新是很重要的，[官方文档的指南](https://webpack.docschina.org/guides/hot-module-replacement/#enabling-hmr)里面就有，其实前面说的端口错误也就是影响到这里的配置了，正常情况应该是 `webpack-dev-server/client?http://localhost:3001` ， 我们会在输出文件里面可以看到它的踪影，后面再说

![start](./img/fixHotUpdate-entry.png "图片地址")

这里要有两个概念，一个是服务端，一个是客户端，服务端就是项目webpack自带的 `wds` (有些项目自研脚手架里面自己搭建node服务，用到 `webpack-dev-middleware` ，其实 `wds` 也是依赖 `wdm` )，就是我们的开发者服务，watch文件变化、和客户端通信、代理请求、自动打开浏览器等等功能后依赖他完成，客户端是我们的浏览器端，项目启动， `webpack` 在编译完成后，会启动 `wds` 服务，看源码吧

> webpack-dev-server/lib/Server.js line:72

```js
class Server {
    constructor(compiler, options = {}, _log) {
        ...

        updateCompiler(this.compiler, this.options);

        ...
    }
```

在初始化的时候会调用上面的 `updateCompiler` 方法，这个方法里面会传入webpack编译后的 `compiler` 对象, 

> webpack-dev-server/lib/utils/updateCompiler.js 

```js
function updateCompiler(compiler, options) {
    ...

    // line:48-57
    addEntries(webpackConfig, options);
    compilers.forEach((compiler) => {
        const config = compiler.options;
        compiler.hooks.entryOption.call(config.context, config.entry);

        const providePlugin = new webpack.ProvidePlugin({
            __webpack_dev_server_client__: getSocketClientPath(options),
        });
        providePlugin.apply(compiler);
    });

    ...
}
```

通过 `addEntries` 修改的 `entry` 配置, 并把 `webpack-dev-server/client/index.js` 添加到 `entry` 配置中，然后调用 `compiler.hooks.entryOption.call(config.context, config.entry);` 使之生效，
这里面还有个很重要的点就是调用了 `webpack.ProvidePlugin()` 方法，向webpack注入了 `__webpack_dev_server_client__` , 类似于全局变量，这个变量名是用来引用 `webpack-dev-server/client/clients/WebsocketClient.js` 的，有木有很眼熟，对，就是上面实例化websocket报错的那个文件

看到这里，其实已经有点藕断丝连的感觉了，不过还不是很清晰，接着往下看

![start](./img/fixHotUpdate-bundles.jpg "图片地址")

wds拉起浏览器，让浏览器请求 `http://localhost:3001/`

可以看到html里面会加载三个bundle，前两个是根据splitChunks配置生成的， `entry.bundle.js` 是根据 `entry` 配置生成的

* `runtime.bundle.js` 是webpack的引导程序，管理着各个chunk，
* `vender.chunk.js` 是项目打包后node_modules代码
* `entry.bundle.js` 主要是项目代码， 

![fixHotUpdate-entry-bundle-js](./img/fixHotUpdate-entry-bundle-js.png "图片地址")

  
`entry.bundle.js` 中除了项目代码还有新发现，webpack在通过 `__webpack_require__` 来引入 `client.js` , 而 `webpack-dev-server/client/index.js?http://localhost:3001"` , 这个路径就是上面配置的entry啊，这就解释了webpack是如何把 `client.js` 导入到浏览器端的，有了 `client.js` ，浏览器才算是真正的客户端，才可以通过websocket和webpack进行交互，下面看看 `client.js`

#### client.js

`createSocketUrl` , 终于看到url的庐山真面目了，这个方法是用来生成websocket的url的，这里有个新名词[__resourceQuery](https://webpack.docschina.org/api/module-variables/#__resourcequery-webpack-specific)，他是webpack的一个全局变量，当引入当前文件时，`__resourceQuery=?http://localhost:3001`，所以端口错误的就是影响到这里正确生成url了

? 后面的参数也只是给 `webpack-dev-server/client/index.js` 传的参数而已，

> webpack-dev-server/client/index.js

```js
...
var socket = require('./socket');

// line:20
var createSocketUrl = require('./utils/createSocketUrl');
...
// line:35
var socketUrl = createSocketUrl(__resourceQuery);

...

// line:176
socket(socketUrl, onSocketMessage);
```
