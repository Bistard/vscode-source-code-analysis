---
layout: post
title:  "Introduction to Event"
categories: [VSCode, Fundamentals]
---

# Background
In the context of a large-scale software application, determining the method of communication across different components is extremely crucial. As a result, VSCode's team choose **Event-Driven architecture (EDA)** as their foundation for communication.

# What is Event-Driven Architecture (EDA)
> An event is not a command, it is a change in state/action.

Event-Driven Architecture (EDA) typically consists of three characteristics:
* **Event producer** - Who produces event(s) and transfers to the event manager.
* **Event manager** - An intermediate, who is responsible for receiving event(s) and broadcasting to the event consumers.
* **Event consumer** - Who consumes the event(s), upon receiving an event, the event consumer will perform a corresponding action or reaction.

<div align="center">
  <img src="https://github.com/Bistard/vscode-source-code-analysis/assets/38385498/54cec28d-dae0-4e2e-bf8a-7966e51b3339" alt="image" />
</div>

There also are some good reasons to use EDA instead of others.
* **Loose Coupling** - Producers and consumers are unaware of each other due to the presence of an intermediate.
  * However, a minor downside to this loose coupling is that it becomes difficult to see the system's overall architecture and dependency relationships purely through the code.
* **Extensibility**: Because of this zero coupling, we can easily add or remove producers or consumers on either side of the intermediate to meet new business requirements without affecting other components.
* **Asynchronous Events**: EDA (Event-Driven Architecture) can implement asynchronicity quite naturally.

> * Event-Driven Architecture (EDA) is a concept rather than a concrete implementation. For instance, we could use callback to implement the event consumer (a.k.a listener),  we could also use message queue (MQ) to implement it.
> * In VSCode's cases, I will introduce how they used a callback to implement their `Emitter` class and achieve EDA in an overall perspective.

# How to Design an Event Manager?
With a solid understanding of what an Event-Driven Architecture (EDA) looks like, it's time to dive into the nuts and bolts of its implementation, particularly focusing on the design of the event manager.

This component can be designed in various ways, and I'll discuss two potential patterns that you might find interesting and useful. They are: 
1. the centralized event dispatcher and 
2. the single-responsibility event emitter.

> P.S. **The above names I used are made up by me**; I'm not certain if there are any official terminologies for these concepts.

> We used the second method to achieve EDA. However, I introduce two different implementations just for you to have a better understanding on the overall perspective.

## Centralized Event Dispatcher
Starting with the centralized event dispatcher, as the name suggests, is a universal hub that manages all the events in the system. It works as a central dispatcher, capable of receiving any event from event producers and forwarding it to the appropriate event consumers. The API of a centralized event dispatcher might look like the following:

```ts
const dispatcher = new Dispatcher();

// producer.ts
button.addEventListener('click', (event: MouseEvent) => {
    dispatcher.dispatch('mouse-click-event', event);
});

// consumer1.ts
dispatcher.listenTo('mouse-click-event', (event: MouseEvent) => {
    // TODO
});

// consumer2.ts
dispatcher.listenTo('mouse-click-event', (event: MouseEvent) => {
    // TODO
});
```
* Given that there's a single event dispatcher, as a consumer, it's necessary for me to designate the event's name accurately to ensure correct listening.
* One of the primary advantages of a global event manager is its simplicity. With a single point of contact for all events, the management of events becomes relatively straightforward. It reduces the complexity of having multiple event managers, and it can offer an easy way to observe and debug event activity throughout the entire system.

> Side Note: In Electron, the communication between the main process and renderer processes is designed in a centralized event dispatcher way. They use global variables like `ipcRenderer` and `ipcMain` to act as a event dispatcher.

## Single-Responsibility Event Emitter - `Emitter`

Now, let's look at the second pattern: the single-responsibility event emitter. This approach divides the responsibility of event management among multiple event emitters, **each responsible for a specific type of event**. The API of a single-responsibility event emitter might look like the following:
```ts
const emitter = new Emitter<MouseEvent>();

// producer.ts
button.addEventListener('click', (event: MouseEvent) => {
    emitter.fire<MouseEvent>(event);
});

// consumer1.ts
emitter.on((event: MouseEvent) => {
    // TODO
});

// consumer2.ts
emitter.listenTo((event: MouseEvent) => {
    // TODO
});
```
* Thanks to the idea of single-responsibility, consumers don't have to specify the event name. This is because the event emitter itself symbolizes a particular type of event.
* The primary advantage here is the separation of concerns. Each event emitter handles a specific type of event, making the system more organized and easier to maintain. It can be an effective way to manage complex systems with various types of events, and it reduces the risk of a single point of failure, as seen in the global event manager approach.

