---
title: setState
description: React中, setState是同步还是异步
keywords: [react, setState, react同步, react异步]
order: 0
---

# React 中, setState 是同步还是异步

所谓同步还是异步指的是调用`setState`之后是否马上能得到最新的 state.

不仅仅是`setState`了, 在对 function 类型组件中的 hook 进行操作时也是一样, 最终决定`setState`是同步渲染还是异步渲染的关键因素是`ReactFiberWorkLoop`工作空间的执行上下文以及当前更新优先级.

> v17 → v19 重大变化:
>
> 1. **`Legacy` 模式与`unstable_batchedUpdates`已删除**: v18 起所有应用必须通过`createRoot`创建, 全部走 Concurrent 渲染. 因此"在`legacy`模式下设置 state 立刻同步 flush"这种行为已经不存在.
> 2. **自动批处理(Automatic Batching)**: v18 起所有更新(无论事件回调、Promise、setTimeout、原生事件中)都通过 microtask 自动批处理, 默认都"异步". 也就是说 v17 中通过`setTimeout / Promise.then`实现"同步 setState"的写法在 v18+ 中已经失效.
> 3. **`flushSync`是唯一的"强制同步"出口**: 业务代码如果确实需要在某个 setState 之后立刻拿到最新 DOM, 必须显式调用`flushSync(() => setState(...))`.
> 4. **同步入口**: 源码层判断同步的依据从 v17 的`expirationTime === Sync && (executionContext & LegacyUnbatchedContext)`变成了 v19 的`includesSyncLane(lane)`和`processRootScheduleInMicrotask`微任务末尾的`performSyncWorkOnRoot`(或更准确地说, `performWorkOnRoot(root, SyncLane, ...)`).

具体代码如下 (v19 简化版):

```js
// v19: ReactFiberWorkLoop.js (节选)
export function scheduleUpdateOnFiber(
  root: FiberRoot,
  fiber: Fiber,
  lane: Lane,
) {
  // 1. 标记 root 上的 pendingLanes
  markRootUpdated(root, lane);

  // 2. 标记 fiber.lanes / 父链 childLanes (v19 由 getRootForUpdatedFiber 内部处理)
  // ...

  // 3. 统一通过 ReactFiberRootScheduler 调度, 不再区分同步/异步入口
  ensureRootIsScheduled(root);

  // 4. v18+ 唯一的"立刻同步冲刷"入口
  //    - 必须当前没有正在渲染 / 没有 commit
  //    - 必须当前没有处于自动批处理上下文中
  //    - 必须是 flushSync() / discrete event 等场景显式提升了优先级
  if (
    executionContext === NoContext &&
    (fiber.mode & ConcurrentMode) === NoMode // 仅 Legacy 兼容路径, v19 已极少触发
  ) {
    resetRenderTimer();
    flushSyncWorkOnLegacyRootsOnly();
  }
}
```

可以看到, v19 中"同步还是异步"已经基本退化为一个统一答案: **默认全部异步(microtask 批处理), 显式 `flushSync` 才同步**.

## 结论 (v18+)

| 写法                                                                 | 行为                                                                     |
| -------------------------------------------------------------------- | ------------------------------------------------------------------------ |
| 事件回调里调`setState`                                               | **异步**(自动批处理)                                                     |
| `setTimeout / Promise.then / requestAnimationFrame`里调`setState`    | **异步**(自动批处理)                                                     |
| 原生 DOM 事件回调里调`setState`(addEventListener 添加)               | **异步**(自动批处理)                                                     |
| `flushSync(() => setState(...))`                                     | **同步**(立刻 commit, 跳过 microtask 等待)                               |
| 渲染期间(在 render 函数中)调当前 fiber 的`setState`                  | render-phase update, 在当前 render 重新进入循环立刻生效                  |
| Class 组件`componentDidMount / componentDidUpdate`里同步调`setState` | 不会立即重新 render, 当前 commit 完成后再进入下一次 commit; 多次调用合并 |

底层判断条件:

1. `lane`是否包含`SyncLane`(或`includesBlockingLane`)?
2. 当前是否处于`executionContext & (RenderContext | CommitContext)`?
3. 是否处于`flushSync`的同步窗口中(`ReactSharedInternals.T === null`且 currentUpdatePriority 是`DiscreteEventPriority`)?

满足前两个条件失败则进入异步路径, 由`ReactFiberRootScheduler`在 microtask 末尾决定如何 flush.

## 演示示例 (v18+)

```jsx
import React, { useState } from 'react';
import { flushSync } from 'react-dom';

export default function App() {
  const [count, setCount] = useState(0);

  // v18+: 自动批处理, 这里 console.log 仍然是旧的 count
  const onClick = () => {
    setCount(count + 1);
    console.log('after setCount, count =', count); // 仍是旧值
  };

  // v18+: setTimeout 中也是异步, 与 v17 行为不同
  const onClickAsync = () => {
    setTimeout(() => {
      setCount((c) => c + 1);
      console.log('after setCount (setTimeout), count =', count); // 仍是旧值
    });
  };

  // v18+ 显式同步: 立刻冲刷, console.log 时 DOM 已更新
  const onClickFlushSync = () => {
    flushSync(() => {
      setCount((c) => c + 1);
    });
    console.log('after flushSync, count =', count); // 在 React 内仍然是旧的闭包变量, 但 DOM 已经更新
  };

  return (
    <div>
      <p>count = {count}</p>
      <button onClick={onClick}>默认 +1</button>
      <button onClick={onClickAsync}>setTimeout 内 +1</button>
      <button onClick={onClickFlushSync}>flushSync 内 +1</button>
    </div>
  );
}
```

[![Edit setstate-v19](https://codesandbox.io/static/img/play-codesandbox.svg)](https://codesandbox.io/s/boring-faraday-m7jtx?fontsize=14&hidenavigation=1&theme=dark)

> 注: v17 时期"`setTimeout`内的 setState 是同步"的著名"奇迹"现象, 在 v18+ 中已经被自动批处理统一为异步, 这是社区里讨论最多的破坏性变化之一.
