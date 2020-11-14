# 什么是 Node.js 

- Node.js 是一个基于 Chrome V8 引擎的 JavaScript 运行环境。
  - 在 Node.js 里写 JS 和在 Chrome 里写 JS ，**几乎没有不一样！**
  - Node.js 没有浏览器 API ，即 document ，window 等。
  - 加了许多 Node.js API 。
  - 你在 Chrome 里写 JavaScript 控制**浏览器。**
  - Node.js 让你用类似的方式，控制**整个计算机。**
- Node.js 使用了一个事件驱动、非阻塞式 I/O 的模型，使其轻量又高效。
  - 搜索引擎优化 + 首屏速度优化 = 服务端渲染
  - 服务端渲染 + 前后端同构 = Node.js



# BFF 层

- BFF全称是Backends For Frontends(服务于前端的后端)
  - 对用户侧提供 HTTP 服务
  - 使用后端 RPC 服务
- 简而言之，BFF就是服务器设计API时会考虑到不同设备的需求，也就是为不同的设备提供不同的API接口，虽然它们可能是实现相同的功能，但因为不同设备的特殊性，它们对服务端的API访问也各有其特点，需要区别处理
- BFF层的维护成本确实比较高，所以一般都是web端和app端使用相同的业务接口





# Node.js 的非阻塞 I/O

- 阻塞 I/O 和非阻塞 I/O 的区别就在于系统接收输入再到输出期间，能不能接收其他输入。



# Node.js 的事件循环

![img](https://user-gold-cdn.xitu.io/2019/11/18/16e7d83cb93ba59e?imageslim)

- 这个图是整个 Node.js 的运行原理，从左到右，从上到下，Node.js 被分为了四层，分别是 `应用层`、`V8引擎层`、`Node API层` 和 `LIBUV层`。

  > - 应用层：   即 JavaScript 交互层，常见的就是 Node.js 的模块，比如 http，fs
  > - V8引擎层：  即利用 V8 引擎来解析JavaScript 语法，进而和下层 API 交互
  > - NodeAPI层：  为上层模块提供系统调用，一般是由 C 语言来实现，和操作系统进行交互 。
  > - LIBUV层： 是跨平台的底层封装，实现了 事件循环、文件操作等，是 Node.js 实现异步的核心 。





# Node.js 异步编程 – async/await

- async function 是 Promise 的语法糖封装

- 异步编程的终极方案 – 以同步的方式写异步

- await 关键字可以“暂停”async function的执行
- await 关键字可以以同步的写法获取 Promise 的执行结果
- try-catch 可以获取 await 所得到的错误





# HTTP 服务框架：Koa

- Koa.js 最为人所知的是基于 `洋葱模型` 的HTTP中间件处理流程。

  在此，洋葱模式可以拆解成一下几个元素。

  - 生命周期
  - 中间件
  - 中间件在生命周期中
    - 前置操作
    - 等待其他中间件操作
    - 后置操作

- 核心功能：

  - 比 Express 更极致的 request/response 简化

    - ctx.status = 200
    - ctx.body = 'hello world'

  - 使用 async function 实现的中间件

    - 有“暂停执行”的能力
    - 在异步的情况下也符合洋葱模型

  - 精简内核，所有额外功能都移到中间件里实现

  - ```javascript
    const fs = require('fs');
    const game = require('./game')
    const koa = require('koa');
    const mount = require('koa-mount')
    
    // 玩家胜利次数，如果超过3，则后续往该服务器的请求都返回500
    var playerWinCount = 0
    // 玩家的上一次游戏动作
    var lastPlayerAction = null;
    // 玩家连续出同一个动作的次数
    var sameCount = 0;
    
    const app = new koa();
    
    app.use(
        mount('/favicon.ico', function (ctx) {
            // koa比express做了更极致的response处理函数
            // 因为koa使用异步函数作为中间件的实现方式
            // 所以koa可以在等待所有中间件执行完毕之后再统一处理返回值，因此可以用赋值运算符
            ctx.status = 200;
        })
    )
    
    const gameKoa = new koa();
    app.use(
        mount('/game', gameKoa)
    )
    gameKoa.use(
        async function (ctx, next) {
            if (playerWinCount >= 3) {
                ctx.status = 500;
                ctx.body = '我不会再玩了！'
                return;
            }
    
            // 使用await 关键字等待后续中间件执行完成
            await next();
    
            // 就能获得一个准确的洋葱模型效果
            if (ctx.playerWon) {
                playerWinCount++;
            }
        }
    )
    gameKoa.use(
        async function (ctx, next) {
            const query = ctx.query;
            const playerAction = query.action;
            if (!playerAction) {
                ctx.status = 400;
                return;
            }
            if (sameCount == 9) {
                ctx.status = 500;
                ctx.body = '我不会再玩了！'
            }
    
            if (lastPlayerAction == playerAction) {
                sameCount++
                if (sameCount >= 3) {
                    ctx.status = 400;
                    ctx.body = '你作弊！我再也不玩了'
                    sameCount = 9
                    return;
                }
    
            } else {
                sameCount = 0;
            }
            lastPlayerAction = playerAction;
            ctx.playerAction = playerAction
            await next();
        }
    )
    gameKoa.use(
        async function (ctx, next) {
            const playerAction = ctx.playerAction;
            const result = game(playerAction);
    
            // 对于一定需要在请求主流程里完成的操作，一定要使用await进行等待
            // 否则koa就会在当前事件循环就把http response返回出去了
            await new Promise(resolve => {
                setTimeout(() => {
                    ctx.status = 200;
                    if (result == 0) {
                        ctx.body = '平局'
    
                    } else if (result == -1) {
                        ctx.body = '你输了'
    
                    } else {
                        ctx.body = '你赢了'
                        ctx.playerWon = true;
    
                    }
                    resolve();
                }, 500)
            })
        }
    )
    
    app.use(
        mount('/', function (ctx) {
            ctx.body = fs.readFileSync(__dirname + '/index.html', 'utf-8')
        })
    )
    app.listen(3000);
    ```

    





# RPC 调用

- 二进制协议
  - 更小的数据包体积
  - 更快的编解码速率

- Node.js Buffer 编解码二进制数据包

- Protocol Buffer

  - Google 研发的二进制协议编解码库

    - 通过协议文件控制 Buffer 的格式
    - 更直观
    - 更好维护
    - 更便于合作

  - ```protobuf
    message Course {
        required float id = 1;
        required string name = 2;
        repeated Lesson lesson = 3;
    }
    
    message Lesson {
        required float id = 1;
        required string title = 2;
    }
    ```

- Node.js net 搭建多路复用的 RPC 通道

  - 单工/半双工的通信通道搭建
  
    













