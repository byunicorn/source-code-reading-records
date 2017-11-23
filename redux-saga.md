[浅析redux-saga实现原理](https://zhuanlan.zhihu.com/p/30098155)

proc里创建了"三个task":

```JavaScript
  // parent task(?)，会被return出去
  const task = newTask(parentEffectId, name, iterator, cont)
  // 当前Generator的main flow
  const mainTask = { name, cancel: cancelMain, isRunning: true }
  // fork task queue
  const taskQueue = forkQueue(name, mainTask, end)
```


```JavaScript
  function newTask(id, name, iterator, cont) {
    iterator._deferredEnd = null
    return {
      [TASK]: true,
      id,
      name,
      get done() {
        if (iterator._deferredEnd) {
          return iterator._deferredEnd.promise
        } else {
          const def = deferred()
          iterator._deferredEnd = def
          if (!iterator._isRunning) {
            iterator._error ? def.reject(iterator._error) : def.resolve(iterator._result)
          }
          return def.promise
        }
      },
      cont,
      joiners: [],
      cancel,
      isRunning: () => iterator._isRunning,
      isCancelled: () => iterator._isCancelled,
      isAborted: () => iterator._isAborted,
      result: () => iterator._result,
      error: () => iterator._error,
      setContext(props) {
        check(props, is.object, createSetContextWarning('task', props))
        object.assign(taskContext, props)
      },
    }
```
