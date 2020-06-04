---
date: "2019-10-28T00:00:00Z"
tags:
- swagger
- openapi
- nodejs
title: swagger/openapi 在 node 项目中的实践
---

Swagger, 现在也称 openapi, 定义了规范的 RESTFUL 接口文档标准，围绕这个标准产生了一个很丰富的[生态圈](https://apis.guru/awesome-openapi3/)。本文谈谈在 node 项目中应用这些工具的一些经验。

首先最基础的是 swagger 的文档功能。如何展示 swagger 文档？swagger-ui 已经不错，有以下的优点。

- 界面较为美观，而且能够交互性地操作。
- 可以产生对应的 curl 命令，很大程度上可以取代 postman 等工具。
- 提供了认证方法，需要登陆态的接口也可以在线试用。

目前的缺点是接口数量多了没有办法有效组织，也缺乏搜索功能，导致查找不便的问题。在 swagger-ui 的文档里面提到了自定义布局的功能，有待挖掘。

解决读的问题后，接着要解决如何写的问题。API文档往往会急骤膨胀，如果写在一个文件里会变得难以书写。可以使用 swagger jsdoc 书写具体的接口，分布于各个接口的具体文件里，也使得代码和文档容易保持一致。其它的 components 可以由多个 yaml 文件书写，swagger-jsdoc 能够将各个文件的内容组装起来。

写文档的时候记得尽可能使用 $ref, allOf 等， 提高文档书写的可复用性。

最后一个很重要的可以利用 swagger 对出参入参进行基本的类型和是否必须进行校验。可以用 oas-chow-chow. 进行校验的 openapi 必须严格遵守规范，最好使用 oas-validator 先进行校验。oas-validator 会在项目启动时检查 swagger 的语法，耗时比较长，可以在生产服务器禁用。

TODO 加例子。
