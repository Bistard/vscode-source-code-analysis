---
layout: post
title:  "VSCode：Centralized Animation Frame Scheduling."
categories: [VSCode, UI]
---

## 简介
今天介绍一个VSCode源码中一个简单且小巧的功能。在此之前我先简单介绍一下`requestAnimationFrame`这个原生API。

## `requestAnimationFrame` API

### 工作原理
`requestAnimationFrame` 会将提供的回调函数放入队列，并在下一次浏览器重绘前调用该函数。与传统的 `setTimeout` 不同，`requestAnimationFrame` 的优势在于：
- **与屏幕刷新同步**：浏览器会在适当的时间调用回调函数，通常是每秒 60 帧（即 16.67ms 间隔）。
- **节能效果**：当页面处于后台或不可见状态时，浏览器会暂停 `requestAnimationFrame` 的调用，从而节省资源。
- **平滑的动画**：由于与浏览器刷新周期一致，动画会显得更加流畅。

### 基本用法
```javascript
function animate() {
    console.log('Animating...');
}
requestAnimationFrame(animate);
```
### 问题点
当应用中的多个模块独立调用 `requestAnimationFrame` 时，可能出现以下问题：
1. **缺乏全局优先级控制**：浏览器无法直接区分任务的优先级，导致关键任务与次要任务并行执行，影响性能。
2. **复杂任务调度**：对于需要动态更新任务优先级或取消任务的场景，原生 API 支持不足。

为了解决这些问题，VSCode 设计了一套中央式动画帧调度器来管理动画和 UI 渲染任务。

## 中央调度器
> 相关文件：`src\vs\base\browser\dom.ts`

`VSCode`整个软件中避免直接使用`window.requestsAnimationFrame`，而是提供了以下两个APIs：
```ts
/**
 * Schedule a callback to be run at the next animation frame.
 * This allows multiple parties to register callbacks that should run at the next animation frame.
 * If currently in an animation frame, `runner` will be executed immediately.
 * @return token that can be used to cancel the scheduled runner (only if `runner` was not executed immediately).
 */
export let runAtThisOrScheduleAtNextAnimationFrame: (targetWindow: Window, runner: () => void, priority?: number) => IDisposable;
/**
 * Schedule a callback to be run at the next animation frame.
 * This allows multiple parties to register callbacks that should run at the next animation frame.
 * If currently in an animation frame, `runner` will be executed at the next animation frame.
 * @return token that can be used to cancel the scheduled runner.
 */
export let scheduleAtNextAnimationFrame: (targetWindow: Window, runner: () => void, priority?: number) => IDisposable;
```

VSCode把函数的定义写在了一个 immediate call function 里面。因为代码量很少，我会直接把大部分代码复制过来。首先是在 function body 中定义了一些map，用来全局储存数据:
```ts
(function () {
    // The runners scheduled at the next animation frame
    const NEXT_QUEUE = new Map<number /* window ID */, AnimationFrameQueueItem[]>();
    // The runners scheduled at the current animation frame
    const CURRENT_QUEUE = new Map<number /* window ID */, AnimationFrameQueueItem[]>();
    // A flag to keep track if the native requestAnimationFrame was already called
    const animFrameRequested = new Map<number /* window ID */, boolean>();
    // A flag to indicate if currently handling a native requestAnimationFrame callback
    const inAnimationFrameRunner = new Map<number /* window ID */, boolean>();

    // ...
})();
```

而`runAtThisOrScheduleAtNextAnimationFrame`和`scheduleAtNextAnimationFrame`的定义如下：
```ts
(function () {
    // ...
    scheduleAtNextAnimationFrame = (targetWindow: Window, runner: () => void, priority: number = 0) => {
        const targetWindowId = getWindowId(targetWindow);
        const item = new AnimationFrameQueueItem(runner, priority);
        let nextQueue = NEXT_QUEUE.get(targetWindowId);
        if (!nextQueue) {
            nextQueue = [];
            NEXT_QUEUE.set(targetWindowId, nextQueue);
        }
        nextQueue.push(item);
        if (!animFrameRequested.get(targetWindowId)) {
            animFrameRequested.set(targetWindowId, true);
            targetWindow.requestAnimationFrame(() => animationFrameRunner(targetWindowId));
        }
        return item;
    };

    runAtThisOrScheduleAtNextAnimationFrame = (targetWindow: Window, runner: () => void, priority?: number) => {
        const targetWindowId = getWindowId(targetWindow);
        if (inAnimationFrameRunner.get(targetWindowId)) {
            const item = new AnimationFrameQueueItem(runner, priority);
            let currentQueue = CURRENT_QUEUE.get(targetWindowId);
            if (!currentQueue) {
                currentQueue = [];
                CURRENT_QUEUE.set(targetWindowId, currentQueue);
            }
            currentQueue.push(item);
            return item;
        } else {
            return scheduleAtNextAnimationFrame(targetWindow, runner, priority);
        }
    };
})();
```
* `AnimationFrameQueueItem` 是任务的基本单位。每个任务都被封装成一个实例，包含了以下信息：
  * **任务逻辑**（`runner`）：需要执行的具体函数。
  * **优先级**（`priority`）：用于控制任务执行的顺序。
  * **取消标志**（`_canceled`）：支持任务的动态取消。
```typescript
class AnimationFrameQueueItem {
    private _runner: () => void;
    public priority: number;
    private _canceled: boolean;

    constructor(runner: () => void, priority: number = 0) {
        this._runner = runner;
        this.priority = priority;
        this._canceled = false;
    }

    dispose(): void {
        this._canceled = true;
    }

    execute(): void {
        if (this._canceled) return;
        this._runner();
    }

    static sort(a: AnimationFrameQueueItem, b: AnimationFrameQueueItem): number {
        return b.priority - a.priority;
    }
}
```
调度的核心逻辑是 `animationFrameRunner` 函数，它通过 `requestAnimationFrame` 在每帧执行任务：
1. 从 `NEXT_QUEUE` 中提取任务到 `CURRENT_QUEUE`。
2. 对 `CURRENT_QUEUE` 按优先级排序。
3. 按顺序依次执行任务。
```typescript
(function () {
    // ...
    const animationFrameRunner = (targetWindowId: number) => {
    animFrameRequested.set(targetWindowId, false);

    const currentQueue = NEXT_QUEUE.get(targetWindowId) ?? [];
    CURRENT_QUEUE.set(targetWindowId, currentQueue);
    NEXT_QUEUE.set(targetWindowId, []);

    while (currentQueue.length > 0) {
        currentQueue.sort(AnimationFrameQueueItem.sort);
        const top = currentQueue.shift()!;
        top.execute(); // actual animation execution
    }
    // ...
};
})();
```

## 额外内容
VSCode还提供了一些简单的helper function方便做测试或者补丁等等：
```ts
export function measure(targetWindow: Window, callback: () => void): IDisposable {
    return scheduleAtNextAnimationFrame(targetWindow, callback, 10000 /* must be early */);
}

export function modify(targetWindow: Window, callback: () => void): IDisposable {
    return scheduleAtNextAnimationFrame(targetWindow, callback, -10000 /* must be late */);
}
```