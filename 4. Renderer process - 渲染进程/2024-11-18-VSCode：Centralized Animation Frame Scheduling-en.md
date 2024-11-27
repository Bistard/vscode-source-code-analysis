---
layout: post
title: "VSCode: Centralized Animation Frame Scheduling"
categories: [VSCode, UI]
---

## Introduction
Today, we'll explore a simple and elegant feature within the VSCode source code. Before diving in, let’s first look at the native `requestAnimationFrame` API.

## `requestAnimationFrame` API

### How It Works
`requestAnimationFrame` queues a callback to be invoked before the next browser repaint. Compared to `setTimeout`, it offers several advantages:
- **Synchronization with screen refresh**: The callback is invoked at an appropriate time, typically 60 frames per second (16.67ms interval).
- **Energy efficiency**: When the page is in the background or hidden, the browser halts `requestAnimationFrame` calls to conserve resources.
- **Smooth animations**: Animations appear smoother due to synchronization with the browser’s refresh cycle.

### Basic Usage
```javascript
function animate() {
    console.log('Animating...');
}
requestAnimationFrame(animate);
```

### Issues
When multiple modules independently call `requestAnimationFrame`, issues can arise:
1. **Lack of global priority control**: The browser cannot distinguish between critical and minor tasks, leading to potential performance bottlenecks.
2. **Complex task scheduling**: The native API offers limited support for dynamically updating priorities or canceling tasks.

To address these challenges, VSCode employs a centralized animation frame scheduler for managing animation and UI rendering tasks.

## Centralized Scheduler
> Relevant File: `src\vs\base\browser\dom.ts`

VSCode avoids directly using `window.requestAnimationFrame`. Instead, it provides the following two APIs:
```typescript
/**
 * Schedule a callback to be run at the next animation frame.
 * Allows multiple parties to register callbacks to run at the next animation frame.
 * If currently in an animation frame, `runner` executes immediately.
 * @return token to cancel the scheduled runner (only if `runner` was not executed immediately).
 */
export let runAtThisOrScheduleAtNextAnimationFrame: (targetWindow: Window, runner: () => void, priority?: number) => IDisposable;

/**
 * Schedule a callback to be run at the next animation frame.
 * Allows multiple parties to register callbacks to run at the next animation frame.
 * If currently in an animation frame, `runner` executes at the next animation frame.
 * @return token to cancel the scheduled runner.
 */
export let scheduleAtNextAnimationFrame: (targetWindow: Window, runner: () => void, priority?: number) => IDisposable;
```

These functions are defined within an immediately invoked function expression (IIFE) for encapsulation. Here’s a simplified breakdown of the code:

### Data Structures
Maps are used to store global data:
```typescript
(function () {
    const NEXT_QUEUE = new Map<number, AnimationFrameQueueItem[]>();
    const CURRENT_QUEUE = new Map<number, AnimationFrameQueueItem[]>();
    const animFrameRequested = new Map<number, boolean>();
    const inAnimationFrameRunner = new Map<number, boolean>();
    // ...
})();
```

### API Implementations
#### `scheduleAtNextAnimationFrame`
Schedules tasks for the next animation frame:
```typescript
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
})();
```

#### `runAtThisOrScheduleAtNextAnimationFrame`
Runs the task immediately if within an animation frame, otherwise schedules it for the next frame:
```typescript
(function () {
    // ...
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

### Task Representation
Each task is encapsulated in an `AnimationFrameQueueItem`:
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

### Core Scheduling Logic
The `animationFrameRunner` function manages task execution:
1. Transfers tasks from `NEXT_QUEUE` to `CURRENT_QUEUE`.
2. Sorts tasks in `CURRENT_QUEUE` by priority.
3. Executes tasks sequentially:
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
            top.execute();
        }
        // ...
    };
})();
```

## Additional Utilities
VSCode provides helper functions for early or late scheduling:
```typescript
export function measure(targetWindow: Window, callback: () => void): IDisposable {
    return scheduleAtNextAnimationFrame(targetWindow, callback, 10000 /* must be early */);
}

export function modify(targetWindow: Window, callback: () => void): IDisposable {
    return scheduleAtNextAnimationFrame(targetWindow, callback, -10000 /* must be late */);
}
```