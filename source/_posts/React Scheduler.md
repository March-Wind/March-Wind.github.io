---
title: React Scheduler
date: 2024-01-10 19:54:06
tags:
  - React
categories: React
index_img: /img/react-logo.png
---

### useState 后的链路

```
dispatchSetState

dispatchSetStateInternal 创建更新对象 Update ,还会判断最后一次值和当前值是否一致，如果一致，跳过渲染，不一致的话

enqueueConcurrentHookUpdate 增加到临时队列，后续会增加到 Fiber.queue.pending 上，

scheduleUpdateOnFiber
  标记 Root 有更新
  处理 Suspense：如果正在等待数据，中断当前渲染
  处理渲染阶段更新：如果在 render 中触发更新（不推荐），特殊处理
  scheduleImmediateRootScheduleTask 微任务回调，去执行processRootScheduleInMicrotask

  processRootScheduleInMicrotask(一次click事件中可能调用多次setState，但是只会执行一次微任务回调，也就是processRootScheduleInMicrotask，从而实现了批处理)
      为每个Root执行scheduler调度(scheduleTaskForRootDuringMicrotask),也就是创建Scheduler Task对象放进timerQueue或者taskQueue去执行。


      设置完调度后，执行同步更新(flushSyncWorkAcrossRoots_impl),此时scheduler并没有调度完成，只是设置调度。要等一个宏任务回调执行。

```

```
processRootScheduleInMicrotask

  处理饥饿的任务，等太久的任务，超时了，就标记为饥饿任务，从而在下一次调度中提升优先级 markStarvedLanesAsExpired(root, currentTime);

  计算这一批要处理的lanes getNextLanes

      同步任务，不新增scheduler任务，由微任务最后的flushSyncWorkAcrossRoots_impl函数执行。

      非同步任务：
          取最高优先级的lane作为callback的优先级，
          取消旧 callback，
          将lane 优先级映射成 scheduler 优先级(userBlocking/normal/idle),
          通过scheduleCallback注册scheduler task，
          callback为performWorkOnRootViaSchedulerTask

      返回本次任务的最高优先级的lane。

```

上面异步任务(非同步任务)调用

```
const newCallbackNode = scheduleCallback(
  schedulerPriorityLevel,
  performWorkOnRootViaSchedulerTask.bind(null, root),
);
```

scheduleCallback 就是 unstable_scheduleCallback 函数

```
  指定要延迟执行的任务执行 requestHostTimeout(handleTimeout, startTime - currentTime); 一定时间过去后，还是调用requestHostCallback();

  否则执行一个宏任务回调 requestHostCallback();
```

### Scheduler 调度器

> React Fiber 将任务拆分成小任务，Scheduler 就是调度这些单元任务的调度器。

### 核心职责

1. 🎯 任务调度：管理若干个不同优先级的任务
2. ⏱️ 时间切片：执行 5ms 时间的若干个单元任务，超过时间，就交给 commit 阶段，以及浏览器事件。
3. 📊 优先级管理：高优先级的先执行，每个单元任务都有优先级，通常对应着不同的超时时间，优先时间紧急的任务，即使一开始时间不紧急的任务，但是被多个时间紧急耽误了之后，自身的超级时间也在减少，而变得紧急，减少的就是等待的时间。
4. 🔄 可中断恢复：执行 5ms 后交给给浏览器，等待下一帧，继续执行若干个任务。

### 优先级

| scheduler 优先级名称 | scheduler 优先级数字 | 超时时间 | 使用场景                                                                                                                                                                                                 |
| -------------------- | -------------------- | -------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| immediatePriority    | 1                    | -1       | ImmediatePriority 基本只用于内部兼容旧环境（setImmediate、MessageChannel 缺失时）。例如 flushSync、useSyncExternalStore、内部事件(错误边界捕获/卸载组件)并没有走 scheduler 而是在 微任务阶段就立即执行了 |
| userBlockingPriority | 2                    | 250ms    | 用户交互(点击、输入、滚动)                                                                                                                                                                               |
| normalPriority       | 3                    | 5s       | 一般更新(网络请求后状态更新)                                                                                                                                                                             |
| lowPriority          | 4                    | 10s      | startTransition、Suspense、(useDeferredValue 不是 Scheduler 调度的任务，所以不在这里)                                                                                                                    |
| IdlePriority         | 5                    | 永不     | 内部使用(比如 React DevTool、Profiler、预加载下一屏数据、Fiber GC)                                                                                                                                       |

### 核心函数

`workLoop`:循环执行任务
`flushWork`:
`performWorkUntilDeadline`
`schedulePerformWorkUntilDeadline`

### 调度过程

```
            从延迟任务队列(timerQueue)中提取过期任务放进taskQueue

                      |
                      |
            从任务队列中(taskQueue中)中获取第一个任务(Task对象)

                      |
                      |
            判断是否继续执行任务(是否终端)

                      |       (1.当前任务没过期，过期的话，继续执行)
                      |       (2. 超过时间切片(5ms)`shouldYieldToHost()`)
                      |       (也就是说当然任务过期或者没超过5ms,都会继续执行)
            ┌────────┴─────────────────┐
            │                          │
            │                          │
            中断                      继续执行
            │                          │
            │                          │
交给浏览器去响应时间或者绘制页面       任务.callback为null是被取消了，直接从最小堆顶推出
                                  不为null，就执行callback，并且传入是否过期的标识

```

### callback 为 performWorkOnRootViaSchedulerTask

此时进来的任务都是要立即渲染的，进行 Fiber 渲染(performWorkOnRoot),甚至 commit.
渲染完之后，再次执行当前 Root 的调度(scheduleTaskForRootDuringMicrotask)
