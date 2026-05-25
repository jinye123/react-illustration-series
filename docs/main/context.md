---
title: context 原理
group: 状态管理
order: 4
---

# React Context 原理

简单来讲, `Context`提供了一种直接访问祖先节点上的状态的方法, 避免了多级组件层层传递`props`.

有关`Context`的用法, 请直接查看官方文档, 本文将从`fiber树构造`的视角, 分析`Context`的实现原理.

> v17 → v19 重大变化:
>
> 1. **删除 `calculateChangedBits / observedBits`**(unstable api, v18 已废弃): `context`对象不再有`_calculateChangedBits`字段, `propagateContextChange`不再需要按位比较, 直接用`Object.is(oldValue, newValue)`判断.
> 2. **Provider 简写**(v19): 直接渲染`<MyContext>`即等同于`<MyContext.Provider>`, 不再需要`.Provider`. `<MyContext.Consumer>`仍然兼容.
> 3. **懒传播(lazy propagation)**(v18 引入): 改变 context value 后, 不再立即向下扫描所有 consumer, 而是分两步: (a) 在 ContextProvider 处仅设置后代搜索 lane; (b) 在`beginWork`阶段如果 fiber 命中 lane, 通过`checkIfContextChanged(currentDependencies)`比对每个 dependency 的`memoizedValue`决定是否进入 render.
> 4. **`use(Context)`**(v19): 新增条件可调用的`use(MyContext)`api, 内部走的也是`readContext`路径.
> 5. **`dependencies`字段简化**: 删除`responders`字段(legacy events api 移除), 删除`observedBits`字段.

## 创建 Context

