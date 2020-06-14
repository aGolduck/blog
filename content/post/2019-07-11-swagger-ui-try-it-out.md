+++
date = "2019-07-11T00:00:00Z"
tags = ["swagger", "openapi", "web"]
categories = ["web"]
title = "Bypass quirk of Swagger-UI try-it-out"

+++

Swagger-UI 有一个很有用的特性，就是可以用 `Try it out` 按钮直接试用，配置好授权策略以后，还可以直接使用登陆态。但是我自己在使用的过程中经常会遇到一个问题。经常会以下的错误。

![firefox error response](/assets/firefox-error-response.jpg)

打开浏览器的调试网络面板，啥都没有。很长一段时间里都不知道解决的办法。直到有一天拿出来跟同事讨论，最后得出个结论，如果地址栏是 `http` 的话就不存在这个问题。到底是为什么呢？

终于有一天开窍了。想起来开发时遇到的很多前端问题都是因为我用 firefox 引起的。不如用 Chrome 试一下。

使用 Chrome 浏览器会返回 `TypeError: Failed to fetch`。

![Chrome failed to fetch](/assets/chrome-failed-to-fetch.jpg)

打开 `Network` 面板比起 firefox 多了一条红红的一条记录，点开 `Response`: `Failed to load response data`。

![Chrome failed to load response data](/assets/chrome-failed-to-load-response-data.jpg)

再点开 `Headers`。看到一条很可疑的东西。

```
Referer Policy: no-referrer-when-downgrade
```

MDN 有[相关文档](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Referrer-Policy)。

> 在没有指定任何策略的情况下用户代理的默认行为。在同等安全级别的情况下，引用页面的地址会被发送(HTTPS->HTTPS)，但是在降级的情况下不会被发送 (HTTPS->HTTP)。

这很可能是 `swagger-ui` 的默认设置了。

教训：调试工具还是 Chrome 的好啊，firefox 这算什么，一个请求出去啥都不显示。
