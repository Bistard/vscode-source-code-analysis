---
layout: post
title:  "微服务架构-Microservice-Driven Architecture"
categories: [VSCode, Fundamentals]
---

# What is a Microservice?
![image](https://github.com/Bistard/vscode-source-code-analysis/assets/38385498/3b1bb539-ad13-42f0-8cf1-66a52afdc6a4)

Microservice-driven architecture, or simply microservices, is an architectural style that structures an application as a collection of loosely coupled services. 

**Each of these services is designed to execute a specific process or business capability, making them easy to understand, develop, and maintain.**

For instance: 
* `FileService` - This service is responsible for handling all file-related operations, such as file creation, reading, updating, and deletion.
* `ConfigurationService` - This service is tasked with managing all the configuration settings of an application.
* `LoggerService` - It captures and stores logs, which are records of events or transactions taking place within the application, typically used for debugging and monitoring purposes.
* `ProcessService` - The ProcessService is involved with the execution and management of application processes.
* `LifecycleService` - It manages the various stages an application or component goes through from initialization to termination.
* `WindowService` - It handles operations such as opening, closing, resizing, and moving windows.

# Microservices and Dependency Injection (DI)
MDA can perfectly work with the idea of Dependency Injection (DI) to achieve a much less coupled system from an overall perspective. In order to understand the concepts in depth, I will show some real applications of what we can benefit from those two concepts.

For instance, consider the following class:
```ts
/**
 * @class The main class of the application. It handles the core business of the 
 * application.
 */
export class ApplicationInstance extends Disposable implements INotaInstance {

    constructor(
        @IInstantiationService private readonly mainInstantiationService: IInstantiationService,
        @IEnvironmentService private readonly environmentService: IMainEnvironmentService,
        @IMainLifecycleService private readonly lifecycleService: IMainLifecycleService,
        @ILogService private readonly logService: ILogService,
        @IFileService private readonly fileService: IFileService,
        @IMainStatusService private readonly statusService: IMainStatusService,
    ) {
        super();
        // ...
    }

    // ...
}
```
* The `ApplicationInstance` depends on all sorts of microservices in abstraction (in our context, depends on interfaces). `ApplicationInstance` is not responsible for creating those microservices (less coupled systems).
* Those prefixes in front of every field like `@ILogService` are known as decorators. These are language-specific techniques used to achieve DI in TypeScript. We will talk about this technique later.

## Microservices Depends on Microservices
Similarly, microservices are not special classes, they can also depend on other microservices as well. As an example, the following `MainWindowService` also depends on all sorts of different microservices:
```ts
class MainWindowService implements IMainWindowService {
    constructor(
        @IInstantiationService private readonly instantiationService: IInstantiationService,
        @ILogService private readonly logService: ILogService,
        @IFileService private readonly fileService: IFileService,
        @IMainLifeCycleService private readonly lifeCycleService: IMainLifeCycleService,
        @IEnvironmentService private readonly mainEnvironmentService: IMainEnvironmentService,
    ) {
        // ...
    }
}
```
From a framework perspective, I can construct `MainWindowService` as follows:
* Regardless of the specific microservice instance passed through the framework, as long as that instance implements the corresponding interface, the system will function seamlessly.
  * For instance, as long as `FileService`, `TestFileService`, and `NullFileService` have all implemented the interface `IFileService`, the system will operate without issues.
```ts
// normal case
const mainWindowService = new MainWindowService(
    new InstantiationService(),
    new LogService(),
    new FileService(),
    new MainLifeCycleService(),
    new MainEnvironmentService(),
);

// unit test case
const mainWindowService = new MainWindowService(
    new TestInstantiationService(),
    new TestLogService(),
    new TestFileService(),
    new TestMainLifeCycleService(),
    new TestMainEnvironmentService(),
);

// null case
const mainWindowService = new MainWindowService(
    new NullInstantiationService(),
    new NullLogService(),
    new NullFileService(),
    new NullMainLifeCycleService(),
    new NullMainEnvironmentService(),
);
```

# Godfather of All Microservices
Let's address some potential questions that might arise so far:
* What happens when I've instantiated a `FileService` somewhere in my program, and another microservice, that depends on a `FileService`, needs to be constructed? From where does the new microservice obtain the previously constructed FileService?
* Does one have to **manually** construct all microservices, or is there a more efficient way to **automate** the construction process?
* Moreover, how do those decorators work, which we've previously seen and which are placed before each constructor parameter?
> All these questions find their answers in a sophisticated solution, a class called `InstantiationService`.
`InstantiationService` lies at the heart of the microservice ecosystem. It's responsible for managing the lifecycle of various services in the system.
* The primary task of `InstantiationService` is to create instances of services, injecting any dependencies as needed.
* `InstantiationService` is a concrete solution to achieve the **Dependency Injection (DI) principle**.

To illustrate how `InstantiationService` operates, consider the following API example:
```ts
// initialization (acting as DI)
const instantiationService = new InstantiationService();

// register a dependency (microservice) into DI (we will talk about it later)
instantiationService.register(IFileService, new FileService());

// create the service by its corresponding dependency tree (how it works?)
const mainWindowService = instantiationService.createInstance(MainWindowService);
```
* The above example provokes some thoughts:
  * First, what is a dependency tree of a microservice?
  * Second, how do we determine the dependency tree of each microservice?
  * Third, how do we create a microservice by its corresponding dependency tree?
Don’t worry, I will explain these concepts to you step by step.

## 1. What Is a Dependency Tree?
A dependency tree represents a hierarchical relationship between different modules (or classes). Recall the previous example, `ApplicationInstance`, its dependency tree would look like the following:
```
ApplicationInstance
├─ IInstantiationService
├─ IEnvironmentService
├─ IMainLifecycleService
├─ ILogService
├─ IFileService
└─ IMainStatusService
    ├─ IInstantiationService
    ├─ ILogService
    ├─ IFileService
    ├─ IMainLifecycleService
    └─ IEnvironmentService
```
## 2. How Do We Determine a Dependency Tree?
Following the earlier example, where `MainWindowService` depends on `IFileService`. We can draw a few conclusions about `IFileService`:
* `IFileService` serves as an abstraction. `IFileService` as an interface, is a syntax sugar from TypeScript, which only exists on compile-time.
* `IFileService` can be implemented as concrete classes, such as `FileService`, `TestFileService` or `NullTestService`, and so forth.

> * Since the interface is just a syntax sugar, it will be removed once compiled. **We need a runtime solution to identify the existence of `IFileService`**. Otherwise, the `InstantiationService` will not be able to establish the connection between `IFileService` and `FileService`/`TestFileService`/`NullFileService`, which is essential for constructing a `MainWindowService` based on `IFileService`.
> * Luckily we can do that by using decorators. Let me introduce `createDecorator` decorator.

### 2.1 Runtime Identifier for Every Microservice - `createDecorator` Decorator
VSCode provide a useful function, `createDecorator`, which creates a unique decorator that can be served as an identifier for each microservice.

> Sidenote: In TypeScript, a decorator is essentially a function.

* The decorator created by `createDecorator`, acts like an identifier, which can be stored in DI to make a connection between the microservice and the concrete class implementation as follows:
```ts
// fileService.ts
const IFileService = createDecorator('file-service'); // note: a variable in TypeScript can have the same name as an interface.

export interface IFileService {
    // ...
}

export class FileService implements IFileService {
    // ...
}

// playground.ts
const instantiationService = new InstantiationService();

// DI registration (⭐)
instantiationService.register(IFileService, new FileService()); // we've seen this line of code before
```
The decorator performs two functionalities in our cases:
1. First, since the decorator is a variable, thus it exists in run-time, it can be used in our DI system (`InstantiationService`). As previously mentioned, It establishes a connection between the abstraction concept (`IFileService`) and a concrete implementation (our case is `FileService`), as we've just done in the above example.
   * At line 16, the DI system now recognizes an abstraction (`IFileService`), and a way to construct its corresponding class (`FileService`).
2. Second, this decorator also helps us to construct a dependency tree at runtime, which leads us to the next sub-topic.

> Variables and interfaces can have the same names. In this case, we have a variable and an interface both named `IFileService`.

> Sidenote: Decorators will be immediately executed once the JavaScript script is loaded.

### 2.2 Build a Dependency Tree at Runtime Using Decorator
Let’s see what can this decorator be used for:
```ts
class MainWindowService implements IMainWindowService {
    constructor(
        @IInstantiationService private readonly instantiationService: IInstantiationService,
        @ILogService private readonly logService: ILogService,
        @IFileService private readonly fileService: IFileService,
        @IMainLifecycleService private readonly lifecycleService: IMainLifecycleService,
        @IEnvironmentService private readonly mainEnvironmentService: IMainEnvironmentService,
    ) {
        // ...
    }
}
```
A decorator, as the name tells, is used to add something extra to a class parameter. It does one key job:
* It marks the decorated class (in this case, `MainWindowService`), which depends on the parameter (e.g. `IFileService`), at runtime.

> You do not need to worry about the black magic behind the decorators created by createDecorator. It works and works elgantly, I can promise you.

> To give you a better illustration of what decorators do, let us consider the following class:
> ```ts
> class TestService {
>     constructor() {}
> }
> ```
> Currently, the class does not depend on anything. We can use an imagined function `getDependencyTreeFor`:
> ```ts
> const dependencies = getDependencyTreeFor(TestService);
> console.log(dependencies);
> // []
> ```
> Now, we let `TestService` depends on two other services:
> ```ts
> class TestService{
>     constructor(
>         @IService1 private readonly service1: IService1,
>         @IService2 private readonly service2: IService2,
>     ) {}
> }
> ```
> If we print out its dependency tree again, we will see different things:
> ```ts
> const dependencies = getDependencyTreeFor(TestService);
> console.log(dependencies);
> /**
> [
>   {
>     id: [Function: serviceIdentifier] {
>       _: undefined,
>       toString: [Function (anonymous)]
>     },
>     index: 1,
>     optional: false
>   },
>   {
>     id: [Function: serviceIdentifier] {
>       _: undefined,
>       toString: [Function (anonymous)]
>     },
>     index: 0,
>     optional: false
>   }
> ]
>  */
> ```
> **In essence, the decorator's job is to create and store the above dependency tree at runtime.**

Continuing our example, the complete dependency tree of `MainWindowService` will look like the following:
```
MainWindowService
└─ IInstantiationService
└─ ILogService
└─ IFileService
└─ IMainLifecycleService
└─ IMainEnvironmentService
```
Furthermore, the dependency tree can be more than just one level. The following dependency tree is also valid:
```
 MainWindowService
└─ IInstantiationService
└─ ILogService
└─ IFileService
└─ IMainLifecycleService
    └─ ILogService
└─ IMainEnvironmentService
```
> However, the DI system will throw a runtime error when encounters a cyclic dependency. That is, encountering A depends on B, B depends on C, and C also depends on A.

## 3. How Do We Create a Microservice?
Now, we already covered a way to create a dependency tree for our classes. We finally reach a stage to discuss how we create our microservices.

### 3.1 Register a Microservice
For every dependency tree, its tree leaf (the dependency that depends on nothing) cannot be constructed automatically. **It means the leaf must be provided by ourselves**. We call this the registration process.

It is quite simple, recall our previous example:
```ts
// initialization (acting as DI)
const instantiationService = new InstantiationService();

// `FileService` does not depend on anything
// register a dependency (microservice) into DI (we just covered)
instantiationService.register(IFileService, new FileService());

// register other dependencies...
```

> Of course, the other tree nodes other than just leafs can also be registered optionally.

### 3.2 Assembly Factory - `InstantiationService`
Here comes the final step of constructing a microservice: **assembling**. This is done inside the method `createInstance`:
```ts
// construct the service (by its corresponding dependency tree)
const mainWindowService = instantiationService.createInstance(MainWindowService);
```

When the `InstantiationService` (the DI System) is constructing a microservice, it will look for its dependency tree, iterate each dependency, and try to perform the following two operations to that dependency:

1. If the dependency already exists, meaning it has been constructed before, then the process is done and it continues to the next dependency.
2. If the dependency has not been constructed yet, the DI system recursively constructs this dependency.

After all the dependencies have been constructed, the `InstantiationService` uses them to construct the final target, the original microservice that was requested to be built.

# Lazy Loading
Recall the second step when assembling in `InstantiationService`, where it checks 'if the dependency has not been constructed yet'. 

One might wonder how a microservice can be registered into the DI system without actually constructing one. For that, VSCode introduce a utility tool named `SyncDescriptor`:
```ts
export class SyncDescriptor<T> {

	readonly ctor: any;
	readonly staticArguments: any[];
	readonly supportsDelayedInstantiation: boolean;

	constructor(ctor: new (...args: any[]) => T, staticArguments: any[] = [], supportsDelayedInstantiation: boolean = false) {
		this.ctor = ctor;
		this.staticArguments = staticArguments;
		this.supportsDelayedInstantiation = supportsDelayedInstantiation;
	}
}
```
With the help of `SyncDescriptor`, we can achieve lazy loading when registering a microservice:
```ts
// initialization (acting as DI)
const instantiationService = new InstantiationService();

// register a dependency into DI using lazy loading
instantiationService.register(IFileService, new SyncDescriptor(FileService));
```

> * When registering a microservice using `SyncDescriptor`, known as lazy loading technique, it ensures that the microservice is only constructed when it is actually needed. 
> * This strategy can significantly improve the efficiency and scalability of an application.








