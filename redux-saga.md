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

- Task
描述generator执行的上下文，通过fork函数来创建。（后面讲proc的时候会展开说

- Effect
plain object，用来描述需要saga处理的指令，例如take，put，call，fork等。

- Channel
> channel是对事件源的抽象，作用是先注册一个take方法，当put触发时，执行一次take方法，然后销毁他。

- Schedule


## Fork Model

Saga的整个执行模型呈现为一棵有多个分支的树，rootSaga为根，每调用一次fork effect就会分产生一个子节点(task)。这些节点之间是并行执行的。

举个例子：

```JavaScript
export default function* root() {
  yield fork(startup)
  yield fork(nextRedditChange)
  yield fork(invalidateReddit)
}
```

上面的代码摘自saga源码`examples/async/src/index.js`, 对应的"执行模型"即为：`root`为根节点，`startup`/`nextRedditChange`/`invalidateReddit`为`root`的子节点。

那么你可能会问，浏览器明明是单线程模型，怎么并行？

这里先说结论，saga上下文中说的并行，并非指多线程间的并行，而是fork effect所产生的task之间的并行。
当一个task遇到阻塞的effect，它会把执行权交还给它的parent task，自己继续等待callback，并在callback中调用当前generator的next方法。

这么说比较难懂，直接看实现。

首先来看创建task的入口，proc函数（`src/internal/proc.js`）:

```JavaScript
function proc(iterator, subscribe, dispatch, getState, parentContext, options, parentEffectId, name, cont) {
    // parent task: 创建一个新的task object，包含当前iterator，即parent iterator的执行上下文
    const task = newTask(parentEffectId, name, iterator, cont)

    // mainTask: 用来控制当前generator的main flow（mainTask本身并没有generator相关的信息，只是用来控制flow
    const mainTask = {
        name,
        cancel: cancelMain,
        isRunning: true
    }

    // taskQueue: 一个包含mainTask 以及forked tasks的队列
    // 当parent task被取消`task.cancel()`，taskQueue里所有的task都会被取消(cancelAll)；
    // 当forked tasks中有没有catch住的error被throw出来，parent task会被暂停(taskQueue.abort)；
    // 当parent task结束(result.done)，mainTask将从taskQueue里移除（它的fork tasks可能还存活）
    const taskQueue = forkQueue(name, mainTask, end)

    // 调用当前generator的next方法，并使用正确的effect runner来处理yield返回的Effect
    next();

    return task;
}
```

```JavaScript

```