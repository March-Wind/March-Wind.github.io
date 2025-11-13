---
title: React Schedule源码阅读
date: 2023-11-20 11:42:08
categories: React
index_img: /img/react-logo.png
---

### 入口 unstable_scheduleCallback 调度 callback

1. 开始时间(startTime = 当前时间 + 传入的 delay) + 优先级时间，计算失效时间(expirationTime)
2. 创建 newTask(来自 unstable_scheduleCallback)
   - id:
   - callback // 传进来的 callback
   - priorityLevel // 传进来的优先级 priorityLevel
   - startTime // 上面计算的时间
   - expirationTime // 上面计算的失效时间
   - sortIndex: -1 // ??? 都是被复制为 expirationTime
3. 如果是`定时任务(当前时间后的一定时间才执行，现在还没有进入调度的资格，放在timerQueue中)`，也就是未来时间执行的任务(startTime > currentTime)

   - 将 startTime 赋值给 sortIndex(newTask.sortIndex = startTime)
   - 将 newTask 加入`定时任务队列中(timerQueue)`
   - 如果只有`定时任务(timeQueue)`,等待延迟执行时间后，触发`处理超时函数(handleTimeout)`,
   - `处理超时函数(handleTimeout)`：
     - `提取定时队列中到时间的的任务到任务队列taskQueue(advanceTimers)`：主要是从定时任务队列(timeQueue)将到时间的任务放到任务队列中(taskQueue)；没有 callback 的释放掉；没有到执行时间的任务，再等等；
     - 由于定时任务已经到执行时间，就放到任务队列，然后查看是否有任务执行，没有任务就执行该任务(唯一执行机制：用`isHostCallbackScheduled`标识来标识之前是否有`下一次事件循环结束立即执行任务`)，有就释放掉。
     - `请求主持回调任务(requestHostCallback)`：启动一个`调度 执行等到最后期限任务(schedulePerformWorkUntilDeadline)`来执行`刷新工作(flushWork)`，并且用`isMessageLoopRunning`来锁定只有一次任务在下一轮事件循环后立即执行。`刷新工作(flushWork)`是挂在`全局变量scheduledHostCallback`上，之后在`等到最后期限时执行任务(performWorkUntilDeadline)`调用，之所以放在`全局变量scheduledHostCallback`上，是为了方便扩展，执行其他函数。
     - `isHostCallbackScheduled`和`isMessageLoopRunning`作用一样， 保证一次 event loop 后只有一个任务执行,并且是串行的(保证了所有的事件占用时间是确定的，空闲也是确定的)，双重保险。

4. 不是定时任务，将该任务放入`任务队列(taskQueue)`中后，`请求主持回调任务(requestHostCallback)`，最终会统一在`等到最后期限时执行任务(performWorkUntilDeadline)`中执行。
5. `等到最后期限时执行任务(performWorkUntilDeadline)`
   - 内部调用`全局变量scheduledHostCallback`即是`flushWork(true,currentTime)`, 并且在函数开始时，记录下当前时间为`开始时间(startTime)`,方便后面判断是否在一帧之内还有时间，作为是否还继续执行任务的依据。
   - 得到`flushWork`返回的结果`是否有更多的工作(hasMoreWork)`，如果有更多的工作，有可能是优先级任务带来的延迟时间导致任务没有到过期时间(???)，也有可能是执行的任务不是 taskQueue 的第一个导致没删除也没处理就自动从新开始，就继续`调度最后期限任务(schedulePerformWorkUntilDeadline)`的`刷新任务(flushWork)`，
