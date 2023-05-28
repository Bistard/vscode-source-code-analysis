# 内存管理/资源管理 - Memory Management
因为JavaScript/TypeScript用的是Garabage Collection，所以做不到手动释放内存。不过由于VSCode整个框架之庞大，所以它们在整个程序中采用了一个设计模式叫做**Dispose Pattern**.

# Dispose Pattern
相关代码位于`vscode\src\vs\base\common\lifecycle.ts`。

这个模式非常简单，在VSCode中，这个文件比较重要的东西有两个，一个是`IDisposable`接口和`Disposable`抽象类。

## `IDisposable`接口
```ts
/**
 * An object that performs a cleanup operation when `.dispose()` is called.
 *
 * Some examples of how disposables are used:
 *
 * - An event listener that removes itself when `.dispose()` is called.
 * - A resource such as a file system watcher that cleans up the resource when `.dispose()` is called.
 * - The return value from registering a provider. When `.dispose()` is called, the provider is unregistered.
 */
export interface IDisposable {
	dispose(): void;
}
```
在拥有GC的JavaScript中，这个接口唯一的函数`dispose()`可以意味着以下事情：
- 清除数据，
- 释放每个resource的reference count到0，


我们可以利用这个函数来模拟释放内存的过程。比如以下案例：
```ts
class FoodContainer implements IDisposable {
    
    private readonly _food = new Set<string>();
    private _childContainer: FoodContainer | null;
    private _isDisposed = false;
    
    constructor(childContainer?: FoodContainer) {
        this._childContainer = childContainer ?? null;
    }
    
    public addFood(food: string): void {
        if (this.isDisposed()) {
            throw new Error('Cannot add food to a disposed food container.');
        }
        this._food.add(food);
    }
    
    public dispose(): void {
        this._food.clear();
        this._childContainer?.dispose();
        this._childContainer = null;
        this._disposed = true;
    }
    
    public isDisposed(): boolean {
        return this._isDisposed;
    }
}
```

## `Disposable`抽象类
下面是源码（`DisposableStore`本质上就是一个`Set`，我就不写在文章里了）：
```ts
/**
 * Abstract base class for a {@link IDisposable disposable} object.
 *
 * Subclasses can {@linkcode _register} disposables that will be automatically cleaned up when this object is disposed of.
 */
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
- `Disposable`抽象类就是提供了一个基类，只要继承了该类，就可以利用`_register`来注册本应当隶属于该instance下的resources（这些resources也必须是`IDisposable`)。然后调用`dispose()`可以一同释放掉已经register过后资源，非常方便于统一管理。
- 这种设计模式也提供了recursive register，是一个很强大也很简单的方法。`IDisposable`接口贯穿了整个VSCode的repository。

### recursive dispose的案例
```ts
const child = new FoodContainer();
const parent = new FoodContainer(child);

parent.dispose();
console.log(parent.isDisposed()); // true
console.log(child.isDisposed()); // true
```
