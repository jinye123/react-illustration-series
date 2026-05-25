---
title: fiber 树渲染
group: 运行核心
order: 7
---

# fiber 树渲染

在正式分析`fiber树渲染`之前, 再次回顾一下[reconciler 运作流程](./reconciler-workflow.md)的 4 个阶段:

![](../../snapshots/reconciler-workflow/reactfiberworkloop.png)

1. 输入阶段: 衔接`react-dom`包, 承接`fiber更新`请求(参考[React 应用的启动过程](./bootstrap.md)).
2. 注册调度任务: 与调度中心(`scheduler`包)交互, 注册调度任务`task`, 等待任务回调(参考[React 调度原理(scheduler)](./scheduler.md)).
3. 执行任务回调: 在内存中构造出`fiber树`和`DOM`对象(参考[fiber 树构造(初次创建)](./fibertree-create.md)和 fiber 树构造(对比更新)).
4. 输出: 与渲染器(`react-dom`)交互, 渲染`DOM`节点.

本节分析其中的第 4 阶段(输出), `fiber树渲染`处于`reconciler 运作流程`这一流水线的最后一环, 或者说前面的步骤都是为了最后一步服务, 所以其重要性不言而喻.

前文已经介绍了`fiber树构造`, 现在分析`fiber树渲染`过程, 这个过程, 实际上是对`fiber树`的进一步处理.

