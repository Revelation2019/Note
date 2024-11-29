# defer

deferred 是一个常见的编程模式，通常用于处理异步操作。它允许你创建一个可以在未来某个时间点手动解决或拒绝的 Promise。这个模式在处理复杂的异步逻辑时非常有用，例如在需要手动控制 Promise 的状态时。

以下是一个示例，展示了如何使用 deferred 模式：

```js
// --run--
const defer = () => {
  let resolve, reject;
  const promise = new Promise((res, rej) => {
    resolve = res;
    reject = rej;
  });
  promise.resolve = resolve;
  promise.reject = reject;

  return promise;
}
```

你可以在需要手动控制 Promise 状态的地方使用 deferred 模式。例如，假设你有一个异步操作，需要在某个条件满足时手动解决或拒绝 Promise。

```js
// --run--
const deferred = defer();

// 模拟一个异步操作
setTimeout(() => {
  const success = true; // 假设这是你的条件
  if (success) {
    deferred.resolve("Operation successful");
  } else {
    deferred.reject("Operation failed");
  }
}, 1000);

// 使用 deferred.promise
deferred
  .then((result) => {
    console.log(result); // 输出: Operation successful
  })
  .catch((error) => {
    console.error(error); // 如果条件不满足，输出: Operation failed
  });
```

这种模式在处理复杂的异步逻辑时非常有用，例如在需要手动控制 Promise 的状态或在多个异步操作之间进行协调时。