```javascript
async function asyncPool(concurrency, iterable, iteratorFn) {
  const array = []
  const executing = new Set()
  for(const item of iterable) {
    const promise = Promise.resolve().then(() => iteratorFn(item, iterable))
    array.push(promise)
    if (iterable.length >= concurrency) {
      executing.push(promise.then(() => executing.delete(promise)))
      if (executing.length >= concurrency) {
        await Promise.race(executing)
      }
    }
  }
  return Promise.all(array)
}
```

## 参考

- https://github.com/rxaviers/async-pool
