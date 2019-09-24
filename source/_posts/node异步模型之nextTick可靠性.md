---
title: node异步模型之nextTick可靠性
date: 2019-09-20 15:57:29
tags: node eventloop nextTick
---

### node异步模型之nextTick

#### eventloop闹明白了？

> 我觉着你可能没明白，当然如果你真的觉得你明白了就别往下看了 😁

提到eventloop，很多都会想到setTimeout、setInterval、Promise这些词，在node里面还有专属的一些异步API

[event loop直通车](https://nodejs.org/zh-cn/docs/guides/event-loop-timers-and-nexttick/#process-nexttick)
看完这里的说明，有一句话是这样说的

当 `Node.js` 启动后，它会初始化事件轮询；处理已提供的输入脚本（或丢入 `REPL`，本文不涉及到），它可能会调用一些异步的 `API` 函数调用，安排任务处理事件，或者调用 `process.nextTick()`，然后开始处理事件循环。

下面的图表显示了事件循环的概述以及操作顺序。

![event loop机制](http://schacker.lijundong.com/WX20190923-170015.png)

> 每个阶段都有一个 `FIFO` 队列来执行回调。虽然每个阶段都是特殊的，但通常情况下，当事件循环进入给定的阶段时，它将执行特定于该阶段的任何操作，然后在该阶段的队列中执行回调，直到队列用尽或最大回调数已执行。当该#`队列已用尽或达到回调限制`#，事件循环将移动到下一阶段，等等。
> 由于这些操作中的任何一个都可能计划 更多的 操作，并且在 轮询 阶段处理的新事件由内核排队，因此在处理轮询事件时，轮询事件可以排队。因此，长时间运行回调可以允许轮询阶段运行大量长于计时器的阈值。有关详细信息，请参阅 计时器 和 轮询 部分。

申明，本文目的不是给大家讲明白`event loop`，论讲理论上面官网比我讲得好多了，本文只是证明一个结论-`process.nextTick`其实没有想象中靠谱，不靠谱来源于上面的一句话 "当该#`队列已用尽或达到回调限制`#，事件循环将移动到下一阶段"

怎么证明？

```ts
const Koa = require('koa');
const app = new Koa();

new Array(100000).fill('').map((v, i) => {
  app.use((ctx, next) => {
    new Array(100).fill(i).map((_v, _i) => {
      process.nextTick(() => {
          ctx.body = i
      })
    })
    next()
  })
})
app.listen(3000);
```

上面启动了一个简单的`node`服务，服务中注册了`100000`个中间件，每个中间件里做了相同的事情，即创建长度为100的数组，内层数组用`nextTick`对`ctx.body`进行响应，响应值为外层数组下标值

哇好激动，见证奇迹的时刻，如果你忽略上面这句话，那结果就是`99999`，因为上面的中间件逻辑是连续的，只有`process.nextTick`是异步API，所以按照主执行栈、微任务、宏任务的顺序来看，我们最后的`ctx.body`响应结果就是`99999`，那到底是不是？不一定

![nextTick-100000](http://schacker.lijundong.com/WX20190924-101436.png)

结果显示`2327`

那这个`2327`怎么来的，`V8`和`OS`的调度来的，那他咋调度的，这我哪知道啊，那调整下中间件个数会怎么样呢？我们调整到`10000`个中间件

笔者这里测试结果一样是`2327`，但出现过一次`3796`

![nextTick-10000](http://schacker.lijundong.com/WX20190924-104759.png)

所以大家通常用`nextTick`的时候，你并不能保证`nextTick`的执行结果是正确的。

最后再来一发[event loop直通车](https://nodejs.org/zh-cn/docs/guides/event-loop-timers-and-nexttick/#process-nexttick)
