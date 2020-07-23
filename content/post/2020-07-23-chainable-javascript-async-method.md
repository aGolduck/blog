+++
title = "可链式调用的 javasrcipt 异步方法"
date = 2020-07-23T18:36:00+08:00
lastmod = 2020-07-23T18:41:13+08:00
tags = ["javascript", "Promise"]
categories = ["javascript"]
draft = false
+++

有一道经典的 js 面试题是这样的：编写一个类 `Walker`, 拥有两个方法，一个是 `walk`,
一个是 `sleep`, 要求两个方法都能接受一个整数参数，表示函数运行的秒数，并且两个方法都能链式调用。比如可以达到以下效果。

```js
const walker = new Walker()
walker.sleep(1).walk(2).sleep(3).walk(4)
```


## 经典 setTimeout {#经典-settimeout}

这道题的核心要求是将程序挂起 N 秒后恢复程序运行。当然你可以用死循环的办法不断地查询系统时间，时间到了就返回，实现同步的 `sleep` 和 `walk`, 但是这样会将 CPU 跑满，实用程序是不可能这么做的，我们需要将控制权交还给系统（不管是操作系统还是 runtime)，时间到了后恢复运行状态。

[Programming Idioms](https://www.programming-idioms.org/) 是一个挺有意思的网站，上面可查看常见语言对一些简单的任务的实现。从 [Pause execution for 5 seconds](https://www.programming-idioms.org/idiom/45/pause-execution-for-5-seconds) 上可以看到除了 js 的绝大多数语言都提供了 `sleep`
方法，作用就是上述的控制权转移。js 也有类似的 `setTimeOut` 方法，不同的在于，其它语言是用语句的先后顺序来实现这个控制权的转移， `setTimeOut` 则是用回调方法，这就给我们实现题目中的链式调用造成了很大困难。

```js
function sleep() {console.log('sleep')}
function walk() {console.log('walk')}
setTimeout(function() {
  sleep()
  setTimeout(function() {
    walk()
    setTimeout(function() {
      sleep()
      setTimeout(
      walk, 4000)
    }, 3000)
  }, 2000)
}, 1000)
```


## async/await {#async-await}

新标准的 async/await 语法为我们提供了用同步的顺序写异步逻辑的可能性。如果将 `sleep`
和 `walk` 实现成 async 函数，可以将代码写成如下。

```js
async function sleep(ms) {
  console.log('sleep')
  return new Promise(resolve => setTimeout(resolve, ms))
}
async function walk(ms) {
  console.log('walk')
  return new Promise(resolve => setTimeout(resolve, ms))
}
async function run() {
  await sleep(1000)
  await walk(2000)
  await sleep(3000)
  await walk(4000)
}
run()
```


## pure promise {#pure-promise}

但是所有的 async function 返回的都是 awaitable 的 Promise. 用来作为类的方法时是不可能返回 this 对象以供链式调用的。实际上跟链式调用长得比较像的反而是不用 async/await
的 Promise. 为了返回 this, 我们也必须完全放弃 async function.

```js
// 这里 async function 与普通 function 的语义其实是一样的，都是返回 promise
function sleep(ms) {
  console.log('sleep')
  return new Promise(resolve => setTimeout(resolve, ms))
}
function walk(ms) {
  console.log('walk')
  return new Promise(resolve => setTimeout(resolve, ms))
}
Promise.resolve()
  .then(sleep.bind(null, 1000))
  .then(walk.bind(null, 2000))
  .then(sleep.bind(null, 3000))
  .then(walk.bind(null, 4000))
```


## chainable promise {#chainable-promise}

可以观察到这个链式调用和我们的目的的链式调用每个步骤都是一一对应的关系

```text
Promise.resolve()                         walker
   .then(sleep.bind(null, 1000))             .sleep(1)
   .then(walk.bind(null, 2000))              .walk(2)
   .then(sleep.bind(null, 3000))             .sleep(3)
   .then(walk.bind(null, 4000))              .walk(4)
```

现在的问题就在于有没有可能用某个方法将两个过程关联起来。

```js
let promise = Promise.resolve()
const walker = {}

walker.sleep = function (s) {
  promise = promise.then(function () {
    console.log('sleep')
    return new Promise(resolve => setTimeout(resolve, 1000 * s))
  })
  return walker
}
walker.walk = function (s) {
  promise = promise.then(function () {
    console.log('walk')
    return new Promise(resolve => setTimeout(resolve, 1000 * s))
  })
  return walker
}

walker.sleep(1).walk(2).sleep(3).walk(4)
```


## final answer {#final-answer}

改写成类的形式可以考虑用 `has-a` 关系组合甚至是 `is-a` 关系继承。

```js
class Walker {
  constructor() {
    this.promise = Promise.resolve()
  }
  sleep(s) {
    this.promise = this.promise.then(function () {
      console.log('sleep')
      return new Promise(resolve => setTimeout(resolve, 1000 * s))
    })
    return this
  }
  walk(s) {
    this.promise = this.promise.then(function () {
      console.log('walk')
      return new Promise(resolve => setTimeout(resolve, 1000 * s))
    })
    return this
  }
}

new Walker().sleep(1).walk(2).sleep(3).walk(4)
```

这其实等于变相实现了一个 [Deferred](http://liubin.org/promises-book/#deferred-and-promise).

为了和其它函数配合，可以加一个 `exec` 方法返回 promise.

```js
class Walker {
  constructor() {}
  sleep() {}
  walk() {}
  exec() {
    return this.promise
  }
}

await new Walker().sleep(1).walk(2).sleep(3).walk(4).exec()
```


## <span class="org-todo todo TODO">TODO</span> 其它解法 {#其它解法}
