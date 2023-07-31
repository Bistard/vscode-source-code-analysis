---
layout: post
title:  "Resource Management - Dispose Pattern"
categories: [VSCode, Fundamentals]
---

# Background

JavaScript utilizes Garbage Collection (GC) to manage memory. As a result, we don't have direct control over resource management—**we can't tell when a resource is actually released once the reference count of a variable drops to zero**. 

However, we can simulate this process (this is also what VSCode does):
> What is a **resource** then? A resource could be file handles, network sockets, or database connections, as well as more direct allocations of memory (number, string, boolean, arrays, objects, etc.).

Let us consider three examples to illustrate your bad resource (or memory) management:

## Example 1 - Manually Releasing Memory

```ts
// example 1

class MyArray {

    public readonly arr: object[] = [];

    constructor() {}

    public push(obj: object): void {
        this.arr.push(obj);
    }
}

let internalArr: object[];
{
    const myArr = new MyArray();
    for (let i = 0; i < 1000000; i++) {
        myArr.push({ number: i });
    }
    internalArr = myArr.arr;
} // scope1

console.log(internalArr.length); // 1000000
```
* When the program reaches the end of `scope1`, the reference count of `myArr` variable will reach zero, but the internal data (`arr`) will not reach a reference count of zero, because it is referenced outside of `scope1`. Thus `arr` cannot be garbage collected immediately.
* While this code does not immediately cause a memory leak, it can potentially lead to one if not handled carefully.

We can solve this problem by using the following solution:
```ts
// example 1 - potential solution
class MyArray {

    public arr: object[] = [];

    constructor() {}

    public push(obj: object): void {
        this.arr.push(obj);
    }

    public destruct(): void {
        this.arr = [];
    }
}

let internalArr: object[];
{
    const myArr = new MyArray();
    for (let i = 0; i < 1000000; i++) {
        myArr.push({ number: i });
    }
    internalArr = myArr.arr;
    myArr.destruct();
}

console.log(internalArr.length); // 0
```
* In the above case, we manually clean the resources held by `MyClass`. We destroyed `myArr` **before it goes out of scope**.
  * The array data `arr` will be garbage collected even if the reference count of itself is still not zero.
  * Remember, even if we solved one problem, this overall design is still not recommended—If the data is designed for internal usage, **it should be private**. If the data is designed for public usage, it should be carefully treated in all cases to avoid holding it too long.

## Example 2 - Destructed Object Should No Longer Being Used
Let’s continue with our previous example, consider we designed `MyArray` to be not valid after `destruct` is invoked:

```ts
// example 2
{
    const myArr = new MyArray();
    for (let i = 0; i < 1000000; i++) {
        myArr.push({ number: i });
    }
    myArr.destruct();
    myArr.push({ number: -1 }); // should be invalid, but it works here.
}
```
* To solve such a potential issue, we can do the following by adding a simple flag for runtime checking:

```ts
// example 2 - solution
class MyArray {

    public arr: object[] = [];
    private _destructed = false;

    constructor() {}

    public push(obj: object): void {
        if (this._destructed) {
            throw new Error('MyArray is destructed, push is no longer used.');
        }
        this.arr.push(obj);
    }

    public destruct(): void {
        this.arr = [];
        this._destructed = true;
    }
}
```

## Example 3 - Release Complicated Resources
```ts
// example 3

import fs from 'fs';

class FileHandler {
    private _filePath: string;
    private _fileDescriptor?: fs.promises.FileHandle;

    constructor(filePath: string) {
        this._filePath = filePath;
        this._fileDescriptor = undefined;
    }

    async open(): Promise<void> {
        this._fileDescriptor = await fs.promises.open(this._filePath, 'r');
    }

    async read(): Promise<string> {
        if (!this._fileDescriptor) {
            throw new Error('File not open');
        }

        const buffer = Buffer.alloc(1024);
        const { bytesRead } = await this._fileDescriptor.read(buffer, 0, buffer.length, 0);

        return buffer.toString('utf-8', 0, bytesRead);
    }

    async close(): Promise<void> {
        if (this._fileDescriptor) {
            await this._fileDescriptor.close();
            this._fileDescriptor = undefined;
        }
    }
}

async function main() {
    const handler = new FileHandler('path/to/your/file');
    await handler.open();
    const data = await handler.read();
    console.log(data);
    
    // await handler.close(); ❌error happens
    
    // Here, the FileHandler object goes out of scope. 
    // If we hadn't manually closed the file, a file descriptor leak could occur.
    // In this case, the garbage collector cannot reclaim the file descriptor resource 
    // because the operating system controls that, not the JavaScript runtime.
}

main();
```
* The `FileHandler.close()` method acts similarly to `MyArray.destruct()` - release resources that holding by themselves.

