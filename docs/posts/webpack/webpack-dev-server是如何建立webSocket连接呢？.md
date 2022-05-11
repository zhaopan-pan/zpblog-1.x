---
sidebar: false
date: "2022-5-5"
tag: webpack
category: 
 - frontEnd  
title: 和webpack-dev-server建立webSocket连接发生了什么？
---

[pre]:./%E7%BB%99%E5%85%AC%E5%8F%B8%E9%A1%B9%E7%9B%AE%E5%8A%A0%E4%B8%8A%E7%83%AD%E6%9B%B4%E6%96%B0%E5%8A%9F%E8%83%BD.md
[ws-npm]:https://www.npmjs.com/package/ws

## 背景

[上一篇][pre]主要是为了解决热更新的问题，通过解决问题，又引申出了另一个问题，就是webpack-dev-server是如何建立webSocket连接呢？

## 创建webSocket服务端以及和dev服务的关系

webSocket的连接也是需要一个服务端，这个在wds创建dev服务成功之后创建的，这里为啥要放在开发服务创建完成后呢？因为后面要通过dev服务来

> node_modules/webpack-dev-server/lib/Server.js 

```js
class Server {
    // line:774
    listen(port, hostname, fn) {
        this.hostname = hostname;

        return this.listeningApp.listen(port, hostname, (err) => {
            // 创建websocket服务
            this.createSocketServer();

            ...

        });
    }
}
```

`this.createSocketServer()` 方法其实调用的一般是下面的这个文件，也就是用[ws](ws-npm)来创建socket服务的库，原理是通过监听dev服务的 `upgrade` 事件，来判断是否是HTTP升级请求，如果是[ws](ws-npm)会对协议进行升级处理，升级完成后webSocket的服务端也就创建成功了，此时已经可以接收客户端的连接请求了

> /node_modules/webpack-dev-server/lib/servers/WebsocketServer.js

```js
const ws = require('ws');
const BaseServer = require('./BaseServer');

module.exports = class WebsocketServer extends BaseServer {
    constructor(server) {
        super(server);
        this.wsServer = new ws.Server({
            noServer: true,
            path: this.server.sockPath,
        });
        // this.server是前面已经创建的dev服务
        this.server.listeningApp.on('upgrade', (req, sock, head) => {
            if (!this.wsServer.shouldHandle(req)) {
                return;
            }

            this.wsServer.handleUpgrade(req, sock, head, (connection) => {
                this.wsServer.emit('connection', connection, req);
            });
        });
    }
}
```

