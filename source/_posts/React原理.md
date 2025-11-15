---
title: React原理
date: 2024-11-03 19:54:06
tags:
  - React
categories: React
index_img: /img/react-logo.png
---

### React Fiber

React 15 之前是 stack Reconciler，也就是从虚拟 DOM 的根节点递归遍历所有节点进行跟旧的节点对比，记录所有差异，进行更新，然后立即执行更改，直到更新完成。但是在大组件树更新的时候，就有长期占据主线程，导致页面响应不了，并且有掉帧。
React 16 开发引入了 Fiber 架构，把比较虚拟 DOM 并记录变更阶段的任务拆分成小任务，然后根据优先级，每执行一小段时间(5ms)，就提交这些更新，交给浏览器先去绘制 DOM、响应用户输入。

#### 辨析

1. React 并没有提高渲染性能，反而降低的渲染性能，因为中间还要对比虚拟 DOM。只是让视图更新更高效、更可维护。高效是声明式 UI 的高效，是开发阶段的高效，不是渲染阶段的高效。
2. 如果执行 DOM 操作，增删改产和样式修改，是不会立刻绘制的，先放在渲染队列，等待屏幕刷新节点到来，再集中一次性渲染更新。参考我的另一篇文章：{% post_link 浏览器渲染机制 %}。所以 React 的 Fiber 架构，就是解决一大堆任务集中渲染带来的掉帧问题和主线程无法响应问题，将任务用 Fiber 对象分成一个个小任务，然后分批操作 DOM，分批来让浏览器渲染。

#### Fiber 对象结构

key: string; React Element 唯一标识。渲染列表时，设置的 key 就是它。
type: :组件构造器,例如"div"
tag: WorkTag; // React Element 类型，比如 ClassComponent、FunctionComponent、SuspenseComponent、HostText(文本节点) 等。
props: ; 组件参数
memoizedState: any; // 存放 Hooks 链表的地方，useState/useEffect 都放在这里，只不过存放的内容不同，分别是 Hook 对象、Effect 对象。
dependencies: Dependencies; //存放依赖的 Context 的链表。

### React Concurrent

Fiber 架构提供中断的能力，React v18 增加 Concurrent Feature，比如 useTransition、useDeferredValue、Suspense。使用`createRoot`创建根节点，自动开启 Concurrent Mode，同时也会对所有更新自动进行批处理。这些都是 React v18 这个版本新特性，React 18 一个大版本，或者叫旗舰版本。

### useState/setState 等状态更新是异步的还是同步的。

同步和异步是指使用`useState`/`setState`更新 state 值，会不会立刻反映到 state 上。

- React 18 之前，useState 的更新行为和 setState 一致：在 React 事件中是批量异步的，在原生事件(onClick、setTimeout)中是同步的。
- React 18 之后，引入了自动批处理，无论在任何场景都是异步的，要想访问同步的 state 值，需要使用回调函数的形式：
  ```JavaScript
    setState(prev => prev + 1);
  ```

### useState/useEffect 都是怎么实现的，为什么不能放进条件判断里

首次渲染 useState 会内部调用的 mountState，会创建一个 Hook 对象放在 Fiber 的 memoizedState 链上，memoizedState 是一个链表，所有的 Hooks 初始创建的时候，都会追加在链表的最后一个节点的 next 上。
更新渲染时，useState 内部调用的 UpdateState，会从 memoizedState 上第一个节点去取更新值，下一个 useSate，就通过当前 hook.next 获取下一个的 Hook 来更新。所以 useState 这种 hooks 函数不能少，因为他们是一一对应的。

> 延伸 1：首次渲染、更新渲染、render(类比 classComponent 的 render 方法里设置 setState)期间,使用的是不同 hooks API，分别是 HooksDispatcherOnMount、HooksDispatcherOnUpdate、HooksDispatcherOnRerender。

> 延伸 2：updateState 最终调用 updateWorkInProgressHook 函数来取下一个 Hook 节点，内部代码如下：

```JavaScript
  if (currentHook === null) {
    const current = currentlyRenderingFiber.alternate;
    if (current !== null) {
      nextCurrentHook = current.memoizedState;
    } else {
      nextCurrentHook = null;
    }
  } else {
    nextCurrentHook = currentHook.next;
  }
```

