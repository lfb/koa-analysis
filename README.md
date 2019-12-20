# koa-analysis
Koa.js 源码分析

### 目录结构
```text
.
├── LICENSE
├── README.md
├── app.js              实例代码
├── application.js      入口文件
├── compose.js          合并中间件
├── context.js          上下文对象
├── request.js          请求对象
└── response.js         响应对象
```

### 一切从这三段代码说起
```JavaScript
const Koa = require('koa')
const app = new Koa()

app.use(async (ctx, next) => {
  ctx.body = 'Hello, Koa'
})

app.listen(3000)
```

大家都知道，Koa 是 Node.js 的一个框架，Koa 主要对 Node.js 的 HTTP 模块进行再次的抽象封装，并基于 Promise 提供控制流程，它本身并不包含任何的附加功能，但是允许用户可以自由组合中间件，从而让 web 应用开发变得更加可控和富有表现力。

Koa 不仅代码精简，且使用起来也很简单，只需要 引入 `koa` 模块使用 new 操作符就可以实例化一个 Koa 对象了， 那么思考一下使用 `new Koa()` 到达做了些什么事情呢？为什么使用了 `use()` 方法传入一个函数就叫中间件函数呢？为什么使用 `listen()` 方法监听一个端口就可以开启一个 HTTP 服务了呢？带着这些的问题，先简单的聊一聊 Koa 的生命周期。

