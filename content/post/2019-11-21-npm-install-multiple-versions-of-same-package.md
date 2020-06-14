+++
date = "2019-11-21T00:00:00Z"
tags = ["node", "npm"]
categories = ["javascript"]
title = "npm 安装同一个包的不同版本"

+++
## 应用场景
不想升级老代码依赖

npm 无法同时安装同一个 package 的不同版本，以往遇到这种需求的时候，我的办法是自己新建一个包，然后在新建的包里安装目标包的一个版本，然后暴露出去。这样，项目就可以依赖新建的包和另一个版本，实现同时依赖两个版本的目的。

在 npm 6.9 以后，npm 可以为安装的包使用别名，可以近乎完美地解决这个问题了。`elasticsearch-js` 项目有[一段](https://github.com/elastic/elasticsearch-js/tree/v7.4.0#install-multiple-versions)详细说明如何使用，我把这一段翻译了一下。

## 同时使用多个版本
如果你同时使用多个版本的 Elasticsearch, 你需要使用对应的多个客户端。同时安装同一个包的多个版本在以前是不可能的，但是 `npm 6.9` 以后，你可以用别名达到目的。

安装不同版本的客户端你必须运行以下命令：

```
npm install <alias>@npm:@elastic/elasticsearch@<version>
```

比方说要同时安装 `7.x` 和 `6.x`, 运行以下命令

```
npm install es6@npm:@elastic/elasticsearch@6
npm install es7@npm:@elastic/elasticsearch@7
```

`package.json` 文件就会多出下面的两行：
```
"dependencies": {
  "es6": "npm:@elastic/elasticsearch@6.7.0",
  "es7": "npm:@elastic/elasticsearch@7.0.0"
}
```

在代码中将刚刚定义的别名 `require` 进来。

```js
const {Client: Client6} = require('es6')
const {Client: Client7} = require('es7')

const client6 = new Client6({node: 'http://localhost:9200'})
const client7 = new Client7({node: 'http://localhost:9201'})

client6.info(console.log)
client7.info(console.log)
```

最后，如果你想使用下一版本 Elasticsearch(即 Elasticsearch 的 master 分支) 的客户端，你可以运行下面的命令：

```
npm install esmaster@github:elastic/elasticsearch-js
```

## 补充

由上文的最后一条命令可看出，别名不局限于从 npm 下载的包。从 github 或者其它自定义 git 地址下载的包也可以使用别名。npm 如何安装其它来源的包可以参照[官方文档](https://docs.npmjs.com/cli/install)
