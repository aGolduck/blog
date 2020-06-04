---
date: "2019-03-03T00:00:00Z"
tags:
- typescript
- javascript
title: 应用 typescript 严格模式
---

最近上手了 nestjs, 开始了 typescript 之旅。趁着项目新开始不久，免得日后积重难返，对代码应用了 strict 模式。strict 模式包括下文的几个选项，下面依次介绍此次迁移中遇到的问题和解决方法。

## noImplicitAny
必须显式声明 any 类型。该选项促使你尽可能地显式声明类型，避免 any 类型的出现。一般情况下，要么选择合适的类型，要么加上 any 声明就好。遇到两个比较棘手的问题。
### 中间结果是 any 类型
有时 typescript 无法推断出表达式的结果类型，错误日志标示整个表达式的类型无法推断, 这时候需要稍微改造代码，将表达式赋值到声明为 any 类型的中间变量。
### TS7053 Element implicitly has an ‘any’ type because expression of type ‘string’ can’t be used to index type xxx
这种情况下需要声明索引的类型是 key of 被索引的目标的类型
```ts
class A {
  a: number
  b: string
}
const o = new A()
const property: keyof A = 'a'
console.log(o[property])
```
### 复杂的类型对应
本质上这涉及到一个复杂的映身关系。对于对象A，它的每个属性p类型t(p)都是已知的，而且对象B有对应的属性(q)，类型是s(t(p))，怎么让编译器知道这个对应关系。

## strictNullChecks
### TS2366: Function lacks ending return statement and return type does not include 'undefined' 
一般是有分支死区，加上异常处理
### TS2532: Object is possibly 'undefined'
typesrcipt 不获取任何运行时信息，有时显得非常蠢。
```ts
const o = {a: 1}
const k = 'a'
if (o[k]) console.log(o[k].toString())
```
尽管已经有非空判断了，还是会报 o[k] 可能 undefined 错误，强制 `o[k]!.toString()` 就好
## strictFunctionTypes
```ts
       [['detailImages', 'detailImagesJoined'],
        ['sellingPoints', 'sellingPointsJoined']
       ].forEach(
         ([list, joined]: ['detailImages' | 'sellingPoints', 'detailImagesJoined' | 'sellingPointsJoined']) => {
           job[joined] = newJob[list].join(',')
         }
       )
```


```
error TS2345: Argument of type '([list, joined]: ["detailImages" | "sellingPoints", "detailImagesJoined" | "sellingPointsJoined"]) => void' is not assignable to parameter of type '(value: string[], index: number, array: string[][]) => void'.
  Types of parameters '__0' and 'value' are incompatible.
    Type 'string[]' is missing the following properties from type '["detailImages" | "sellingPoints", "detailImagesJoined" | "sellingPointsJoined"]': 0, 1
```
可能与之前一篇博文 forEach 可选参数 index, array 相关。暂时用 for ... of 绕过
## strictBindCallApply
## strictPropertyInitialization
TS2564: Property 'name' has no initializer and is not definitely assigned in the constructor.

大量 typeorm, graphql 注入类。使用 !: 就好了。

## noImplicitThis
## alwaysStrict
