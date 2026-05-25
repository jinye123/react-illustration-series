---
title: fiber 树构造(基础准备)
group: 运行核心
order: 4
---

# fiber 树构造(基础准备)

在 React 运行时中, `fiber树构造`位于`react-reconciler`包.

在正式解读`fiber树构造`之前, 再次回顾一下[reconciler 运作流程](./reconciler-workflow.md)的 4 个阶段:

![](../../snapshots/reconciler-workflow/reactfiberworkloop.png)

1. 输入阶段: 衔接`react-dom`包, 承接`fiber更新`请求(可以参考[React 应用的启动过程](./bootstrap.md)).
2. 注册调度任务: 与调度中心(`scheduler`包)交互, 注册调度任务`task`, 等待任务回调(可以参考[React 调度原理(scheduler)](./scheduler.md)).
3. 执行任务回调: 在内存中构造出`fiber树`和`DOM`对象, 也是**fiber 树构造的重点内容**.
4. 输出: 与渲染器(`react-dom`)交互, 渲染`DOM`节点.

`fiber树构造`处于上述第 3 个阶段, 可以通过不同的视角来理解`fiber树构造`在`React`运行时中所处的位置:

- 从`scheduler`调度中心的角度来看, 它是任务队列`taskQueue`中的一个具体的任务回调(`task.callback`).
- 从[React 工作循环](./workloop.md)的角度来看, 它属于`fiber树构造循环`.

由于`fiber 树构造`源码量比较大, 本系列根据`React`运行的`内存状态`, 分为 2 种情况来说明:

1. 初次创建: 在`React`应用首次启动时, 界面还没有渲染, 此时并不会进入对比过程, 相当于直接构造一棵全新的树.
2. 对比更新: `React`应用启动后, 界面已经渲染. 如果再次发生更新, 创建`新fiber`之前需要和`旧fiber`进行对比. 最后构造的 fiber 树有可能是全新的, 也可能是部分更新的.

无论是`初次创建`还是`对比更新`, 基础概念都是通用的, 本节将介绍这些基础知识, 为正式进入`fiber树构造`做准备.

## ReactElement, Fiber, DOM 三者的关系

在[React 应用中的高频对象](./object-structure.md)一文中, 已经介绍了`ReactElement`和`Fiber`对象的数据结构. 这里我们梳理出`ReactElement, Fiber, DOM`这 3 种对象的关系