首先在入口文件的 `application.js` 文件里是暴露导出了一个 Application 类，使用导出的类使用 new 操作符调用，就会得到了一个 Koa 实例化对象，Application 类的构造函数 ( constructor ) 会进行一系列的初始化工作，其中比较重要的有 中间件 middleware 数组，context、request、response对象，和use、listen等方法，这就完成了 `new Koa()`的工作。
![koa-life-cycle-constructor](https://cdn.boblog.com/koa-life-cycle-constructor.png)

然后得到了 Koa的实例对象，我们命名为 app，这个 app 实例对象会拥有 Koa 类的所有方法，其中有个方法叫 `use()`，这个 `use(fn)` 方法接收一个 fn 函数的参数，这个方法主要做的工作就是把传入的函数新增到（ push ）到中间件数组 ( middleware ) 中，然后 return this，可以保持链式调用。

![koa-life-cycle-app-use](https://cdn.boblog.com/koa-life-cycle-app-use.png)

还一个 `listen()` 的方法，这个方法主要的工作做了2个：第一、使用了 Node.js 原生的 HTTP 模块创建了一个服务器，第二、在创建服务器函数里，传入一个 `this.callback()` 方法，这个 `this.callback()` 方法做了大量的工作，比如处理中间件，创建上下文，处理请求对象，响应请求，返回数据等工作。下面来详细聊聊 callback 方法做了具体的内容。

`this.callback()` 方法第一步是使用 `koa-compose` 中间件的 compose  函数进行把多个中间件函数合并成为一个大的中间件函数，然后进行创建一个上下文对象，开始处理请求、执行中间件函数、最后响应请求返回数据，整个生命周期的流程就结束了。下面的简单描述了Koa生命周期的流程图：


![koa-life-cycle](https://cdn.boblog.com/koa-life-cycle.png)

## Application 类

application.js 里面导出了一个 `Application`类，这个类继承了Node.js 的 events 模块 `Emitter`，所以拥有了事件系统的能力，`Application` 类里面的 constructor 构造函数进行了一系列的初始化，列出了较重要的属性：
- middleware 中间件数组，下面用来装使用use传入的函数
- context 上下文对象，继承从 context.js 文件导出的对象
- request 请求对象，继承从 request.js 文件导出的对象
- response 响应对象，继承从 response.js 文件导出的对象
```JavaScript
/**
 * 暴露一个 Application 类
 * 继承了 Emitter，拥有了事件系统的能力
 */
module.exports = class Application extends Emitter {
  constructor() {
    // 调用父类 Emitter，拥有了事件系统的能力
    super();
    // 中间件数组
    this.middleware = [];
    // 上下文对象，继承从 context.js 文件导出的对象
    this.context = Object.create(context);
    // 请求对象，继承从 request.js 文件导出的对象
    this.request = Object.create(request);
    // 响应对象，继承从 response.js 文件导出的对象
    this.response = Object.create(response);
  }
  
  // 主要的功能是把函数放入到中间件数组中
  use(fn) {
    
  }
  
  /** 主要的功能是创建一个 HTTP 服务器
   * 且创建服务器同时传入一个 callback 函数
   */
  listen(...args) {
  }
  
  /* 处理请求 */
  handleRequest(ctx, fnMiddleware) {
  }
  
  /* 创建一个上下文对象 */
  createContext(req, res) {
  }
}

/* 处理响应数据 */
function respond(ctx) {
}
```

其中有个知识点需要学习一下：`Object.create()`方法创建一个新对象，使用现有的对象来提供新创建的对象的\_\_proto__。它的实现原理：

```JavaScript
if (typeof Object.create !== "function") {
    Object.create = function (proto) {
        if (typeof proto !== 'object') {
            throw new TypeError('需要传入对象');
        }

        function F() {}
        F.prototype = proto;
        
        return new F();
    };
}
```


## use(fn) 方法

use 方法开始是对 fn 参数做了一些判断，比如：
- 首先判断传入的 fn 参数是否为一个函数，如果不是函数，则抛出错误。
- 然后判断传入的 fn 参数是否是一个 Generator 函数，如果是则使用 convert 方法转化为返回一个 promise 函数。

```JavaScript
use(fn) {
  // 首先判断传入的 fn 参数是否为一个函数，如果不是函数，则抛出错误。
  if (typeof fn !== 'function') throw new TypeError('middleware must be a function!');
  
  // 然后判断传入的 fn 参数是否是一个 Generator 函数
  // 如果是则使用 convert 方法转化为返回一个 promise 函数。
  if (isGeneratorFunction(fn)) {
    deprecate('Support for generators will be removed in v3. ' +
              'See the documentation for examples of how to convert old middleware ' +
              'https://github.com/koajs/koa/blob/master/docs/migration.md');
    fn = convert(fn);
  }
  debug('use %s', fn._name || fn.name || '-');
  
  // 把传入的函数新增到中间件数组中
  this.middleware.push(fn);
  // 保持 use 方法的链式调用
  return this;
}
```

最后，我们只需要知道 `use()` 的方法最重要的是做了 2 件事情：
- 把传入的函数新增到中间件数组中：`this.middleware.push(fn)`
- 保持 use 方法的链式调用：`return this`

```JavaScript
use(fn) {
  // 把传入的函数新增到中间件数组中
  this.middleware.push(fn)
  // 保持 use 方法的链式调用
  return this
}
```

## listen(...args) 方法
```js
listen(...args) {
  debug('listen');
  // 创建一个服务器
  const server = http.createServer(this.callback());
  // 开始服务器监听
  return server.listen(...args);
}
```
 由以上代码我们可以轻易的看出 listen 方法里面做的事情就是调用了 Node.js 的 HTTP 模块创建了一个服务器，然后进行监听传入的 `...args` 参数。这就是为什么使用 listen 方法传入一个端口号就能启动一个服务的原因了。
 
 
 其中，我们注意到，在创建一个服务器里传入了一个 `this.callback()` 方法，这个 callback 方法非常关键，在这个方法里面做了很多事情，比如中间件处理，从接收到请求开始处理，到响应数据完毕的整个流程都在这里面完成了，那么我们来好好分析这个 callback 是何方神圣。
 
 ## callback() 方法
首先 callback 方法进入的第一行代码是使用一个 compose 方法来处理中间件数组，那么我们思考 2 个问题：
1. compose 方法是什么，有什么用？
2. compose 方法如何处理中间件的？
 ```JavaScript
callback() {
  const fn = compose(this.middleware);

  // ...
}
 ```
 
 #### compose 方法
 compose 方法其实是一个 Koa 的一个中间件`koa-compose`，这个中间件的源码在当前项目中的：`koa-compose.js`，里面就暴露了一个方法：`compose`，在使用 compose 方法处理时会进行一个判断：
 1. 判断 middleware 参数必须是一个栈数组（先进后出）
 2. 判断 middleware 数组里面的每一项确保都是函数

```JavaScript
module.exports = compose

function compose(middleware) {
  // middleware 必须是一个栈数组
  if (!Array.isArray(middleware)) throw new TypeError('Middleware stack must be an array!')
  // middleware 栈数组的每一项必须是函数
  for (const fn of middleware) {
    if (typeof fn !== 'function') throw new TypeError('Middleware must be composed of functions!')
  }
  // 核心代码...
}
```

**请继续看核心内容：敲黑板，重点！重点！重点来了！**

```JavaScript
function compose(middleware) {
  // 判断代码...
  
  return function (context, next) {
    let index = -1
    return dispatch(0)

    function dispatch(i) {
      if (i <= index) return Promise.reject(new Error('next() called multiple times'))
      index = i
      let fn = middleware[i]
      if (i === middleware.length) fn = next
      if (!fn) return Promise.resolve()
      try {
        return Promise.resolve(fn(context, dispatch.bind(null, i + 1)));
      } catch (err) {
        return Promise.reject(err)
      }
    }
  }
}
 ```
 
 ## 未定稿，持续更稿子中...