这里面有个小插曲，除了上面提到的[ws](ws-npm)，通过 `options.transportMode.server='sockjs'` 配置，wds会用[sockjs](https://www.npmjs.com/package/sockjs)来创建socket服务的，当然了，相对应的浏览器端也会用[sockjs-client](https://www.npmjs.com/package/sockjs-client)来做连接，好处是可以对于不支持webSocket协议的浏览器可以降级为轮询处理，如果要解决浏览器兼容问题，可以用这个

## socket地址是如何生成的呢？

从配置开始看，有个很重要的点，在webpack的配置参数里面 `entry` , 对于启用热更新是很重要的，[官方文档的指南](https://webpack.docschina.org/guides/hot-module-replacement/#enabling-hmr)里面就有，其实[上一篇][pre]的端口错误也就是影响到这里的配置了，正常情况应该是 `webpack-dev-server/client?http://localhost:3001` ， 我们会在输出文件里面可以看到它的踪影，后面会说到

![start](./img/fixHotUpdate-entry.png "图片地址")

这里要有两个概念，一个是服务端，一个是客户端，服务端就是项目webpack自带的 `wds` (有些项目自研脚手架里面自己搭建node服务，用到 `webpack-dev-middleware` ，其实 `wds` 也是依赖 `wdm` )，就是我们的开发者服务，watch文件变化、和客户端通信、代理请求、自动打开浏览器等等功能后依赖他完成，客户端是我们的浏览器端，项目启动， `webpack` 在编译完成后，会启动 `wds` 服务，看源码吧

> node_modules/webpack-dev-server/lib/Server.js 

```js
class Server {
    constructor(compiler, options = {}, _log) {
        ...
        // line:72
        updateCompiler(this.compiler, this.options);

        ...
    }
```

在初始化的时候会调用上面的 `updateCompiler` 方法，这个方法里面会传入webpack编译后的 `compiler` 对象, 

> node_modules/webpack-dev-server/lib/utils/updateCompiler.js 

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

通过 `addEntries` 修改的 `entry` 配置, 并把 `webpack-dev-server/client/index.js` 添加到 `entry` 配置中，然后调用 `compiler.hooks.entryOption.call(config.context, config.entry);` 使之生效。
这里面还有个很重要的点就是调用了 `webpack.ProvidePlugin()` 方法，向webpack注入了 `__webpack_dev_server_client__` , 类似于全局变量，这个变量名是通过 `getSocketClientPath()` 来引用 `WebsocketClient.js` 的，有木有很眼熟，对，就是[上一篇][pre]实例化websocket报错的那个文件，后面在客户端会用到这个文件

> /node_modules/webpack-dev-server/client/clients/WebsocketClient.js

```js
module.exports = /*#__PURE__*/ function(_BaseClient) {
    _inherits(WebsocketClient, _BaseClient);

    var _super = _createSuper(WebsocketClient);

    function WebsocketClient(url) {
        var _this;

        _classCallCheck(this, WebsocketClient);

        _this = _super.call(this);
        _this.client = new WebSocket(url.replace(/^http/, 'ws'));

        _this.client.onerror = function(err) { // TODO: use logger to log the error event once client and client-src
            // are reorganized to have the same directory structure
        };

        return _this;
    }
}
```

wds拉起浏览器，让浏览器请求 `http://localhost:3001/`

可以看到html里面会加载三个bundle，前两个是根据splitChunks配置生成的， `entry.bundle.js` 是根据 `entry` 配置生成的

* `runtime.bundle.js` 是webpack的引导程序，管理着各个chunk，
* `vender.chunk.js` 是项目打包后node_modules代码
* `entry.bundle.js` 主要是项目代码， 

![start](./img/fixHotUpdate-bundles.jpg "图片地址")

  
`entry.bundle.js` 中除了项目代码还有新发现，webpack在通过 `__webpack_require__` 来引入 `client.js` , 而 `webpack-dev-server/client/index.js?http://localhost:3001"` , 这个路径就是上面配置的entry啊，这就解释了webpack是如何把 `client.js` 导入到浏览器端的，有了 `client.js` ，浏览器才算是真正的客户端，才可以通过websocket和wds进行交互，下面看看 `client.js`

![fixHotUpdate-entry-bundle-js](./img/fixHotUpdate-entry-bundle-js.png "图片地址")

## 客户端如何建立连接

`createSocketUrl` , 终于看到url的庐山真面目了，这个方法就是用来生成websocket的url的。

简单介绍下这个新名词[__resourceQuery](https://webpack.docschina.org/api/module-variables/#__resourcequery-webpack-specific)，他是webpack的一个变量，比如当引入一个文件时， `file.js?test` , `file.js` 是文件名，此时在 `file.js` 中就可以取到 `__resourceQuery` 而 `__resourceQuery=?test` ， 所以 `?` 后面的参数就是给文件传入得信息。

同理[上面](#socket%E5%9C%B0%E5%9D%80%E6%98%AF%E5%A6%82%E4%BD%95%E7%94%9F%E6%88%90%E7%9A%84%E5%91%A2)配置的 `entry` 就是 `webpack-dev-server/client?http://localhost:3001` ，这里的 `?` 后面的参数就是给 `webpack-dev-server/client` 传入得信息，所以在 `client` 中通过 `__resourceQuery` 拿到参数，就可以生成webSocket的连接地址

> node_modules/webpack-dev-server/client/index.js

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

[上一篇][pre]端口错误的就是影响到这里正确生成url了, 至于 `createSocketUrl` 方法也只是校验协议、端口、地址等信息，最后返回一个合法的url。

### 开始连接webSocket

上面拿到url之后，会调用 `socket` 方法，这里就是连接webSocket的地方，首先会拿到 `Client` , 这里会发现一个熟悉的面孔 `__webpack_dev_server_client__` ，这就对应上了上面的 `updateCompiler` 文件对应上了，会拿到之前注入的 `WebsocketClient.js` 客户端文件，然后进行实例化操作

> node_modules/webpack-dev-server/client/socket.js

```js
var Client = typeof __webpack_dev_server_client__ !== 'undefined' ? __webpack_dev_server_client__ : // eslint-disable-next-line import/no-unresolved
    require('./clients/SockJSClient');
var retries = 0;
var client = null;

var socket = function initSocket(url, handlers) {
    client = new Client(url);
    client.onOpen(function() {
        retries = 0;
    });
    client.onClose(function() {});
    client.onMessage(function(data) {
        var msg = JSON.parse(data);

        if (handlers[msg.type]) {
            handlers[msg.type](msg.data);
        }
    });
};

module.exports = socket;
```

如果地址没有问题， `webSocket` 实例创建成功后，客户端也就可以服务端进行消息交互了

## 总结

本文从以下几个方面整理了在连接 `webSocket` 的过程中发生的事情： 
1. 创建ws服务端以及和dev服务的关系
2. socket地址的生成
3. 客户端如何建立连接 

了解了上面的流程后，对于[上一篇][pre]中端口错误的问题也有了更深的认识，以及对客户端和服务端交互比较清晰了，弄明白了这些， `webpack-dev-server` 建立 `webSocket` 连接的过程也就没什么神秘了
