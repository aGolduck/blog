+++
date = "2019-06-19T00:00:00Z"
tags = ["javascript", "nodejs"]
categories = ["javascript"]
title = "koa-compose"

+++

koa 的精华统共就这么几行。

```js
// compose :: [Context -> Next -> Promise] -> (Context -> Promise)
function compose (middleware) {
  // validate middleware
  // ..

  return function (context, next) {
    // last called middleware #
    let index = -1
    return dispatch(0)
    function dispatch (i) {
      if (i <= index) return Promise.reject(new Error('next() called multiple times'))
      index = i
      let fn = middleware[i]
      if (i === middleware.length) fn = next
      if (!fn) return Promise.resolve()
      try {
        return Promise.resolve(fn(context, dispatch.bind(null, i + 1)));
      } catch (err) {
        return Promise.reject(err)
      }
    }
  }
}
```

用一个简单的 middleware 数组 `[middleware0, middleware1]` 来推演一遍。

-   index = -1, dispatch(0)

    ```js
    i = 0
    index = i = 0
    fn = middleware0

    dispatch(0) =  Promise.resolve(middleware0(context, function() { return dispatch(1) }));
    ```

-   index = 0, dispatch(1)

    ```js
    i = 1
    index = i = 1
    fn = middleware1

    dispatch(1) = Promise.resolve(middleware1(context, function() {return dispatch(2)}));
    ```

-   index = 1, dispatch(2)

    ```js
    i = 2
    index = i = 2
    fn = next

    dispatch(2) = Promise.resolve(next(context, function() {return dispatch(3)}));
    ```

-   index = 2, dispath(3)

    ```js
    i = 3
    index = i = 3
    fn = next

    dispatch(3) = return Promise.resolve()
    ```


    整合起来。

    ```js
    return Promise.resolve(
      middleware0(ctx, () => Promise.resolve(
        middleware1(ctx, () => Promise.resolve(
          next(ctx, () => Promise.resolve())
        ))
      ))
    );
    ```

    It's nothing more than a recursive function, just wrapped in many `Promise.resolve`.
