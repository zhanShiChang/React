# legacy 模式下调度和调和的理解(理解批量更新以及打破批量更新)

在 React 中，"legacy 模式"（Legacy Mode）指的是旧版本的 React 特性和行为，这些特性和行为在较新的 React 版本中可能已经被弃用或替换。使用 legacy 模式通常是为了兼容旧代码，确保在升级到新版本时不会立即破坏现有应用程序的功能。

本章理解什么是调度和调和？ 调度和调和是怎么实现的？React的更新机制？
调度的本质（时间分片，请求帧）和调和的流程（两大阶段 render 和 commit ）。

## 举例
如果有很多顾客同时点餐，你不能一次性完成所有订单，因为这样会让厨房过载，影响其他工作的正常进行。

## 调度过程

### 排队

所有顾客的订单会先排队，按照先来后到的顺序处理。

### 分批处理

你会先处理一个订单，然后暂停一下，检查厨房的其他工作是否正常，比如检查炉火、清理工作台等。确保一切正常后，再继续处理下一个订单。

### 优先级

如果有VIP顾客（高优先级任务）和普通顾客（低优先级任务），你会优先处理VIP顾客的订单。但如果普通顾客等得太久，你也会插空处理一些普通顾客的订单，避免他们等得太久。

## 例子总结

调度：就是在多个订单（任务）之间合理分配时间，确保厨房（系统）正常运转，不会因为一次性处理所有订单而过载。

调和：一旦开始处理某个订单（任务），就会进入具体的烹饪（更新）流程，直到完成这道菜（更新视图）。

# React 更新

React更新本质有两种：初始化和state更新(触发点击事件，setState)

## 初始化更新流程

```js
import ReactDOM from 'react-dom'
/* 通过 ReactDOM.render  */
ReactDOM.render(
    <App />,
    document.getElementById('app')
)
```
在 ReactDOM.render 做的事情是形成一个 Fiber Tree 挂载到 app 上。

> react-dom/src/client/ReactDOMLegacy.js -> legacyRenderSubtreeIntoContainer
```js
function legacyRenderSubtreeIntoContainer(
    parentComponent,  // null
    children,         // <App/> 跟部组件
    container,        // app dom 元素
    forceHydrate,
    callback          // ReactDOM.render 第三个参数回调函数。
){
    let root = container._reactRootContainer
    let fiberRoot
    if(!root){
        /* 创建 fiber Root */
        root = container._reactRootContainer = legacyCreateRootFromDOMContainer(container,forceHydrate);
        fiberRoot = root._internalRoot;
        /* 处理 callback 逻辑，这里可以省略 */
        /* 注意初始化这里用的是 unbatch */
        unbatchedUpdates(() => {
            /*  开始更新  */
            updateContainer(children, fiberRoot, parentComponent, callback);
        });
    }
}
```
调用  ReactDOM.render 本质上就是 `legacyRenderSubtreeIntoContainer` 方法。这个方法的主要做的事情是：
* 创建整个应用的 `FiberRoot` 。
* 然后调用 `updateContainer` 开始初始化更新。
* 这里注意⚠️的是，用的是 **`unbatch`** （非批量的情况），并不是批量更新的 `batchUpdate` 。

接下来开始更新，调用updateContainer方法

> react-reonciler/src/ReactFiberReconciler.js -> updateContainer
```js
export function updateContainer(element,container,parentComponent,callback){
    /* 计算优先级，在v16及以下版本用的是 expirationTime ，在 v17 ,v18 版本，用的是 lane。  */
    const lane = requestUpdateLane(current);
    /* 创建一个 update */
    const update = createUpdate(eventTime, lane);
    enqueueUpdate(current, update, lane);
    /* 开始调度更新 */
    const root = scheduleUpdateOnFiber(current, lane, eventTime);
}
```

* 首先计算更新优先级 `lane` ，老版本用的是 `expirationTime`。
* 然后创建一个 `update` ，通过 `enqueueUpdate` 把当前的 update 放入到待更新队列 `updateQueue` 中。
* 接下来开始调用 `scheduleUpdateOnFiber` ，开始进入调度更新流程中。

## state更新(触发点击事件，setState)
state的更新分为类组件和函数组件
**类组件之 `setState`**：
> react-reconciler/src/ReactFiberClassComponent.js -> enqueueSetState

```js
enqueueSetState(inst,payload,callback){
    const update = createUpdate(eventTime, lane);
    enqueueUpdate(fiber, update, lane);
    const root = scheduleUpdateOnFiber(fiber, lane, eventTime);
}
```

**函数组件之 `useState`**

> react-reconciler/src/ReactFiberHooks.js -> dispatchAction
```js
function dispatchAction(fiber, queue, action) {
    var lane = requestUpdateLane(fiber);
    scheduleUpdateOnFiber(fiber, lane, eventTime);
}
```
从上面可得知最后都进入了scheduleUpdateOnFiber，也就是调度的入口。

