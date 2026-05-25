---
title: 调度原理
group: 运行核心
order: 3
---

# React 调度原理(scheduler)

在 React 运行时中, 调度中心(位于`scheduler`包), 是整个 React 运行时的中枢(其实是心脏), 所以理解`scheduler`调度, 就基本把握了 React 的命门.

在深入分析之前, 建议回顾一下往期与`scheduler`相关的文章(这 3 篇文章不长, 共 10 分钟能浏览完):

- [React 工作循环](./workloop.md): 从宏观的角度介绍 React 体系中两个重要的循环, 其中`任务调度循环`就是本文的主角.
- [reconciler 运作流程](./reconciler-workflow.md): 从宏观的角度介绍了`react-reconciler`包的核心作用, 并把`reconciler`分为了 4 个阶段. 其中第 2 个阶段`注册调度任务`串联了`scheduler`包和`react-reconciler`包, 其实就是`任务调度循环`中的一个任务(`task`).
- [React 中的优先级管理](./priority.md): 介绍了 React 体系中的优先级管理. v19 中只剩 `Lane / EventPriority / SchedulerPriority` 三层, 其中`SchedulerPriority`控制`任务调度循环`中循环的顺序.

了解上述基础知识之后, 再谈`scheduler`原理, 其实就是在大的框架下去添加实现细节, 相对较为容易. 下面就正式进入主题.