6. `刷新工作(flushWork)`
   - 测试环境时 enableProfiling 为 true，会加 try{}catch 包裹`wookLoop`,把报错显示在屏幕,生产环境时,直接执行`wookLoop`
   - `工作循环(wookLoop)`:
     - `提取定时队列中到时间的的任务到任务队列taskQueue(advanceTimers)`
     - 拿出最早的任务，如果任务存在，并且不是调度 debug 模式，那么就是正常流程
       - 判断是否中断执行：`过期时间(expirationgTime)` 大于 `当前时间(currentTime)` ,或者`刷新工作`的开始时间到现在时间 大于 一帧时间，就中断。(hasTimeRemaining 这个字段暂时没用)。这里才用到优先级产生的过期时间，如果没有到期，就不执行任务，直到返回 true 给父级函数，然后继续`调度最后期限任务(schedulePerformWorkUntilDeadline)`
       - 没有中断执行的话，就检查 task.callback 是否是个函数，如果不是直接释放掉 task；
       - 如果 task.callback 是函数，将 task.callback 的地址复制给当前局部变量 callback，然后将 task.callback 置为 null,将`任务的优先级(task.priorityLevel)`设置为`当前优先级(currentPriorityLevel)`,然后执行 callback,并传入参数过期标识(就是 true)
         - 如果`callback(true)`返回为一个函数，将该函数赋值给`task.callback`，将再次进入循环，执行 callback 任务,再次进入循环的方法就是不从 taskQueue 中去掉。
         - 如果`callback(true)`返回不是一个函数，将剔除当前任务，也就是剔除任务队列中的第一个任务，这样就没办法循环自身了。因为当前的任务总是任务队列最优先执行的，所以就是当前执行掉的。所以就是任务队列的第一个。
         - 然后做一遍`提取定时队列中到时间的的任务到任务队列taskQueue(advanceTimers)`，之后取优先级最高的任务，也就是第一个任务来继续循环。
     - 以防`当前的任务(currentTask)`不是 `任务队列(taskQueue)`中的第一个，那么就有可能没从`任务队列(taskQueue)`中删除掉，再次检查 currentTask 是否是 null，如果是，就返回 true 给`执行过期的任务(performWorkUntilDeadline)`,它会产生一个`调度过期任务(schedulePerformWorkUntilDeadline)`；同时如果任务没有到过期时间，是不会执行的，此时 currentTask 还是第一个任务，就返回 true 给`执行到过期的任务(performWorkUntilDeadline)`，它会产生一个`调度过期任务(schedulePerformWorkUntilDeadline)`
     - 如果为空，那么只剩定时任务，就重复定时，然后触发`处理超时函数(handleTimeout)`

## 路程上的疑惑，看完后解答

1. `定时任务`,`优先级产生的时间间隔造成的过期时间`，一开始是等待的`定时时间`，等到判断否是执行的时候用的是`过期时间`来检查是否大于当前时间，如果还没到，那么就返回有任务
2. 取消前一个`定时任务`，会在 workLoop 里面执行`提取到时间的定时任务到任务队列(advanceTimers)`,和执行完任务队列后也会拿`定时任务队列`再次设置`定时器(requestHostTimeout)`来再次执行`超时处理函数`来检查到期任务。

## 属性

1. 优先级对应的时间延迟
   - ImmediatePriority： -1 // 也就是立即执行
   - UserBlockingPriority： 250 // 用户锁定，也就是用户一直操作的间隔
   - IdlePriority：maxSigned31BitInt，也就是 1073741823，也就是 2^30 - 1 // 空闲
   - LowPriority：10000 // 低优先级
   - NormalPriority：5000 // 正常优先级
2. taskQueue 是同步任务队列
3. timerQueue 定时任务，还没有执行，所以也不在调度之中，等到执行时间后才进入调度循环。
4. scheduledHostCallback
   - requestHostCallback 里赋值
   - performWorkUntilDeadline 里执行
5. schedulePerformWorkUntilDeadline 产生宏任务，将执行权力移交浏览器
   - 关于赋值是优先 setImmediate，因为是经过一轮 event loop 立即执行，但是兼容性不好，可以使用 MessageChannel 来实现，如果没有 MessageChannel，再用 setTimeOut,但是浏览器的 setTimeout(() => {...},0)的 delay 起码是 4，也就是会被修改成 setTimeout(() => {...},4)，就浪费了 4ms
   - schedulePerformWorkUntilDeadline 的回调是 performWorkUntilDeadline
6. isHostCallbackScheduled 锁定`下一轮事件循环立即执行的宏任务(schedulePerformWorkUntilDeadline)`在一次 event loop 循环只有一个任务执行。
7. isMessageLoopRunning 跟 isHostCallbackScheduled 的作用一样，双重保险
8. isHostTimeoutScheduled 是用来打断之前的等待任务，每次进来的任务是延迟任务，并且只有延迟任务，那么久打断上一个延迟任务的等待时间。
9. hasTimeRemaining：代表当前帧是否还有时间留给 react
10. 这里的 callback 也就是外部传进来的执行函数也就是执行任务
11. `下一次时间循环结束立即执行任务` 就是一个执行事件的时机

## 堆排序(Schedule 利用最小堆来获取最早的任务)

1. 调度就是用堆排序来获得最早过期任务的，最小堆，最小的时间戳在 index=0 位置
2. 之所以用堆排序，是因为时间复杂度更低，最小堆构建的时间复杂度是 O(n)精确时间复杂度是 n-log(n+1),比 n 还小。每次加入一个任务之后就构建最小堆，事件复杂度是 O(n)，比冒泡排序 O(n^2)快很多。
3. [讲解链接](https://github.com/March-Wind/tool/tree/master/src/algorithm/heap-sort)