> From the perspective of a large-scale desktop application, it's crucial to know which resources have been released or are unused, and which are still active. Resource management is always a challenge in a programmer's life.

# Introduction to Dispose Pattern
Luckily, we can improve this situation by leveraging a design pattern called the **Dispose Pattern**. This pattern allows us to explicitly manage resource disposal, adding a layer of control to help manage memory more efficiently.

> The Dispose Pattern is a design pattern often used in programming languages that explicitly manage memory, such as C#. 
> 
> However, it can also be beneficial in languages with garbage collection, like JavaScript, particularly when dealing with resources other than just memory, such as file handles, network sockets, or database connections. 
> 
> You may found out that this pattern is used everywhere across entirely in VSCode.

> All the related codes can be found at `dispose.ts`.

## `IDisposable` Interface
In VScode's repository, they provide an interface `IDisposable` that includes a `dispose` method. This `dispose` method should be responsible for cleaning up any resources (or just direct memory) the object uses when it's no longer needed.

Let's have a look at a basic example:
```ts
interface IDisposable {
    dispose(): void;
}

class Resource implements IDisposable {
    private resourceHandle: any; // This could be a file handle, database connection, etc.

    constructor() {
        this.resourceHandle = this.acquireResource();
    }

    private acquireResource(): any {
        // Logic to acquire the resource
    }

    dispose(): void {
        // Logic to release or clean up the resource
        this.resourceHandle = null;
    }
}

// Using the Resource class
const resource = new Resource();
// Use the resource...
resource.dispose(); // Don't forget to dispose of it when done
```
* In the above example, the `Resource` class implements the `IDisposable` interface. The `dispose` method in the `Resource` class is responsible for cleaning up or releasing the resource. The consumer of the `Resource` class is responsible for calling `dispose` when they're finished using the resource.

> This pattern provides a way to explicitly manage resources, allowing you to free up resources as soon as they're no longer needed, rather than waiting for the garbage collector to do it at some undetermined time in the future. It can be particularly useful in scenarios where resources are scarce or expensive, or where the timing of garbage collection could negatively impact performance.

> **This design pattern is not as complicated as imagined. If you are familiar with C++, then essentially, the role of the dispose function is similar to the destructor function.**

> `dispose()` method is just a universal name for methods like `MyArray.destruct()` or `FileHandler.close()`.

## Utility Tool - `Disposable` Class
> The related code can be found at `lifecycle.ts` and `lifecycle.test.ts`.

In addition, VSCode also provide a simple but powerful base class named `Disposable`. Here is the code:
```ts
export interface IDisposable {
	dispose(): void;
}

export abstract class Disposable implements IDisposable {

	/**
	 * A disposable that does nothing when it is disposed of.
	 *
	 * TODO: This should not be a static property.
	 */
	static readonly None = Object.freeze<IDisposable>({ dispose() { } });

	protected readonly _store = new DisposableStore();

	constructor() {
		trackDisposable(this);
		setParentOfDisposable(this._store, this);
	}

	public dispose(): void {
		markAsDisposed(this);

		this._store.dispose();
	}

	/**
	 * Adds `o` to the collection of disposables managed by this object.
	 */
	protected _register<T extends IDisposable>(o: T): T {
		if ((o as unknown as Disposable) === this) {
			throw new Error('Cannot register a disposable on itself!');
		}
		return this._store.add(o);
	}
}
```

> For a better understanding, I highly recommend combining this blog with the `lifecycle.test.ts` which is a unit test file.

Here is one of the unit test:
```ts
test('dispose recursively', () => {
	const mainDisposable = new DisposableManager();
	
	const disposable2 = new Disposable();
	const disposable3 = new DisposableManager();

	mainDisposable.register(disposable2);
	mainDisposable.register(disposable3);

	const disposable4 = new Disposable();
	disposable3.register(disposable4);

	mainDisposable.dispose();

	assert.ok(mainDisposable.disposed);
	assert.ok(disposable2.isDisposed());
	assert.ok(disposable3.disposed);
	assert.ok(disposable4.isDisposed());
});
```

> In practice, how do we make use of the `Disposable`?
>
> * When a microservice holds a certain amount of resources, it can extend the base `Disposable` class and use the protected `__register` method to register all its resources.
>
> * As a client of that microservice, all one needs to do is call the `this.dispose()` method to release all the resources it has acquired. 
> 	* This practice is efficient and effective, simplifying the resource management process and preventing resource leaks, which could otherwise hamper the performance of our application. 
>
> * This is why we consider the Dispose Pattern to be an invaluable tool for resource and memory management.