> v17 → v19 关键变化:
>
> - v18 起将原`SchedulerHostConfig.default.js`合并到[`Scheduler.js`](https://github.com/facebook/react/blob/v19.2.6/packages/scheduler/src/forks/Scheduler.js)中, host 调度代码不再单独分文件;
> - 浏览器宿主调用方式增加了`scheduler.postTask`(在支持的浏览器中优先使用); 否则降级到`MessageChannel`;
> - `yieldInterval`重命名为`frameInterval`(默认仍为 5ms), 同时新增`continuousYieldTime`、`maxYieldInterval = 300ms`等阈值, 在`shouldYieldToHost`中综合使用;
> - 节流防抖路径下沉到[`ReactFiberRootScheduler.js`](https://github.com/facebook/react/blob/v19.2.6/packages/react-reconciler/src/ReactFiberRootScheduler.js), 不再写在`ensureRootIsScheduled`第一行.

## 调度实现

`调度中心`最核心的代码, 在[`Scheduler.js`](https://github.com/facebook/react/blob/v19.2.6/packages/scheduler/src/forks/Scheduler.js)中.

### 内核

该 js 文件中, host 相关的核心逻辑集中在一组本地变量与函数中:

```js
// 用于与 Scheduler.js 主体通信的几个全局开关
let scheduledHostCallback = null; // 当前需要执行的回调 (= flushWork)
let isMessageLoopRunning = false; // 是否已经在事件循环中等待执行
let taskTimeoutID = -1; // 延时任务的 timer id

// 时间切片相关
let frameInterval = 5; // v19: 时间切片默认 5ms (原 yieldInterval)
const continuousInputInterval = 50; // 连续输入容忍上限
const maxInterval = 300; // 极端兜底
let startTime = -1; // 当前切片开始时间
let needsPaint = false;
const scheduling = typeof navigator !== 'undefined' && navigator.scheduling;

// 对外暴露的关键能力
// requestHostCallback / cancelHostCallback / requestHostTimeout / cancelHostTimeout
// shouldYieldToHost / requestPaint / getCurrentTime
```

我们知道 react 可以在 nodejs 环境中使用, 所以在不同的 js 执行环境中, 这些函数的实现会有区别. 下面基于普通浏览器环境逐一分析:

1. **调度相关: 请求或取消调度**

```js
// v19: 接收 MessageChannel 消息 (或 scheduler.postTask 回调), 签名只剩 currentTime, 不再传 hasTimeRemaining
const performWorkUntilDeadline = () => {
  if (scheduledHostCallback !== null) {
    const currentTime = getCurrentTime();
    // 重置切片起点
    startTime = currentTime;
    try {
      const hasMoreWork = scheduledHostCallback(currentTime);
      if (!hasMoreWork) {
        isMessageLoopRunning = false;
        scheduledHostCallback = null;
      } else {
        // 还有任务: 通过 schedulePerformWorkUntilDeadline 触发下一次切片
        schedulePerformWorkUntilDeadline();
      }
    } catch (error) {
      schedulePerformWorkUntilDeadline();
      throw error;
    }
  } else {
    isMessageLoopRunning = false;
  }
  needsPaint = false;
};

// v19 中根据宿主能力选择不同的调度通道
let schedulePerformWorkUntilDeadline;
if (typeof localSetImmediate === 'function') {
  // node / 老 IE: setImmediate
  schedulePerformWorkUntilDeadline = () =>
    localSetImmediate(performWorkUntilDeadline);
} else if (typeof MessageChannel !== 'undefined') {
  // 现代浏览器: MessageChannel (宏任务, 不会被 microtask 队头阻塞)
  const channel = new MessageChannel();
  const port = channel.port2;
  channel.port1.onmessage = performWorkUntilDeadline;
  schedulePerformWorkUntilDeadline = () => port.postMessage(null);
} else {
  // 兜底: setTimeout 0
  schedulePerformWorkUntilDeadline = () =>
    localSetTimeout(performWorkUntilDeadline, 0);
}

// 请求回调
function requestHostCallback(callback) {
  scheduledHostCallback = callback;
  if (!isMessageLoopRunning) {
    isMessageLoopRunning = true;
    schedulePerformWorkUntilDeadline();
  }
}
function cancelHostCallback() {
  scheduledHostCallback = null;
}
```

很明显, 请求回调之后`scheduledHostCallback = callback`, 然后通过`MessageChannel`(或宿主等价物)发消息的方式触发`performWorkUntilDeadline`函数, 最后执行回调`scheduledHostCallback`.

此处需要注意: `MessageChannel`在浏览器事件循环中属于`宏任务`, 所以调度中心永远是`异步执行`回调函数. v19 之所以坚持使用宏任务通道而不是 microtask, 是为了在每次切片之间留出渲染/输入的窗口.

> **关于`scheduler.postTask`**: 在支持 [Prioritized Task Scheduling API](https://wicg.github.io/scheduling-apis/) 的浏览器中, react 19 提供了另一个 fork [`SchedulerPostTask.js`](https://github.com/facebook/react/blob/v19.2.6/packages/scheduler/src/forks/SchedulerPostTask.js), 使用浏览器原生的`scheduler.postTask({priority})`代替 MessageChannel. 但默认导出的`Scheduler.js`仍然以 MessageChannel 为主.

2. **时间切片(`time slicing`)相关: 让出主线程**

```js
function getCurrentTime() {
  return localPerformance.now();
}

// v19 中关键判断: 是否让出主线程
function shouldYieldToHost() {
  const timeElapsed = getCurrentTime() - startTime;
  if (timeElapsed < frameInterval) {
    // 时间切片还没用完, 继续工作
    return false;
  }
  // 切片已超出, 看更多条件再决定
  if (enableIsInputPending) {
    if (needsPaint) return true;
    if (timeElapsed < continuousInputInterval) {
      if (scheduling.isInputPending !== undefined) {
        return scheduling.isInputPending();
      }
    } else if (timeElapsed < maxInterval) {
      if (scheduling.isInputPending !== undefined) {
        return scheduling.isInputPending(continuousOptions);
      }
    } else {
      return true; // 兜底: 超过 300ms 强制让出
    }
  }
  return true;
}

function requestPaint() {
  needsPaint = true;
}

// 强制设置帧率 (仅用于测试)
function forceFrameRate(fps) {
  if (fps < 0 || fps > 125) {
    console.error('forceFrameRate takes a positive int between 0 and 125');
    return;
  }
  frameInterval = fps > 0 ? Math.floor(1000 / fps) : 5;
}
```

这几个函数代码都很简洁:

- `frameInterval`默认是`5ms`, 只能通过`forceFrameRate`函数修改(v19 中仅 facebook 内部某些场景使用).
- 切片超时后, react 进一步借助 `navigator.scheduling.isInputPending()` 这一 facebook 官方贡献给 Chromium 的 api(已纳入 W3C 标准, [具体解释](https://engineering.fb.com/2019/04/22/developer-tools/isinputpending-api/)) 判断是否需要让位给输入/绘制.
- `continuousInputInterval = 50ms`、`maxInterval = 300ms` 用于在持续渲染场景下做更宽松的让出, 避免对每帧都过度阻塞.

分析到这里, 可以得到调度中心的内核实现图:

![](../../snapshots/scheduler/core.png)

> 注: 此图绘制于 v17 时期, 当时调度通道仅有`MessageChannel`一条; v19 中可选`postTask / MessageChannel / setImmediate / setTimeout`四条, 但消息循环的整体结构(请求 → 分片 → 回调 → 可选继续)保持一致.

### 任务队列管理

通过上文的分析, 我们已经知道请求和取消调度的实现原理. 调度的目的是为了消费任务, 接下来就具体分析任务队列是如何管理与实现的.

在[`Scheduler.js`](https://github.com/facebook/react/blob/v19.2.6/packages/scheduler/src/forks/Scheduler.js)中, 维护了两个队列, 任务队列管理就是围绕`taskQueue`展开:

```js
// Tasks are stored on a min heap
var taskQueue = [];
var timerQueue = [];
```

注意:

- `taskQueue`是一个小顶堆数组, 关于堆排序的详细解释, 可以查看[React 算法之堆排序](../algorithm/heapsort.md).
- `timerQueue`存储延时任务(`unstable_scheduleCallback(priority, cb, { delay })`), 当`startTime`到来时由`advanceTimers`迁移到`taskQueue`. 与 v17 不同, v18+ 中 react-reconciler 偶尔会使用延时(主要用于 `TransitionTracing` 等场景).

#### 创建任务

在`unstable_scheduleCallback`函数中([源码链接](https://github.com/facebook/react/blob/v19.2.6/packages/scheduler/src/forks/Scheduler.js)):

```js
// 省略部分无关代码
function unstable_scheduleCallback(priorityLevel, callback, options) {
  // 1. 获取当前时间
  var currentTime = getCurrentTime();
  var startTime;
  if (typeof options === 'object' && options !== null) {
    var delay = options.delay;
    startTime =
      typeof delay === 'number' && delay > 0
        ? currentTime + delay
        : currentTime;
  } else {
    startTime = currentTime;
  }
  // 2. 根据传入的优先级, 设置任务的过期时间 expirationTime
  var timeout;
  switch (priorityLevel) {
    case ImmediatePriority:
      // v18+: -1 表示已过期, 立即执行
      timeout = -1;
      break;
    case UserBlockingPriority:
      timeout = userBlockingPriorityTimeout; // 250ms
      break;
    case IdlePriority:
      timeout = maxSigned31BitInt; // 几乎永不过期
      break;
    case LowPriority:
      timeout = lowPriorityTimeout; // 10000ms
      break;
    case NormalPriority:
    default:
      timeout = normalPriorityTimeout; // 5000ms
      break;
  }
  var expirationTime = startTime + timeout;
  // 3. 创建新任务
  var newTask = {
    id: taskIdCounter++,
    callback,
    priorityLevel,
    startTime,
    expirationTime,
    sortIndex: -1,
  };
  if (startTime > currentTime) {
    // 延时任务: 放入 timerQueue
    newTask.sortIndex = startTime;
    push(timerQueue, newTask);
    if (peek(taskQueue) === null && newTask === peek(timerQueue)) {
      if (isHostTimeoutScheduled) cancelHostTimeout();
      else isHostTimeoutScheduled = true;
      requestHostTimeout(handleTimeout, startTime - currentTime);
    }
  } else {
    newTask.sortIndex = expirationTime;
    // 4. 加入任务队列
    push(taskQueue, newTask);
    // 5. 请求调度
    if (!isHostCallbackScheduled && !isPerformingWork) {
      isHostCallbackScheduled = true;
      requestHostCallback(flushWork);
    }
  }
  return newTask;
}
```

逻辑很清晰(在注释中已标明), 重点分析`task`对象的各个属性:

```js
var newTask = {
  id: taskIdCounter++, // id: 一个自增编号
  callback, // callback: 传入的回调函数
  priorityLevel, // priorityLevel: 优先级等级
  startTime, // startTime: 创建task时的当前时间 (或 currentTime + delay)
  expirationTime, // expirationTime: task的过期时间, 优先级越高 expirationTime = startTime + timeout 越小
  sortIndex: -1,
};
newTask.sortIndex = expirationTime; // sortIndex: 排序索引, 全等于过期时间. 保证过期时间越小, 越紧急的任务排在最前面
```

#### 消费任务

创建任务之后, 最后请求调度`requestHostCallback(flushWork)`(`创建任务`源码中的第 5 步), `flushWork`函数作为参数被传入调度中心内核等待回调. `requestHostCallback`函数在上文调度内核中已经介绍过了, 在调度中心中, 只需下一个事件循环就会执行回调, 最终执行`flushWork`.

```js
// v19: flushWork 签名只剩 initialTime
function flushWork(initialTime) {
  // 1. 做好全局标记, 表示现在已经进入调度阶段
  isHostCallbackScheduled = false;
  if (isHostTimeoutScheduled) {
    isHostTimeoutScheduled = false;
    cancelHostTimeout();
  }
  isPerformingWork = true;
  const previousPriorityLevel = currentPriorityLevel;
  try {
    // 2. 循环消费队列
    return workLoop(initialTime);
  } finally {
    // 3. 还原全局标记
    currentTask = null;
    currentPriorityLevel = previousPriorityLevel;
    isPerformingWork = false;
  }
}
```

`flushWork`中调用了`workLoop`. 队列消费的主要逻辑是在`workLoop`函数中, 这就是[React 工作循环](./workloop.md)一文中提到的`任务调度循环`.

```js
// v19 简化版
function workLoop(initialTime) {
  let currentTime = initialTime;
  advanceTimers(currentTime); // 把到期的延时任务从 timerQueue 搬到 taskQueue
  currentTask = peek(taskQueue);
  while (currentTask !== null) {
    if (currentTask.expirationTime > currentTime && shouldYieldToHost()) {
      // 当前任务未过期, 但切片已用完: 让出主线程
      break;
    }
    const callback = currentTask.callback;
    if (typeof callback === 'function') {
      currentTask.callback = null;
      currentPriorityLevel = currentTask.priorityLevel;
      const didUserCallbackTimeout = currentTask.expirationTime <= currentTime;
      // 执行回调
      const continuationCallback = callback(didUserCallbackTimeout);
      currentTime = getCurrentTime();
      if (typeof continuationCallback === 'function') {
        // 产生了连续回调(如 fiber 树太大, 出现了中断渲染), 保留 currentTask
        currentTask.callback = continuationCallback;
        advanceTimers(currentTime);
        return true;
      } else {
        if (currentTask === peek(taskQueue)) {
          pop(taskQueue);
        }
        advanceTimers(currentTime);
      }
    } else {
      // 任务被取消(currentTask.callback = null), 移出队列
      pop(taskQueue);
    }
    currentTask = peek(taskQueue);
  }
  // 如果 taskQueue 没有清空, 返回 true; 否则尝试启动延时计时器
  if (currentTask !== null) {
    return true;
  } else {
    const firstTimer = peek(timerQueue);
    if (firstTimer !== null) {
      requestHostTimeout(handleTimeout, firstTimer.startTime - currentTime);
    }
    return false;
  }
}
```

`workLoop`就是一个大循环, 虽然代码也不多, 但是非常精髓, 在此处实现了`时间切片(time slicing)`和`fiber树的可中断渲染`. 这 2 大特性的实现, 都集中于这个`while`循环.

每一次`while`循环的退出就是一个时间切片, 深入分析`while`循环的退出条件:

1. 队列被完全清空: 这种情况就是很正常的情况, 一气呵成, 没有遇到任何阻碍.
2. 执行超时: 在消费`taskQueue`时, 在执行`task.callback`之前, 都会检测是否超时, 所以超时检测是以`task`为单位.
   - 如果某个`task.callback`执行时间太长(如: `fiber树`很大, 或逻辑很重)也会造成超时
   - 所以在执行`task.callback`过程中, 也需要一种机制检测是否超时, 如果超时了就立刻暂停`task.callback`的执行.

#### 时间切片原理

消费任务队列的过程中, 可以消费`1~n`个 task, 甚至清空整个 queue. 但是在每一次具体执行`task.callback`之前都要进行超时检测, 如果超时可以立即退出循环并等待下一次调用.

#### 可中断渲染原理

在时间切片的基础之上, 如果单个`task.callback`执行时间就很长(假设 200ms). 就需要`task.callback`自己能够检测是否超时, 所以在 fiber 树构造过程中, 每构造完成一个单元, 都会检测一次`shouldYield()`(见[`ReactFiberWorkLoop.js`](https://github.com/facebook/react/blob/v19.2.6/packages/react-reconciler/src/ReactFiberWorkLoop.js)). 一旦超时, 就退出`fiber树构造循环`, 并返回一个新的回调函数(就是此处的`continuationCallback`)并等待下一次回调继续未完成的`fiber树构造`.

## 节流防抖 {#throttle-debounce}

通过上文的分析, 已经覆盖了`scheduler`包中的核心原理. 现在再次回到`react-reconciler`包中, 在调度过程中的关键路径中, 我们还需要理解一些细节.

在[reconciler 运作流程](./reconciler-workflow.md)中总结的 4 个阶段中, `注册调度任务`属于第 2 个阶段. v18 起这部分逻辑被拆到[`ReactFiberRootScheduler.js`](https://github.com/facebook/react/blob/v19.2.6/packages/react-reconciler/src/ReactFiberRootScheduler.js)中, 节流防抖也在此处:

```js
// v19 ReactFiberRootScheduler.js (简化版)
function scheduleTaskForRootDuringMicrotask(root, currentTime) {
  const existingCallbackNode = root.callbackNode;
  const nextLanes = getNextLanes(
    root,
    root === workInProgressRoot ? workInProgressRootRenderLanes : NoLanes,
  );
  if (nextLanes === NoLanes) {
    // 没有任何更新, 把已注册的 callback 撤回
    if (existingCallbackNode !== null) cancelCallback(existingCallbackNode);
    root.callbackNode = null;
    root.callbackPriority = NoLane;
    return NoLane;
  }

  // 节流防抖
  const newCallbackPriority = getHighestPriorityLane(nextLanes);
  const existingCallbackPriority = root.callbackPriority;
  if (
    existingCallbackPriority === newCallbackPriority &&
    !(__DEV__ && /* ...act 例外 */ false)
  ) {
    // 优先级一致, 复用已有 callback
    return newCallbackPriority;
  } else if (existingCallbackNode !== null) {
    // 优先级变了, 取消旧的, 注册新的
    cancelCallback(existingCallbackNode);
  }

  // 注册新 callback
  let schedulerPriorityLevel;
  switch (lanesToEventPriority(nextLanes)) {
    case DiscreteEventPriority:
      schedulerPriorityLevel = ImmediateSchedulerPriority;
      break;
    case ContinuousEventPriority:
      schedulerPriorityLevel = UserBlockingSchedulerPriority;
      break;
    case DefaultEventPriority:
      schedulerPriorityLevel = NormalSchedulerPriority;
      break;
    case IdleEventPriority:
      schedulerPriorityLevel = IdleSchedulerPriority;
      break;
    default:
      schedulerPriorityLevel = NormalSchedulerPriority;
      break;
  }
  // v19: 回调统一为 performWorkOnRoot(root, lanes, forceSync), 此处由 Scheduler 异步调度, forceSync=false
  const newCallbackNode = scheduleCallback(
    schedulerPriorityLevel,
    performWorkOnRoot.bind(null, root),
  );
  root.callbackPriority = newCallbackPriority;
  root.callbackNode = newCallbackNode;
  return newCallbackPriority;
}
```

正常情况下, 该函数会与`scheduler`包通信, 最后注册一个`task`并等待回调:

1. 在`task`注册完成之后, 会设置`fiberRoot`对象上的属性(`fiberRoot`是 react 运行时中的重要全局对象, 可参考[React 应用的启动过程](./bootstrap.md#create-global-obj)), 代表现在已经处于调度进行中
2. 再次进入时(比如连续 2 次`setState`, 第 2 次`setState`同样会触发`reconciler 运作流程`中的调度阶段), 如果发现处于调度中, 则需要一些节流和防抖措施, 进而保证调度性能:
   1. 节流(判断条件: `existingCallbackPriority === newCallbackPriority`, 新旧更新的优先级相同, 如连续多次执行`setState`), 则无需注册新`task`(继续沿用上一个优先级相同的`task`), 直接退出调用.
   2. 防抖(判断条件: `existingCallbackPriority !== newCallbackPriority`, 新旧更新的优先级不同), 则取消旧`task`, 重新注册新`task`.

> v18 后此函数被包装在 microtask 中调用, 同一 tick 内即使触发数百次 setState, 也只会在 microtask 末尾走一次"决策 → 注册"流程, 这是 React 18 [automatic batching](https://react.dev/blog/2022/03/29/react-v18#new-feature-automatic-batching) 在调度层的核心实现.

## 总结

本节主要分析了`scheduler`包中`调度原理`, 也就是`React两大工作循环`中的`任务调度循环`. 并介绍了`时间切片`和`可中断渲染`等特性在`任务调度循环`中的实现. `scheduler`包是`React`运行时的心脏, 为了提升调度性能, 注册`task`之前, 在`react-reconciler`包中做了 microtask 决策、节流和防抖等措施.
