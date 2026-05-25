---
title: Hook 原理(状态Hook)
group: 状态管理
order: 2
---

# Hook 原理(状态 Hook)

首先回顾一下前文[Hook 原理(概览)](./hook-summary.md), 其主要内容有:

1. `function`类型的`fiber`节点, 它的处理函数是[updateFunctionComponent](https://github.com/facebook/react/blob/v19.2.6/packages/react-reconciler/src/ReactFiberBeginWork.js), 其中再通过[renderWithHooks](https://github.com/facebook/react/blob/v19.2.6/packages/react-reconciler/src/ReactFiberHooks.js)调用`function`.
2. 在`function`中, 通过`Hook Api`(如: `useState, useEffect`)创建`Hook`对象.
   - `状态Hook`实现了状态持久化(等同于`class组件`维护`fiber.memoizedState`).
   - `副作用Hook`则实现了维护`fiber.flags`,并提供`副作用回调`(类似于`class组件`的生命周期回调)
3. 多个`Hook`对象构成一个`链表结构`, 并挂载到`fiber.memoizedState`之上.
4. `fiber树`更新阶段, 把`current.memoizedState`链表上的所有`Hook`按照顺序克隆到`workInProgress.memoizedState`上, 实现数据的持久化.

在此基础之上, 本节将深入分析`状态Hook`的特性和实现原理.

## 创建 Hook

在`fiber`初次构造阶段, [useState](https://github.com/facebook/react/blob/v19.2.6/packages/react-reconciler/src/ReactFiberHooks.js)对应源码[mountState](https://github.com/facebook/react/blob/v19.2.6/packages/react-reconciler/src/ReactFiberHooks.js), [useReducer](https://github.com/facebook/react/blob/v19.2.6/packages/react-reconciler/src/ReactFiberHooks.js)对应源码[mountReducer](https://github.com/facebook/react/blob/v19.2.6/packages/react-reconciler/src/ReactFiberHooks.js)

> v18 起: `dispatchAction`被拆分为 2 个函数, `useState`的 dispatcher 是`dispatchSetState`, `useReducer`的 dispatcher 是`dispatchReducerAction`. 这样做的原因是: `useState`内部知道 reducer 就是`basicStateReducer`, 可以提前做`eagerState`优化; 而`useReducer`使用外部 reducer, 走完整路径.

`mountState` (v19):

```js
function mountState<S>(
  initialState: (() => S) | S,
): [S, Dispatch<BasicStateAction<S>>] {
  // 1. 创建hook
  const hook = mountStateImpl(initialState);
  // 2. queue 已在 mountStateImpl 中初始化, 取出
  const queue = hook.queue;
  // 3. 设置 hook.dispatch (v18 起改用 dispatchSetState)
  const dispatch: Dispatch<BasicStateAction<S>> = (dispatchSetState.bind(
    null,
    currentlyRenderingFiber,
    queue,
  ): any);
  queue.dispatch = dispatch;
  // 4. 返回[当前状态, dispatch函数]
  return [hook.memoizedState, dispatch];
}

function mountStateImpl<S>(initialState: (() => S) | S): Hook {
  const hook = mountWorkInProgressHook();
  if (typeof initialState === 'function') initialState = initialState();
  hook.memoizedState = hook.baseState = initialState;
  hook.queue = {
    pending: null,
    lanes: NoLanes,
    dispatch: null,
    lastRenderedReducer: basicStateReducer,
    lastRenderedState: (initialState: any),
  };
  return hook;
}
```

`mountReducer` (v19):

```js
function mountReducer<S, I, A>(
  reducer: (S, A) => S,
  initialArg: I,
  init?: (I) => S,
): [S, Dispatch<A>] {
  // 1. 创建hook
  const hook = mountWorkInProgressHook();
  let initialState;
  if (init !== undefined) {
    initialState = init(initialArg);
  } else {
    initialState = ((initialArg: any): S);
  }
  // 2. 初始化hook的属性
  hook.memoizedState = hook.baseState = initialState;
  const queue: UpdateQueue<S, A> = {
    pending: null,
    lanes: NoLanes,
    dispatch: null,
    lastRenderedReducer: reducer,
    lastRenderedState: (initialState: any),
  };
  hook.queue = queue;
  // 3. 设置 hook.dispatch (v18 起改用 dispatchReducerAction)
  const dispatch: Dispatch<A> = (queue.dispatch = (dispatchReducerAction.bind(
    null,
    currentlyRenderingFiber,
    queue,
  ): any));
  // 4. 返回[当前状态, dispatch函数]
  return [hook.memoizedState, dispatch];
}
```

`mountState`和`mountReducer`逻辑简单: 主要负责创建`hook`, 初始化`hook`的属性, 最后返回`[当前状态, dispatch函数]`.

主要不同点有两个:

1. **`hook.queue.lastRenderedReducer`**:

   - `mountState`使用的是内置的[basicStateReducer](https://github.com/facebook/react/blob/v19.2.6/packages/react-reconciler/src/ReactFiberHooks.js)

     ```js
     function basicStateReducer<S>(state: S, action: BasicStateAction<S>): S {
       return typeof action === 'function' ? action(state) : action;
     }
     ```

   - `mountReducer`使用的是外部传入自定义`reducer`.

2. **dispatch 函数(v18 起拆分)**:
   - `useState`绑定`dispatchSetState`: 内部已知 reducer 是`basicStateReducer`, 可立即 try-eager-bailout.
   - `useReducer`绑定`dispatchReducerAction`: 走完整调度路径, 不做 eager.

可见`mountState`是`mountReducer`的一种特殊情况, 即`useState`也是`useReducer`的一种特殊情况, 也是最简单的情况.

`useState`可以转换成`useReducer`:

```js
const [state, dispatch] = useState({ count: 0 });

// 等价于
const [state, dispatch] = useReducer(
  function basicStateReducer(state, action) {
    return typeof action === 'function' ? action(state) : action;
  },
  { count: 0 },
);

// 当需要更新state时, 有2种方式
dispatch({ count: 1 }); // 1.直接设置
dispatch((state) => ({ count: state.count + 1 })); // 2.通过回调函数设置
```

`useReducer`的[官网示例](https://zh-hans.reactjs.org/docs/hooks-reference.html#usereducer):

```js
const [state, dispatch] = useReducer(
  function reducer(state, action) {
    switch (action.type) {
      case 'increment':
        return { count: state.count + 1 };
      case 'decrement':
        return { count: state.count - 1 };
      default:
        throw new Error();
    }
  },
  { count: 0 },
);

// 当需要更新state时, 只有1种方式
dispatch({ type: 'decrement' });
```

可见, `useState`就是对`useReducer`的基本封装, 内置了一个特殊的`reducer`(后文不再区分`useState, useReducer`, 都以`useState`为例).`创建hook`之后返回值`[hook.memoizedState, dispatch]`中的`dispatch`实际上会调用`reducer`函数.

## 状态初始化

在`useState(initialState)`函数内部, 设置`hook.memoizedState = hook.baseState = initialState;`, 初始状态被同时保存到了`hook.baseState`,`hook.memoizedState`中.

1. `hook.memoizedState`: 当前状态
2. `hook.baseState`: `基础`状态, 作为合并`hook.baseQueue`的初始值(下文介绍).

最后返回`[hook.memoizedState, dispatch]`, 所以在`function`中使用的是`hook.memoizedState`.

## 状态更新

有如下代码:[![Edit hook-status](https://codesandbox.io/static/img/play-codesandbox.svg)](https://codesandbox.io/s/hook-status-vhlf8?fontsize=14&hidenavigation=1&theme=dark)

```jsx
import React, { useState } from 'react';
export default function App() {
  const [count, dispatch] = useState(0);
  return (
    <button
      onClick={() => {
        dispatch(1);
        dispatch(3);
        dispatch(2);
      }}
    >
      {count}
    </button>
  );
}
```

初次渲染时`count = 0`, 这时`hook`对象的内存状态如下:

![](../../snapshots/hook-state/initial-state.png)

点击`button`, 通过`dispatch`函数进行更新, `dispatch`实际就是[dispatchSetState](https://github.com/facebook/react/blob/v19.2.6/packages/react-reconciler/src/ReactFiberHooks.js)(v18 起替代了原来的`dispatchAction`):

```js
// v19 简化版
function dispatchSetState<S, A>(
  fiber: Fiber,
  queue: UpdateQueue<S, A>,
  action: A,
) {
  // 1. 计算优先级 (v18 起不再传 eventTime, 由 ReactFiberRootScheduler 在微任务中统一打时间戳)
  const lane = requestUpdateLane(fiber);

  // 2. 创建 update 对象 (v18+ 字段)
  const update: Update<S, A> = {
    lane,
    revertLane: NoLane, // 配合 Transition 回滚使用
    action,
    hasEagerState: false, // v18 起替代 eagerReducer 标志
    eagerState: null,
    next: (null: any),
  };

  if (isRenderPhaseUpdate(fiber)) {
    // 渲染期更新: 做全局标记, 走 enqueueRenderPhaseUpdate
    enqueueRenderPhaseUpdate(queue, update);
  } else {
    const alternate = fiber.alternate;
    if (
      fiber.lanes === NoLanes &&
      (alternate === null || alternate.lanes === NoLanes)
    ) {
      // 性能优化: eager state (见下文)
      const lastRenderedReducer = queue.lastRenderedReducer;
      if (lastRenderedReducer !== null) {
        try {
          const currentState: S = (queue.lastRenderedState: any);
          const eagerState = lastRenderedReducer(currentState, action);
          update.hasEagerState = true;
          update.eagerState = eagerState;
          if (is(eagerState, currentState)) {
            // 快通道: 将 update 入队但不调度
            enqueueConcurrentHookUpdateAndEagerlyBailout(fiber, queue, update);
            return;
          }
        } catch (error) {
          // 渲染期会重新抛
        }
      }
    }
    // 3. 入队 (v18 起统一的 enqueueConcurrentHookUpdate)
    const root = enqueueConcurrentHookUpdate(fiber, queue, update, lane);
    if (root !== null) {
      // 4. 发起调度
      scheduleUpdateOnFiber(root, fiber, lane);
      entangleTransitionUpdate(root, queue, lane); // 配合 useTransition
    }
  }
}
```

逻辑十分清晰:

1. 计算`lane`(`requestUpdateLane`内部会处理`Transition`、`Discrete event`、`Continuous event`等场景, 详见[priority.md](./priority.md)).
2. 创建`update`对象. v18+ 的`update`不再有`eventTime`字段(时间统一在`processRootScheduleInMicrotask`中标记), 并将`eagerReducer`替换为`hasEagerState` boolean.
3. 通过`enqueueConcurrentHookUpdate`把`update`加入`concurrentQueues`待提交队列, 同时返回`root`. 该函数会让`hook.queue.pending`的修改延迟到下一次进入 render 之前生效, 避免渲染中修改造成可见性问题.
4. 调用`scheduleUpdateOnFiber(root, fiber, lane)`, 进入`reconciler 运作流程`中的输入阶段.
5. `entangleTransitionUpdate`: 如果当前队列上正在处理一个`Transition lane`, 把新 update 也纠缠进同一批, 保证 transition 内部所有`setState`原子提交.

> 与 v17 的差异:
>
> - 参数: `(fiber, queue, action)`, 不再传`eventTime`.
> - 入队方式: 不再直接修改`queue.pending`, 改为先放入全局`concurrentQueues`, 在`finishQueueingConcurrentUpdates`时统一提交.
> - 调度: 调用`scheduleUpdateOnFiber(root, fiber, lane)`, 第一个参数是`root`(便于`ReactFiberRootScheduler`管理多 root).
> - `useReducer`走的是几乎相同的`dispatchReducerAction`, 区别仅在于不做 eager 优化.

从调用`scheduleUpdateOnFiber`开始, 进入了`react-reconciler`包, 其中的所有逻辑可回顾[reconciler 运作流程](./reconciler-workflow.md), 本节只讨论`状态Hook`相关逻辑.

注意: 本示例中虽然同时执行了 3 次 dispatch, 会请求 3 次调度. v18 起所有更新天然走 microtask 自动批处理(`automatic batching`), 最后只会执行一次渲染. 详见[scheduler.md 节流防抖](./scheduler.md#throttle-debounce)和[fibertree-prepare.md 自动批处理](./fibertree-prepare.md).

在`fiber树构造(对比更新)`过程中, 再次调用`function`, 这时`useState`对应的函数是[updateState](https://github.com/facebook/react/blob/v19.2.6/packages/react-reconciler/src/ReactFiberHooks.js)

```js
function updateState<S>(
  initialState: (() => S) | S,
): [S, Dispatch<BasicStateAction<S>>] {
  return updateReducer(basicStateReducer, (initialState: any));
}
```

实际调用[updateReducer](https://github.com/facebook/react/blob/v19.2.6/packages/react-reconciler/src/ReactFiberHooks.js).

在执行`updateReducer`之前, `hook`相关的内存结构如下:

![](../../snapshots/hook-state/before-basequeue-combine.png)

```js
// v19 简化版
function updateReducer<S, I, A>(
  reducer: (S, A) => S,
  initialArg: I,
  init?: (I) => S,
): [S, Dispatch<A>] {
  // 1. 获取workInProgressHook对象
  const hook = updateWorkInProgressHook();
  return updateReducerImpl(hook, ((currentHook: any): Hook), reducer);
}

function updateReducerImpl<S, A>(
  hook: Hook,
  current: Hook,
  reducer: (S, A) => S,
): [S, Dispatch<A>] {
  const queue = hook.queue;
  queue.lastRenderedReducer = reducer;

  // 2. 链表拼接: 将 hook.queue.pending 拼接到 current.baseQueue
  let baseQueue = hook.baseQueue;
  const pendingQueue = queue.pending;
  if (pendingQueue !== null) {
    if (baseQueue !== null) {
      const baseFirst = baseQueue.next;
      const pendingFirst = pendingQueue.next;
      baseQueue.next = pendingFirst;
      pendingQueue.next = baseFirst;
    }
    current.baseQueue = baseQueue = pendingQueue;
    queue.pending = null;
  }

  // 3. 状态计算
  const baseState = hook.baseState;
  if (baseQueue === null) {
    hook.memoizedState = baseState;
  } else {
    const first = baseQueue.next;
    let newState = baseState;
    let newBaseState = null;
    let newBaseQueueFirst = null;
    let newBaseQueueLast = null;
    let update = first;
    let didReadFromEntangledAsyncAction = false;

    do {
      const updateLane = removeLanes(update.lane, OffscreenLane);
      const isHiddenUpdate = updateLane !== update.lane;

      const shouldSkipUpdate = isHiddenUpdate
        ? !isSubsetOfLanes(getWorkInProgressRootRenderLanes(), updateLane)
        : !isSubsetOfLanes(renderLanes, updateLane);

      if (shouldSkipUpdate) {
        // 优先级不够: 加入到 baseQueue 中, 等待下一次 render
        const clone: Update<S, A> = {
          lane: updateLane,
          revertLane: update.revertLane,
          action: update.action,
          hasEagerState: update.hasEagerState,
          eagerState: update.eagerState,
          next: (null: any),
        };
        if (newBaseQueueLast === null) {
          newBaseQueueFirst = newBaseQueueLast = clone;
          newBaseState = newState;
        } else {
          newBaseQueueLast = newBaseQueueLast.next = clone;
        }
        currentlyRenderingFiber.lanes = mergeLanes(
          currentlyRenderingFiber.lanes,
          updateLane,
        );
        markSkippedUpdateLanes(updateLane);
      } else {
        // v18+ Transition 回退处理 (useOptimistic / useTransition)
        const revertLane = update.revertLane;
        if (revertLane !== NoLane) {
          // ... 详见源码中的 transition entangle 逻辑
        }

        if (newBaseQueueLast !== null) {
          const clone: Update<S, A> = {
            lane: NoLane,
            revertLane: NoLane,
            action: update.action,
            hasEagerState: update.hasEagerState,
            eagerState: update.eagerState,
            next: (null: any),
          };
          newBaseQueueLast = newBaseQueueLast.next = clone;
        }

        // 计算 newState (复用 eagerState 优化)
        if (update.hasEagerState) {
          newState = ((update.eagerState: any): S);
        } else {
          newState = reducer(newState, update.action);
        }
      }
      update = update.next;
    } while (update !== null && update !== first);

    // 3.2. 更新属性
    if (newBaseQueueLast === null) {
      newBaseState = newState;
    } else {
      newBaseQueueLast.next = (newBaseQueueFirst: any);
    }
    if (!is(newState, hook.memoizedState)) {
      markWorkInProgressReceivedUpdate();
    }
    hook.memoizedState = newState;
    hook.baseState = newBaseState;
    hook.baseQueue = newBaseQueueLast;
    queue.lastRenderedState = newState;
  }

  const dispatch: Dispatch<A> = (queue.dispatch: any);
  return [hook.memoizedState, dispatch];
}
```

`updateReducer`函数, 代码相对较长, 但是逻辑分明:

1. 调用`updateWorkInProgressHook`获取`workInProgressHook`对象
2. 链表拼接: 将 `hook.queue.pending` 拼接到 `current.baseQueue`

   ![](../../snapshots/hook-state/after-basequeue-combine.png)

3. 状态计算

   1. `update`优先级不够: 加入到 baseQueue 中, 等待下一次 render
   2. `update`优先级足够: 状态合并
   3. 更新属性

      ![](../../snapshots/hook-state/state-compute.png)

### 性能优化

`dispatchSetState`函数中, 在调用`scheduleUpdateOnFiber`之前, 针对`update`对象做了性能优化(`useReducer`走的`dispatchReducerAction`**不做**此优化).

1. `fiber.lanes === NoLanes`时(意味着该 fiber 没有其他 pending update), 当前`update`就是`queue.pending`中的第一个`update`.
2. 直接调用`queue.lastRenderedReducer`计算出`update`之后的 state, 记为`eagerState`.
3. 如果`eagerState`与`currentState`相同(`Object.is`), 则**入队但不调度**, 直接返回. 之后如果有其他更新需要 render, 该 update 仍会参与状态计算, 不会丢失.
4. 否则, 标记`update.hasEagerState = true`, 在 render 阶段直接使用`eagerState`避免再次调用 reducer.

```js
// dispatchSetState 性能优化部分 (v19)
if (
  fiber.lanes === NoLanes &&
  (alternate === null || alternate.lanes === NoLanes)
) {
  const lastRenderedReducer = queue.lastRenderedReducer;
  if (lastRenderedReducer !== null) {
    try {
      const currentState: S = (queue.lastRenderedState: any);
      const eagerState = lastRenderedReducer(currentState, action);
      // v18 起标志位字段变更: eagerReducer → hasEagerState (boolean)
      update.hasEagerState = true;
      update.eagerState = eagerState;
      if (is(eagerState, currentState)) {
        // 快速通道, 入队但不调度
        enqueueConcurrentHookUpdateAndEagerlyBailout(fiber, queue, update);
        return;
      }
    } catch (error) {
      // 此处忽略, 错误在 render 时还会再次抛出
    }
  }
}
```

为了验证上述优化, 可以查看这个 demo: [![Edit hook-throttle](https://codesandbox.io/static/img/play-codesandbox.svg)](https://codesandbox.io/s/hook-throttle-58ly5?fontsize=14&hidenavigation=1&theme=dark)

### 异步更新与 Concurrent 模式

> v17 时期默认是`Legacy`模式, 同步更新, 所以`update`对象会被全量合并, `hook.baseQueue`和`hook.baseState`并没有起到实质作用.
>
> v18 起所有通过`createRoot`创建的应用默认开启 `Concurrent` 模式, 但是否会"按优先级跳过 update"取决于业务代码: 只有在`useTransition / startTransition / useDeferredValue / useOptimistic / Server Component refresh`等场景中, update 才会被打上低优先级(`TransitionLane`/`DeferredLane`/`OffscreenLane`等)与高优先级 update 共存.
>
> 本节示意图同样适用于"高优先级 update + 低优先级 transition update"的混合场景.

假设有一个`queue.pending`链表, 其中`update`优先级不同, `绿色`表示高优先级, `灰色`表示低优先级(例如`TransitionLane`), `红色`表示最高优先级.

在执行`updateReducer`之前, `hook.memoizedState`有如下结构(其中`update3, update4`是低优先级):

![](../../snapshots/hook-state/async-update-before-combine.png)

链表拼接:

- 和同步更新时一致, 直接把`queue.pending`拼接到`current.baseQueue`

![](../../snapshots/hook-state/async-update-after-combine.png)

状态计算:

- 只会提取`update1, update2`这 2 个高优先级的`update`, 所以最后`memoizedState=2`
- 保留其余低优先级的`update`, 等待下一次`render`
- 从第一个低优先级`update3`开始, 随后的所有`update`都会被添加到`baseQueue`, 由于`update2`已经是高优先级, 会设置`update2.lane=NoLane`将优先级升级到最高(红色表示).
- 而`baseState`代表第一个低优先级`update3`之前的`state`, 在本例中, `baseState=1`

![](../../snapshots/hook-state/async-update-state-compute.png)

`function`节点被处理完后, 高优先级的`update`, 会率先被使用(`memoizedState=2`). 一段时间后, 低优先级`update3, update4`符合渲染, 这种情况下再次执行`updateReducer`重复之前的步骤.

链表拼接:

- 由于`queue.pending = null`, 故拼接前后没有实质变化

![](../../snapshots/hook-state/async-final-combine.png)

状态计算:

- 现在所有`update.lane`都符合`渲染优先级`, 所以最后的内存结构与同步更新一致(`memoizedState=4,baseState=4`).

![](../../snapshots/hook-state/async-final-compute.png)

> 结论: 尽管`update`链表的优先级不同, 中间的`render`可能有多次, 但最终的更新结果等于`update`链表`按顺序合并`.

## 与其他状态类 Hook 的关系

v18 起新增的`useTransition / useDeferredValue / useOptimistic / useActionState`本质都建立在`useReducer`的基础上, 区别只在`update.lane`与`update.revertLane`的取值:

| Hook                    | 内部实现                                           | update.lane 特点                                                              |
| ----------------------- | -------------------------------------------------- | ----------------------------------------------------------------------------- |
| `useState / useReducer` | `dispatchSetState / dispatchReducerAction`         | 由`requestUpdateLane`返回, 通常是同步/事件优先级                              |
| `useTransition`         | `useReducer` + `startTransition` 包裹的 `dispatch` | dispatch 时切到`TransitionLane`(由全局`ReactSharedInternals.T`提供)           |
| `useDeferredValue`      | `useReducer` + `getDeferredLane`                   | 上一次旧值立即生效, 新值放入`DeferredLane`懒提交                              |
| `useOptimistic`         | `useReducer` + `setOptimistic`                     | 在 transition 内 dispatch, 同时设置`update.revertLane`, transition 完成时撤回 |
| `useActionState`        | `useOptimistic` + `useReducer` 组合                | action 是 async 函数, dispatch 时进入 transition                              |

也就是说: **`useReducer / useState`是"状态 Hook 之根"**, 上述并发能力都是在它的`update`字段上"加车道"实现的, 没有引入新的状态结构.

## 总结

本节深入分析`状态Hook`即`useState`的内部原理, 从`同步,异步`更新理解了`update`对象的合并方式, 最终结果存储在`hook.memoizedState`供给`function`使用.

v17 → v19 关键变化:

1. **`dispatchAction`拆分**: `useState`→`dispatchSetState`(带 eager 优化), `useReducer`→`dispatchReducerAction`.
2. **`update`字段变更**: 去掉`eventTime`, `eagerReducer`变为`hasEagerState`(boolean), 新增`revertLane`(配合 Transition / Optimistic).
3. **`UpdateQueue`字段变更**: 新增`lanes`聚合字段, 用于`enqueueConcurrentHookUpdate`期间快速比对优先级.
4. **入队机制**: 不再直接改`queue.pending`, 改为`enqueueConcurrentHookUpdate`先放进全局`concurrentQueues`, 由`finishQueueingConcurrentUpdates`在 render 准备阶段一次性提交, 杜绝渲染期间被外部修改.
5. **自动批处理**: 所有 dispatch 都通过 microtask 调度, 同一 tick 内的多次 dispatch 自动合并为一次 render.
