[浅析redux-saga实现原理](https://zhuanlan.zhihu.com/p/30098155)

proc里创建了"三个task":

```JavaScript
  // parent task(?)，包含当前iterator的执行上下文
  const task = newTask(parentEffectId, name, iterator, cont)
  // 当前Generator的main flow
  const mainTask = { name, cancel: cancelMain, isRunning: true }
  // fork task queue
  const taskQueue = forkQueue(name, mainTask, end)
  // next会调用runEffect 方法来处理yeild的返回值 / effect
  next();
  
  return task;
```

```JavaScript
    // Completion callback passed to the appropriate effect runner
    function currCb(res, isErr) {
      if (effectSettled) {
        return
      }

      effectSettled = true
      cb.cancel = noop // defensive measure
      if (sagaMonitor) {
        isErr ? sagaMonitor.effectRejected(effectId, res) : sagaMonitor.effectResolved(effectId, res)
      }
      cb(res, isErr)
    }
```

```JavaScript
  function runForkEffect({ context, fn, args, detached }, effectId, cb) {
    const taskIterator = createTaskIterator({ context, fn, args })

    try {
      suspend()
      const task = proc(
        taskIterator,
        subscribe,
        dispatch,
        getState,
        taskContext,
        options,
        effectId,
        fn.name,
        detached ? null : noop,
      )

      if (detached) {
        cb(task)
      } else {
        if (taskIterator._isRunning) {
          taskQueue.addTask(task)
          cb(task)
        } else if (taskIterator._error) {
          taskQueue.abort(taskIterator._error)
        } else {
          cb(task)
        }
      }
    } finally {
      flush()
    }
    // Fork effects are non cancellables
  }
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
