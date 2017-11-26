# TL;TR;
- ES6 Generator：[function* - JavaScript | MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/function*)
- Redux-Saga Doc：[Read Me · Redux-Saga](https://redux-saga.js.org/)
- [浅析redux-saga实现原理](https://zhuanlan.zhihu.com/p/30098155)

开始后续的阅读之前，有几点需要注意：
- 需要有Generator，Saga的基本上下文，没有的话请戳上面给出的几个链接
- 下面的代码分析基于版本： [v0.15.6](https://github.com/redux-saga/redux-saga/tree/v0.15.6)

# File Structure

```
src
├── effects.js
├── index.js
├── internal
│   ├── buffers.js    // 
│   ├── channel.js    // 
│   ├── channels-trans-table.png
│   ├── io.js         // 描述了saga中各种Effect(plain object)
│   ├── middleware.js // 定义了createSagaMiddleware function
│   ├── proc.js       // 包含主要逻辑
│   ├── runSaga.js    // 定义了sagaMiddleware.run function
│   ├── sagaHelpers
│   │   ├── fsmIterator.js 
│   │   ├── index.js
│   │   ├── takeEvery.js
│   │   ├── takeLatest.js
│   │   └── throttle.js
│   ├── scheduler.js  // 
│   └── utils.js      // general的辅助函数
└── utils.js
```

# Basic Concept
[名词解释 | Redux-saga 中文文档](http://leonshi.com/redux-saga-in-chinese/docs/Glossary.html)

- Effect
plain object，用来描述需要saga处理的指令，例如take，put，call，fork等

- Task
描述generator执行的上下文，通过fork函数来创建

- Channel

# Source Code

## `proc.js`

首先来看proc.js里面最主要的函数`function proc(iterator, subscribe, dispatch, getState, parentContext, options, parentEffectId, name, cont)`。

proc里创建了"三个task"，Parent Task / Main Task / Forked Tasks:

```JavaScript

// Parent task = Main Tasks + Forked Tasks 
// taskQueue用来记录parent task和它的forked tasks
// forked tasks默认会被attach到它的parent task上 
// main task是当前generator的main flow
// 整个执行的模型呈现为一棵有多个分支的树

// 一个parent task有以下几个定义：

// 如果它所有的forks都结束/取消了，那么它就结束了
// 如果它被取消了，它的所有forks都会被取消
// 如果从forks里有没有被catch住的error throw出来，那么它会被暂停
// 如果它结束，那么返回的是main task的返回值

// 创建一个新的task object，包含当前iterator的执行上下文
const task = newTask(parentEffectId, name, iterator, cont) 
const mainTask = { name, cancel: cancelMain, isRunning: true }

// fork task queue
const taskQueue = forkQueue(name, mainTask, end)

// next会调用runEffect 方法来处理yeild的返回值 / effect
next();

return task;
```

cb是上文的next
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

cb是上文的currCb
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
