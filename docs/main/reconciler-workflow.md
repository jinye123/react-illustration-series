---
title: reconciler 运作流程
group:
  title: 运行核心
  order: 1
order: 1
---

# reconciler 运作流程

## 概览

通过前文[宏观包结构](./macro-structure.md)和[两大工作循环](./workloop.md)中的介绍, 对`react-reconciler`包有一定了解.

此处先归纳一下`react-reconciler`包的主要作用, 将主要功能分为 4 个方面:

1. 输入: 暴露`api`函数(如: `scheduleUpdateOnFiber`), 供给其他包(如`react`包)调用.
2. 注册调度任务: 与调度中心(`scheduler`包)交互, 注册调度任务`task`, 等待任务回调.
3. 执行任务回调: 在内存中构造出`fiber树`, 同时与与渲染器(`react-dom`)交互, 在内存中创建出与`fiber`对应的`DOM`节点.
4. 输出: 与渲染器(`react-dom`)交互, 渲染`DOM`节点.

以上功能源码都集中在[ReactFiberWorkLoop.js](https://github.com/facebook/react/blob/v19.2.6/packages/react-reconciler/src/ReactFiberWorkLoop.js)中. 现在将这些功能(从输入到输出)串联起来, 用下图表示:

![](../../snapshots/reconciler-workflow/reactfiberworkloop.png)

> 注: 该图绘制于 v17 时期, 主体 4 步骤(`输入 → 注册调度任务 → 执行任务回调 → 输出`)在 v19 中依然成立; 但 v18 起`performSyncWorkOnRoot / performConcurrentWorkOnRoot`已合并为单一入口`performWorkOnRoot(root, lanes, forceSync)`, 同时副作用链表(`firstEffect / nextEffect`)被`subtreeFlags`取代.

下方为 v19 版本的等价流程图(Mermaid, 如未渲染请复制到 [mermaid.live](https://mermaid.live)):

```mermaid
flowchart LR
  A[("① 输入<br/>setState / dispatchSetState<br/>root.render(...)")] --> B[scheduleUpdateOnFiber<br/>root, fiber, lane]
  B --> C[markRootUpdated<br/>root.pendingLanes |= lane]
  C --> D{② 注册调度<br/>ensureRootIsScheduled}
  D -->|"SyncLane / DefaultLane"| E["microtask<br/>processRootScheduleInMicrotask"]
  D -->|"TransitionLane / IdleLane"| F["Scheduler.scheduleCallback<br/>(schedulerPriority)"]
  E --> G[("③ 任务回调<br/>performWorkOnRoot<br/>(root, lanes, forceSync=true)")]
  F --> H[("③ 任务回调<br/>performWorkOnRoot<br/>(root, lanes, forceSync=false)")]
  G --> I[renderRootSync<br/>workLoopSync]
  H --> J[renderRootConcurrent<br/>workLoopConcurrent<br/>+ shouldYield]
  I --> K[fiber 树构造<br/>beginWork / completeWork<br/>沿父链冒泡 subtreeFlags]
  J --> K
  K --> L[("④ 输出<br/>commitRoot")]
  L --> M[commitBeforeMutationEffects<br/>DFS by subtreeFlags]
  L --> N[commitMutationEffects<br/>含 useInsertionEffect]
  L --> O[commitLayoutEffects<br/>componentDidMount<br/>useLayoutEffect]
  L --> P[scheduleCallback NormalPriority<br/>flushPassiveEffects<br/>useEffect]
```

图中的`1,2,3,4`步骤可以反映`react-reconciler`包`从输入到输出`的运作流程,这是一个固定流程, 每一次更新都会运行.

## 分解

图中只列举了最核心的函数调用关系(其中的每一步都有各自的实现细节, 会在后续的章节中逐一展开). 将上述 4 个步骤逐一分解, 了解它们的主要逻辑.

### 输入

在`ReactFiberWorkLoop.js`中, 承接输入的函数只有`scheduleUpdateOnFiber`[源码地址](https://github.com/facebook/react/blob/v19.2.6/packages/react-reconciler/src/ReactFiberWorkLoop.js). 在`react-reconciler`对外暴露的 api 函数中, 只要涉及到需要改变 fiber 的操作(无论是`首次渲染`或`后续更新`操作), 最后都会间接调用`scheduleUpdateOnFiber`, 所以`scheduleUpdateOnFiber`函数是输入链路中的`必经之路`.

> 关键变更(v17 → v19):
>
> 1. v18 起函数签名变更为`scheduleUpdateOnFiber(root, fiber, lane)`, 由调用方提前算好`root`传入(`getRootForUpdatedFiber`).
> 2. v17 中"在`LegacyUnbatchedContext`下绕过调度直接执行`performSyncWorkOnRoot`"的分支已删除. 因为 v18 已经废弃`ReactDOM.render`, 不再有`LegacyUnbatchedContext`.
> 3. 自 v18 起, 所有更新统一通过`ensureRootIsScheduled`进入调度, **同步更新走 microtask**, 并发更新走 Scheduler 包.

```js
// 唯一接收输入信号的函数 (v19 实际签名)
export function scheduleUpdateOnFiber(
  root: FiberRoot,
  fiber: Fiber,
  lane: Lane,
) {
  // 1. 标记 root 上对应 lane 的更新, 同时把 lane 沿 return 指针冒泡到根
  markRootUpdated(root, lane);

  // 2. 注册调度任务: 无论同步/并发都走 ensureRootIsScheduled
  //    内部会区分同步(microtask) 与并发(Scheduler) 两条调度路径
  ensureRootIsScheduled(root);
}
```

逻辑进入到`scheduleUpdateOnFiber`之后, 不再像 v17 那样区分"是否绕过调度":

- **同步车道(`SyncLane / DefaultLane`等)**: 在 microtask 中冲刷, 实际仍调用`performSyncWorkOnRoot`(内部由`performWorkOnRoot(root, lanes, true)`实现).
- **过渡 / 异步车道(`TransitionLane / IdleLane`等)**: 通过`Scheduler.scheduleCallback`注册一个可被中断的回调`performWorkOnRoot(root, lanes, false)`.

### 注册调度任务

与`输入`环节紧密相连, `scheduleUpdateOnFiber`函数之后, 立即进入`ensureRootIsScheduled`函数([源码地址](https://github.com/facebook/react/blob/v19.2.6/packages/react-reconciler/src/ReactFiberRootScheduler.js)). v18 起, 这套逻辑被独立到了`ReactFiberRootScheduler.js`, 并且不再依赖 v17 的 3 套优先级转换体系:

```js
// v19 简化版 (基于 ReactFiberRootScheduler.js)
function ensureRootIsScheduled(root: FiberRoot) {
  // 1. 把 root 挂到全局 rootsWithUpdate 链表上, 等待 processRootScheduleInMicrotask 处理
  if (root === mightHavePendingSyncWork) return;
  mightHavePendingSyncWork = true;
  if (!didScheduleMicrotask) {
    didScheduleMicrotask = true;
    scheduleImmediateRootScheduleTask(); // 内部走 queueMicrotask / Promise.resolve().then
  }
}

function processRootScheduleInMicrotask() {
  // 2. microtask 中, 遍历所有有更新的 root, 依次决定调度方式
  for (let root = firstScheduledRoot; root !== null; root = root.next) {
    const nextLanes = getNextLanes(root, NoLanes);
    if (nextLanes === NoLanes) continue;

    // 3. 区分同步/并发两条调度路径
    if (includesSyncLane(nextLanes)) {
      // 同步: 直接在 microtask 末尾冲刷
      performSyncWorkOnRoot(root, nextLanes);
    } else {
      // 并发: 走 Scheduler 包, 转换为 SchedulerPriority
      const schedulerPriorityLevel = lanesToEventPriority(nextLanes);
      scheduleCallback(
        schedulerPriorityLevel,
        performConcurrentWorkOnRoot.bind(null, root),
      );
    }
  }
}
```

`ensureRootIsScheduled`在 v19 中的核心思想可以总结为:

1. **延迟到 microtask 决策**: 同一 tick 内多次 setState 不再每次都走调度判断, 而是先把 root 标记到全局链表, 在 microtask 中统一处理一次, 这就是 v18 [#26512](https://github.com/facebook/react/pull/26512) 引入的关键优化.
2. **同步走 microtask, 并发走 Scheduler**: 同步车道在 microtask 末尾直接冲刷, 不再向 Scheduler 注册任务; 并发车道才使用`scheduleCallback`.
3. **`LanePriority`已删除**: v17 中`SyncLanePriority / SyncBatchedLanePriority / lanePriorityToSchedulerPriority`整套已被`lanesToEventPriority + eventPriorityToSchedulerPriority`两步替代, 详见[优先级管理](./priority.md).

### 执行任务回调

任务回调, 实际上就是执行`performSyncWorkOnRoot`或`performConcurrentWorkOnRoot`. v18 起这两个函数底层共用同一个`performWorkOnRoot(root, lanes, forceSync)`, 通过`forceSync`参数区分同步/并发两种行为. 简单看一下它的源码(在`fiber树构造`章节再深入分析), 将主要逻辑剥离出来, 单个函数的代码量并不多.

[performWorkOnRoot](https://github.com/facebook/react/blob/v19.2.6/packages/react-reconciler/src/ReactFiberWorkLoop.js):

```js
// ... 省略部分无关代码 (v19 ReactFiberWorkLoop.js)
function performWorkOnRoot(root, lanes, forceSync) {
  // 1. 同步前先冲刷 pending 的 passive effects (useEffect cleanup/setup)
  //    有可能在 cleanup 中产生新更新, 影响本轮要处理的 lanes
  const didFlushPassiveEffects = flushPassiveEffects();
  if (didFlushPassiveEffects) {
    // pending effects 已冲刷, 重新选取 lanes
    return ensureRootIsScheduled(root);
  }

  // 2. 选择 render 模式: 同步 / 并发
  //    - forceSync 或 lanes 包含 SyncLane: renderRootSync
  //    - 否则: renderRootConcurrent (可被时间切片中断)
  const shouldTimeSlice =
    !forceSync &&
    !includesBlockingLane(lanes) &&
    !includesExpiredLane(root, lanes);
  let exitStatus = shouldTimeSlice
    ? renderRootConcurrent(root, lanes)
    : renderRootSync(root, lanes, true);

  // 3. 异常 / 中断处理
  if (exitStatus === RootInProgress) {
    // 渲染被中断, 直接返回 (Scheduler 会重新回调本函数继续未完成的工作)
    return;
  }
  if (exitStatus === RootErrored) {
    // ... 错误恢复, 必要时降级到同步渲染
  }

  // 4. 输出: 提交 fiber 树
  const finishedWork = root.current.alternate;
  root.finishedWork = finishedWork;
  root.finishedLanes = lanes;
  commitRoot(root, finishedWork, lanes);

  // 5. 退出前再次检测, 是否还有其他更新需要发起新调度
  ensureRootIsScheduled(root);
}
```

`performWorkOnRoot`的逻辑可总结为 5 步:

1. **冲刷 passive effects**: 先把上一次提交遗留的`useEffect`冲掉, 避免它们影响本轮 render.
2. **决定 render 模式**: 根据`forceSync`、`Blocking lane`、`Expired lane`等条件, 选择`renderRootSync`(不可中断)或`renderRootConcurrent`(可中断).
3. **fiber 树构造**: 在`renderRoot*`内部驱动`fiber构造循环`(`workLoop*` + `performUnitOfWork`), 详见[fiber 树构造(初次创建)](./fibertree-create.md).
4. **异常 / 中断处理**: `RootInProgress`表示被时间切片中断, 此时直接返回, Scheduler 会再回调; `RootErrored`走错误恢复路径.
5. **commit + 再调度**: 完成的 fiber 树进入`commitRoot`, 完成后调用`ensureRootIsScheduled`查看有没有遗留更新.

> `可中断渲染`的核心机制:
>
> - **时间切片**: `renderRootConcurrent`内部循环执行`performUnitOfWork`, 每个单元结束后通过`shouldYield()`检查是否需要让出主线程; 让出后函数返回`RootInProgress`, Scheduler 重新派发同一回调继续未完成的工作.
> - **新更新打断**: 如果在 render 过程中产生了更高优先级的更新, `prepareFreshStack`会丢弃当前 workInProgress 重新开始构造.

### 输出

[`commitRoot`](https://github.com/facebook/react/blob/v19.2.6/packages/react-reconciler/src/ReactFiberWorkLoop.js):

```js
// ... 省略部分无关代码 (v19 ReactFiberWorkLoop.js)
function commitRootImpl(root, finishedWork, lanes) {
  // 清空 FiberRoot 上的提交相关字段
  root.finishedWork = null;
  root.finishedLanes = NoLanes;
  root.callbackNode = null;

  // 标记 root 上 lanes 已完成
  markRootFinished(root, lanes);

  if (
    (finishedWork.subtreeFlags & BeforeMutationMask) !== NoFlags ||
    (finishedWork.flags & BeforeMutationMask) !== NoFlags
  ) {
    // 阶段1: DOM 突变之前
    //   - 处理 Snapshot 标记 (getSnapshotBeforeUpdate)
    //   - 调度 Passive effect (useEffect)
    commitBeforeMutationEffects(root, finishedWork);
  }

  if (
    (finishedWork.subtreeFlags & MutationMask) !== NoFlags ||
    (finishedWork.flags & MutationMask) !== NoFlags
  ) {
    // 阶段2: DOM 突变, 界面发生改变
    //   - Placement / Update / ChildDeletion / Ref
    //   - 同步执行 useInsertionEffect (v18 新增)
    commitMutationEffects(root, finishedWork, lanes);
  }

  // 切换 current 指针, 此后 root.current 指向已提交的树
  root.current = finishedWork;

  if (
    (finishedWork.subtreeFlags & LayoutMask) !== NoFlags ||
    (finishedWork.flags & LayoutMask) !== NoFlags
  ) {
    // 阶段3: layout 阶段
    //   - componentDidMount / componentDidUpdate
    //   - useLayoutEffect 同步执行
    commitLayoutEffects(finishedWork, root, lanes);
  }

  ensureRootIsScheduled(root);
}
```

在输出阶段,`commitRoot`的实现逻辑是在`commitRootImpl`函数中, 其主要逻辑是基于`finishedWork.subtreeFlags`从根 DFS, 将最新的 fiber 树结构反映到 DOM 上.

核心逻辑分为 3 个步骤(命名沿用 v17, 但内部实现 v18 起完全基于`subtreeFlags + flags`剪枝, 而非旧的 effect 链表):

1. `commitBeforeMutationEffects`
   - DOM 变更之前, 主要处理子树中带有`Snapshot`、`Passive`标记的`fiber`节点(调用`getSnapshotBeforeUpdate`、调度`useEffect`).
2. `commitMutationEffects`
   - DOM 变更, 界面得到更新. 主要处理子树中带有`Placement`、`Update`、`ChildDeletion`、`Ref`、`Hydrating`标记的`fiber`节点; **同步执行`useInsertionEffect`**(v18 新增, 用于 CSS-in-JS 库在 layout 读取样式之前注入样式).
3. `commitLayoutEffects`
   - DOM 变更后, 主要处理子树中带有`Update | Callback | Ref`标记的`fiber`节点(`componentDidMount/Update`、`useLayoutEffect`、ref 挂载).

> 关于 passive effects (`useEffect`): 它**不在**三阶段中同步执行, 而是在阶段 1 通过`schedulePassiveEffectCallback`注册一个`NormalSchedulerPriority`的 Scheduler 回调, 在浏览器空闲后异步冲刷`commitPassiveMountEffects / commitPassiveUnmountEffects`. 详见[hook 原理(副作用 Hook)](./hook-effect.md).

## 总结

本节从宏观上分析了`reconciler 运作流程`, 并将其分为了 4 个步骤, 基本覆盖了`react-reconciler`包的核心逻辑.