> This approach is also what VSCode's team primarily adopted throughout the software development process.

# `Emitter<T>` Class in VSCode
The `Emitter<T>` class plays a vital component in our Event-Driven Architecture (EDA):
* It works as a single-responsibility event emitter, of a particular event with generic type `T`, notifying all registered listeners when the event occurs.

Here is its TypeScript interface:
```ts
export interface IEmitter<T> {
    event: Event<T>;
    fire(event: T): void;
    hasListeners(): boolean;
    dispose(): void;
}
```

* Since there are no restrictions on the generic type `T`, it means it could be interpreted into different means depending on the type `T`. 
  * When `T` is defined as `void`, this emitter merely signals the occurrence of the event, without conveying any additional information.
  * If `T` is a `boolean`, the emitter essentially behaves like a switch, which only triggers a `true` or `false` value.
  * when `T` is an `object` type, the emitter is capable of sending supplementary metadata to the consumers.
* When you want to respond/listen/consume to the event `T`, you can use the `event` method of `Emitter` to register a callback function: `emitter.event(CALLBACK)`.
* To trigger the event and notify all the listeners, the `fire` method is used. This method takes an event of type `T` and notifies all the listeners about it: `emitter.fire(YOUR_EVENT)`.

> However, I personally would refactor its interface as:
> ```ts
> export interface IEmitter<T> {
>     registerListener: Register<T>;
> }
> ```
> So that its API would be look more nicely:
> ```ts
> emitter.registerListener(CALLBACK);
> ```

> The implementation of `Emitter` is not hard. It wraps over a linked list to store all the callback functions. Once it is fired, it simply iterate all the callbacks and invoke them one by one. The discussed codes are located at `event.ts`. There is a important class named `Emitter`.

# Application of `Emitter`
Recall from before, that VSCode are using single-responsibility event emitters across the entirely codebase:
```ts
// construct a emitter that fires a boolean event.
const onMouseClick = new Emitter<boolean>();

// register a callback to listen to the events
const onMouseClick.event((event: boolean) => {
    console.log('on mouse click:', event);
});

// deconstruct the emitter
onMouseClick.dispose();
```

We can assembly the `Emitters` inside our classes to simplify the registering process. Here is an example of `Sash` class:
```ts
export class Sash extends Disposable implements ISash {

    /** An event which fires whenever the user starts dragging the sash. */
      private readonly _onDidStart = this.__register(new Emitter<ISashEvent>());
    public readonly onDidStart: Event<ISashEvent> = this._onDidStart.event;

    /** An event which fires whenever the user moves the mouse while dragging the sash. */
    private readonly _onDidMove = this.__register(new Emitter<ISashEvent>());
    public readonly onDidMove: Event<ISashEvent> = this._onDidMove.event;

    /** An event which fires whenever the user stops dragging the sash. */
    private readonly _onDidEnd = this.__register(new Emitter<void>());
    public readonly onDidEnd: Event<void> = this._onDidEnd.event;

    /** An event which fires whenever the user double clicks the sash. */
    private readonly _onDidReset = this.__register(new Emitter<void>());
    public readonly onDidReset: Event<void> = this._onDidReset.event;
}
```
* `_onDidStart` (note it is a private field) is the actual `Emitter` that when a event is detected (e.g. A click action is captured). 
  * `Emitter` tells every consumer (or listeners) that registered to it. Since we know the listeners are just callback functions, telling simply means invoking those callback functions. 
* `onDidStart` is an open public registrant that registers listeners to the emitter.

**By doing this we can achieve the following coding convention for simplicity**:
```ts
const sash = new Sash(/** */);
sash.onDidStart((event: ISashEvent) => {
    console.log('event triggers');
});

// better than:
// sash.onDidStart.event(() => {});
```

# Wrapping Up
There's no definitive answer to whether one should opt for a centralized event dispatcher or a single-responsibility event emitter. Both approaches have their unique strengths and potential drawbacks. The decision ultimately depends on your specific needs and requirements.

