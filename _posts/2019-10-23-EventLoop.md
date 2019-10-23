---
layout: post
title: "浅谈javascript的执行机制(EventLoop)"
date: 2019-10-23 
description: "前端技术"
tag: 前端技术
---   

## 关于javascript

javascript是一门单线程语言，在最新的HTML5中提出了[Web-Worker](https://developer.mozilla.org/zh-CN/docs/Web/API/Web_Workers_API/Using_web_workers)，
但javascript是单线程这一核心仍未改变。所以一切javascript版的"多线程"都是用单线程模拟出来的，一切javascript多线程都是纸老虎！


## 为什么JavaScript是单线程

JavaScript的单线程，与它的用途有关。作为浏览器脚本语言，JavaScript的主要用途是与用户互动，以及操作DOM。
这决定了它只能是单线程，否则会带来很复杂的同步问题。比如，假定JavaScript同时有两个线程，一个线程在某个DOM
节点上添加内容，另一个线程删除了这个节点，这时浏览器应该以哪个线程为准？
所以，为了避免复杂性，从一诞生，JavaScript就是单线程，这已经成了这门语言的核心特征，将来也不会改变。

为了利用多核CPU的计算能力，HTML5提出Web Worker标准，允许JavaScript脚本创建多个线程，但是子线程完全受主线程控制，
且不得操作DOM。所以，这个新标准并没有改变JavaScript单线程的本质。


## 浏览器中的Event Loop

在了解事件循环之前，我们先来猜猜下面两个代码片段的输出结果。  
**代码片段1**
```
let a = '1';
console.log(a);

let b = '2';
console.log(b);
```
**代码片段2**
```
setTimeout(function(){
    console.log('定时器开始啦')
}, 0);

new Promise(function(resolve){
    console.log('马上执行for循环啦');
    for(var i = 0; i < 10000; i++){
        i == 99 && resolve();
    }
}).then(function(){
    console.log('执行then函数啦')
});

console.log('代码执行结束');
```
因为javascript是一门单线程语言，所以我们可以得出结论：javascript是按照语句出现的顺序执行的。
因此，我相信第一个代码片段的输出结果难倒不了大家；但看到第二个代码片段，可能大家猜出的结果
都各不相同。这里先不公布正确答案，先继续往下看。

其实大家应该看出上面两个代码片段的不同之处：  
1、 第一个代码片段属于同步任务  
2、 第二个代码片段属于异步任务  
当我们打开一个网站的背后也是这两种任务在共同配合着，网页的渲染过程就是一大堆同步任务，比如页面骨架和页面元素的渲染。
而像加载图片音乐之类占用资源大耗时久的任务，就是异步任务。我们用一张导图来说明两种任务的执行机制：
![企业微信截图_15688794526597.png](https://i.loli.net/2019/10/23/kx79Mwc6QhtKNrI.png)
下面用一段代码来表述导图要表达的内容
```
let data = [];
$.ajax({
    url:www.javascript.com,
    data:data,
    success:() => {
        console.log('发送成功!');
    }
})
console.log('代码执行结束');
```
上面是一段简易的ajax请求代码：  
1、 ajax进入Event Table，注册回调函数success。  
2、 执行console.log('代码执行结束')。  
3、 ajax事件完成，回调函数success进入Event Queue。  
4、 主线程从Event Queue读取回调函数success并执行。  
相信通过上面的文字和代码，你已经对js的执行顺序有了初步了解。接下来我们来研究进阶话题：setTimeout。


## 聊聊setTimeout

对于setTimeout，我们对它的印象就是可以延迟执行代码。但用多了我们会发现明明设置了2s后执行，可实际却
是3-4s后才执行函数，咋回事呢？
先看一下这段代码：
```
setTimeout(() => {
    task()
},2000)

sleep(10000000)
```
说说上述代码是怎么执行的：  
1、 task()进入Event Table并注册,计时开始。  
2、 执行sleep函数，很慢，非常慢，计时仍在继续。（这里可以理解为我们写的很多需要同步执行的语句）  
3、 2秒到了，计时事件timeout完成，task()进入Event Queue，但是sleep也太慢了吧，还没执行完，只好等着。  
4、 sleep终于执行完了，task()终于从Event Queue进入了主线程执行。  

上述的流程走完，我们知道setTimeout这个函数，是经过指定时间后，把要执行的任务(本例中为task())加入到
Event Queue中，又因为是单线程任务要一个一个执行，如果前面的任务需要的时间太久，那么只能等着，导致真正
的延迟时间远远大于2秒。

我们还经常遇到setTimeout(fn,0)这样的代码，0秒后执行又是什么意思呢？是不是可以立即执行呢？
答案是不会的，setTimeout(fn,0)的含义是，指定某个任务在主线程最早可得的空闲时间执行，意思就是不用再等
多少秒了，只要主线程执行栈内的同步任务全部执行完成，栈为空就马上执行
关于setTimeout要补充的是，即便主线程为空，0毫秒实际上也是达不到的。根据HTML的标准，最低是4毫秒。（最低延时的设置是为了给CPU留下休息时间）

## Promise
关于Promise的定义和功能，大家可以戳[Promise](https://es6.ruanyifeng.com/#docs/promise)了解
接下来进入本文的正题，除了广义的同步任务和异步任务，我们对任务有更精细的定义：  
1、 macro-task(宏任务)：包括整体代码script，setTimeout，setInterval，setImmediate ，I/O ，UI rendering  
2、 micro-task(微任务)：process.nextTick ，promise ，Object.observe ，MutationObserver我们用本文最开始的一段代码来描述一下Event Loop的过程  

```
setTimeout(function() {
    console.log('setTimeout');
})

new Promise(function(resolve) {
    console.log('promise');
}).then(function() {
    console.log('then');
})

console.log('console');
```

1、 这段代码作为宏任务，进入主线程。  
2、 先遇到setTimeout，那么将其回调函数注册后分发到宏任务Event Queue。(注册过程与上同，下文不再描述)  
3、 接下来遇到了Promise，new Promise立即执行，then函数分发到微任务Event Queue。  
4、 遇到console.log()，立即执行。  
好啦，整体代码script作为第一个宏任务执行结束，看看有哪些微任务？我们发现了then在微任务Event Queue里面，执行。
ok，第一轮事件循环结束了，我们开始第二轮循环，当然要从宏任务Event Queue开始。我们发现了宏任务Event Queue中setTimeout对应的回调函数，立即执行。
结束。
事件循环，宏任务，微任务的关系如图所示：
![企业微信截图_15693143768661.png](https://i.loli.net/2019/10/23/MLUTuGEf9yaVwgd.png)

## 拓展

### Node中的Event loop

Node 中的 Event loop 和浏览器中的不相同。

Node 的 Event loop 分为 6 个阶段，它们会按照顺序反复运行

```
┌───────────────────────┐
┌─>│        timers         │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
│  │     I/O callbacks     │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
│  │     idle, prepare     │
│  └──────────┬────────────┘      ┌───────────────┐
│  ┌──────────┴────────────┐      │   incoming:   │
│  │         poll          │<──connections───     │
│  └──────────┬────────────┘      │   data, etc.  │
│  ┌──────────┴────────────┐      └───────────────┘
│  │        check          │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
└──┤    close callbacks    │
   └───────────────────────┘
```


### 进一步了解event loops

知道了event loops大致做什么的，我们再深入了解下event loops。

有两种event loops，一种在浏览器上下文，一种在workers中。  
1、 每个线程都有自己的event loop。  
2、 浏览器可以有多个event loop，browsing contexts(浏览器上下文)和web workers就是相互独立的。  
3、 所有同源的browsing contexts可以共用event loop，这样它们之间就可以相互通信。


### 了解vue中的nextTick是如何实现的

在 Vue 源码 2.5+ 后，nextTick 的实现单独有一个 JS 文件来维护它，它的源码并不多，总共也就 100 多行。
接下来我们来看一下它的实现，在 src/core/util/next-tick.js 中：
```
import { noop } from 'shared/util'
import { handleError } from './error'
import { isIOS, isNative } from './env'

const callbacks = []
let pending = false

function flushCallbacks () {
  pending = false
  const copies = callbacks.slice(0)
  callbacks.length = 0
  for (let i = 0; i < copies.length; i++) {
    copies[i]()
  }
}

let microTimerFunc
let macroTimerFunc
let useMacroTask = false

if (typeof setImmediate !== 'undefined' && isNative(setImmediate)) {
  macroTimerFunc = () => {
    setImmediate(flushCallbacks)
  }
} else if (typeof MessageChannel !== 'undefined' && (
  isNative(MessageChannel) ||
  // PhantomJS
  MessageChannel.toString() === '[object MessageChannelConstructor]'
)) {
  const channel = new MessageChannel()
  const port = channel.port2
  channel.port1.onmessage = flushCallbacks
  macroTimerFunc = () => {
    port.postMessage(1)
  }
} else {
  /* istanbul ignore next */
  macroTimerFunc = () => {
    setTimeout(flushCallbacks, 0)
  }
}

// Determine microtask defer implementation.
/* istanbul ignore next, $flow-disable-line */
if (typeof Promise !== 'undefined' && isNative(Promise)) {
  const p = Promise.resolve()
  microTimerFunc = () => {
    p.then(flushCallbacks)
    if (isIOS) setTimeout(noop)
  }
} else {
  // fallback to macro
  microTimerFunc = macroTimerFunc
}

export function withMacroTask (fn: Function): Function {
  return fn._withTask || (fn._withTask = function () {
    useMacroTask = true
    const res = fn.apply(null, arguments)
    useMacroTask = false
    return res
  })
}

export function nextTick (cb?: Function, ctx?: Object) {
  let _resolve
  callbacks.push(() => {
    if (cb) {
      try {
        cb.call(ctx)
      } catch (e) {
        handleError(e, ctx, 'nextTick')
      }
    } else if (_resolve) {
      _resolve(ctx)
    }
  })
  if (!pending) {
    pending = true
    if (useMacroTask) {
      macroTimerFunc()
    } else {
      microTimerFunc()
    }
  }
  // $flow-disable-line
  if (!cb && typeof Promise !== 'undefined') {
    return new Promise(resolve => {
      _resolve = resolve
    })
  }
}
```
next-tick.js 申明了 microTimerFunc 和 macroTimerFunc 2 个变量，它们分别对应的是 micro task 的函数和 macro task 的函数。对于 macro task 的实现，优先检测是否支持原生 setImmediate，这是一个高版本 IE 和 Edge 才支持的特性，不支持的话再去检测是否支持原生的 MessageChannel，如果也不支持的话就会降级为 setTimeout 0；而对于 micro task 的实现，则检测浏览器是否原生支持 Promise，不支持的话直接指向 macro task 的实现。

next-tick.js 对外暴露了 2 个函数，先来看 nextTick，这就是我们在上一节执行 nextTick(flushSchedulerQueue) 所用到的函数。它的逻辑也很简单，把传入的回调函数 cb 压入 callbacks 数组，最后一次性地根据 useMacroTask 条件执行 macroTimerFunc 或者是 microTimerFunc，而它们都会在下一个 tick 执行 flushCallbacks，flushCallbacks 的逻辑非常简单，对 callbacks 遍历，然后执行相应的回调函数。

这里使用 callbacks 而不是直接在 nextTick 中执行回调函数的原因是保证在同一个 tick 内多次执行 nextTick，不会开启多个异步任务，而把这些异步任务都压成一个同步任务，在下一个 tick 执行完毕。

next-tick.js 还对外暴露了 withMacroTask 函数，它是对函数做一层包装，确保函数执行过程中对数据任意的修改，触发变化执行 nextTick 的时候强制走 macroTimerFunc。比如对于一些 DOM 交互事件，如 v-on 绑定的事件回调函数的处理，会强制走 macro task。

## 写在最后
1、 javascript是一门单线程语言  
2、 Event Loop是js实现异步的一种方法，也是js的执行机制。

## 趣味探讨

如何实现一个准时的定时器？

最简单有效的方式就是使用requestAnimationFrame动画去实现（借助的是requestAnimationFrame的一个特性，即符合w3c规范的浏览器都会以“浏览器屏幕刷新次数”的频率去调用回调函数。而一般屏幕的刷新频率是60hz，即1秒60次）
使用这种方案，只需要结合Date（如Date.now()）的时间戳去做一定的时间控制和逻辑处理即可。




  
