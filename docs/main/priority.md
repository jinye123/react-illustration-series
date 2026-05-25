---
title: 优先级管理
group: 运行核心
order: 2
---

# React 中的优先级管理

> `React`内部对于`优先级`的管理, 自 v18 起经历了一次精简: 删除了 v17 的`LanePriority`枚举(以及`SchedulerWithReactIntegration.js`的转换层), 现在只剩 `Lane(Lanes)` 与 `SchedulerPriority` 两套核心模型, 通过一个轻量的`EventPriority`枚举进行衔接. 本文基于`react@19.2.6`, 梳理源码中的优先级管理体系.

`React`是一个声明式, 高效且灵活的用于构建用户界面的 JavaScript 库. React 团队一直致力于实现高效渲染, 其中有 2 个十分有名的演讲:

1. [2017 年 Lin Clark 的演讲](https://conf2017.reactjs.org/speakers/lin)中介绍了`fiber`架构和`可中断渲染`.
2. [2018 年 Dan 在 JSConf 冰岛的演讲](https://legacy.reactjs.org/blog/2018/03/01/sneak-peek-beyond-react-16.html)进一步介绍了时间切片(`time slicing`)和异步渲染(`suspense`)等特性.

演讲中所展示的`可中断渲染`、`时间切片(time slicing)`、`异步渲染(suspense)`等特性, 在源码中得以实现都依赖于`优先级管理`. 这些特性也在`react@18.0.0`以后真正落地到了稳定版.

在`react@19.2.6`源码中, 优先级被组织为 **`Lane(Lanes)` ↔ `EventPriority` ↔ `SchedulerPriority`** 三层关系, 在深入分析之前, 再次回顾一下([reconciler 运作流程](./reconciler-workflow.md)):

![](../../snapshots/reconciler-workflow/reactfiberworkloop.png)

`React`内部对于`优先级`的管理, 贯穿运作流程的 4 个阶段(从输入到输出). v19 中根据其功能的不同, 可以分为 2 种类型 + 1 个转换层:

1. **`Lane(Lanes)`**: 位于`react-reconciler`包, 即[`Lane(车道模型)`](https://github.com/facebook/react/pull/18796). 直接挂在 `fiber.lanes / fiber.childLanes / update.lane / root.pendingLanes` 等字段上, 是 fiber 树构造的核心.
2. **`SchedulerPriority`**: 位于`scheduler`包. 用于向 Scheduler 注册任务的优先级.
3. **`EventPriority`**(转换层): 位于`react-reconciler`包的[`ReactEventPriorities.js`](https://github.com/facebook/react/blob/v19.2.6/packages/react-reconciler/src/ReactEventPriorities.js). 它本身是一个 `Lane` 子集, 把"事件来源的语义"映射到"车道", 并间接桥接到 SchedulerPriority.

> v17 → v19 关键变化:
>
> - 删除 `LanePriority` 枚举(原 18 种)与 `SchedulerWithReactIntegration.js` 转换函数.
> - 删除 `SyncBatchedLane / SyncBatchedLanePriority`(v18 全面 automatic batching, 不再需要"批同步"这个中间档).
> - 新增 `DiscreteEventPriority / ContinuousEventPriority / DefaultEventPriority / IdleEventPriority` 4 档 `EventPriority`.
> - 转换函数由旧的 `lanePriorityToSchedulerPriority` 改为 `lanesToEventPriority` + 直接 switch case 到 SchedulerPriority.

## 预备知识

在深入分析之前, 为了理解 `Lane`, 这是`react@17.0.0`引入、`react@18+`继续打磨的核心模型.

### Lane (车道模型)

> 英文单词`lane`翻译成中文表示"车道, 航道"的意思, 所以很多文章都将`Lanes`模型称为`车道模型`.

`Lane`模型的源码在[`ReactFiberLane.js`](https://github.com/facebook/react/blob/v19.2.6/packages/react-reconciler/src/ReactFiberLane.js), 源码中大量使用了位运算(有关位运算的讲解, 可以参考[React 算法之位运算](../algorithm/bitfield.md)).

首先引入作者对`Lane`的解释([相应的 pr](https://github.com/facebook/react/pull/18796)), 这里简单概括如下:

1. `Lane`类型被定义为二进制变量, 利用了位掩码的特性, 在频繁运算的时候占用内存少, 计算速度快.
   - `Lane`和`Lanes`就是单数和复数的关系, 代表单个任务的定义为`Lane`, 代表多个任务的定义为`Lanes`.
2. `Lane`是对于`expirationTime`的重构, 以前使用`expirationTime`表示的字段, 都改为了`lane`.

   ```js
     renderExpirationtime -> renderLanes
     update.expirationTime -> update.lane
     fiber.expirationTime -> fiber.lanes
     fiber.childExpirationTime -> fiber.childLanes
     root.firstPendingTime and root.lastPendingTime -> root.pendingLanes
   ```

3. 使用`Lanes`模型相比`expirationTime`模型的优势:

   1. `Lanes`把任务优先级从批量任务中分离出来, 可以更方便的判断单个任务与批量任务的优先级是否重叠.

      ```js
      // 判断: 单task与batchTask的优先级是否重叠
      //1. 通过expirationTime判断
      const isTaskIncludedInBatch = priorityOfTask >= priorityOfBatch;
      //2. 通过Lanes判断
      const isTaskIncludedInBatch = (task & batchOfTasks) !== 0;
      ```

   2. `Lanes`使用单个 32 位二进制变量即可代表多个不同的任务, 也就是说一个变量即可代表一个组(`group`), 如果要在一个 group 中分离出单个 task, 非常容易.

      ```js
      // 从group中删除或增加task
      // 1) 删除单个task
      batchOfTasks &= ~task;
      // 2) 增加单个task
      batchOfTasks |= task;
      // 3) 比较task是否在group中
      const isTaskIncludedInBatch = (task & batchOfTasks) !== 0;
      ```

      通过上述伪代码, 可以看到`Lanes`的优越性, 运用起来代码量少, 简洁高效.

4. `Lanes`是一个不透明的类型, 只能在[`ReactFiberLane.js`](https://github.com/facebook/react/blob/v19.2.6/packages/react-reconciler/src/ReactFiberLane.js)这个模块中维护. 如果要在其他文件中使用, 只能通过`ReactFiberLane.js`中提供的工具函数来使用.

分析车道模型的源码([`ReactFiberLane.js`](https://github.com/facebook/react/blob/v19.2.6/packages/react-reconciler/src/ReactFiberLane.js))中, 可以得到如下结论:

1. 可以使用的比特位一共有 31 位(为什么? 可以参考[React 算法之位运算](../algorithm/bitfield.md)中的说明).
2. 共定义了 30+ 种车道(`Lane/Lanes`)变量, 每一个变量占有 1 个或多个比特位, 分别定义为`Lane`和`Lanes`类型. 相比 v17 的 18 种, v19 主要新增了`TransitionLanes`(16 条)、`RetryLanes`(4 条)、`DeferredLane`等. **`LanePriority`枚举与配套的转换函数已删除**, 优先级直接由"哪些比特位被点亮"决定, 占有低位比特位的`Lane`变量优先级越高.
3. 占有低位比特位的`Lane`变量对应的优先级越高
   - 最高优先级为`SyncLane = 0b0000000000000000000000000000010`(注意 v18 起增加了`SyncHydrationLane`占据位 0, `SyncLane`挪到位 1).
   - 最低优先级为`OffscreenLane`/`DeferredLane`等高位车道.

v19 中的 Lane 大致按优先级分组如下:

```js
// 同步组
SyncHydrationLane; // 0b0000000000000000000000000000001
SyncLane; // 0b0000000000000000000000000000010
InputContinuousHydrationLane; // 0b0000000000000000000000000000100
InputContinuousLane; // 0b0000000000000000000000000001000
DefaultHydrationLane; // 0b0000000000000000000000000010000
DefaultLane; // 0b0000000000000000000000000100000

// 过渡组 (16 条)
TransitionHydrationLane; // 0b0000000000000000000000010000000
TransitionLane1; // 0b0000000000000000000000100000000
// ... TransitionLane2 ~ TransitionLane15

// 重试组 (4 条)
RetryLane1; // 0b0000000010000000000000000000000
// ... RetryLane2 ~ RetryLane4

// 其他
SelectiveHydrationLane; // 0b0000010000000000000000000000000
IdleHydrationLane; // 0b0000100000000000000000000000000
IdleLane; // 0b0001000000000000000000000000000
OffscreenLane; // 0b0010000000000000000000000000000
DeferredLane; // 0b0100000000000000000000000000000
```

## 优先级区别和联系

在源码中, 2 种核心优先级 + 1 个事件优先级转换层位于不同的 js 文件, 是相互独立的.

### Lane(s)

`Lane / Lanes`定义于[`ReactFiberLane.js`](https://github.com/facebook/react/blob/v19.2.6/packages/react-reconciler/src/ReactFiberLane.js):

```js
export const NoLane: Lane = /*                          */ 0b0000000000000000000000000000000;
export const SyncHydrationLane: Lane = /*               */ 0b0000000000000000000000000000001;
export const SyncLane: Lane = /*                        */ 0b0000000000000000000000000000010;
export const InputContinuousHydrationLane: Lane = /*    */ 0b0000000000000000000000000000100;
export const InputContinuousLane: Lanes = /*            */ 0b0000000000000000000000000001000;
export const DefaultHydrationLane: Lane = /*            */ 0b0000000000000000000000000010000;
export const DefaultLane: Lanes = /*                    */ 0b0000000000000000000000000100000;
// ... 省略后续 30+ 个 Lane 定义
export const OffscreenLane: Lane = /*                   */ 0b0010000000000000000000000000000;
export const DeferredLane: Lane = /*                    */ 0b0100000000000000000000000000000;
```

与`fiber`构造过程相关的优先级(如`fiber.updateQueue`、`fiber.lanes`)都使用`Lane / Lanes`.

由于本节重点介绍优先级体系以及它们的转换关系, 关于`Lane(车道模型)`在`fiber树构造`时的具体使用, 在`fiber 树构造`章节详细解读.

### EventPriority

`EventPriority`是 v18 引入的"语义化"优先级桥, 定义于[`ReactEventPriorities.js`](https://github.com/facebook/react/blob/v19.2.6/packages/react-reconciler/src/ReactEventPriorities.js):

```js
export type EventPriority = Lane;

export const DiscreteEventPriority: EventPriority = SyncLane; // click, keyDown, input ...
export const ContinuousEventPriority: EventPriority = InputContinuousLane; // scroll, drag, mouseMove ...
export const DefaultEventPriority: EventPriority = DefaultLane; // setTimeout, Promise, idle render ...
export const IdleEventPriority: EventPriority = IdleLane;
```

EventPriority **本身就是 Lane**(类型别名), 它的存在是为了把"事件来源"的语义直接映射成 Lane: 在合成事件入口处, `react-dom` 会根据原生事件名挑选对应的`EventPriority`, 再传给`setCurrentUpdatePriority` → `requestUpdateLane`. 详见[合成事件原理](./synthetic-event.md).

### SchedulerPriority

`SchedulerPriority`属于`scheduler`包, 定义于[`SchedulerPriorities.js`](https://github.com/facebook/react/blob/v19.2.6/packages/scheduler/src/SchedulerPriorities.js)中:

```js
export const NoPriority = 0;
export const ImmediatePriority = 1;
export const UserBlockingPriority = 2;
export const NormalPriority = 3;
export const LowPriority = 4;
export const IdlePriority = 5;
```

与`scheduler`调度中心相关的优先级使用`SchedulerPriority`.

### 转换关系

v18 起优先级转换被精简为两步, 不再需要中间的`ReactPriorityLevel`枚举:

1. **`Lanes → EventPriority`**: 用`lanesToEventPriority(lanes)`返回"最高优先级"对应的`EventPriority`(本质是 Lane 的某个子集).
2. **`EventPriority → SchedulerPriority`**: 直接 switch case, 见 `ensureRootIsScheduled` 内部.

```js
// ReactEventPriorities.js: 从 Lanes 中选出对应的 EventPriority
export function lanesToEventPriority(lanes: Lanes): EventPriority {
  const lane = getHighestPriorityLane(lanes);
  if (!isHigherEventPriority(DiscreteEventPriority, lane)) {
    return DiscreteEventPriority;
  }
  if (!isHigherEventPriority(ContinuousEventPriority, lane)) {
    return ContinuousEventPriority;
  }
  if (includesNonIdleWork(lane)) {
    return DefaultEventPriority;
  }
  return IdleEventPriority;
}
```

```js
// ReactFiberRootScheduler.js: 在 ensureRootIsScheduled 内将 lanes 转换为 SchedulerPriority
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
// v19: 回调统一为 performWorkOnRoot(root, lanes, forceSync), 异步路径下 forceSync = false
scheduleCallback(schedulerPriorityLevel, performWorkOnRoot.bind(null, root));
```

> 对应在 v17 中由 `lanePriorityToSchedulerPriority` 一个函数承担两次转换, v19 已拆为更直观的"先用`lanesToEventPriority`选出语义档位, 再 switch 到 SchedulerPriority". 同时 `react-reconciler/src/SchedulerWithReactIntegration.js` 整个文件已被删除.

## 优先级使用

通过[reconciler 运作流程](./reconciler-workflow.md)中的归纳, `reconciler`从输入到输出一共经历了 4 个阶段, 在每个阶段中都会涉及到与`优先级`相关的处理. 正是通过`优先级`的灵活运用, `React`实现了`可中断渲染`、`时间切片(time slicing)`、`异步渲染(suspense)`、`Transitions`等特性.

v18+ 中典型的优先级流转链路如下:

1. **事件入口**: 原生事件触发 → `react-dom` 根据事件类型选`EventPriority`(`Discrete/Continuous/Default`) → `setCurrentUpdatePriority(...)`.
2. **触发更新**: 业务调用 `setState / dispatch / startTransition` → `requestUpdateLane()` 读取`currentUpdatePriority`(transition 场景会读取`currentEntangledLane`)生成`update.lane` → `enqueueUpdate`.
3. **注册调度**: `scheduleUpdateOnFiber(root, fiber, lane)` → `markRootUpdated` → `ensureRootIsScheduled` → microtask 中决定走同步还是 `scheduleCallback(schedulerPriority, ...)`.
4. **构造与提交**: `getNextLanes(root)` 选取本次要处理的`renderLanes` → `renderRootSync/Concurrent` → `commitRoot`.

在理解了优先级的基本思路之后, 接下来就正式进入 react 源码分析中的硬核部分(`scheduler 调度原理`和`fiber树构造`).

## 总结

本文介绍了 react 源码中有关优先级的部分, 并梳理了 `Lane / EventPriority / SchedulerPriority` 三层关系. 它们贯穿了[reconciler 运作流程](./reconciler-workflow.md)中的 4 个阶段, 在 react 源码中所占用的代码量比较高, 理解它们的设计思路, 为接下来分析`调度原理`和`fiber构造`打下基础.
