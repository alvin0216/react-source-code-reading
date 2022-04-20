---
title: scheduler 任务调度
order: 2
---

> 排序优先级，让优先级高的任务先进行 `reconcile`

## Lane 优先级

原理是通过二进制掩码的方式 & 位运算 进行计算，不展开，推荐阅读 [React 中的优先级管理](https://7kms.github.io/react-illustration-series/main/priority)

## 时间分片

```js
// 接收 MessageChannel 消息
const performWorkUntilDeadline = () => {
  // ...省略无关代码
  if (scheduledHostCallback !== null) {
    const currentTime = getCurrentTime();
    // 更新deadline
    deadline = currentTime + yieldInterval;
    // 执行callback
    scheduledHostCallback(hasTimeRemaining, currentTime);
  } else {
    isMessageLoopRunning = false;
  }
};

const channel = new MessageChannel();
const port = channel.port2;
channel.port1.onmessage = performWorkUntilDeadline;

// 请求回调
requestHostCallback = function (callback) {
  // 1. 保存callback
  scheduledHostCallback = callback;
  if (!isMessageLoopRunning) {
    isMessageLoopRunning = true;
    // 2. 通过 MessageChannel 发送消息
    port.postMessage(null);
  }
};
// 取消回调
cancelHostCallback = function () {
  scheduledHostCallback = null;
};
```

很明显, 请求回调之后`scheduledHostCallback = callback`, 然后通过`MessageChannel`发消息的方式触发`performWorkUntilDeadline`函数, 最后执行回调`scheduledHostCallback`.

此处需要注意: `MessageChannel`在浏览器事件循环中属于`宏任务`, 所以调度中心永远是`异步执行`回调函数.

时间切片(`time slicing`)相关: 执行时间分割, 让出主线程(把控制权归还浏览器, 浏览器可以处理用户输入, UI 绘制等紧急任务).

- [getCurrentTime](https://github.com/facebook/react/blob/v17.0.2/packages/scheduler/src/forks/SchedulerHostConfig.default.js#L22-L24): 获取当前时间
- [shouldYieldToHost](https://github.com/facebook/react/blob/v17.0.2/packages/scheduler/src/forks/SchedulerHostConfig.default.js#L129-L152): 是否让出主线程
- [requestPaint](https://github.com/facebook/react/blob/v17.0.2/packages/scheduler/src/forks/SchedulerHostConfig.default.js#L154-L156): 请求绘制
- [forceFrameRate](https://github.com/facebook/react/blob/v17.0.2/packages/scheduler/src/forks/SchedulerHostConfig.default.js#L168-L183): 强制设置 `yieldInterval`(从源码中的引用来看, 算一个保留函数, 其他地方没有用到)

```js
const localPerformance = performance;
// 获取当前时间
getCurrentTime = () => localPerformance.now();

// 时间切片周期, 默认是5ms(如果一个task运行超过该周期, 下一个task执行之前, 会把控制权归还浏览器)
let yieldInterval = 5;

let deadline = 0;
const maxYieldInterval = 300;
let needsPaint = false;
const scheduling = navigator.scheduling;
// 是否让出主线程
shouldYieldToHost = function () {
  const currentTime = getCurrentTime();
  if (currentTime >= deadline) {
    if (needsPaint || scheduling.isInputPending()) {
      // There is either a pending paint or a pending input.
      return true;
    }
    // There's no pending input. Only yield if we've reached the max
    // yield interval.
    return currentTime >= maxYieldInterval; // 在持续运行的react应用中, currentTime肯定大于300ms, 这个判断只在初始化过程中才有可能返回false
  } else {
    // There's still time left in the frame.
    return false;
  }
};

// 请求绘制
requestPaint = function () {
  needsPaint = true;
};

// 设置时间切片的周期
forceFrameRate = function (fps) {
  if (fps < 0 || fps > 125) {
    // Using console['error'] to evade Babel and ESLint
    console['error'](
      'forceFrameRate takes a positive int between 0 and 125, ' +
        'forcing frame rates higher than 125 fps is not supported'
    );
    return;
  }
  if (fps > 0) {
    yieldInterval = Math.floor(1000 / fps);
  } else {
    // reset the framerate
    yieldInterval = 5;
  }
};
```

![](https://7kms.github.io/react-illustration-series/static/reactfiberworkloop.59263ed5.png)
