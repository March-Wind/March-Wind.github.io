---
title: React render阶段
date: 2024-02-27 16:07:36
tags:
  - React
categories: React
index_img: /img/react-logo.png
---

### 回顾

前面经过一个微任务之后，进行批量更新收集，然后再经过一个宏任务调度器进行调度。再往下就是 render 阶段，生成新的 Fiber 树。 scheduler 调度的函数是 Task.callback 也就是 performWorkOnRootViaSchedulerTask 函数，在 Root 上执行任务。然后就会走到 performWorkOnRoot 执行 render 阶段，然后再次调用 scheduleTaskForRootDuringMicrotask 执行调度。

### 渲染阶段

渲染阶段就是创建新的 Fiber 树，diff 就是比较新旧两个 Fiber 节点，来创新新的 Fiber 树的。当前页面的是 current 树，正在更新的是 workInProcess 树，两者通过 alternate 来指向对方。如果遇到更高优先级的任务，进行中断的话，就会清除 workInProcess,然后去构建高优先级任务的 Fiber 树；如果只是常规 5ms 检查，但是没有被高优先级任务打断，并不会清除 workInProcess 树，而是保留当前要执行的 Fiber。

### 实际场景

1. 多次输入，每次中断 startTransition，中断列表更新

```
import { useState, useTransition } from 'react';

const bigList = Array.from({ length: 50000 }, (_, i) => `Item ${i}${i % 2 === 0 ? 'e' : 'o'}`);

export default function App1() {
  const [query, setQuery] = useState('');
  const [filtered, setFiltered] = useState(bigList);
  const [isPending, startTransition] = useTransition();

  function handleChange(e) {
    const value = e.target.value;
    setQuery(value);

    // 这是低优先级任务，会被中断
    startTransition(() => {
      const result = bigList.filter((item) => item.includes(value));
      setFiltered(result);
      // setQuery(1); // 这里是后执行，输入框就会变成1
      // setQuery((prev) => prev + 1); // 这里是后执行，输入框就会变成query + 1
    });
  }

  return (
    <div>
      <input value={query} onChange={handleChange} />

      {isPending && <div>计算中...</div>}

      <ul>
        {filtered.map((item) => (
          <li key={item}>{item}</li>
        ))}
      </ul>
    </div>
  );
}
```
