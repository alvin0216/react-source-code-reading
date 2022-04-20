---
title: createRoot 启动过程
order: 1
---

## 启动过程

- 推荐阅读 [React 应用的启动过程](https://7kms.github.io/react-illustration-series/main/bootstrap)

> `React.render` -> `createRootImpl` -> `createFiberRoot`

1. `createRootImpl` 创建 `fiberRoot` 对象
   - `legacy` `blocking` `concurrent` 分别通过 `React.render`、`React.createBlockingRoot`、 `createRoot`
2. `createFiberRoot` 创建 `HostRootFiber` 对象

```js
export function createHostRootFiber(tag: RootTag): Fiber {
  let mode;
  if (tag === ConcurrentRoot) {
    mode = ConcurrentMode | BlockingMode | StrictMode;
  } else if (tag === BlockingRoot) {
    mode = BlockingMode | StrictMode;
  } else {
    mode = NoMode;
  }
  return createFiber(HostRoot, null, null, mode); // 注意这里设置的mode属性是由RootTag决定的
}
```

## 为什么某些生命周期标记为 unsafe

componentWillMount、componentWillMount、componentWillUpdate 为什么标记 UNSAFE?

原因是为了可中断渲染做准备

react 中最广为人知的可中断渲染(render 可以中断, **部分生命周期函数有可能执行多次**。

`UNSAFE_componentWillMount`,`UNSAFE_componentWillReceiveProps`)只有在 `HostRootFiber.mode === ConcurrentRoot | BlockingRoot` 才会开启.

如果使用的是 `legacy`, 即通过 `ReactDOM.render(<App/>, dom)`这种方式启动时 `HostRootFiber.mode = NoMode`, 这种情况下无论是首次 render 还是后续 update 都只会进入同步工作循环, reconciliation 没有机会中断, 所以生命周期函数只会调用一次.

在最新稳定版 v17.0.2 中, 可中断渲染虽然实现, 但是并没有在稳定版暴露出 api. 只能安装 `alpha` 版本才能体验该特性.