根据官网示例, 通过`React.createContext`这个 api 来创建`context`对象. 在[createContext](https://github.com/facebook/react/blob/v19.2.6/packages/react/src/ReactContext.js)中, 可以看到`context`对象的数据结构:

```js
// v19 简化版
export function createContext<T>(defaultValue: T): ReactContext<T> {
  const context: ReactContext<T> = {
    $$typeof: REACT_CONTEXT_TYPE,
    // 保存 2 个 value 是为了支持多个渲染器并发渲染 (Primary / Secondary)
    _currentValue: defaultValue,
    _currentValue2: defaultValue,
    _threadCount: 0,
    Provider: (null: any),
    Consumer: (null: any),
  };

  if (enableRenderableContext) {
    // v19 默认开启: context 自身可以作为 Provider 使用
    context.Provider = context;
    context.Consumer = {
      $$typeof: REACT_CONSUMER_TYPE,
      _context: context,
    };
  } else {
    // v18 旧路径
    context.Provider = {
      $$typeof: REACT_PROVIDER_TYPE,
      _context: context,
    };
    context.Consumer = context;
  }
  return context;
}
```

`createContext`核心逻辑:

- 其初始值保存在`context._currentValue`(同时保存到`context._currentValue2`. 保存 2 个 value 是为了支持多个渲染器并发渲染).
- 字段缩减: v19 中**移除**了`_calculateChangedBits`字段(已废弃).
- v19 起开启`enableRenderableContext`后, `context.Provider === context`, 也就是说`<MyContext value={...}>{children}</MyContext>`和`<MyContext.Provider value={...}>{children}</MyContext.Provider>`完全等价.

比如, 创建`const MyContext = React.createContext(defaultValue);`, 之后使用以下任一形式都可声明一个`ContextProvider`类型的组件:

```jsx
// v19 推荐: Provider 简写
<MyContext value={someValue}>{children}</MyContext>

// 兼容写法
<MyContext.Provider value={someValue}>{children}</MyContext.Provider>
```

在`fiber树渲染`时, 在`beginWork`中`ContextProvider`类型的节点对应的处理函数是[updateContextProvider](https://github.com/facebook/react/blob/v19.2.6/packages/react-reconciler/src/ReactFiberBeginWork.js):

```js
function beginWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes,
): Fiber | null {
  const updateLanes = workInProgress.lanes;
  workInProgress.lanes = NoLanes;
  // ...省略无关代码
  switch (workInProgress.tag) {
    case ContextProvider:
      return updateContextProvider(current, workInProgress, renderLanes);
    case ContextConsumer:
      return updateContextConsumer(current, workInProgress, renderLanes);
  }
}

function updateContextProvider(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes,
) {
  // ...省略无关代码
  const providerType: ReactProviderType<any> = workInProgress.type;
  const context: ReactContext<any> = providerType._context;

  const newProps = workInProgress.pendingProps;
  const oldProps = workInProgress.memoizedProps;
  // 接收新value
  const newValue = newProps.value;

  // 更新 ContextProvider._currentValue
  pushProvider(workInProgress, newValue);

  if (oldProps !== null) {
    // ... 省略更新context的逻辑, 下文讨论
  }

  const newChildren = newProps.children;
  reconcileChildren(current, workInProgress, newChildren, renderLanes);
  return workInProgress.child;
}
```

`updateContextProvider()`在`fiber初次创建`时十分简单, 仅仅就是保存了`pendingProps.value`做为`context`的最新值, 之后这个最新的值用于供给消费.

### context.\_currentValue 存储

注意`updateContextProvider -> pushProvider`中的[pushProvider(providerFiber, context, nextValue)](https://github.com/facebook/react/blob/v19.2.6/packages/react-reconciler/src/ReactFiberNewContext.js):

```js
// v19 简化版: 入参从 (providerFiber, nextValue) 改为 (providerFiber, context, nextValue),
// 因为 v19 起 Provider 可以直接就是 context 本身, 不再能从 providerFiber.type._context 推断
export function pushProvider<T>(
  providerFiber: Fiber,
  context: ReactContext<T>,
  nextValue: T,
): void {
  if (isPrimaryRenderer) {
    push(valueCursor, context._currentValue, providerFiber);
    context._currentValue = nextValue;
  } else {
    push(valueCursor, context._currentValue2, providerFiber);
    context._currentValue2 = nextValue;
  }
}
```

`pushProvider`实际上是一个存储函数, 利用`栈`的特性, 先把`context._currentValue`压栈, 之后更新`context._currentValue = nextValue`.

与`pushProvider`对应的还有[popProvider](https://github.com/facebook/react/blob/v19.2.6/packages/react-reconciler/src/ReactFiberNewContext.js), 同样利用`栈`的特性, 把`栈`中的值弹出, 还原到`context._currentValue`中.

本节重点分析`Context Api`在`fiber树构造`过程中的作用. 有关`pushProvider/popProvider`的具体实现过程(栈存储), 在[React 算法之栈操作](../algorithm/stack.md#context)中有详细图解.

## 消费 Context

使用了`MyContext.Provider`组件之后, 在`fiber树构造`过程中, context 的值会被`ContextProvider`类型的`fiber`节点所更新. 在后续的过程中, 如何读取`context._currentValue`?

在`react`中, 共提供了 4 种方式可以消费`Context`:

1. 使用`MyContext.Consumer`组件: 用于`JSX`. 如, `<MyContext.Consumer>{(value) => ...}</MyContext.Consumer>`

   - `beginWork`中, 对于`ContextConsumer`类型的节点, 对应的处理函数是[updateContextConsumer](https://github.com/facebook/react/blob/v19.2.6/packages/react-reconciler/src/ReactFiberBeginWork.js)

   ```js
   function updateContextConsumer(
     current: Fiber | null,
     workInProgress: Fiber,
     renderLanes: Lanes,
   ) {
     let context: ReactContext<any> = workInProgress.type;
     // v19 启用 renderableContext 后, workInProgress.type 指向的是 ConsumerObject, 需要拿 _context
     if (enableRenderableContext) {
       context = (workInProgress.type: any)._context;
     }
     const newProps = workInProgress.pendingProps;
     const render = newProps.children;

     // 读取 context (v19 无 observedBits 参数)
     prepareToReadContext(workInProgress, renderLanes);
     const newValue = readContext(context);

     // ...省略无关代码
   }
   ```

2. 使用`useContext(MyContext)`: 用于`function`组件中.

   - 进入`updateFunctionComponent`后, 会调用`prepareToReadContext`.
   - 无论是初次[创建阶段](https://github.com/facebook/react/blob/v19.2.6/packages/react-reconciler/src/ReactFiberHooks.js)还是[更新阶段](https://github.com/facebook/react/blob/v19.2.6/packages/react-reconciler/src/ReactFiberHooks.js), `useContext`都直接调用了`readContext`.

3. **v19 新增: `use(MyContext)`**: 唯一允许在条件/循环中调用的 hook. 与`useContext`的区别在于`use`不在 hook 调用顺序上, 而是在调用现场即时读取, 适合在`if`分支中按需读 context.

   ```js
   function MyComp() {
     if (someCondition) {
       const value = use(MyContext); // ✅ 合法
       return value;
     }
     return null;
   }
   ```

4. `class`组件中, 使用一个静态属性`contextType`: 用于`class`组件中获取`context`. 如, `MyClass.contextType = MyContext;`
   - 进入`updateClassComponent`后, 会调用`prepareToReadContext`.
   - 无论`constructClassInstance / mountClassInstance / updateClassInstance`内部都调用`context = readContext((contextType: any));`(详见[ReactFiberClassComponent.js](https://github.com/facebook/react/blob/v19.2.6/packages/react-reconciler/src/ReactFiberClassComponent.js)).

所以这 4 种方式只是`react`根据不同使用场景封装的`api`, 内部都会调用[prepareToReadContext](https://github.com/facebook/react/blob/v19.2.6/packages/react-reconciler/src/ReactFiberNewContext.js)和[readContext](https://github.com/facebook/react/blob/v19.2.6/packages/react-reconciler/src/ReactFiberNewContext.js).

```js
// v19 简化版
export function prepareToReadContext(
  workInProgress: Fiber,
  renderLanes: Lanes,
): void {
  currentlyRenderingFiber = workInProgress;
  lastContextDependency = null;
  // v18+: 移除了 lastContextWithAllBitsObserved (observedBits 已删除)

  const dependencies = workInProgress.dependencies;
  if (dependencies !== null) {
    if (dependencies.firstContext !== null) {
      if (includesSomeLane(dependencies.lanes, renderLanes)) {
        markWorkInProgressReceivedUpdate();
      }
      // Reset the work-in-progress list
      dependencies.firstContext = null;
    }
  }
}

export function readContext<T>(context: ReactContext<T>): T {
  return readContextForConsumer(currentlyRenderingFiber, context);
}

function readContextForConsumer<T>(
  consumer: Fiber | null,
  context: ReactContext<T>,
): T {
  const value = isPrimaryRenderer
    ? context._currentValue
    : context._currentValue2;
  const contextItem = {
    context: ((context: any): ReactContext<mixed>),
    memoizedValue: value, // v18+ 新增: 记录读取时的 value, 用于懒传播比对
    next: null,
  };

  if (lastContextDependency === null) {
    lastContextDependency = contextItem;
    consumer.dependencies = {
      lanes: NoLanes,
      firstContext: contextItem,
    };
  } else {
    lastContextDependency = lastContextDependency.next = contextItem;
  }
  return value;
}
```

> v17 → v19 字段变更:
>
> - `contextItem.observedBits` 删除.
> - `contextItem.memoizedValue` 新增, 是懒传播的关键: 在 beginWork 时通过`Object.is(memoizedValue, latestValue)`决定该 consumer 是否需要 re-render.
> - `dependencies.responders` 删除.

核心逻辑:

1. `prepareToReadContext`: 设置`currentlyRenderingFiber = workInProgress`, 并重置`lastContextDependency`等全局变量.
2. `readContext`: 返回`context._currentValue`, 并构造一个`contextItem`添加到`workInProgress.dependencies`链表之后. 同时把当前读到的`value`记录到`contextItem.memoizedValue`供更新阶段比对.

注意: 这个`readContext`并不是纯函数, 它还有一些副作用, 会更改`workInProgress.dependencies`. 这个`dependencies`属性会在更新时使用, 用于判定是否依赖了`ContextProvider`中的值, 以及判断该 consumer 是否需要 re-render.

返回`context._currentValue`之后, 继续进行`fiber树构造`直到全部完成即可.

## 更新 Context

来到更新阶段, 再次进入`updateContextProvider`. v18+ 这一段代码做了显著简化(`changedBits`已删除):

```js
// v19 简化版
function updateContextProvider(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes,
) {
  let context: ReactContext<any>;
  let newValue;
  if (enableRenderableContext) {
    context = (workInProgress.type: any);
    newValue = workInProgress.pendingProps.value;
  } else {
    context = workInProgress.type._context;
    newValue = workInProgress.pendingProps.value;
  }
  const oldProps = workInProgress.memoizedProps;

  // 进入栈帧 (v19 入参从 (fiber, value) 改为 (fiber, context, value))
  pushProvider(workInProgress, context, newValue);

  if (oldProps !== null) {
    const oldValue = oldProps.value;
    if (is(oldValue, newValue)) {
      // value 没有变动, 进入 Bailout 逻辑
      if (
        oldProps.children === workInProgress.pendingProps.children &&
        !hasLegacyContextChanged()
      ) {
        return bailoutOnAlreadyFinishedWork(
          current,
          workInProgress,
          renderLanes,
        );
      }
    } else {
      // value 变动, 触发懒传播标记
      propagateContextChange(workInProgress, context, renderLanes);
    }
  }
  const newChildren = workInProgress.pendingProps.children;
  reconcileChildren(current, workInProgress, newChildren, renderLanes);
  return workInProgress.child;
}
```

核心逻辑:

1. `value`没有改变(`Object.is(oldValue, newValue) === true`), 直接进入`Bailout`(可以回顾[fiber 树构造(对比更新)](./fibertree-update.md#bailout)中对`bailout`的解释).
2. `value`改变, 调用`propagateContextChange`(v18+ 是"懒传播"版本).

### 懒传播 (Lazy Propagation)

v18 起, `propagateContextChange`的策略发生了重大变化([#20890](https://github.com/facebook/react/pull/20890)):

- **v17 (Eager Propagation)**: 在`updateContextProvider`时立即向下深度遍历整棵子树, 找到所有依赖该 context 的 consumer fiber, 给它们的`fiber.lanes`标上`renderLanes`, 并沿`fiber.return`路径上把`childLanes`补齐. 这种做法在 consumer 数量多时会造成 O(N) 的同步开销.
- **v18+ (Lazy Propagation)**: `propagateContextChange`不再立即下钻到 consumer, 而是:
  1. 将子树所有"已 bailout 过的"分支重新打上`renderLanes`(只标 lane, 不修改 dependencies).
  2. 然后由后续的`beginWork`正常往下走, 在每个 fiber 的 dependencies 上做`memoizedValue`和`latestValue`的逐项`Object.is`比对. 命中变化的 consumer 才会`markWorkInProgressReceivedUpdate`触发 re-render. 这样跳过整棵无关子树时仍然是 O(1).

[propagateContextChange_eager / propagateParentContextChanges](https://github.com/facebook/react/blob/v19.2.6/packages/react-reconciler/src/ReactFiberNewContext.js):

```js
// v19 简化版 propagateContextChange
function propagateContextChange<T>(
  workInProgress: Fiber,
  context: ReactContext<T>,
  renderLanes: Lanes,
): void {
  // 注意: 这是懒传播版本, 不再向 consumer 节点写 lanes
  // 只把所有已经 bailout 过的子树重新打上 renderLanes, 让它们进入 beginWork 检查 dependencies
  propagateContextChange_eager(workInProgress, context, renderLanes);
}

// beginWork 时, 检查 fiber 是否需要因 context 变化而 re-render
export function checkIfContextChanged(
  currentDependencies: Dependencies,
): boolean {
  let dependency = currentDependencies.firstContext;
  while (dependency !== null) {
    const context = dependency.context;
    const newValue = isPrimaryRenderer
      ? context._currentValue
      : context._currentValue2;
    const oldValue = dependency.memoizedValue;
    if (!is(newValue, oldValue)) {
      return true; // 命中变化, 需要 re-render
    }
    dependency = dependency.next;
  }
  return false;
}
```

`scheduleContextWorkOnParentPath`(v18+ 重命名)与[markUpdateLaneFromFiberToRoot](https://github.com/facebook/react/blob/v19.2.6/packages/react-reconciler/src/ReactFiberConcurrentUpdates.js)的作用相似, 具体可以回顾[fiber 树构造(对比更新)](./fibertree-update.md#markUpdateLaneFromFiberToRoot).

通过以上设计, 保证了所有消费该`context`的子节点都会被重新检查, 进而保证了状态的一致性, 实现了`context`更新, 同时把"扫描代价"延后到了真正的`beginWork`阶段, 大大降低了 provider 切换时的同步开销.

## 总结

`Context`的实现思路还是比较清晰, v19 中总体分为 3 步.

1. **消费**: `ContextConsumer / useContext / use / contextType`内部都调用`readContext(MyContext)`, 把当前 value 写入`fiber.dependencies`链表的`memoizedValue`字段.
2. **变更标记**: `Provider`收到新 value 时调用`propagateContextChange`(懒传播), 仅把已 bailout 的分支重新打上`renderLanes`, **不**直接修改 consumer 的`fiber.lanes`.
3. **变更校验**: 后续`beginWork`沿正常路径执行, 遇到打过 lanes 的分支时通过`checkIfContextChanged(currentDependencies)`逐项比对`memoizedValue`和最新`_currentValue`, 命中变化的 consumer 才 re-render.

v17 → v19 重要差异汇总:

1. `_calculateChangedBits / observedBits / changedBits` 全部删除.
2. `pushProvider`签名从`(fiber, value)`变更为`(fiber, context, value)`.
3. v19 起`<MyContext value>`简写等价于`<MyContext.Provider value>`.
4. 引入 `dependency.memoizedValue` 字段, 配合"懒传播 + beginWork 阶段校验"模式.
5. 新增 `use(Context)`api, 支持条件读取.