### React 合成事件

合成事件是指，基于原生事件，对不同浏览器，进行标准化处理，在进行功能强化。事件委托在 Root 上，也就是在 Root 上监听所有事件，然后再去 Fiber 树中找到监听对应事件的回调函数，再执行。所有事件注册在 Root 时，就带有优先级，分为离散事件(DiscreteEventPriority)，比如 click、input 事件都是立即执行的、连续事件(ContinuousEventPriority)，比如 scroll、mousemove、drag 等都是可延迟的。

> 优势：解决浏览器兼容性问题；事件委托，减少内存消耗；统一管理事件，方便根据优先级进调度
> 执行顺序：document、Root 捕获事件、然后是 ReactElement 的捕获事件、然后是 ReactElement 的冒泡事件、最后是 Root、document 的冒泡事件。这里并没有阻止 DOM 的原生事件，注册 DOM 的原生事件还是会执行，捕获事件会在 ReactElement 事件之后执行，冒泡事件会在 ReactElement 之前执行。

### 组件通信

React 是单向数据流，父组件的数据和行数可以通过 props 转给子组件。父组件可以通过 forwardRef+useImperativeHandle 方法来查阅子组件数据和调用子组件函数。此外还可以通过 context API 跨层级通信，还有 zustand 等外部库，通过发布订阅形式+useSyncExternalStore 来进行通信。

#### context 实现机制

1. 使用 createContext 来创建 Context 对象，值就是放在 context.currentValue 的中，Provider 是提供者，Customer 是消费者，现在常用 useContext 来消费。
2. 所有嵌套的 Context 都放在一个栈结构中，用数组表示的栈，全局公用这个栈。遇到 Provider 就压入栈，离开 Provider 就推出栈。

   > 同一个 Provider 进行嵌套,相同的 context 在不同作用里可以设置不同值。离开 Provider 的时候会恢复上一个值。

   ```JavaScript
    // 问题：嵌套Provider如何正确恢复值？
    <ThemeContext.Provider value="dark">
      <Child1 />  {/* 应该读到 "dark" */}

      <ThemeContext.Provider value="light">
        <Child2 />  {/* 应该读到 "light" */}
      </ThemeContext.Provider>

      <Child3 />  {/* 应该读到 "dark"（恢复了！） */}
    </ThemeContext.Provider>

    // 栈操作图解

    // 初始状态：
    // index = -1
    // valueCursor.current = null
    // valueStack = []

    // 1.push: index = 0、valueCursor.current = "dark"、valueStack = [null]、context._currentValue = "dark"
    // 2.push: index = 1、valueCursor.current = "light"、valueStack = [null, "dark"]  // 保存了历史值、context._currentValue = "light"
    // 3.pop: index =0 、valueCursor.current = dark // 从valueStack恢复"dark"、valueStack = [null]、context._currentValue = "dark"
   ```

3. 在 useContext 时，会在 Fiber 的 dependencies 上记录以来的 context，Provider 的值更新的时候，会遍历所有子树的 dependencies，如果存在就触发更新。
   > 性能优化：1. 使用 useMemo 对值进行缓存；2. 拆分单独的 context。

### diff 算法

按照新节点是单节点或多节点两种情况 diff。单节点 Diff,遍历旧节点链表，找到 key 相同并且 type 也相同或者 ElementType 相同的节点，如果找到就复用，并且删除其他兄弟节点，如果没有找到，删除旧的，创建新的。多节点 diff，同时遍历新旧两个链表，index=0 开始，找到 index 相等，key 也相等的 Fiber 就复用,遇到不相等的就跳出第一轮循环；然后看看新链表是否还有剩余，没剩余，就删除旧链表上的其他节点，有剩余的话，看就旧链表是否还有剩余，没有剩余，就直接创建新的剩余节点；如果旧链接也有剩余节点，那么把旧链表上的剩余节点整理整 Map 对象，遍历新链表从 Map 中看看能不能找到可复用的，找到就复用，找不到的就创建新的。多节点时，不写 key，会增加比较次数，影响性能。

### 并发模式下 UI 一致性（Consistency） 如何保证(双缓冲机制)