> v17 → v19 重大变化:
>
> 1. **删除副作用链表**: v17 时期通过`firstEffect / nextEffect / lastEffect`链表把所有需要在 commit 阶段处理的 fiber 串成一条线性链, v18 起这条链表被彻底移除([#19388](https://github.com/facebook/react/pull/19388)). 取而代之的是: 在 complete 阶段把每个节点的`flags`往父节点冒泡, 父节点用`subtreeFlags`记录子树是否有副作用; commit 阶段则从根 DFS, 沿途遇到`subtreeFlags === NoFlags`的子树直接跳过.
> 2. **删除节点单独存放**: 被删除的子 fiber 不再共享在 effect 链上, 而是直接放在父节点的`fiber.deletions: Fiber[]`数组中, 父节点打上`ChildDeletion`标志位.
> 3. **新增 useInsertionEffect 阶段**: v18 起 mutation 阶段执行 DOM 突变之前, 先同步执行所有`useInsertionEffect`(给 CSS-in-JS 库一个"DOM 已经创建/更新但还没插入 layout 阶段"的窗口).
> 4. **commit 阶段不再 runWithPriority(ImmediateSchedulerPriority)**: v17 会切到 Immediate 调度优先级再调 `commitRootImpl`, v18 起整个 commit 过程都同步进行, 不再切换优先级.

## fiber 树特点

通过前文`fiber树构造`的解读, 可以总结出`fiber树`的基本特点:

- 无论是`首次构造`或者是`对比更新`, 最终都会在内存中生成一棵用于渲染页面的`fiber树`(即`fiberRoot.finishedWork`).
- 这棵将要被渲染的`fiber树`有 2 个特点:
  1. **子树副作用标志位挂载在父节点上**(即`fiber.subtreeFlags`). commit 阶段从根 DFS 时, 凡是`subtreeFlags === NoFlags && flags === NoFlags`的分支可整体剪枝跳过.
  2. 代表最新页面的`DOM`对象挂载在`fiber树`中首个`HostComponent`类型的节点上(具体来讲`DOM`对象是挂载在`fiber.stateNode`属性上).

这里再次回顾前文使用过的 2 棵 fiber 树, 可以验证上述特点:

1. 初次构造

![](../../snapshots/fibertree-create/fibertree-beforecommit.png)

2. 对比更新

![](../../snapshots/fibertree-update/fibertree-beforecommit.png)

> 注: 上面两张图绘制于 v17 时期, 图中带绿色虚线箭头的`firstEffect/nextEffect`链路在 v19 中已删除. 子树副作用走`fiber.subtreeFlags`位掩码, 父 → 子方向沿 child/sibling 指针 DFS. 图待重绘.

## commitRoot

整个渲染逻辑都在[commitRoot 函数中](https://github.com/facebook/react/blob/v19.2.6/packages/react-reconciler/src/ReactFiberWorkLoop.js):

```js
// v19 简化版
function commitRoot(root, finishedWork, lanes, recoverableErrors, transitions) {
  // v18 起 commit 同步运行, 不再切换到 ImmediateSchedulerPriority
  const prevTransition = ReactSharedInternals.T;
  const previousUpdateLanePriority = getCurrentUpdatePriority();
  try {
    ReactSharedInternals.T = null;
    setCurrentUpdatePriority(DiscreteEventPriority);
    commitRootImpl(
      root,
      finishedWork,
      lanes,
      recoverableErrors,
      transitions,
      previousUpdateLanePriority,
    );
  } finally {
    ReactSharedInternals.T = prevTransition;
    setCurrentUpdatePriority(previousUpdateLanePriority);
  }
  return null;
}
```

> 与 v17 的差异: 不再使用`runWithPriority(ImmediateSchedulerPriority, ...)`, 而是直接通过`setCurrentUpdatePriority(DiscreteEventPriority)`提升当前更新优先级. 因为`UpdatePriority`是一个 Lane(在 commit 期间触发的`setState`会落到`SyncLane`, 立即冲刷).

最后的实现是通过`commitRootImpl`函数:

```js
// ... 省略部分无关代码 (v19 ReactFiberWorkLoop.js)
function commitRootImpl(
  root,
  finishedWork,
  lanes,
  recoverableErrors,
  transitions,
  renderPriorityLevel,
) {
  // ============ 渲染前: 准备 ============

  // 1. 清空 FiberRoot 对象上的属性
  root.finishedWork = null;
  root.finishedLanes = NoLanes;
  root.callbackNode = null;
  root.callbackPriority = NoLane;
  root.cancelPendingCommit = null;

  // 2. 标记 root 上 lane 已完成
  let remainingLanes = mergeLanes(finishedWork.lanes, finishedWork.childLanes);
  markRootFinished(root, lanes, remainingLanes /* ...*/);

  if (root === workInProgressRoot) {
    // 重置全局变量
    workInProgressRoot = null;
    workInProgress = null;
    workInProgressRootRenderLanes = NoLanes;
  }

  // 3. 调度 passive effects (如有需要)
  //    使用 schedulePassiveEffectCallback, 默认在 commit 完成后异步冲刷
  if (
    (finishedWork.subtreeFlags & PassiveMask) !== NoFlags ||
    (finishedWork.flags & PassiveMask) !== NoFlags
  ) {
    if (!rootDoesHavePassiveEffects) {
      rootDoesHavePassiveEffects = true;
      pendingPassiveEffectsRemainingLanes = remainingLanes;
      pendingPassiveTransitions = transitions;
      scheduleCallback(NormalSchedulerPriority, () => {
        flushPassiveEffects();
        return null;
      });
    }
  }

  // ============ 渲染: 三阶段 DFS, 全部基于 subtreeFlags 剪枝 ============

  const subtreeHasEffects =
    (finishedWork.subtreeFlags &
      (BeforeMutationMask | MutationMask | LayoutMask | PassiveMask)) !==
    NoFlags;
  const rootHasEffect =
    (finishedWork.flags &
      (BeforeMutationMask | MutationMask | LayoutMask | PassiveMask)) !==
    NoFlags;

  if (subtreeHasEffects || rootHasEffect) {
    const prevTransition = ReactSharedInternals.T;
    ReactSharedInternals.T = null;
    const previousPriority = getCurrentUpdatePriority();
    setCurrentUpdatePriority(DiscreteEventPriority);
    const prevExecutionContext = executionContext;
    executionContext |= CommitContext;

    // 阶段1: DOM 突变之前
    //   - Snapshot 标志 → getSnapshotBeforeUpdate
    commitBeforeMutationEffects(root, finishedWork);

    // 阶段2: DOM 突变
    //   - useInsertionEffect 同步冲刷 (v18 新增)
    //   - Placement / Update / ChildDeletion / Hydrating / Ref
    commitMutationEffects(root, finishedWork, lanes);

    // 切换 current 指针 (v17 一致): 此后 root.current 指向已提交的树
    root.current = finishedWork;

    // 阶段3: layout 阶段
    //   - componentDidMount / componentDidUpdate / useLayoutEffect
    //   - 重新挂 ref
    commitLayoutEffects(finishedWork, root, lanes);

    // 请求重绘 (帮助 Scheduler.shouldYieldToHost 在下一帧让出主线程)
    requestPaint();

    executionContext = prevExecutionContext;
    setCurrentUpdatePriority(previousPriority);
    ReactSharedInternals.T = prevTransition;
  } else {
    // 没有任何副作用, 直接切换 current 指针
    root.current = finishedWork;
  }

  // ============ 渲染后: 重置与清理 ============

  // v18+ 不再有"分解副作用链表"的代码 (链表已删除)
  // 唯一需要做的是: 重置全局状态, 检查嵌套更新

  // 1. 检测常规任务, 如果有则发起新调度
  ensureRootIsScheduled(root);

  // 2. 检测渲染期间产生的嵌套更新, 防止死循环
  if (rootWithPendingPassiveEffects !== null && passiveEffectDuration !== 0) {
    // ...
  }

  return null;
}
```

`commitRootImpl`函数中, 可以根据是否调用渲染, 把整个`commitRootImpl`分为 3 段(分别是`渲染前`, `渲染`, `渲染后`).

### 渲染前

为接下来正式渲染, 做一些准备工作. 主要包括:

1. 清空`FiberRoot`对象上的属性(`finishedWork / finishedLanes / callbackNode / callbackPriority`等).
2. 调用`markRootFinished(root, lanes, remainingLanes)`, 把已经完成的 lane 从`pendingLanes`中移除, 重置`suspendedLanes / pingedLanes / warmLanes`等.
3. 重置全局变量(`workInProgressRoot / workInProgress / workInProgressRootRenderLanes`等).
4. **调度 passive effects**: 检查`subtreeFlags & PassiveMask`, 如有则通过`scheduleCallback(NormalSchedulerPriority, flushPassiveEffects)`异步冲刷 `useEffect`.

注意:

- v17 时期需要"将根节点添加到副作用队列的末尾"的特殊处理, v19 中**无需**: 因为根节点本身的`flags / subtreeFlags`本来就会被三阶段 DFS 处理到.
- `firstEffect / nextEffect`链表已删除, 整张图`fiber-effectlist.png` 在 v19 中已不再适用, 图待重绘.

### 渲染

`commitRootImpl`函数中, 渲染阶段的主要逻辑是基于`finishedWork.subtreeFlags`从根 DFS, 处理所有有副作用的`fiber`节点, 将最新的 DOM 节点(已经在内存中, 只是还没渲染)渲染到界面上.

整个渲染过程被分为 3 个函数分布实现:

1. `commitBeforeMutationEffects`
   - DOM 变更之前, 处理子树中带有`Snapshot`标记的`fiber`节点(`ClassComponent.getSnapshotBeforeUpdate / HostRoot 清容器`).
2. `commitMutationEffects`
   - DOM 变更, 界面得到更新. 处理子树中带有`Placement`, `Update`, `ChildDeletion`, `Hydrating`, `Ref`标记的`fiber`节点; 同步执行`useInsertionEffect`.
3. `commitLayoutEffects`
   - DOM 变更后, 处理子树中带有`Update | Callback | Ref | LayoutStatic`标记的`fiber`节点(`componentDidMount / Update`、`useLayoutEffect`、ref 挂载).

通过上述源码分析, 可以把`commitRootImpl`的职责概括为 2 个方面:

1. **沿 fiber 树 DFS 处理副作用**(三阶段都做, 只是处理的`fiber.flags`位不同).
2. **调用渲染器, 输出最终结果**(在阶段 2: `commitMutationEffects`中执行).

所以`commitRootImpl`是处理`fiberRoot.finishedWork`这棵即将被渲染的`fiber`树, 理论上无需关心这棵`fiber`树是如何产生的(可以是`首次构造`产生, 也可以是`对比更新`产生). 为了清晰简便, 在下文的所有图示都使用`初次创建的fiber树结构`来进行演示.

这 3 个函数处理的对象是`fiber树本身`和`DOM对象`(不再有"副作用队列").

所以无论`fiber树`结构有多么复杂, 到了`commitRoot`阶段, 实际起作用的关键节点是:

- 整棵 fiber 树(从根开始 DFS, 但每个分支被`subtreeFlags`剪枝)
- `DOM对象`所在节点: 从上至下首个`HostComponent`类型的`fiber`节点, 此节点 `fiber.stateNode`实际上指向最新的 DOM 树.

下图为了清晰, 省略了一些无关引用, 只留下`commitRoot`阶段实际会用到的`fiber`节点:

![](../../snapshots/fibertree-commit/fiber-noredundant.png)

> 注: 此图绘制于 v17 时期, 图中`fiberRoot.finishedWork.firstEffect` → ... → ... 这条线性链路在 v19 中已删除. 实际 DFS 走的是父 → 子 + sibling 指针. 图待重绘.

#### commitBeforeMutationEffects

第一阶段: DOM 变更之前, 处理子树中带有`Snapshot`、`Passive`标记的`fiber`节点. v19 使用 begin/complete 双指针 DFS:

```js
// v19 ReactFiberCommitWork.js (简化版)
let nextEffect: Fiber | null = null;

function commitBeforeMutationEffects(root: FiberRoot, firstChild: Fiber) {
  nextEffect = firstChild;
  commitBeforeMutationEffects_begin();
}

function commitBeforeMutationEffects_begin() {
  while (nextEffect !== null) {
    const fiber = nextEffect;
    const child = fiber.child;
    // 关键剪枝: 如果子树标志位中包含 BeforeMutationMask, 就继续往下走
    if (
      (fiber.subtreeFlags & BeforeMutationMask) !== NoFlags &&
      child !== null
    ) {
      nextEffect = child;
    } else {
      commitBeforeMutationEffects_complete();
    }
  }
}

function commitBeforeMutationEffects_complete() {
  while (nextEffect !== null) {
    const fiber = nextEffect;
    try {
      commitBeforeMutationEffectsOnFiber(fiber); // 处理本节点的 Snapshot 标志
    } catch (error) {
      captureCommitPhaseError(fiber, fiber.return, error);
    }
    const sibling = fiber.sibling;
    if (sibling !== null) {
      nextEffect = sibling;
      return; // 切回 _begin, 处理 sibling
    }
    nextEffect = fiber.return; // 回到父节点
  }
}
```

接下来看每个节点上的具体处理`commitBeforeMutationEffectsOnFiber`:

```js
function commitBeforeMutationEffectsOnFiber(finishedWork: Fiber) {
  const current = finishedWork.alternate;
  const flags = finishedWork.flags;

  switch (finishedWork.tag) {
    case ClassComponent: {
      if ((flags & Snapshot) !== NoFlags) {
        if (current !== null) {
          const prevProps = current.memoizedProps;
          const prevState = current.memoizedState;
          const instance = finishedWork.stateNode;
          const snapshot = instance.getSnapshotBeforeUpdate(
            finishedWork.elementType === finishedWork.type
              ? prevProps
              : resolveDefaultProps(finishedWork.type, prevProps),
            prevState,
          );
          instance.__reactInternalSnapshotBeforeUpdate = snapshot;
        }
      }
      return;
    }
    case HostRoot: {
      if ((flags & Snapshot) !== NoFlags) {
        const root = finishedWork.stateNode;
        clearContainer(root.containerInfo);
      }
      return;
    }
    case HostComponent:
    case HostText:
    case HostPortal:
    case IncompleteClassComponent:
      return;
  }
}
```

从源码中可以看到, 与`Snapshot`标记相关的类型只有`ClassComponent`和`HostRoot`:

- 对于`ClassComponent`类型节点, 调用了`instance.getSnapshotBeforeUpdate`生命周期函数.
- 对于`HostRoot`类型节点, 调用`clearContainer`清空了容器节点(即`div#root`这个 dom 节点).

关于`Passive`标记 (`useEffect`): v19 不再在这个函数里"按节点 schedule", 而是在`commitRootImpl`开头做了一次性的`scheduleCallback(NormalSchedulerPriority, flushPassiveEffects)`(见前文"渲染前"小节). 但提示性质的小例子依然成立:

```js
// 以下示例代码中的输出顺序为 1, 3, 4, 2
function Test() {
  console.log(1);
  useEffect(() => {
    console.log(2);
  });
  console.log(3);
  Promise.resolve().then(() => {
    console.log(4);
  });
  return <div>test</div>;
}
```

> 在 v18+ 中, `useEffect`的 cleanup 与 setup 仍由 Scheduler 在浏览器空闲时异步冲刷, 所以它依旧晚于同 tick 的 microtask(`Promise.then`).

#### commitMutationEffects

第二阶段: DOM 变更, 界面得到更新. 处理子树中带有`Placement`、`Update`、`ChildDeletion`、`Hydrating`、`Ref`、`ContentReset`标记的`fiber`节点.

v18 起这一阶段不再用 effect list 顺序处理, 而是用一个`recursivelyTraverseMutationEffects + commitMutationEffectsOnFiber`的双层递归 DFS:

```js
// v19 ReactFiberCommitWork.js (简化版)
function commitMutationEffects(
  root: FiberRoot,
  finishedWork: Fiber,
  committedLanes: Lanes,
) {
  inProgressLanes = committedLanes;
  inProgressRoot = root;
  commitMutationEffectsOnFiber(finishedWork, root, committedLanes);
}

function commitMutationEffectsOnFiber(
  finishedWork: Fiber,
  root: FiberRoot,
  lanes: Lanes,
) {
  const current = finishedWork.alternate;
  const flags = finishedWork.flags;

  switch (finishedWork.tag) {
    case FunctionComponent:
    case ForwardRef:
    case SimpleMemoComponent: {
      // 1. 先递归处理子树
      recursivelyTraverseMutationEffects(root, finishedWork, lanes);
      // 2. 处理 Ref 清除 (新 ref 在 layout 阶段挂)
      commitReconciliationEffects(finishedWork);

      // 3. v18 新增: 同步执行 useInsertionEffect cleanup, 然后是 useInsertionEffect setup
      if (flags & Update) {
        commitHookEffectListUnmount(
          HookInsertion | HookHasEffect,
          finishedWork,
          finishedWork.return,
        );
        commitHookEffectListMount(HookInsertion | HookHasEffect, finishedWork);
        // 4. 同步执行 useLayoutEffect cleanup (setup 在 layout 阶段)
        commitHookEffectListUnmount(
          HookLayout | HookHasEffect,
          finishedWork,
          finishedWork.return,
        );
      }
      return;
    }
    case HostComponent: {
      recursivelyTraverseMutationEffects(root, finishedWork, lanes);
      commitReconciliationEffects(finishedWork);

      if (flags & Ref) {
        const current = finishedWork.alternate;
        if (current !== null) safelyDetachRef(current, current.return); // 卸载旧 ref
      }
      if (flags & ContentReset) {
        const instance = finishedWork.stateNode;
        resetTextContent(instance);
      }
      if (flags & Update) {
        // v19 起 diff 在这里同步完成 (不再读 fiber.updateQueue 上的 updatePayload, 因为 prepareUpdate 已删除)
        const instance = finishedWork.stateNode;
        if (instance != null) {
          const newProps = finishedWork.memoizedProps;
          const oldProps = current !== null ? current.memoizedProps : newProps;
          const type = finishedWork.type;
          try {
            commitUpdate(instance, type, oldProps, newProps, finishedWork);
          } catch (error) {
            captureCommitPhaseError(finishedWork, finishedWork.return, error);
          }
        }
      }
      return;
    }
    case HostRoot: {
      recursivelyTraverseMutationEffects(root, finishedWork, lanes);
      commitReconciliationEffects(finishedWork);
      return;
    }
    // ... 其他 case: HostPortal / SuspenseComponent / OffscreenComponent / Profiler / ...
  }
}

function recursivelyTraverseMutationEffects(root, parentFiber, lanes) {
  // 1. 先处理 deletions (v18 起删除节点放在父节点的 deletions 数组里)
  const deletions = parentFiber.deletions;
  if (deletions !== null) {
    for (let i = 0; i < deletions.length; i++) {
      commitDeletionEffects(root, parentFiber, deletions[i]);
    }
  }
  // 2. 再 DFS 子节点 (用 subtreeFlags 剪枝)
  if (parentFiber.subtreeFlags & MutationMask) {
    let child = parentFiber.child;
    while (child !== null) {
      commitMutationEffectsOnFiber(child, root, lanes);
      child = child.sibling;
    }
  }
}

function commitReconciliationEffects(finishedWork) {
  const flags = finishedWork.flags;
  if (flags & Placement) {
    commitPlacement(finishedWork);
    finishedWork.flags &= ~Placement; // 注意 Placement 标记会被清除
  }
  if (flags & Hydrating) {
    finishedWork.flags &= ~Hydrating;
  }
}
```

处理 DOM 突变(v17 与 v19 调用链一致, 命名也一致):

1. **新增**: `commitReconciliationEffects(...) -> commitPlacement -> insertOrAppendPlacementNode -> appendChildToContainer / appendChild`.
2. **更新**: `commitMutationEffectsOnFiber(case HostComponent) -> commitUpdate -> updateProperties`(在`react-dom-bindings/src/client/ReactFiberConfigDOM.js`).
3. **删除**: `recursivelyTraverseMutationEffects 开头 -> commitDeletionEffects -> commitDeletionEffectsOnFiber`(v18 新名), 内部最终调用`removeChild / removeChildFromContainer`.

最终会调用`appendChild`、`commitUpdate`、`removeChild`这些`react-dom`(实际是`react-dom-bindings`)包中的函数. 它们是[`HostConfig`协议](https://github.com/facebook/react/blob/v19.2.6/packages/react-reconciler/README.md#practical-examples)中规定的标准函数, 在渲染器`react-dom-bindings`中实现(源码位于[`ReactFiberConfigDOM.js`](https://github.com/facebook/react/blob/v19.2.6/packages/react-dom-bindings/src/client/ReactFiberConfigDOM.js)). 这些函数就是直接操作 DOM, 所以执行之后, 界面也会得到更新.

注意: `commitMutationEffects`执行之后, 在`commitRootImpl`函数中切换当前`fiber`树(`root.current = finishedWork`), 保证`fiberRoot.current`指向代表当前界面的`fiber树`.

![](../../snapshots/fibertree-commit/fiber-switch.png)

#### commitLayoutEffects

第三阶段: DOM 变更后, 处理子树中带有`Update | Callback | Ref | LayoutStatic`标记的`fiber`节点.

```js
// v19 ReactFiberCommitWork.js (简化版)
function commitLayoutEffects(
  finishedWork: Fiber,
  root: FiberRoot,
  committedLanes: Lanes,
) {
  inProgressLanes = committedLanes;
  inProgressRoot = root;
  const current = finishedWork.alternate;
  commitLayoutEffectOnFiber(root, current, finishedWork, committedLanes);
  inProgressLanes = null;
  inProgressRoot = null;
}

function commitLayoutEffectOnFiber(
  finishedRoot,
  current,
  finishedWork,
  committedLanes,
) {
  const flags = finishedWork.flags;
  switch (finishedWork.tag) {
    case FunctionComponent:
    case ForwardRef:
    case SimpleMemoComponent: {
      recursivelyTraverseLayoutEffects(
        finishedRoot,
        finishedWork,
        committedLanes,
      );
      if (flags & Update) {
        // 同步执行 useLayoutEffect setup
        commitHookEffectListMount(HookLayout | HookHasEffect, finishedWork);
      }
      return;
    }
    case ClassComponent: {
      recursivelyTraverseLayoutEffects(
        finishedRoot,
        finishedWork,
        committedLanes,
      );
      if (flags & Update) {
        const instance = finishedWork.stateNode;
        if (current === null) {
          instance.componentDidMount(); // 初次渲染
        } else {
          const prevProps =
            finishedWork.elementType === finishedWork.type
              ? current.memoizedProps
              : resolveDefaultProps(finishedWork.type, current.memoizedProps);
          const prevState = current.memoizedState;
          instance.componentDidUpdate(
            // 更新阶段
            prevProps,
            prevState,
            instance.__reactInternalSnapshotBeforeUpdate,
          );
        }
      }
      if (flags & Callback) {
        const updateQueue: UpdateQueue<*> | null =
          (finishedWork.updateQueue: any);
        if (updateQueue !== null) {
          commitCallbacks(updateQueue, instance); // this.setState({}, callback) 的 callback
        }
      }
      if (flags & Ref) commitAttachRef(finishedWork);
      return;
    }
    case HostComponent: {
      recursivelyTraverseLayoutEffects(
        finishedRoot,
        finishedWork,
        committedLanes,
      );
      if (current === null && flags & Update) {
        const type = finishedWork.type;
        const props = finishedWork.memoizedProps;
        commitMount(finishedWork.stateNode, type, props, finishedWork); // 设置 focus 等
      }
      if (flags & Ref) commitAttachRef(finishedWork);
      return;
    }
    // ...其他 case
  }
}
```

在`commitLayoutEffectOnFiber`函数中:

- 对于`FunctionComponent`等节点, 同步调用所有`useLayoutEffect`的 setup.
- 对于`ClassComponent`节点, 调用生命周期函数`componentDidMount`或`componentDidUpdate`, 并执行`update.callback`回调.
- 对于`HostComponent`节点, 如有`Update`标记, 需要设置一些原生状态(如: `focus`等).
- 所有节点的 ref 在此阶段重新挂载.

> v17 → v19 关键变化: ref 挂载顺序更细化为先卸载旧 ref(`commitMutationEffects`内)→ 应用 DOM mutation → 挂载新 ref(`commitLayoutEffects`内). 这样保证父 → 子卸载、子 → 父挂载, 避免父组件 ref 还在指向旧 DOM.

### 渲染后

执行完上述步骤之后, 本次渲染任务就已经完成了. 在渲染完成后, 需要做一些重置和清理工作:

1. **不再需要清除副作用队列**: 在 v17 时期, 副作用是一个单向链表, 由于`fiber.firstEffect / nextEffect / lastEffect`的相互引用, 需要主动拆链让`fiber`对象能被 gc 回收. 在 v18+ 中链表已删除, 所有副作用信息都存放在`fiber.flags / subtreeFlags / deletions`三个字段上, 在 commit 完成、`root.current`切换后, 旧的 alternate fiber 树会被替换为新的, 老的`fiber`自然失去引用被 gc.

2. **检测更新**:
   - 在整个渲染过程中, 有可能产生新的`update`(比如在`componentDidMount`函数中, 再次调用`setState()`).
   - 调用`ensureRootIsScheduled(root)`确保新任务已经加入待调度链表, 由 microtask 末尾的`processRootScheduleInMicrotask`统一处理同步/并发. 不再有"`flushSyncCallbackQueue`主动冲刷"的额外调用.

```js
// v19 末尾收尾代码 (简化)
ensureRootIsScheduled(root);

// 检测渲染期间产生的嵌套更新, 防止死循环 (NESTED_UPDATE_LIMIT = 50)
if (
  workInProgressRootRenderPhaseUpdatedLanes !== NoLanes &&
  // ...
) {
  nestedUpdateScheduled = true;
  // ...
}
```

> 旧图`clear-effectlist.png`在 v19 中已不再适用(没有 effect list 可拆), 图待重绘.

## 总结

本节分析了`fiber 树渲染`的处理过程, 从宏观上看`fiber 树渲染`位于`reconciler 运作流程`中的输出阶段, 是整个`reconciler 运作流程`的链路中最后一环(从输入到输出). 本节根据 v19 源码, 具体从`渲染前, 渲染, 渲染后`三个方面分解了`commitRootImpl`函数. 其中最核心的`渲染`逻辑又分为了 3 个函数, 这 3 个函数共同处理了有副作用的`fiber`节点, 并通过渲染器`react-dom`把最新的 DOM 对象渲染到界面上.

v18 起, 整个 commit 流水线最大的变化在于:

1. **副作用链表删除**, 改为基于`subtreeFlags`位掩码的 DFS 剪枝.
2. **删除节点**通过`fiber.deletions`数组在父节点统一处理.
3. **新增 useInsertionEffect**, 在 mutation 阶段同步执行, 给 CSS-in-JS 库一个干净的"DOM 已变更但还没 layout 测量"窗口.
4. **commit 不再切换调度优先级**, 通过`setCurrentUpdatePriority(DiscreteEventPriority)`让 commit 期间触发的更新落入同步车道.