1. [ReactElement 对象](https://github.com/facebook/react/blob/v19.2.6/packages/react/src/jsx/ReactJSXElement.js)(type 定义在[shared 包中](https://github.com/facebook/react/blob/v19.2.6/packages/shared/ReactElementType.js))

   - 所有采用`jsx`语法书写的节点, 都会被编译器转换, 最终会以新的 jsx-runtime(`jsx`/`jsxs`)创建出来一个与之对应的`ReactElement`对象

2. [fiber 对象](https://github.com/facebook/react/blob/v19.2.6/packages/react-reconciler/src/ReactFiber.js)(type 类型的定义在[ReactInternalTypes.js](https://github.com/facebook/react/blob/v19.2.6/packages/react-reconciler/src/ReactInternalTypes.js)中)

   - `fiber对象`是通过`ReactElement`对象进行创建的, 多个`fiber对象`构成了一棵`fiber树`, `fiber树`是构造`DOM树`的数据模型, `fiber树`的任何改动, 最后都体现到`DOM树`.

3. [DOM 对象](https://developer.mozilla.org/zh-CN/docs/Web/API/Document_Object_Model): 文档对象模型
   - `DOM`将文档解析为一个由节点和对象（包含属性和方法的对象）组成的结构集合, 也就是常说的`DOM树`.
   - `JavaScript`可以访问和操作存储在 DOM 中的内容, 也就是操作`DOM对象`, 进而触发 UI 渲染.

它们之间的关系反映了我们书写的 JSX 代码到 DOM 节点的转换过程:

![](../../snapshots/fibertree-create/code2dom.png)

注意:

- 开发人员能够控制的是`JSX`, 也就是`ReactElement`对象.
- `fiber树`是通过`ReactElement`生成的, 如果脱离了`ReactElement`,`fiber树`也无从谈起. 所以是`ReactElement`树(不是严格的树结构, 为了方便也称为树)驱动`fiber树`.
- `fiber树`是`DOM树`的数据模型, `fiber树`驱动`DOM树`

开发人员通过编程只能控制`ReactElement`树的结构, `ReactElement树`驱动`fiber树`, `fiber树`再驱动`DOM树`, 最后展现到页面上. 所以`fiber树`的构造过程, 实际上就是`ReactElement`对象到`fiber`对象的转换过程.

## 全局变量

从[React 工作循环](./workloop.md)的角度来看, 整个构造过程被包裹在`fiber树构造循环`中(对应源码位于[ReactFiberWorkLoop.js](https://github.com/facebook/react/blob/v19.2.6/packages/react-reconciler/src/ReactFiberWorkLoop.js)).

在`React`运行时, `ReactFiberWorkLoop.js`闭包中的`全局变量`会随着`fiber树构造循环`的进行而变化, 现在查看其中重要的全局变量([源码链接](https://github.com/facebook/react/blob/v19.2.6/packages/react-reconciler/src/ReactFiberWorkLoop.js)):

```js
// 当前React的执行栈(执行上下文)
let executionContext: ExecutionContext = NoContext;

// 当前root节点
let workInProgressRoot: FiberRoot | null = null;
// 正在处理中的fiber节点
let workInProgress: Fiber | null = null;
// 正在渲染的车道(复数)
let workInProgressRootRenderLanes: Lanes = NoLanes;

// v18 新增: 在 render 期间出现的、与本次渲染车道纠缠(entangled)在一起的车道集合
// 一旦发生纠缠, 这些车道必须在同一次提交中一起处理 (用于实现 useTransition / useDeferredValue 的语义)
let entangledRenderLanes: Lanes = NoLanes;

// 包含所有子节点的优先级, 是workInProgressRootRenderLanes的超集
// 大多数情况下: 在工作循环整体层面会使用workInProgressRootRenderLanes, 在begin/complete阶段层面会使用 subtreeRenderLanes
let subtreeRenderLanes: Lanes = NoLanes;
// 一个栈结构: 专门存储当前节点的 subtreeRenderLanes
const subtreeRenderLanesCursor: StackCursor<Lanes> = createCursor(NoLanes);

// fiber构造完后, root节点的状态: completed, errored, suspended等
let workInProgressRootExitStatus: RootExitStatus = RootInProgress;
// 重大错误
let workInProgressRootFatalError: mixed = null;
// 整个render期间所使用到的所有lanes
let workInProgressRootIncludedLanes: Lanes = NoLanes;
// 在render期间被跳过(由于优先级不够)的lanes: 只包括未处理的updates, 不包括被复用的fiber节点
let workInProgressRootSkippedLanes: Lanes = NoLanes;
// v18 新增: 在并发 render 期间, 由穿插更新产生的 lanes
let workInProgressRootInterleavedUpdatedLanes: Lanes = NoLanes;
// v18 新增: 跨 render 累计的 interleaved 更新 (用于决定是否需要打断)
let workInProgressRootRenderPhaseUpdatedLanes: Lanes = NoLanes;
// 在render期间被修改过的lanes
let workInProgressRootUpdatedLanes: Lanes = NoLanes;
// v18 新增: 在 render 期间被 pinged 的 lanes (Suspense 解除挂起触发)
let workInProgressRootPingedLanes: Lanes = NoLanes;

// 防止无限循环和嵌套更新
const NESTED_UPDATE_LIMIT = 50;
let nestedUpdateCount: number = 0;
let rootWithNestedUpdates: FiberRoot | null = null;

const NESTED_PASSIVE_UPDATE_LIMIT = 50;
let nestedPassiveUpdateCount: number = 0;
let rootWithPassiveNestedUpdates: FiberRoot | null = null;

// v18 起 currentEventTime 已被删除 (因为统一使用 lane 模型, 不再用 eventTime 计算优先级)
```

在源码中, 大部分变量都带有英文注释(读者可自行查阅), 此处只列举了`fiber树构造循环`中最核心的变量.

> v17 → v19 关键变化:
>
> - 删除 `currentEventTime / currentEventWipLanes / currentEventPendingLanes` 三个事件时间字段(`Lane`模型不再依赖`expirationTime`衍生).
> - 新增 `entangledRenderLanes` 用于 Transition / useDeferredValue 等"纠缠车道"的实现.
> - 新增 `workInProgressRootInterleavedUpdatedLanes / workInProgressRootRenderPhaseUpdatedLanes / workInProgressRootPingedLanes` 等记录在 render 期间发生的更新.
> - `RootIncomplete` 状态枚举重命名为 `RootInProgress`(语义更准确).

### 执行上下文

在全局变量中有`executionContext`, 代表`渲染期间`的`执行栈`(或叫做`执行上下文`), 它也是一个二进制表示的变量, 通过位运算进行操作(参考[React 算法之位运算](../algorithm/bitfield.md)). v19 中只保留了 4 种执行栈位:

```js
type ExecutionContext = number;
export const NoContext = /*             */ 0b000;
const BatchedContext = /*               */ 0b001;
const RenderContext = /*                */ 0b010;
const CommitContext = /*                */ 0b100;
```

> v17 → v19 删除位: `EventContext / DiscreteEventContext / LegacyUnbatchedContext`. 因为 v18 起所有更新都自动批处理(automatic batching), 不再需要"是否处于离散事件"这类上下文标记; `LegacyUnbatchedContext`随`ReactDOM.render`一同退役.

上文回顾了`reconciler 运作流程`的 4 个阶段, 这 4 个阶段只是一个整体划分. v19 中所有更新都统一进入第 2 阶段(注册调度任务), 不再有"legacy 直接跳到第 3 阶段"的快捷路径.

事实上正是`executionContext`在操控`reconciler 运作流程`(源码体现在[scheduleUpdateOnFiber 函数](https://github.com/facebook/react/blob/v19.2.6/packages/react-reconciler/src/ReactFiberWorkLoop.js)).

```js
// v19 实际签名: 不再有 LegacyUnbatchedContext 分支
export function scheduleUpdateOnFiber(
  root: FiberRoot,
  fiber: Fiber,
  lane: Lane,
) {
  // 检测是否处于无效的嵌套更新
  if (
    (executionContext & RenderContext) !== NoLanes &&
    root === workInProgressRoot
  ) {
    workInProgressRootRenderPhaseUpdatedLanes |= lane;
  }

  // 标记 root 上的 update lane
  markRootUpdated(root, lane);

  // 统一进入调度链路: 同步走 microtask, 并发走 Scheduler
  ensureRootIsScheduled(root);
}
```

在 render 过程中, 每一个阶段都会改变`executionContext`(render 之前, 会设置`executionContext |= RenderContext`; commit 之前, 会设置`executionContext |= CommitContext`), 假设在`render`过程中再次发起更新(如在`UNSAFE_componentWillReceiveProps`生命周期中调用`setState`), 则可通过`executionContext`来判断当前的`render`状态, 并将 lane 累积到`workInProgressRootRenderPhaseUpdatedLanes`中, 用于在 render 结束后决定是否需要重新开始构造.

### 双缓冲技术(double buffering)

在全局变量中有`workInProgress`, 还有不少以`workInProgress`来命名的变量. `workInProgress`的应用实际上就是`React`的双缓冲技术(`double buffering`).

在上文我们梳理了`ReactElement, Fiber, DOM三者的关系`, `fiber树`的构造过程, 就是把`ReactElement`转换成`fiber树`的过程. 在这个过程中, 内存里会同时存在 2 棵`fiber树`:

- 其一: 代表当前界面的`fiber`树(已经被展示出来, 挂载到`fiberRoot.current`上). 如果是初次构造(`初始化渲染`), 页面还没有渲染, 此时界面对应的 fiber 树为空(`fiberRoot.current = null`).
- 其二: 正在构造的`fiber`树(即将展示出来, 挂载到`HostRootFiber.alternate`上, 正在构造的节点称为`workInProgress`). 当构造完成之后, 重新渲染页面, 最后切换`fiberRoot.current = workInProgress`, 使得`fiberRoot.current`重新指向代表当前界面的`fiber`树.

此处涉及到 2 个全局对象`fiberRoot`和`HostRootFiber`, 在[React 应用的启动过程](./bootstrap.md)中有详细的说明.

用图来表述`double buffering`的概念如下:

1. 构造过程中, `fiberRoot.current`指向当前界面对应的`fiber`树.

![](../../snapshots/fibertree-create/fibertreecreate1-progress.png)

2. 构造完成并渲染, 切换`fiberRoot.current`指针, 使其继续指向当前界面对应的`fiber`树(原来代表界面的 fiber 树, 变成了内存中).

![](../../snapshots/fibertree-create/fibertreecreate2-complete.png)

### 优先级 {#lanes}

在全局变量中有不少变量都以 Lanes 命名(如`workInProgressRootRenderLanes`,`subtreeRenderLanes`其作用见上文注释), 它们都与优先级相关.

在前文[React 中的优先级管理](./priority.md)中, 我们介绍了`React`中的`Lane / EventPriority / SchedulerPriority`三层模型, 并了解了它们之间的关联. 现在`fiber树构造`过程中, 将要深入分析车道模型`Lane`的具体应用.

在整个`react-reconciler`包中, `Lane`的应用可以分为 3 个方面:

#### `update`优先级(update.lane) {#update-lane}

在[React 应用中的高频对象](./object-structure.md#Update)一文中, 介绍过`update`对象, 它是一个环形链表. 对于单个`update`对象来讲, `update.lane`代表它的优先级, 称之为`update`优先级.

观察其构造函数([源码链接](https://github.com/facebook/react/blob/v19.2.6/packages/react-reconciler/src/ReactFiberClassUpdateQueue.js)), 其优先级是由外界传入. **注意 v18 起删除了`eventTime`参数**:

```js
export function createUpdate(lane: Lane): Update<*> {
  const update: Update<*> = {
    lane,
    tag: UpdateState,
    payload: null,
    callback: null,
    next: null,
  };
  return update;
}
```

在`React`体系中, 有 2 种情况会创建`update`对象:

1. 应用初始化: 在`react-reconciler`包中的`updateContainer`函数中([源码](https://github.com/facebook/react/blob/v19.2.6/packages/react-reconciler/src/ReactFiberReconciler.js))

   ```js
   export function updateContainer(
     element: ReactNodeList,
     container: OpaqueRoot,
     parentComponent: ?React$Component<any, any>,
     callback: ?Function,
   ): Lane {
     const current = container.current;
     const lane = requestUpdateLane(current); // 选取 update 优先级
     const update = createUpdate(lane); // 创建 update 对象
     update.payload = { element };
     const root = enqueueUpdate(current, update, lane);
     if (root !== null) {
       scheduleUpdateOnFiber(root, current, lane);
       entangleTransitions(root, current, lane); // v18 新增: 处理 transition 纠缠
     }
     return lane;
   }
   ```

2. 发起组件更新: 假设在 class 组件中调用`setState`([源码](https://github.com/facebook/react/blob/v19.2.6/packages/react-reconciler/src/ReactFiberClassComponent.js))

```js
const classComponentUpdater = {
  isMounted,
  enqueueSetState(inst, payload, callback) {
    const fiber = getInstance(inst);
    const lane = requestUpdateLane(fiber);
    const update = createUpdate(lane);
    update.payload = payload;
    const root = enqueueUpdate(fiber, update, lane);
    if (root !== null) {
      scheduleUpdateOnFiber(root, fiber, lane);
      entangleTransitions(root, fiber, lane);
    }
  },
};
```

可以看到, 无论是`应用初始化`或者`发起组件更新`, 创建`update.lane`的逻辑都是一样的, 都是根据当前的"事件优先级"来生成一个`Lane`.

[requestUpdateLane](https://github.com/facebook/react/blob/v19.2.6/packages/react-reconciler/src/ReactFiberWorkLoop.js):

```js
// v19 实际实现, 大幅简化
export function requestUpdateLane(fiber: Fiber): Lane {
  // 1. legacy 模式仅用于内部桥接, 业务无法触达
  const mode = fiber.mode;
  if ((mode & ConcurrentMode) === NoMode) {
    return (SyncLane: Lane);
  }

  // 2. 处于 render 阶段的更新: 复用当前 renderLanes 中的某条 lane
  if (
    (executionContext & RenderContext) !== NoContext &&
    workInProgressRootRenderLanes !== NoLanes
  ) {
    return pickArbitraryLane(workInProgressRootRenderLanes);
  }

  // 3. 处于 transition: 选取一条 TransitionLane (16 条循环复用)
  const transition = requestCurrentTransition();
  if (transition !== null) {
    if (transition._updatedFibers !== undefined) {
      transition._updatedFibers.add(fiber);
    }
    const actionScopeLane = peekEntangledActionLane();
    return actionScopeLane !== NoLane
      ? actionScopeLane
      : requestTransitionLane(transition);
  }

  // 4. 普通更新: 用当前事件优先级
  const updateLane: Lane = (getCurrentUpdatePriority(): any);
  if (updateLane !== NoLane) {
    return updateLane;
  }

  // 5. 事件外的更新 (Promise/setTimeout): 回落到 DefaultLane (v18 起 setTimeout 也会自动批处理)
  const eventLane: Lane = (getCurrentEventPriority(): any);
  return eventLane;
}
```

可以看到`requestUpdateLane`的作用是返回一个合适的 update 优先级:

1. 业务代码不再进入"legacy / blocking"分支(只剩 ConcurrentMode 一条主路).
2. **render 阶段触发的更新**会被立刻挑到当前 renderLanes 之中(避免被跳过).
3. **transition 中的更新**走专门的 `TransitionLane1 ~ TransitionLane15`(共 15 条循环复用), 并能与`peekEntangledActionLane()`所返回的 action lane 纠缠在一起(实现 Actions 的"同一 Transition 共享 lane"语义).
4. **事件内的更新**直接使用`getCurrentUpdatePriority()`(由合成事件入口提前 set 好).
5. **事件外的更新(Promise / setTimeout)**回落到`DefaultLane`. 注意 v18 起这种"事件外的更新"也会被自动批处理(automatic batching).

最后通过`scheduleUpdateOnFiber(root, current, lane);`函数, 把`update.lane`正式带入到了`输入`阶段.

`scheduleUpdateOnFiber`是`输入`阶段的必经函数, 在本系列的文章中已经多次提到, 此处以`update.lane`的视角分析:

```js
export function scheduleUpdateOnFiber(
  root: FiberRoot,
  fiber: Fiber,
  lane: Lane,
) {
  // 检测 render 阶段嵌套更新
  if (
    (executionContext & RenderContext) !== NoLanes &&
    root === workInProgressRoot
  ) {
    workInProgressRootRenderPhaseUpdatedLanes |= lane;
  }
  // 标记 root 上的 lane
  markRootUpdated(root, lane);
  // 进入第2阶段: 不分同步/并发, 一律走 ensureRootIsScheduled
  ensureRootIsScheduled(root);
}
```

> 与 v17 相比, 这里有 3 个核心差异:
>
> 1. 没有"legacy 同步绕过调度"的快捷分支(`LegacyUnbatchedContext`已删除).
> 2. 同步车道不再调用`flushSyncCallbackQueue()`立即冲刷, 而是延迟到 microtask 末尾由`processRootScheduleInMicrotask`统一处理. 这就是 React 18 [automatic batching](https://react.dev/blog/2022/03/29/react-v18#new-feature-automatic-batching) 的内核.
> 3. 因此, "`setState`到底是同步还是异步"在 v18+ 中已经统一为: **始终在 microtask 末尾批量冲刷**, 当下一行代码同步执行时, DOM 还没有更新(行为表现为"异步").

#### `渲染`优先级(renderLanes)

这是一个全局概念, 每一次`render`之前, 首先要确定本次`render`的优先级. 具体对应到源码如下:

```js
// v19 实际入口: performWorkOnRoot(root, lanes, forceSync)
function performWorkOnRoot(root, lanes, forceSync) {
  // 1. 选择 render 模式: 同步 / 并发
  const shouldTimeSlice =
    !forceSync &&
    !includesBlockingLane(lanes) &&
    !includesExpiredLane(root, lanes);
  let exitStatus = shouldTimeSlice
    ? renderRootConcurrent(root, lanes)
    : renderRootSync(root, lanes, true);
}
```

v18 起所有进入 render 的入口都通过`performWorkOnRoot`(详见[reconciler 运作流程](./reconciler-workflow.md)). 在调用前, 上层(`ensureRootIsScheduled` / `processRootScheduleInMicrotask`) 已经调用过`getNextLanes(root, ...)`挑出本次要处理的`lanes`. 该函数([源码链接](https://github.com/facebook/react/blob/v19.2.6/packages/react-reconciler/src/ReactFiberLane.js)) 主要逻辑可简化为:

```js
// v19 简化版 getNextLanes
export function getNextLanes(root: FiberRoot, wipLanes: Lanes): Lanes {
  const pendingLanes = root.pendingLanes;
  if (pendingLanes === NoLanes) return NoLanes;

  let nextLanes = NoLanes;
  const suspendedLanes = root.suspendedLanes;
  const pingedLanes = root.pingedLanes;
  const warmLanes = root.warmLanes; // v19 新增: 已"预热"但还未点亮的 lanes

  // 1. 优先处理非 idle 的"非挂起"任务
  const nonIdlePendingLanes = pendingLanes & NonIdleLanes;
  if (nonIdlePendingLanes !== NoLanes) {
    const nonIdleUnblockedLanes = nonIdlePendingLanes & ~suspendedLanes;
    if (nonIdleUnblockedLanes !== NoLanes) {
      nextLanes = getHighestPriorityLanes(nonIdleUnblockedLanes);
    } else {
      // 全部被挂起, 看 pinged 是否解除挂起
      const nonIdlePingedLanes = nonIdlePendingLanes & pingedLanes;
      if (nonIdlePingedLanes !== NoLanes) {
        nextLanes = getHighestPriorityLanes(nonIdlePingedLanes);
      }
    }
  } else {
    // 2. 只剩 idle, 按 idle 选取
    const unblockedLanes = pendingLanes & ~suspendedLanes;
    if (unblockedLanes !== NoLanes) {
      nextLanes = getHighestPriorityLanes(unblockedLanes);
    } else if (pingedLanes !== NoLanes) {
      nextLanes = getHighestPriorityLanes(pingedLanes);
    }
  }
  // 3. 如果正在 render 的 wipLanes 优先级更高, 沿用 wipLanes 避免打断自己
  if (
    wipLanes !== NoLanes &&
    wipLanes !== nextLanes &&
    (wipLanes & suspendedLanes) === NoLanes
  ) {
    const nextLane = getHighestPriorityLane(nextLanes);
    const wipLane = getHighestPriorityLane(wipLanes);
    if (
      nextLane >= wipLane ||
      (nextLane === DefaultLane && (wipLane & TransitionLanes) !== NoLanes)
    ) {
      return wipLanes;
    }
  }
  return nextLanes;
}
```

`getNextLanes`会根据`fiberRoot`对象上的属性(`expiredLanes`, `suspendedLanes`, `pingedLanes`等), 确定出当前最紧急的`lanes`.

此处返回的`lanes`会作为全局渲染的优先级, 用于`fiber树构造过程`中. 针对`fiber对象`或`update对象`, 只要它们的优先级(如: `fiber.lanes`和`update.lane`)比`渲染优先级`低, 都将会被忽略.

#### `fiber`优先级(fiber.lanes)

在[React 应用中的高频对象](./object-structure.md)一文中, 介绍过`fiber`对象的数据结构. 其中有 2 个属性与优先级相关:

1. `fiber.lanes`: 代表本节点的优先级
2. `fiber.childLanes`: 代表子节点的优先级
   从`FiberNode`的构造函数中可以看出, `fiber.lanes`和`fiber.childLanes`的初始值都为`NoLanes`, 在`fiber树构造`过程中, 使用全局的渲染优先级(`renderLanes`)和`fiber.lanes`判断`fiber`节点是否更新([源码地址](https://github.com/facebook/react/blob/v19.2.6/packages/react-reconciler/src/ReactFiberBeginWork.js)).
   - 如果全局的渲染优先级`renderLanes`不包括`fiber.lanes`, 证明该`fiber`节点没有更新, 可以复用.
   - 如果不能复用, 进入创建阶段.

```js
function beginWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes,
): Fiber | null {
  const updateLanes = workInProgress.lanes;
  if (current !== null) {
    const oldProps = current.memoizedProps;
    const newProps = workInProgress.pendingProps;
    if (
      oldProps !== newProps ||
      hasLegacyContextChanged() ||
      // Force a re-render if the implementation changed due to hot reload:
      (__DEV__ ? workInProgress.type !== current.type : false)
    ) {
      didReceiveUpdate = true;
    } else if (!includesSomeLane(renderLanes, updateLanes)) {
      didReceiveUpdate = false;
      // 本`fiber`节点的没有更新, 可以复用, 进入bailout逻辑
      return bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes);
    }
  }
  // 不能复用, 创建新的fiber节点
  workInProgress.lanes = NoLanes; // 重置优先级为 NoLanes
  switch (workInProgress.tag) {
    case ClassComponent: {
      const Component = workInProgress.type;
      const unresolvedProps = workInProgress.pendingProps;
      const resolvedProps =
        workInProgress.elementType === Component
          ? unresolvedProps
          : resolveDefaultProps(Component, unresolvedProps);

      return updateClassComponent(
        current,
        workInProgress,
        Component,
        resolvedProps,
        // 正常情况下渲染优先级会被用于fiber树的构造过程
        renderLanes,
      );
    }
  }
}
```

### 栈帧管理

在`React`源码中, 每一次执行`fiber树`构造(v19 统一入口为`performWorkOnRoot(root, lanes, forceSync)`, 它替代了 v17 的`performSyncWorkOnRoot / performConcurrentWorkOnRoot`两个独立入口)的过程, 都需要一些全局变量来保存状态. 在上文中已经介绍最核心的全局变量.

如果从单个变量来看, 它们就是一个个的全局变量. 如果将这些全局变量组合起来, 它们代表了当前`fiber树`构造的活动记录. 通过这一组全局变量, 可以还原`fiber树`构造过程(比如时间切片的实现过程(参考[React 调度原理](./scheduler.md#内核)), `fiber树`构造过程被打断之后需要还原进度, 全靠这一组全局变量). 所以每次`fiber树`构造是一个独立的过程, 需要`独立的`一组全局变量, 在`React`内部把这一个独立的过程封装为一个栈帧`stack`(简单来说就是每次构造都需要独立的空间. 对于`栈帧`的深入理解, 请读者自行参考其他资料).

所以在进行`fiber树`构造之前, 如果不需要恢复上一次构造进度, 都会刷新栈帧(源码在[prepareFreshStack 函数](https://github.com/facebook/react/blob/v19.2.6/packages/react-reconciler/src/ReactFiberWorkLoop.js))

```js
function renderRootConcurrent(root: FiberRoot, lanes: Lanes) {
  const prevExecutionContext = executionContext;
  executionContext |= RenderContext;
  const prevDispatcher = pushDispatcher(root.containerInfo);
  const prevAsyncDispatcher = pushAsyncDispatcher();
  // 如果fiberRoot变动, 或者update.lane变动, 都会刷新栈帧, 丢弃上一次渲染进度
  if (workInProgressRoot !== root || workInProgressRootRenderLanes !== lanes) {
    workInProgressTransitions = getTransitionsForLanes(root, lanes);
    resetRenderTimer();
    // 刷新栈帧
    prepareFreshStack(root, lanes);
  }
}

/**
刷新栈帧: 重置 FiberRoot 上的全局属性 和 `fiber树构造`循环过程中的全局变量
*/
function prepareFreshStack(root: FiberRoot, lanes: Lanes): Fiber {
  // 重置FiberRoot对象上的属性
  root.finishedWork = null;
  root.finishedLanes = NoLanes;
  const timeoutHandle = root.timeoutHandle;
  if (timeoutHandle !== noTimeout) {
    root.timeoutHandle = noTimeout;
    cancelTimeout(timeoutHandle);
  }
  // v19 新增: 用 cancelPendingCommit 取消那些"已经准备 commit 但还在等待资源(suspendy images / fonts)"的提交
  const cancelPendingCommit = root.cancelPendingCommit;
  if (cancelPendingCommit !== null) {
    root.cancelPendingCommit = null;
    cancelPendingCommit();
  }
  // 中断上一次正在构造的栈
  if (workInProgress !== null) {
    let interruptedWork = workInProgress.return;
    while (interruptedWork !== null) {
      const current = interruptedWork.alternate;
      unwindInterruptedWork(
        current,
        interruptedWork,
        workInProgressRootRenderLanes,
      );
      interruptedWork = interruptedWork.return;
    }
  }
  // 重置全局变量
  workInProgressRoot = root;
  const rootWorkInProgress = createWorkInProgress(root.current, null);
  workInProgress = rootWorkInProgress; // 给HostRootFiber对象创建一个alternate, 并将其设置成全局 workInProgress
  workInProgressRootRenderLanes = lanes;
  workInProgressSuspendedReason = NotSuspended;
  workInProgressThrownValue = null;
  workInProgressRootDidSkipSuspendedSiblings = false;
  workInProgressRootIsPrerendering = checkIfRootIsPrerendering(root, lanes);
  workInProgressRootDidAttachPingListener = false;
  workInProgressRootExitStatus = RootInProgress;
  workInProgressSuspendedRetryLanes = NoLanes;
  workInProgressRootSkippedLanes = NoLanes;
  workInProgressRootInterleavedUpdatedLanes = NoLanes;
  workInProgressRootRenderPhaseUpdatedLanes = NoLanes;
  workInProgressRootPingedLanes = NoLanes;
  workInProgressRootConcurrentErrors = null;
  workInProgressRootRecoverableErrors = null;
  workInProgressAppearingViewTransitions = null;
  workInProgressRootIncludedLanes = lanes;
  // v18 起 entangledRenderLanes 在 prepareFreshStack 时计算, 并通过 push 到栈管理
  entangledRenderLanes = getEntangledLanes(root, lanes);

  finishQueueingConcurrentUpdates(); // v18 新增: 把 interleaved 更新冲入正式 updateQueue

  return rootWorkInProgress;
}
```

> v17 → v19 关键差异:
>
> - 新增 `cancelPendingCommit` 流程, 处理 Suspensey resources(图片/字体等)未就绪时的"挂起 commit".
> - 新增 `workInProgressSuspendedReason / workInProgressThrownValue` 等字段, 用于 Suspense sibling pre-warming 的 throw-and-retry 流程.
> - 新增 `entangledRenderLanes` 与 `finishQueueingConcurrentUpdates`, 把并发场景下穿插到队列里的 update 冲入正式 updateQueue.
> - 删除了 `startWorkOnPendingInteractions`(`scheduler/tracing` 已移除).

注意其中的`createWorkInProgress(root.current, null)`, 其参数`root.current`即`HostRootFiber`, 作用是给`HostRootFiber`创建一个`alternate`副本.`workInProgress`指针指向这个副本(即`workInProgress = HostRootFiber.alternate`), 在上文`double buffering`中分析过, `HostRootFiber.alternate`是`正在构造的fiber树`的根节点.

## 总结

本节是`fiber树构造`的准备篇, 首先在宏观上从不同的视角(`任务调度循环`, `fiber树构造循环`)介绍了`fiber树构造`在`React`体系中所处的位置, 然后深入`react-reconciler`包分析`fiber树构造`过程中需要使用到的全局变量, 并解读了`双缓冲技术`和`优先级(车道模型)`的使用, 最后解释`栈帧管理`的实现细节. 有了这些基础知识, `fiber树构造`的具体实现过程会更加简单清晰.
