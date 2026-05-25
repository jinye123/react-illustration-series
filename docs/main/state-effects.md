---
title: 状态与副作用
group:
  title: 状态管理
  order: 2
order: 0
---

# 状态与副作用

在前文我们已经分析了`fiber树`从`构造`到`渲染`的关键过程. 本节我们站在`fiber`对象的视角, 考虑一个具体的`fiber`节点如何影响最终的渲染.

回顾[fiber 数据结构](https://github.com/facebook/react/blob/v19.2.6/packages/react-reconciler/src/ReactInternalTypes.js), 并结合前文`fiber树构造`系列的解读, 我们注意到`fiber`众多属性中, 有 2 类属性十分关键:

1. `fiber`节点的自身状态: 在`renderRootSync[Concurrent]`阶段, 为子节点提供确定的输入数据, 直接影响子节点的生成.

2. `fiber`节点的副作用: 在`commitRoot`阶段, 如果`fiber`被标记有副作用, 则副作用相关函数会被(同步/异步)调用.

```js
export type Fiber = {
  // 1. fiber节点自身状态相关
  pendingProps: any,
  memoizedProps: any,
  updateQueue: mixed,
  memoizedState: any,

  // 2. fiber节点副作用(Effect)相关 (v18 起 effectList 链表已删除)
  flags: Flags, // 本节点的副作用标志位
  subtreeFlags: Flags, // 子树聚合标志位 (取代 firstEffect / nextEffect / lastEffect)
  deletions: Array<Fiber> | null, // 本节点的待删除子节点列表
};
```

## 状态

与`状态`相关有 4 个属性:

1. `fiber.pendingProps`: 输入属性, 从`ReactElement`对象传入的 props. 它和`fiber.memoizedProps`比较可以得出属性是否变动.
2. `fiber.memoizedProps`: 上一次生成子节点时用到的属性, 生成子节点之后保持在内存中. 向下生成子节点之前叫做`pendingProps`, 生成子节点之后会把`pendingProps`赋值给`memoizedProps`用于下一次比较.`pendingProps`和`memoizedProps`比较可以得出属性是否变动.
3. `fiber.updateQueue`: 存储`update更新对象`的队列, 每一次发起更新, 都需要在该队列上创建一个`update对象`.
4. `fiber.memoizedState`: 上一次生成子节点之后保持在内存中的局部状态.

它们的作用只局限于`fiber树构造`阶段, 直接影响子节点的生成.

## 副作用

与`副作用`相关有 3 个属性(v17 时期还包含`nextEffect / firstEffect / lastEffect`三个链表指针, v18 起已删除):

1. `fiber.flags`: 标志位, 表明该`fiber`节点有副作用. 在 v19.2.6 中共定义了[30+ 种副作用](https://github.com/facebook/react/blob/v19.2.6/packages/react-reconciler/src/ReactFiberFlags.js)(常见: `Placement`、`Update`、`ChildDeletion`、`Snapshot`、`Passive`、`Hydrating`、`Ref`、`Visibility`、`StoreConsistency`、`DidCapture`、`Forked`、`ScheduleRetry`等), 以及一组"静态标志"`StaticMask / PassiveStatic / LayoutStatic / RefStatic`用于在 bailout 时仍保留必要的子树检查.
2. `fiber.subtreeFlags`: 子树聚合标志位. 在 complete 阶段沿父链冒泡(`returnFiber.subtreeFlags |= wip.flags | wip.subtreeFlags`), commit 阶段据此从根 DFS 剪枝.
3. `fiber.deletions`: 在 reconcile 阶段被父节点删除的子 fiber 数组, 父节点同时打上`ChildDeletion`位.

通过前文`fiber树构造`我们知道, v19 中所有副作用信息都聚合在`fiber.flags / subtreeFlags / deletions`三个字段, 由根节点开始 DFS 处理. 在`commitRoot`阶段中, `react`提供了 3 种处理副作用的方式(详见[fiber 树渲染](./fibertree-commit.md#渲染)).

另外, `副作用`的设计可以理解为对`状态`功能不足的补充.

- `状态`是一个`静态`的功能, 它只能为子节点提供数据源.
- 而`副作用`是一个`动态`功能, 由于它的调用时机是在`fiber树渲染阶段`, 故它拥有更多的能力, 能轻松获取`突变前快照, 突变后的DOM节点等`. 甚至通过`调用api`发起新的一轮`fiber树构造`, 进而改变更多的`状态`, 引发更多的`副作用`.

## 外部 api

`fiber`对象的这 2 类属性, 可以影响到渲染结果, 但是`fiber`结构始终是一个内核中的结构, 对于外部来讲是无感知的, 对于调用方来讲, 甚至都无需知道`fiber`结构的存在. 所以正常只有通过暴露`api`来直接或间接的修改这 2 类属性.

从`react`包暴露出的`api`来归纳, 只有 2 类组件支持修改:

> 本节只讨论使用`api`的目的是修改`fiber`的`状态`和`副作用`, 进而可以改变整个渲染结果. 本节先介绍 api 与`状态`和`副作用`的联系, 有关`api`的具体实现会在`class组件`,`Hook原理`章节中详细分析.

### class 组件

```js
class App extends React.Component {
  constructor() {
    this.state = {
      // 初始状态
      a: 1,
    };
  }
  changeState = () => {
    this.setState({ a: ++this.state.a }); // 进入reconciler流程
  };

  // 生命周期函数: 状态相关
  static getDerivedStateFromProps(nextProps, prevState) {
    console.log('getDerivedStateFromProps');
    return prevState;
  }

  // 生命周期函数: 状态相关
  shouldComponentUpdate(newProps, newState, nextContext) {
    console.log('shouldComponentUpdate');
    return true;
  }

  // 生命周期函数: 副作用相关 fiber.flags |= Update
  componentDidMount() {
    console.log('componentDidMount');
  }

  // 生命周期函数: 副作用相关 fiber.flags |= Snapshot
  getSnapshotBeforeUpdate(prevProps, prevState) {
    console.log('getSnapshotBeforeUpdate');
  }

  // 生命周期函数: 副作用相关 fiber.flags |= Update
  componentDidUpdate() {
    console.log('componentDidUpdate');
  }

  render() {
    // 返回下级ReactElement对象
    return <button onClick={this.changeState}>{this.state.a}</button>;
  }
}
```

1. 状态相关: `fiber树构造`阶段.

   1. 构造函数: `constructor`实例化时执行, 可以设置初始 state, 只执行一次.
   2. 生命周期: `getDerivedStateFromProps`在`fiber树构造`阶段(`renderRootSync[Concurrent]`)执行, 可以修改 state([链接](https://github.com/facebook/react/blob/v19.2.6/packages/react-reconciler/src/ReactFiberClassComponent.js)).
   3. 生命周期: `shouldComponentUpdate`在`fiber树构造`阶段(`renderRootSync[Concurrent]`)执行, 返回值决定是否执行 render([链接](https://github.com/facebook/react/blob/v19.2.6/packages/react-reconciler/src/ReactFiberClassComponent.js)).

2. 副作用相关: `fiber树渲染`阶段.
   1. 生命周期: `getSnapshotBeforeUpdate`在`fiber树渲染`阶段(`commitRoot->commitBeforeMutationEffects->commitBeforeMutationEffectsOnFiber`)执行([链接](https://github.com/facebook/react/blob/v19.2.6/packages/react-reconciler/src/ReactFiberCommitWork.js)).
   2. 生命周期: `componentDidMount`在`fiber树渲染`阶段(`commitRoot->commitLayoutEffects->commitLayoutEffectOnFiber`)执行([链接](https://github.com/facebook/react/blob/v19.2.6/packages/react-reconciler/src/ReactFiberCommitWork.js)).
   3. 生命周期: `componentDidUpdate`在`fiber树渲染`阶段(`commitRoot->commitLayoutEffects->commitLayoutEffectOnFiber`)执行([链接](https://github.com/facebook/react/blob/v19.2.6/packages/react-reconciler/src/ReactFiberCommitWork.js)).

可以看到, 官方`api`提供的`class组件`生命周期函数实际上也是围绕`fiber树构造`和`fiber树渲染`来提供的.

### function 组件

注: `function组件`与`class组件`最大的不同是: `class组件`会实例化一个`instance`所以拥有独立的局部状态; 而`function组件`不会实例化, 它只是被直接调用, 故无法维护一份独立的局部状态, 只能依靠`Hook`对象间接实现局部状态(有关更多`Hook`实现细节, 在`Hook原理`章节中详细讨论).

在`v19.2.6`中共定义了 25+ 种 Hook(详见[ReactFiberHooks.js](https://github.com/facebook/react/blob/v19.2.6/packages/react-reconciler/src/ReactFiberHooks.js)), 相较 v17 时期的 14 种, 新增的 hooks 大致可以分为 4 类:

- **状态类**(都基于 `useReducer` 实现, 都会创建 hook 对象并参与 fiber 树构造):
  - `useTransition`: 把更新降为`TransitionLane`, 配合`isPending`显示挂起状态.
  - `useDeferredValue`: 把状态降级为延迟值, 落到`DeferredLane`.
  - `useId`: 跨服务端/客户端稳定 id, 解决 hydration 不一致.
  - `useSyncExternalStore`: 让外部 store(redux/zustand 等)安全配合 concurrent rendering.
  - `useActionState` (v19): 配合 Server Actions, 自动管理 pending / state.
  - `useOptimistic` (v19): 在 Transition 期间显示乐观更新值.
  - `use` (v19): 渲染期挂起读取 `Promise / Context`, 可条件调用.
  - `useFormStatus` (v19, `react-dom`): 读取最近的`<form>` Action 的 pending 状态.
- **副作用类**:
  - `useEffect`: 异步 (Passive), 不阻塞 paint.
  - `useLayoutEffect`: 同步 (Layout), 阻塞 paint.
  - `useInsertionEffect` (v18 新增): 同步, 在 mutation 阶段执行, **早于** useLayoutEffect, 专门给 CSS-in-JS 库注入样式(此时 DOM 已突变但还没读 layout).
  - `useEffectEvent` (v19.2 experimental): 把闭包变量从 effect 依赖中剥离.
- **缓存类**: `useMemo / useCallback / useRef / useImperativeHandle`(语义不变).
- **上下文类**: `useContext`(语义不变, 但 v19 已支持`<Context value>`简写).

```js
function App() {
  // 状态相关: 初始状态
  const [a, setA] = useState(1);
  const changeState = () => {
    setA((prev) => prev + 1); // 进入 reconciler 流程
  };

  // 副作用相关: fiber.flags |= Passive (异步)
  useEffect(() => {
    console.log(`useEffect`);
  }, []);

  // 副作用相关: fiber.flags |= Update (同步, layout 阶段)
  useLayoutEffect(() => {
    console.log(`useLayoutEffect`);
  }, []);

  // v18 新增, 副作用相关: fiber.flags |= Update (同步, mutation 阶段)
  useInsertionEffect(() => {
    console.log(`useInsertionEffect`);
  }, []);

  // 返回下级 ReactElement 对象
  return <button onClick={changeState}>{a}</button>;
}
```

1. 状态相关: `fiber树构造`阶段.
   1. `useState`、`useReducer`、`useTransition`、`useDeferredValue`、`useId`、`useSyncExternalStore`、`useActionState`、`useOptimistic`等都在`renderRootSync[Concurrent]`阶段执行, 修改`Hook.memoizedState`并可能触发新的更新.
   2. `use(promise|context)` (v19): 在渲染期间挂起或读取上下文; 当读取的是 Promise 且未 resolve 时, 会`throw`一个 thenable, 由 React 接住并切到 Suspense 路径.
2. 副作用相关: `fiber树渲染`阶段.
   1. `useEffect`在`fiber树渲染`阶段(`commitRoot->commitBeforeMutationEffects`)调度`scheduleCallback(NormalSchedulerPriority, flushPassiveEffects)`, **异步执行**([链接](https://github.com/facebook/react/blob/v19.2.6/packages/react-reconciler/src/ReactFiberWorkLoop.js)).
   2. `useInsertionEffect`在`fiber树渲染`阶段(`commitRoot->commitMutationEffects->commitMutationEffectsOnFiber->commitHookEffectListMount(HookInsertion | HookHasEffect)`)**同步执行**, 早于 layout([链接](https://github.com/facebook/react/blob/v19.2.6/packages/react-reconciler/src/ReactFiberCommitHookEffects.js)).
   3. `useLayoutEffect`在`fiber树渲染`阶段(`commitRoot->commitLayoutEffects->commitLayoutEffectOnFiber->commitHookEffectListMount(HookLayout | HookHasEffect)`)**同步执行**([链接](https://github.com/facebook/react/blob/v19.2.6/packages/react-reconciler/src/ReactFiberCommitWork.js)).

### 细节与误区

这里有几个细节:

1. `useEffect(function(){}, [])`中的函数是**异步执行**, 因为它经过了调度中心(具体实现可以回顾[调度原理](./scheduler.md)).
2. `useLayoutEffect`和`Class组件`中的`componentDidMount, componentDidUpdate`从调用时机上来讲是等价的, 因为他们都在`commitRoot->commitLayoutEffects`函数中被调用.
   - 误区: 虽然官网文档推荐尽可能使用标准的`useEffect`以避免阻塞视觉更新, 所以很多开发者使用`useEffect`来代替`componentDidMount, componentDidUpdate`是不准确的. 如果完全类比, `useLayoutEffect`比`useEffect`更符合`componentDidMount, componentDidUpdate`的定义.
3. 三类同步/异步副作用的执行顺序(以同一个 fiber 节点为例):
   - **mutation 阶段**: `useInsertionEffect cleanup → useInsertionEffect setup → useLayoutEffect cleanup → DOM 突变`.
   - **layout 阶段**: `useLayoutEffect setup → componentDidMount/Update`.
   - **commit 完成后异步**: `useEffect cleanup → useEffect setup`(由 Scheduler 在空闲时冲刷).

## 总结

本节从`fiber`视角出发, 总结了`fiber`节点中可以影响最终渲染结果的 2 类属性(`状态`和`副作用`).并且归纳了`class`和`function`组件中, 直接或间接更改`fiber`属性的常用方式. 最后从`fiber树构造和渲染`的角度对`class的生命周期函数`与`function的Hooks函数`进行了比较.
