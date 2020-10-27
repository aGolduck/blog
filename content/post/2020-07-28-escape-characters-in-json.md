+++
title = "JSON 如何存储换行符"
date = 2020-10-27T10:58:00+08:00
publishDate = 2020-07-28T00:00:00+08:00
lastmod = 2020-10-27T10:58:59+08:00
tags = ["javascript", "json"]
categories = ["javascript"]
draft = false
+++

今天遇到了一个 JSON 字符串的错误，简单地说就是 js 的 `JSON.parse` 无法解析换行符，比如下面的例子 `Line 1\nLine 2` 是一个有效的 js 字符串，但是
JSON 无法解析。

```js
try {
   console.log("Line 1\nLine 2")
   console.log('-------------------------------------------')
   console.log(JSON.parse('"Line 1\nLine 2"'))
} catch(error) {
   console.log(error)
}
```

```text
Line 1
Line 2
-------------------------------------------
SyntaxError: Unexpected token
 in JSON at position 7
    at JSON.parse (<anonymous>)
    at /tmp/babel-Opt7jV/js-script-KGvIxD:5:21
    at Object.<anonymous> (/tmp/babel-Opt7jV/js-script-KGvIxD:9:2)
    at Module._compile (internal/modules/cjs/loader.js:1015:30)
    at Object.Module._extensions..js (internal/modules/cjs/loader.js:1035:10)
    at Module.load (internal/modules/cjs/loader.js:879:32)
    at Function.Module._load (internal/modules/cjs/loader.js:724:14)
    at Function.executeUserEntryPoint [as runMain] (internal/modules/run_main.js:60:12)
    at internal/main/run_main_module.js:17:47
undefined
```

JSON 脱胎于 js 的对象，一般认为 JSON 是 js 的子集，是一个有效的 js 数组，对象或者字符串，数字。但是 js 总有各种意外情况。[JSON - JavaScript | MDN](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global%5FObjects/JSON) 有一个链接 [JSON: The JavaScript subset that isn't - Timeless](http://timelessrepo.com/json-isnt-a-javascript-subset)，讲的是 `U+2028` 与 `U+2029` 两个字符不能出现在 js 字符串中，但却是有效的 json 字符，也即是说不是所有的 json 字符串都是 js 字符串。那反过来说，由我们前面的例子是不是说明不是所有的 js 字符串都是 json 字符串。

那么 JSON 如何存储换行符呢？

首先 `\n` 是一个字符，用 unicode 码点的表示法是 `\u000a` ，下面两个语句打印结果是一样的。

```js
console.log("Line 1\nLine 2")
console.log('-------------------------------------------')
console.log("Line 1\u000ALine 2")
```

```text
Line 1
Line 2
-------------------------------------------
Line 1
Line 2
undefined
```

然后我们利用 Buffer 看下序列化前后都是什么。

```js
console.log(Buffer.from('Line 1\nLine 2'))
console.log(Buffer.from('Line 1\u000aLine 2'))
console.log(Buffer.from(JSON.stringify('Line 1\nLine 2')))
console.log(Buffer.from(JSON.stringify('Line 1\u000aLine 2')))
```

```text
: <Buffer 4c 69 6e 65 20 31 0a 4c 69 6e 65 20 32>
: <Buffer 4c 69 6e 65 20 31 0a 4c 69 6e 65 20 32>
: <Buffer 22 4c 69 6e 65 20 31 5c 6e 4c 69 6e 65 20 32 22>
: <Buffer 22 4c 69 6e 65 20 31 5c 6e 4c 69 6e 65 20 32 22>
: undefined
```

从 Buffer 可以看出，序列化的时候作了转义: escape(`\n`) = escape(`\u000a`) = `\` (`5c`) + `n` (`6e`)

[RFC 8259 - The JavaScript Object Notation (JSON) Data Interchange Format](https://tools.ietf.org/html/rfc8259)
对字符串的定义是：

```text
string = quotation-mark *char quotation-mark

char = unescaped /
    escape (
        %x22 /          ; "    quotation mark  U+0022
        %x5C /          ; \    reverse solidus U+005C
        %x2F /          ; /    solidus         U+002F
        %x62 /          ; b    backspace       U+0008
        %x66 /          ; f    form feed       U+000C
        %x6E /          ; n    line feed       U+000A
        %x72 /          ; r    carriage return U+000D
        %x74 /          ; t    tab             U+0009
        %x75 4HEXDIG )  ; uXXXX                U+XXXX

escape = %x5C              ; \

quotation-mark = %x22      ; "

unescaped = %x20-21 / %x23-5B / %x5D-10FFFF
```

可以理解为这些控制字符序列化的时候转成了 `\` + `"\/bfnrt` 的格式，而在 js 的字符串中，
`\` 必须写为 `\\`.

```js
console.log('\\')
```

```text
\
undefined
```

所以，下面的字符串才能得到正确解析

```js
console.log(JSON.parse('"Line 1\\nLine 2"'))
```

```text
Line 1
Line 2
undefined
```

注意这里的 `\\n` 不是对 `\` 转义再存 `n` 哦，而是如上所述，转义 `\u000a` 这个换行符存储为 `\` (写为 `\\`)加 `n`.
