+++
date = "2019-10-23T00:00:00Z"
tags = ["npm", "javascript"]
categories = ["javascript"]
title = "npm 包私藏"

+++

稍微有点开发实战经验的都知道 DRY(Do not Repeat Yourself). 说白了就是能抄就抄。如果有第三方模块能够使用，就用。其次改。实在不行再考虑自己写。但事情到了 js 这里稍有些不一样。js 有臭名昭著的 Dependency Hell, 很大程度上是因为小包文化盛行。很多人都在知乎上吃过瓜，且看 justjavac [驳《我不是很懂 Node.js 社区的 DRY 文化》](https://zhuanlan.zhihu.com/p/35864087) 英文世界里[sindresorhus](https://github.com/sindresorhus/ama/issues/10) 的观念很有代表性。他说代码模块就像乐高一样，不是按代码行来算的，而是按功能点来算的。这其实有点诡辩。乐高也是一组一组包装的。包是我们能用的最大的单元了，敢情他们家乐高全是散装的，都是一片一片买的。人的记忆容量是有限的，而且用树形结构来组织信息。这样散乱的扁平化的包想用时都不知道怎么找起。理想的情况下，应该把各种小包组织成 `lodash` 这样的工具集合到一起。

在现有的生态下，平时还是要注意积累有用的包。下面的表格会不断地更新。


| package                    | tags                | description                                   |
|----------------------------|---------------------|-----------------------------------------------|
| ajv                        | json-schema         | another json schema validator                 |
| debug                      | debug,log           |                                               |
| delay                      | timer               | aka, sleep                                    |
| elastic-apm-node           | apm                 | apm with ELK(elastic, logstash, kibana)       |
| express-mung               | express             | express response hooker                       |
| image-size                 | image               | get width, height, and type of any image file |
| js-yaml                    | yaml,parser         | yaml parser and dumper                        |
| json-schema-ref-parser     | json-schema, parser |                                               |
| lerna                      | package             | monorepo                                      |
| node-mock-stdin            | test                |                                               |
| node-pool                  | pool                | generic resource pool with Promise based API. |
| oas-kit                    | swagger,openapi     | converter, validator, linter, resolver        |
| really-need                | package             | require utils                                 |
| required-in-the-middle     | hook,module,require | module to hook into `require` function        |
| semver                     | package             | 用于语义化版本的各种操作                      |
| sinon                      | test                | mock, fake, spy, stub                         |
| stackman                   | error               | error stack tracer                            |
| supertest                  | test,http,api       | for http test                                 |
| swagger-jsdoc              | openapi,jsdoc       | generate swagger doc based on jsdoc           |
| swagger-express-middleware | openapi,express     | mock and parse/validate request               |
| swagger-parser             | openapi,parser      | openapi parser/validator                      |
| swagger-cli                | openapi,cli         | command line openapi validator and bundler    |
| @open-api/xxx              | openapi             | 似乎还不完全支持 openapi 3.0                  |
