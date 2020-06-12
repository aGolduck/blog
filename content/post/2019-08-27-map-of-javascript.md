+++
date = "2019-08-27T00:00:00Z"
tags = ["javascript", "functional-programming"]
title = "从 js 神奇的 map 说到函数的元数"

+++

从 lujun9972 的博客看到一则[博文](http://blog.lujun9972.win/blog/2019/07/08/javascript%E4%B8%AD%E7%A5%9E%E5%A5%87%E7%9A%84map/), 说在 https://medium.com/dailyjs/parseint-mystery-7c4368ef7b21 看到 js 的 map 神奇的一面。

运行下面的 js 代码：
```js
['1', '7', '11'].map(parseInt)
```
得到的是

```js
[1, NaN, 3]
```
？？？

js 真是到处都是坑啊。原因在哪里呢？下面是 MDN `array.prototype.map` 和 `parseInt` 的文档。
```js
var new_array = arr.map(function callback(currentValue[, index[, array]]) {
 // Return element for new_array
}[, thisArg])
```
```js
parseInt(string, radix=10)
```

可以看到 map 的回调函数其实有三个参数：数组元素，元素的索引和完整的数组。parseInt 也是一个双参数的函数。
那么实际上上面的代码的中间结果是
```js
[parseInt('1', 0), parseInt('7', 1), parseInt('11', 2)]
```
同样的，执行以下代码
```js
[1, 2, 3].map(console.log)
```
打印出来是：
```
1 0 [ 1, 2, 3 ]
2 1 [ 1, 2, 3 ]
3 2 [ 1, 2, 3 ]
[ undefined, undefined, undefined ]
```

js 实参跟形参数量是可以不对应，多余的实参被忽略，不够的用 undefined 补上。我们说 js 可以很灵活地进行函数式编程，但是这种参数特性其实和经典的 haskell 是相悖的。一个重要的原因是柯里化(curry)和函数的元数(arity)是息息相关的。使用受 haskell 影响比较大的 `ramdajs` 时，函数的元数是受到严格要求的。之前
在我使用 ramda 结合 parseInt 时就遇到了坑。

直接在 R.ifElse 里面使用 parseInt, 结果是得到函数
```js
const page = ifElse(
  identity,
  Number.parseInt,
  always(1)
)(this.ctx.request.query.page);
```
但是如果是把 parseInt 包装一下就正常了
```js
const page = ifElse(
  identity,
  (s) => Number.parseInt(s),
  always(1)
)(this.ctx.request.query.page);
```

我在 stackoverflow 上提了[问题](https://stackoverflow.com/questions/47196675/not-able-to-use-parseint-as-a-ontrue-or-onfalse-function-for-ifelse-of-ramdajs)，得到很完美的回答。由于 ramdajs 的所有的函数都是柯里化的，总要有一个标准判断什么时候真正执行函数，这个标准就是参数个数是一个的时候。这里可以用 `Number` 函数来替代 `parseInt`(但是我后来发现这玩意也有坑，Number 和 parseInt 的语义并不完全一致). ramdajs 的[作者之一](https://github.com/CrossEye)，还给出了另一个解决方案，就是用 R.unary 包装一个，其实也可以等价于说是我上面的方法的简写版了。
