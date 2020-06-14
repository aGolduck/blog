+++
date = "2020-05-15T00:00:00Z"
title = "集成 SOFA 到 nestjs"
tags = ["nestjs"]
categories = ["javascript"]
+++

[https://github.com/Urigo/SOFA](SOFA) 是一个 express 中间件，
可以将 graphql 查询转换成 restful 的 api. 最近将其集成到 nestjs
的时候遇到一些问题，下面记录下解决的过程。

有人在 SOFA github 的 [https://github.com/Urigo/SOFA/issues/250](issue)
下直接提问过如何集成到 nestjs 中。作者的回答是 nestjs graphql 的
schema 是可以直接获取的，SOFA 只需要这个 schema 就可以了。但是在原文档中
我发现这个 schema 得在 `app.listen()` 或者 `app.init()` 后才可以获取。
意味着 express 程序已经启动，是没法再应用任何中间件了。

这是 sofa 的基本用法

```ts
interface SofaConfig {
  schema: GraphQLSchema;
  context?: Context;
  execute?: ExecuteFn;
  ignore?: Ignore;
  onRoute?: OnRoute;
  depthLimit?: number;
  errorHandler?: ErrorHandler;
  method?: MethodMap;
}

function useSofa(config: SofaConfig): ExpressRouter {
  return createRouter(createSofa(config));
}

app.use(
  'api',
  useSofa({
    schema,
    ignore: ['User.favoriteBook'],
    onRoute(info) {
      openApi.addRoute(info, {
        basePath: '',
      });
    },
  })
)
```

`SofaConfig` 是使用这个中间件的关键。其中最重要的是 schema 和 execute
参数。execute 默认使用 'graphql-js' 的 graphql 函数。schema 在 SOFA
的 example 中使用的是 makeExecutebaleSchema. 只要能生成有效的 schema
就行。至少要 typeDefs. 如果使用默认 graphql 至少还要有 resolvers.

```ts
export declare function makeExecutableSchema<TContext = any>({ typeDefs, resolvers, connectors, logger, allowUndefinedInResolve, resolverValidationOptions, directiveResolvers, schemaDirectives, parseOptions, inheritResolversFromInterfaces, }: IExecutableSchemaDefinition<TContext>): GraphQLSchema;
```

生成 schema 的方案。

自己生成。typeDefs 都可以，resolvers 7.0 版本后可在 app.listen() 后获得。
或者自己模仿过程 type-graphql 或 nest/typescript 7 生成。

升级就是个大坑，弃。

自己模仿可以在 app.listen 前生成，但工作量大，
几乎肯定会丢失 nest 框架的很多信息。
listen 后获取只能重启一个 express 实例，Guard 等机制不知道是否仍能应用。

最后一个绕道的方法，request 调起 /graphql。重新发起了一个本地 http 请求，
有时间损耗，有衔接工作。从 args 获取 source, contextValue.headers,
variableValues 就可以发起远程请求。Sofa 文档也提到

> Thanks to that you can even use a remote GraphQL Server (with Fetch or through Apollo Links).