### 更新入口 scheduleUpdateOnFiber
> react-reconciler/src/ReactFiberWorkLoop.js -> scheduleUpdateOnFiber
```js
export function scheduleUpdateOnFiber(fiber,lane,eventTime){
    if (lane === SyncLane) {
        if (
            (executionContext & LegacyUnbatchedContext) !== NoContext && // unbatch 情况，比如初始化
            (executionContext & (RenderContext | CommitContext)) === NoContext) {
            /* 开始同步更新，进入到 workloop 流程 */    
            performSyncWorkOnRoot(root);
         }else{
               /* 进入调度，把任务放入调度中 */
               ensureRootIsScheduled(root, eventTime);
               if (executionContext === NoContext) {
                   /* 当前的执行任务类型为 NoContext ，说明当前任务是非可控的，那么会调用 flushSyncCallbackQueue 方法。 */
                   flushSyncCallbackQueue();
               }
         }
    }
}
```
首先判断是否为正常任务(SyncLane)，无论同步任务或异步任务(Promise或setTimeout)都属于`SyncLane`。
**那什么不属于SyncLane？**
低优先级的任务。
例如 TransitionLane(页面切换时的动画效果),IdleLane(后台数据同步、日志记录等),DefaultLane(一般的状态更新，不需要立即响应用户操作),OffscreenLane(预加载隐藏的组件或内容),BlockingLane(表单验证、输入框的自动完成等),这些一般不需要立即更新的任务。
那么在 `scheduleUpdateOnFiber` 内部主要做的事情是：

* 在 `unbatch` 情况下，会直接进入到 performSyncWorkOnRoot ，接下来会进入到 **调和流程**，比如 `render` ，`commit`。
* 那么任务是 `useState` 和 `setState`，那么会进入到 `else` 流程，那么会进入到 `ensureRootIsScheduled` 调度流程。
* 当前的执行任务类型为 `NoContext` ，说明当前任务是非可控的，那么会调用 `flushSyncCallbackQueue` 方法。

**`legacy` 模式下的可控任务和非可控任务。**

* 可控任务：在事件系统章节和 state 章节讲到过，对于 React 事件系统中发生的任务，会被标记 `EventContext`，在 batchUpdate api 里面的更新任务，会被标记成 `BatchedContext`，那么这些任务是 React 可以检测到的，所以 `executionContext !== NoContext`，那么不会执行 `flushSyncCallbackQueue`。

* 非可控任务：如果在**延时器（timer）队列**或者是**微任务队列（microtask）**，那么这种更新任务，React 是无法控制执行时机的，所以说这种任务就是非可控的任务。比如 `setTimeout` 和 `promise` 里面的更新任务，那么 `executionContext === NoContext` ，接下来会执行一次 `flushSyncCallbackQueue` 。

## 进入调度更新
非初始化类型的更新任务，那么最终会走到 ensureRootIsScheduled 流程中。
> react-reconciler/src/ReactFiberWorkLoop.js -> ensureRootIsScheduled
```js
function ensureRootIsScheduled(root,currentTime){
    /* 计算一下执行更新的优先级 */
    var newCallbackPriority = returnNextLanesPriority();
    /* 当前 root 上存在的更新优先级 */
    const existingCallbackPriority = root.callbackPriority;
    /* 如果两者相等，那么说明是在一次更新中，那么将退出 */
    if(existingCallbackPriority === newCallbackPriority){
        return 
    }
    if (newCallbackPriority === SyncLanePriority) {
        /* 在正常情况下，会直接进入到调度任务中。 */
        newCallbackNode = scheduleSyncCallback(performSyncWorkOnRoot.bind(null, root));
    }else{
        /* 这里先忽略 */
    }
    /* 此时已调度完成 避免再次调度相同优先级的任务 给当前 root 的更新优先级，绑定到最新的优先级  */
    root.callbackPriority = newCallbackPriority;
}
```
ensureRootIsScheduled 主要做的事情有：

* 首先会计算最新的调度更新优先级 `newCallbackPriority`，接下来获取当前 root 上的 `callbackPriority` 判断两者是否相等。如果两者相等，那么将直接退出不会进入到调度中。
* 如果不想等那么会真正的进入调度任务 `scheduleSyncCallback` 中。注意的是放入调度中的函数就是**调和流程**的入口函数 `performSyncWorkOnRoot`。
* 函数最后会将 newCallbackPriority 赋值给 callbackPriority。

**什么情况下会存在 existingCallbackPriority === newCallbackPriority，退出调度的情况？**

