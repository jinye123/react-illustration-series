---
title: 合成事件
group:
  title: 交互
  order: 3
order: 0
---

# React 合成事件

## 概览

从`v17.0.0`开始, React 不会再将事件处理添加到 `document` 上, 而是将事件处理添加到渲染 React 树的根 DOM 容器中.

引入官方提供的图片:

![](https://zh-hans.reactjs.org/static/bb4b10114882a50090b8ff61b3c4d0fd/1e088/react_17_delegation.png)

图中清晰的展示了`v17.0.0`的改动, 无论是在`document`还是`根 DOM 容器`上监听事件, 都可以归为`事件委托(代理)`([mdn](https://developer.mozilla.org/zh-CN/docs/Learn/JavaScript/Building_blocks/Events)).

> v17 → v19 重大变化:
>
> 1. **事件源代码迁移**: 整个 events 子目录从`react-dom/src/events/`迁移到`react-dom-bindings/src/events/`(v18 拆包后).
> 2. **优先级模型重构**: v17 的三档`DiscreteEvent / UserBlockingEvent / ContinuousEvent`(整型枚举)被`EventPriority`(`DiscreteEventPriority / ContinuousEventPriority / DefaultEventPriority / IdleEventPriority`)替代, 这套枚举本身就是 Lane(直接复用车道值), 详见[priority.md](./priority.md).
> 3. **listenerWrapper 命名变更**: `dispatchUserBlockingUpdate → dispatchContinuousEvent`, 同时 v18+ 在 listener 包装中直接`setCurrentUpdatePriority(EventPriority)`, 而不是过 v17 的`runWithPriority(LanePriority, ...)`.
> 4. **事件绑定开关删除**: v18 起删除`enableEagerRootListeners`feature flag, 启动时统一一次性注册全部代理事件.
> 5. **Hydration 处理迁移**: hydration root 的事件回放走单独的`ReactDOMEventReplaying.js`, v19 中`attemptHydrationAtCurrentPriority`等钩子接入了`ReactFiberRootScheduler`.

注意: `react`的事件体系, 不是全部都通过`事件委托`来实现的. 有一些[特殊情况](https://github.com/facebook/react/blob/v19.2.6/packages/react-dom-bindings/src/client/ReactDOMComponent.js), 是直接绑定到对应 DOM 元素上的(如:`scroll`, `load`), 它们都通过[listenToNonDelegatedEvent](https://github.com/facebook/react/blob/v19.2.6/packages/react-dom-bindings/src/events/DOMPluginEventSystem.js)函数进行绑定.

上述特殊事件最大的不同是监听的 DOM 元素不同, 除此之外, 其他地方的实现与正常事件大体一致.

本节讨论的是可以被`根 DOM 容器`代理的正常事件.

## 事件绑定

在前文[React 应用的启动过程](./bootstrap.md#create-global-obj)中介绍了`React`在启动时会创建全局对象, 其中在创建[fiberRoot](./bootstrap.md#create-root-impl)对象时, 调用[createRootImpl](https://github.com/facebook/react/blob/v19.2.6/packages/react-dom/src/client/ReactDOMRoot.js)(代码已迁移到`ReactDOMRoot.js`和`ReactFiberReconciler.js`):

```js
// v19 简化版
function createContainerImpl(
  container: Container,
  tag: RootTag,
  // ...
) {
  // ... 省略无关代码
  // v18 起删除了 enableEagerRootListeners 开关, 启动时无条件注册全部代理事件
  const rootContainerElement =
    container.nodeType === COMMENT_NODE ? container.parentNode : container;
  listenToAllSupportedEvents(rootContainerElement);
  // ... 省略无关代码
}
```

[listenToAllSupportedEvents](https://github.com/facebook/react/blob/v19.2.6/packages/react-dom-bindings/src/events/DOMPluginEventSystem.js)函数, 实际上完成了事件代理:

```js
// v19 简化版 (路径: react-dom-bindings/src/events/DOMPluginEventSystem.js)
export function listenToAllSupportedEvents(rootContainerElement: EventTarget) {
  // 1. 节流优化, 保证全局注册只被调用一次
  if ((rootContainerElement: any)[listeningMarker]) return;
  (rootContainerElement: any)[listeningMarker] = true;

  // 2. 遍历 allNativeEvents 监听冒泡和捕获阶段的事件
  allNativeEvents.forEach((domEventName) => {
    // selectionchange 是 document 级别的事件, 单独处理
    if (domEventName !== 'selectionchange') {
      if (!nonDelegatedEvents.has(domEventName)) {
        listenToNativeEvent(
          domEventName,
          false, // 冒泡阶段监听
          ((rootContainerElement: any): Element),
        );
      }
      listenToNativeEvent(
        domEventName,
        true, // 捕获阶段监听
        ((rootContainerElement: any): Element),
      );
    }
  });
  // v19 新增: selectionchange 绑定到 ownerDocument
  const ownerDocument =
    (rootContainerElement: any).nodeType === DOCUMENT_NODE
      ? rootContainerElement
      : (rootContainerElement: any).ownerDocument;
  if (ownerDocument !== null) {
    if (!(ownerDocument: any)[listeningMarker]) {
      (ownerDocument: any)[listeningMarker] = true;
      listenToNativeEvent('selectionchange', false, ownerDocument);
    }
  }
}
```

核心逻辑:

1. 节流优化, 保证全局注册只被调用一次.
2. 遍历`allNativeEvents`, 调用`listenToNativeEvent`监听冒泡和捕获阶段的事件.
   - `allNativeEvents`包括了大量的原生事件名称, 它在`DOMPluginEventSystem.js`中由各 Plugin (`SimpleEventPlugin / EnterLeaveEventPlugin / ChangeEventPlugin / SelectEventPlugin / BeforeInputEventPlugin / FormActionEventPlugin (v19 新增)`) 通过`registerEvents()`初始化.
3. v19 新增`selectionchange`监听到 ownerDocument, 用于 contentEditable 场景.

[listenToNativeEvent](https://github.com/facebook/react/blob/v19.2.6/packages/react-dom-bindings/src/events/DOMPluginEventSystem.js):

```js
// v19 简化版
export function listenToNativeEvent(
  domEventName: DOMEventName,
  isCapturePhaseListener: boolean,
  target: EventTarget,
): void {
  let eventSystemFlags = 0;
  if (isCapturePhaseListener) {
    eventSystemFlags |= IS_CAPTURE_PHASE;
  }
  addTrappedEventListener(
    target,
    domEventName,
    eventSystemFlags,
    isCapturePhaseListener,
  );
}
```

[addTrappedEventListener](https://github.com/facebook/react/blob/v19.2.6/packages/react-dom-bindings/src/events/DOMPluginEventSystem.js):

```js
// ... 省略无关代码
function addTrappedEventListener(
  targetContainer: EventTarget,
  domEventName: DOMEventName,
  eventSystemFlags: EventSystemFlags,
  isCapturePhaseListener: boolean,
  isDeferredListenerForLegacyFBSupport?: boolean,
) {
  // 1. 构造listener
  let listener = createEventListenerWrapperWithPriority(
    targetContainer,
    domEventName,
    eventSystemFlags,
  );
  let unsubscribeListener;
  // 2. 注册事件监听
  if (isCapturePhaseListener) {
    unsubscribeListener = addEventCaptureListener(
      targetContainer,
      domEventName,
      listener,
    );
  } else {
    unsubscribeListener = addEventBubbleListener(
      targetContainer,
      domEventName,
      listener,
    );
  }
}

// 注册原生事件 冒泡
export function addEventBubbleListener(
  target: EventTarget,
  eventType: string,
  listener: Function,
): Function {
  target.addEventListener(eventType, listener, false);
  return listener;
}

// 注册原生事件 捕获
export function addEventCaptureListener(
  target: EventTarget,
  eventType: string,
  listener: Function,
): Function {
  target.addEventListener(eventType, listener, true);
  return listener;
}
```

从`listenToAllSupportedEvents`开始, 调用链路比较长, 最后调用`addEventBubbleListener`和`addEventCaptureListener`监听了原生事件.

### 原生 listener

在注册原生事件的过程中, 需要重点关注一下监听函数, 即`listener`函数. 它实现了把原生事件派发到`react`体系之内, 非常关键.

> 比如点击 DOM 触发原生事件, 原生事件最后会被派发到`react`内部的`onClick`函数. `listener`函数就是这个`由外至内`的关键环节.

`listener`是通过`createEventListenerWrapperWithPriority`函数产生 (v19 版本):

```js
// v19: 路径 react-dom-bindings/src/events/ReactDOMEventListener.js
export function createEventListenerWrapperWithPriority(
  targetContainer: EventTarget,
  domEventName: DOMEventName,
  eventSystemFlags: EventSystemFlags,
): Function {
  // 1. 根据 EventPriority 设置 listenerWrapper
  const eventPriority = getEventPriority(domEventName);
  let listenerWrapper;
  switch (eventPriority) {
    case DiscreteEventPriority:
      listenerWrapper = dispatchDiscreteEvent;
      break;
    case ContinuousEventPriority:
      listenerWrapper = dispatchContinuousEvent; // v18 重命名: 原 dispatchUserBlockingUpdate
      break;
    case DefaultEventPriority:
    default:
      listenerWrapper = dispatchEvent;
      break;
  }
  // 2. 返回绑定参数后的 listenerWrapper
  return listenerWrapper.bind(
    null,
    domEventName,
    eventSystemFlags,
    targetContainer,
  );
}

function dispatchDiscreteEvent(
  domEventName,
  eventSystemFlags,
  container,
  nativeEvent,
) {
  const previousPriority = getCurrentUpdatePriority();
  const prevTransition = ReactSharedInternals.T;
  ReactSharedInternals.T = null;
  try {
    // v18+ 直接 set, 不再 runWithPriority
    setCurrentUpdatePriority(DiscreteEventPriority);
    dispatchEvent(domEventName, eventSystemFlags, container, nativeEvent);
  } finally {
    setCurrentUpdatePriority(previousPriority);
    ReactSharedInternals.T = prevTransition;
  }
}

function dispatchContinuousEvent(
  domEventName,
  eventSystemFlags,
  container,
  nativeEvent,
) {
  const previousPriority = getCurrentUpdatePriority();
  const prevTransition = ReactSharedInternals.T;
  ReactSharedInternals.T = null;
  try {
    setCurrentUpdatePriority(ContinuousEventPriority);
    dispatchEvent(domEventName, eventSystemFlags, container, nativeEvent);
  } finally {
    setCurrentUpdatePriority(previousPriority);
    ReactSharedInternals.T = prevTransition;
  }
}
```

可以看到, 不同的`domEventName`调用[getEventPriority](https://github.com/facebook/react/blob/v19.2.6/packages/react-dom-bindings/src/events/ReactDOMEventListener.js)后返回不同的`EventPriority`(本质上就是 Lane), 最终会有 3 种情况:

| EventPriority             | 对应 Lane             | 典型事件                                                                             | listenerWrapper                                                          |
| ------------------------- | --------------------- | ------------------------------------------------------------------------------------ | ------------------------------------------------------------------------ |
| `DiscreteEventPriority`   | `SyncLane`            | `click / keydown / pointerdown / input / submit / focusin / focusout` 等离散用户输入 | `dispatchDiscreteEvent`                                                  |
| `ContinuousEventPriority` | `InputContinuousLane` | `drag / mousemove / scroll / wheel / touchmove / pointermove` 等连续用户输入         | `dispatchContinuousEvent` (v18 重命名, 旧名`dispatchUserBlockingUpdate`) |
| `DefaultEventPriority`    | `DefaultLane`         | `animationend / loadeddata / message` 等系统事件                                     | `dispatchEvent`                                                          |

> v17 → v19 关键变化:
>
> - 事件优先级常量从`DiscreteEvent / UserBlockingEvent / ContinuousEvent`(纯枚举值)迁移到`DiscreteEventPriority / ContinuousEventPriority / DefaultEventPriority`(直接复用 Lane 数值).
> - `dispatchUserBlockingUpdate` → `dispatchContinuousEvent`.
> - 不再使用`runWithPriority(LanePriority, ...)`, 改为`setCurrentUpdatePriority(EventPriority)`+ `try/finally`恢复. 这与`startTransition`/Concurrent 渲染统一在`ReactSharedInternals.T`/`currentUpdatePriority`两个全局变量上.
> - 事件优先级 → Lane 映射统一通过`eventPriorityToLane = eventPriority`(它本身就是 Lane), 不再走"三层换算".

这 3 种`listener`实际上都是对[dispatchEvent](https://github.com/facebook/react/blob/v19.2.6/packages/react-dom-bindings/src/events/ReactDOMEventListener.js)的包装:

```js
// v19 简化版
export function dispatchEvent(
  domEventName: DOMEventName,
  eventSystemFlags: EventSystemFlags,
  targetContainer: EventTarget,
  nativeEvent: AnyNativeEvent,
): void {
  if (!_enabled) return;
  const blockedOn = findInstanceBlockingEvent(nativeEvent);
  if (blockedOn === null) {
    dispatchEventForPluginEventSystem(
      domEventName,
      eventSystemFlags,
      nativeEvent,
      return_targetInst,
      targetContainer,
    );
    return;
  }
  // ... 处理 hydration 期间的事件回放, 见 ReactDOMEventReplaying.js
}
```

## 事件触发

当原生事件触发之后, 首先会进入到`dispatchEvent`这个回调函数. 而`dispatchEvent`函数是`react`事件体系中最关键的函数, 其调用链路较长, 核心步骤如图所示:

![](../../snapshots/synthetic-event/dispatch-event.png)

重点关注其中 3 个核心环节:

1. `attemptToDispatchEvent`
2. `SimpleEventPlugin.extractEvents`
3. `processDispatchQueue`

### 关联 fiber

v18 起函数从`attemptToDispatchEvent`重命名为[findInstanceBlockingEvent](https://github.com/facebook/react/blob/v19.2.6/packages/react-dom-bindings/src/events/ReactDOMEventListener.js), 它的职责是判断当前事件是否被某个 hydration 中的 Suspense / Container 阻塞, 如果没有阻塞才往下派发:

```js
// v19 简化版
export function findInstanceBlockingEvent(
  nativeEvent: AnyNativeEvent,
): null | Container | SuspenseInstance {
  const nativeEventTarget = getEventTarget(nativeEvent);
  return findInstanceBlockingTarget(nativeEventTarget);
}

export function findInstanceBlockingTarget(
  targetNode: Node,
): null | Container | SuspenseInstance {
  return_targetInst = null;
  let targetInst = getClosestInstanceFromNode(targetNode);
  if (targetInst !== null) {
    const nearestMounted = getNearestMountedFiber(targetInst);
    if (nearestMounted === null) {
      targetInst = null;
    } else if (nearestMounted.tag === SuspenseComponent) {
      // ... 处理 SuspenseComponent 还没 hydrate 完毕的情况
    } else if (nearestMounted.tag === HostRoot) {
      // ... 处理 root 还没 hydrate 完毕的情况
    } else if (nearestMounted !== targetInst) {
      targetInst = null;
    }
  }
  return_targetInst = targetInst;
  return null;
}
```

核心逻辑:

1. 定位原生 DOM 节点: 调用`getEventTarget`.
2. 获取与 DOM 节点对应的 fiber 节点: 调用`getClosestInstanceFromNode`.
3. 检查是否被 Suspense / Root hydration 阻塞; 若没有阻塞, 把目标 fiber 写到模块级`return_targetInst`变量, 由`dispatchEvent`继续往下走.
4. 通过插件系统派发事件: 调用`dispatchEventForPluginEventSystem`.

### 收集 fiber 上的 listener

`dispatchEvent`函数的调用链路中, 通过不同的插件, 处理不同的事件. 其中最常见的事件都会由`SimpleEventPlugin.extractEvents`进行处理:

```js
function extractEvents(
  dispatchQueue: DispatchQueue,
  domEventName: DOMEventName,
  targetInst: null | Fiber,
  nativeEvent: AnyNativeEvent,
  nativeEventTarget: null | EventTarget,
  eventSystemFlags: EventSystemFlags,
  targetContainer: EventTarget,
): void {
  const reactName = topLevelEventsToReactNames.get(domEventName);
  if (reactName === undefined) {
    return;
  }
  let SyntheticEventCtor = SyntheticEvent;
  let reactEventType: string = domEventName;

  const inCapturePhase = (eventSystemFlags & IS_CAPTURE_PHASE) !== 0;
  const accumulateTargetOnly = !inCapturePhase && domEventName === 'scroll';
  // 1. 收集所有监听该事件的函数.
  const listeners = accumulateSinglePhaseListeners(
    targetInst,
    reactName,
    nativeEvent.type,
    inCapturePhase,
    accumulateTargetOnly,
  );
  if (listeners.length > 0) {
    // 2. 构造合成事件, 添加到派发队列
    const event = new SyntheticEventCtor(
      reactName,
      reactEventType,
      null,
      nativeEvent,
      nativeEventTarget,
    );
    dispatchQueue.push({ event, listeners });
  }
}
```

核心逻辑:

1. 收集所有`listener`回调

   - 这里的是`fiber.memoizedProps.onClick/onClickCapture`等绑定在`fiber`节点上的回调函数
   - 具体逻辑在[accumulateSinglePhaseListeners](https://github.com/facebook/react/blob/v19.2.6/packages/react-dom-bindings/src/events/DOMPluginEventSystem.js):

     ```js
     export function accumulateSinglePhaseListeners(
       targetFiber: Fiber | null,
       reactName: string | null,
       nativeEventType: string,
       inCapturePhase: boolean,
       accumulateTargetOnly: boolean,
     ): Array<DispatchListener> {
       const captureName = reactName !== null ? reactName + 'Capture' : null;
       const reactEventName = inCapturePhase ? captureName : reactName;
       const listeners: Array<DispatchListener> = [];

       let instance = targetFiber;
       let lastHostComponent = null;

       // 从targetFiber开始, 向上遍历, 直到 root 为止
       while (instance !== null) {
         const { stateNode, tag } = instance;
         // 当节点类型是HostComponent时(如: div, span, button等类型)
         if (tag === HostComponent && stateNode !== null) {
           lastHostComponent = stateNode;
           if (reactEventName !== null) {
             // 获取标准的监听函数 (如onClick , onClickCapture等)
             const listener = getListener(instance, reactEventName);
             if (listener != null) {
               listeners.push(
                 createDispatchListener(instance, listener, lastHostComponent),
               );
             }
           }
         }
         // 如果只收集目标节点, 则不用向上遍历, 直接退出
         if (accumulateTargetOnly) {
           break;
         }
         instance = instance.return;
       }
       return listeners;
     }
     ```

2. 构造合成事件(`SyntheticEvent`), 添加到派发队列(`dispatchQueue`)

### 构造合成事件

[SyntheticEvent](https://github.com/facebook/react/blob/v19.2.6/packages/react-dom-bindings/src/events/SyntheticEvent.js), 是`react`内部创建的一个对象, 是原生事件的跨浏览器包装器, 拥有和浏览器原生事件相同的接口(`stopPropagation`,`preventDefault`), 抹平不同浏览器 api 的差异, 兼容性好.

> v17 → v19: SyntheticEvent **不再做事件池(event pooling)回收**(v17 已经移除), 用完即丢, 也不会在 callback 结束后被复用. 同时移除了`event.persist()`方法(v18 起空操作).

具体的构造过程并不复杂, 可以直接[查看源码](https://github.com/facebook/react/blob/v19.2.6/packages/react-dom-bindings/src/events/SyntheticEvent.js).

此处我们需要知道, 在`Plugin.extractEvents`过程中, 遍历`fiber树`找到`listener`之后, 就会创建`SyntheticEvent`, 加入到`dispatchQueue`中, 等待派发.

### 执行派发

`extractEvents`完成之后, 逻辑来到[processDispatchQueue](https://github.com/facebook/react/blob/v19.2.6/packages/react-dom-bindings/src/events/DOMPluginEventSystem.js), 终于要真正执行派发了.

```js
export function processDispatchQueue(
  dispatchQueue: DispatchQueue,
  eventSystemFlags: EventSystemFlags,
): void {
  const inCapturePhase = (eventSystemFlags & IS_CAPTURE_PHASE) !== 0;
  for (let i = 0; i < dispatchQueue.length; i++) {
    const { event, listeners } = dispatchQueue[i];
    processDispatchQueueItemsInOrder(event, listeners, inCapturePhase);
  }
  // ...省略无关代码
}

function processDispatchQueueItemsInOrder(
  event: ReactSyntheticEvent,
  dispatchListeners: Array<DispatchListener>,
  inCapturePhase: boolean,
): void {
  let previousInstance;
  if (inCapturePhase) {
    // 1. capture事件: 倒序遍历listeners
    for (let i = dispatchListeners.length - 1; i >= 0; i--) {
      const { instance, currentTarget, listener } = dispatchListeners[i];
      if (instance !== previousInstance && event.isPropagationStopped()) {
        return;
      }
      executeDispatch(event, listener, currentTarget);
      previousInstance = instance;
    }
  } else {
    // 2. bubble事件: 顺序遍历listeners
    for (let i = 0; i < dispatchListeners.length; i++) {
      const { instance, currentTarget, listener } = dispatchListeners[i];
      if (instance !== previousInstance && event.isPropagationStopped()) {
        return;
      }
      executeDispatch(event, listener, currentTarget);
      previousInstance = instance;
    }
  }
}
```

在[processDispatchQueueItemsInOrder](https://github.com/facebook/react/blob/v19.2.6/packages/react-dom-bindings/src/events/DOMPluginEventSystem.js)遍历`dispatchListeners`数组, 执行[executeDispatch](https://github.com/facebook/react/blob/v19.2.6/packages/react-dom-bindings/src/events/DOMPluginEventSystem.js)派发事件, 在`fiber`节点上绑定的`listener`函数被执行.

在`processDispatchQueueItemsInOrder`函数中, 根据`捕获(capture)`或`冒泡(bubble)`的不同, 采取了不同的遍历方式:

1. `capture`事件: `从上至下`调用`fiber树`中绑定的回调函数, 所以`倒序`遍历`dispatchListeners`.
2. `bubble`事件: `从下至上`调用`fiber树`中绑定的回调函数, 所以`顺序`遍历`dispatchListeners`.

## 事件优先级 → Lane 映射 (v19)

v18+ 中事件优先级与 Lane 的关系十分直接, 三者基本"一一对应":

| 用户输入 / 系统事件                                                                   | `getEventPriority()` 返回 | 对应 Lane                |
| ------------------------------------------------------------------------------------- | ------------------------- | ------------------------ |
| `click / keydown / submit / focusin / pointerdown / input(textarea/input)` 等离散事件 | `DiscreteEventPriority`   | `SyncLane`(2)            |
| `drag / mousemove / scroll / wheel / touchmove / pointermove` 等连续事件              | `ContinuousEventPriority` | `InputContinuousLane`(8) |
| `animationend / load / message / canplay / playing / waiting` 等系统事件              | `DefaultEventPriority`    | `DefaultLane`(32)        |
| `idle` 类任务                                                                         | `IdleEventPriority`       | `IdleLane`               |

事件 listener 在`dispatchDiscreteEvent / dispatchContinuousEvent / dispatchEvent`中调用`setCurrentUpdatePriority(EventPriority)`, 此后该事件处理器中触发的任意`setState`会通过`requestUpdateLane(fiber)`读取`getCurrentUpdatePriority()`, 直接拿到对应 Lane(参见[priority.md 优先级使用](./priority.md#优先级使用)).

## 总结

从架构上来讲, [SyntheticEvent](https://github.com/facebook/react/blob/v19.2.6/packages/react-dom-bindings/src/events/SyntheticEvent.js)打通了从外部`原生事件`到内部`fiber树`的交互渠道, 使得`react`能够感知到浏览器提供的`原生事件`, 进而做出不同的响应, 修改`fiber树`, 变更视图等.

从实现上讲, 主要分为 3 步:

1. 监听原生事件: 对齐`DOM元素`和`fiber元素`.
2. 收集`listeners`: 遍历`fiber树`, 收集所有监听本事件的`listener`函数.
3. 派发合成事件: 构造合成事件, 遍历`listeners`进行派发.

v17 → v19 关键变化:

1. 事件源代码迁移到`react-dom-bindings`包.
2. 优先级常量由`DiscreteEvent / UserBlockingEvent / ContinuousEvent`改为`DiscreteEventPriority / ContinuousEventPriority / DefaultEventPriority / IdleEventPriority`(本质即 Lane).
3. `dispatchUserBlockingUpdate` → `dispatchContinuousEvent`.
4. 不再走`runWithPriority(LanePriority, ...)`, 改为`setCurrentUpdatePriority(EventPriority) + try/finally`.
5. SyntheticEvent 移除事件池, `event.persist()`变为空操作.
6. hydration 事件阻塞改名为`findInstanceBlockingEvent / findInstanceBlockingTarget`.
7. v19 引入`FormActionEventPlugin`, 处理`<form action={...}>`的提交事件, 把 action 派发为 transition.