我们注意到在一次更新中最后 callbackPriority 会被赋值成 newCallbackPriority 。那么如果在正常模式下（非异步）一次更新中触发了多次 `setState` 或者 `useState` ，那么第一个 setState 进入到 ensureRootIsScheduled 就会有 root.callbackPriority = newCallbackPriority，那么接下来如果还有 setState | useState，那么就会退出，将不进入调度任务中，**原来这才是批量更新的原理，多次触发更新只有第一次会进入到调度中。**

### 进入调度任务

上面的例子进入了scheduleSyncCallback，scheduleSyncCallback做了什么？
> react-reconciler/src/ReactFiberSyncTaskQueue.js -> scheduleSyncCallback
```js
function scheduleSyncCallback(callback) {
    if (syncQueue === null) {
        /* 如果队列为空 */
        syncQueue = [callback];
        /* 放入调度任务 */
        immediateQueueCallbackNode = Scheduler_scheduleCallback(Scheduler_ImmediatePriority, flushSyncCallbackQueueImpl);
    }else{
        /* 如果任务队列不为空，那么将任务放入队列中。 */
        syncQueue.push(callback);
    }
} 
```

`flushSyncCallbackQueueImpl` 会真正的执行 `callback` ，本质上就是调和函数 `performSyncWorkOnRoot`。 

`Scheduler_scheduleCallback` 就是在调度章节讲的调度的执行方法，本质上就是通过 **`MessageChannel`** 向浏览器请求下一空闲帧，在空闲帧中执行更新任务。

接下来有一个问题就是，**比如在浏览器空闲状态下发生一次 state 更新，那么最后一定会进入调度，等到下一次空闲帧执行吗？** 

答案是否定的，如果这样，那么就是一种性能的浪费，因为正常情况下，发生更新希望的是在一次事件循环中执行完更新到视图渲染，如果在下一次事件循环中执行，那么更新肯定会延时。但是 `React` 是如何处理这个情况的呢？

### 空闲期的同步任务

在没有更新任务空闲期的条件下，为了让更新变成同步的，也就是本次更新不在调度中执行，那么 React 对于更新，会用 `flushSyncCallbackQueue` 立即执行更新队列，发起更新任务，**目的就是让任务不延时到下一帧**。但是此时调度会正常执行，不过调度中的任务已经被清空，

那么有的同学可以会产生疑问，既然不让任务进入调度，而选择同步执行任务，那么调度意义是什么呢? 

调度的目的是处理存在多个更新任务的情况，比如发生了短时间内的连续的点击事件，每次点击事件都会更新 state ，那么对于这种更新并发的情况，第一个任务以同步任务执行，那么接下来的任务将放入调度，等到调度完成后，在下一空闲帧时候执行。

#### 可控更新任务

那么知道了，发生一次同步任务之后，React 会让调度执行，但是会立即执行同步任务。原理就是通过 `flushSyncCallbackQueue` 方法。对于可控的更新任务，比如事件系统里的同步的 setState 或者 useState，再比如 batchUpdate，如果此时处理空闲状态，在内部都会触发一个 `flushSyncCallbackQueue`来立即更新。我们看一下:


**事件系统中的**
> react-reconciler/src/ReactFiberWorkLoop.js -> batchedEventUpdates
```js
function batchedEventUpdates(fn, a){
     /* 批量更新流程，没有更新状态下，那么直接执行任务 */
     var prevExecutionContext = executionContext;
     executionContext |= EventContext;
    try {
        return fn(a) /* 执行事件本身，React 事件在这里执行，useState 和 setState 也会在这里执行 */
    } finally {
     /* 重置状态 */ 
    executionContext = prevExecutionContext;
    if (executionContext === NoContext) { 
      /* 批量更新流程，没有更新状态下，那么直接执行任务 */
      flushSyncCallbackQueue();
    }
  }
}
```

**ReactDOM暴露的api `batchedUpdates`**
> react-reconciler/src/ReactFiberWorkLoop.js -> batchedUpdates
```js
function batchedUpdates(fn, a) {
    /* 和上述流程一样 */
    if (executionContext === NoContext) {
      flushSyncCallbackQueue();
    }
}
```

如上可以看到，如果浏览器没有调度更新任务，那么如果发生一次可控更新任务，最后会默认执行一次 `flushSyncCallbackQueue` 来让任务同步执行。


#### 非可控更新任务

如果是非可控的更新任务，比如在 `setTimeout` 或者 `Promise` 里面的更新，那么在 scheduleUpdateOnFiber 中已经讲过。

```js
if (executionContext === NoContext) {
    /* 执行 flushSyncCallbackQueue ，立即执行更新 */
    flushSyncCallbackQueue();
}
```
综上这也就说明了，为什么在异步内部的 `setState` | `useState` 会打破批量更新的原则，本质上是因为，执行一次 `setState` | `useState` 就会触发一次 `flushSyncCallbackQueue` 立即触发更新，所以就会进入到调和阶段，去真正的更新 fiber 树。
