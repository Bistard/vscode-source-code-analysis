
## 背景介绍
一个软件中，「右键菜单」是非常平凡的一个操作：
1. 你需要编辑一个文件/文件夹，你就右键它弹出针对文件树的文件/文件夹内容的「右键菜单」。
2. 你需要编辑一段文字，你选中并右键它，会弹出一个针对文本内容的「右键菜单」。
3. 你需要编辑一个xxx，你就右键它，会弹出一个针对xxx内容的「右键菜单」。
4. ...

> **一个软件内部，针对不同的「右键对象」，会弹出不同的「右键菜单」。** 比方说，VSCode整个软件里，一共有189个不同内容的「右键菜单」。

VSCode这么多「右键菜单」，它是怎么统一管理操作逻辑，统一修改内容的呢？

## VSCode：`MenuRegistry`和`MenuId`

VSCode把这种「**右键菜单**」**里所呈现的内容**在代码里面称呼为`Menu`。
我第一次意识到VSCode有统一管理`Menu`里的内容的倾向是看到了源码里有两个重要的工具叫做：
1. `MenuRegistry`
2. `MenuId`

`MenuRegistry`和`MenuId`都隶属于同一个文件：`src\vs\platform\actions\common\actions.ts`（不得不吐槽这个文件的名字真的感觉跟它没什么血缘关系。。。）`MenuRegistry`它本质就是一个全局变量，用来储存注册信息的，仅此而已。它的API如下：
```ts
export interface IMenuRegistry {
   readonly onDidChangeMenu: Event<IMenuRegistryChangeEvent>;
	addCommand(userCommand: ICommandAction): IDisposable;
	getCommand(id: string): ICommandAction | undefined;
	getCommands(): ICommandsMap;

	/**
	 * @deprecated Use `appendMenuItem` or most likely use `registerAction2` instead. There should be no strong
	 * reason to use this directly.
	 */
	appendMenuItems(items: Iterable<{ id: MenuId; item: IMenuItem | ISubmenuItem; }>): IDisposable;
	appendMenuItem(menu: MenuId, item: IMenuItem | ISubmenuItem): IDisposable;
	getMenuItems(loc: MenuId): Array<IMenuItem | ISubmenuItem>;
}
```
## VSCode: `MenuRegistry`和`MenuId`的使用案例
这里面的`addCommand`系列不是很重要，所以不属于今天讨论的话题，~~当然不是我读不懂是干嘛~~的。主要看`appendMenuItem`。我们来看看VSCode源码里是怎么用这个API的：
```ts
// Menu registration - explorer
MenuRegistry.appendMenuItem(MenuId.ExplorerContext, {
	group: 'navigation',
	order: 4,
	command: {
		id: NEW_FILE_COMMAND_ID,
		title: NEW_FILE_LABEL,
		precondition: ExplorerResourceNotReadonlyContext
	},
	when: ExplorerFolderContext
});
// ...
MenuRegistry.appendMenuItem(MenuId.ExplorerContext, {
	group: '3_compare',
	order: 30,
	command: compareSelectedCommand,
	when: ContextKeyExpr.and(ExplorerFolderContext.toNegated(), ResourceContextKey.HasResource, WorkbenchListDoubleSelection)
});
// ...
MenuRegistry.appendMenuItem(MenuId.ExplorerContext, {
	group: '5_cutcopypaste',
	order: 10,
	command: {
		id: COPY_FILE_ID,
		title: COPY_FILE_LABEL,
	},
	when: ExplorerRootContext.toNegated()
});
MenuRegistry.appendMenuItem(MenuId.ExplorerContext, {
	group: '5_cutcopypaste',
	order: 20,
	command: {
		id: PASTE_FILE_ID,
		title: PASTE_FILE_LABEL,
		precondition: ContextKeyExpr.and(ExplorerResourceNotReadonlyContext, FileCopiedContext)
	},
	when: ExplorerFolderContext
});
// a bunch of all other registrations...
```
我只列出来4条注册信息，其实这个文件里还注册了非常多其他的功能。其中`MenuId.ExplorerContext`其实所指代的就是文件树里的「右键菜单」。最终在VSCode里呈现的效果如下图所示：
![alt text](/assets/vscode-analysis/menu/menu1.png)
我来总结一下这4条在干什么：
1. 4条注册信息都在往一个叫做`MenuId.ExplorerContext`的`MenuId`里注册一条item。每一条item其实就是「右键菜单」的一行按钮。
2. 比如第一条注册信息，就是注册了一个名字（title）叫做`NEW_FILE_LABEL`，点击按钮之后会运行一个命令（command）叫做`NEW_FILE_COMMAND_ID`。这行按钮的group叫做`navigation`，在渲染的时候会和其他被标记为`navigation`的item一起渲染在一起，然后画一个横线和其他的group分开。而`when`里填写的则是告诉软件这一行item要在`ExplorerFolderContext`条件为true的情况下才渲染。

具体的注册API`appendMenuItem(menu: MenuId, item: IMenuItem | ISubmenuItem): IDisposable`其中的类型定义如下：
```ts
export interface IMenuItem {
	command: ICommandAction;
	alt?: ICommandAction;
	/**
	 * Menu item is hidden if this expression returns false.
	 */
	when?: ContextKeyExpression;
	group?: 'navigation' | string;
	order?: number;
	isHiddenByDefault?: boolean;
}
export interface ISubmenuItem {
	title: string | ICommandActionTitle;
	submenu: MenuId;
	icon?: Icon;
	when?: ContextKeyExpression;
	group?: 'navigation' | string;
	order?: number;
	isSelection?: boolean;
	// for dropdown menu: if true the last executed action is remembered as the default action
	rememberDefaultAction?: boolean;
}
```
## `MenuId`
`MenuId`的定义如下：
```ts
export class MenuId {

	private static readonly _instances = new Map<string, MenuId>();

	static readonly CommandPalette = new MenuId('CommandPalette');
	static readonly DebugBreakpointsContext = new MenuId('DebugBreakpointsContext');
	static readonly DebugCallStackContext = new MenuId('DebugCallStackContext');
	static readonly DebugConsoleContext = new MenuId('DebugConsoleContext');
	static readonly DebugVariablesContext = new MenuId('DebugVariablesContext');
	// ...other 180+ menus
}
```
其实这个`MenuId`吧，虽然在VSCode源码里写成了class形式，但我觉得换成`const Enum`，换成`Symbol`之类的也完全可以。class本身的prototype并没有提供任何有用的API。就是`new MenuId(SOME_STRING_ID)`然后拿返回值当作unique identifier使用。

## 如何实现`SubMenu`功能
以下代码来自`src\vs\editor\contrib\clipboard\browser\clipboard.ts`:
```ts
// ...
MenuRegistry.appendMenuItem(MenuId.ExplorerContext, { submenu: MenuId.ExplorerContextShare, title: nls.localize2('share', "Share"), group: '11_share', order: -1 });
// ...
```
其中argument中填入了`submenu: MenuId.ExplorerContextShare`。意思就是这里将会渲染一行叫做`share`的按钮，然后它在UI的呈现上是一个submenu的形式。如刚刚较早呈现的图片里所示（你能看到share按钮右边有个submenu的指示器）。

那么你该如何往这个叫做`share`的`Submenu`里注册item呢？
* 非常简单，就如同上方是如何往`MenuId.ExplorerContext`里注册items一样，你也应该用同样的方法调用`MenuRegistry.addMenuItem()`往`MenuId.ExplorerContextShare`注册items。
* 在UI渲染的时候，会去检测该menu下的每一个item是否属于submenu，我们注册menu的时候只需要标记一下即可。

## `MenuRegistry`不仅仅是为了「右键菜单」服务的

这里我得提一个在vscode源码中大量使用的一个函数，据目前写文章位置，这个函数在源码中出现了`1303`次。
函数定义如下：
```ts
export function registerAction2(ctor: { new(): Action2; }): IDisposable {
	const disposables: IDisposable[] = []; // not using `DisposableStore` to reduce startup perf cost
	const action = new ctor();
	const { f1, menu, keybinding, ...command } = action.desc;
	// command ...
	// menu
	if (Array.isArray(menu)) {
		for (const item of menu) {
			disposables.push(MenuRegistry.appendMenuItem(item.id, { command: { ...command, precondition: item.precondition === null ? undefined : command.precondition }, ...item }));
		}

	} else if (menu) {
		disposables.push(MenuRegistry.appendMenuItem(menu.id, { command: { ...command, precondition: menu.precondition === null ? undefined : command.precondition }, ...menu }));
	}
	if (f1) {
		disposables.push(MenuRegistry.appendMenuItem(MenuId.CommandPalette, { command, when: command.precondition }));
		disposables.push(MenuRegistry.addCommand(command));
	}

	// keybinding
	if (Array.isArray(keybinding)) {
		// ...
	} else if (keybinding) {
		// ...
	}
	// ...
}
```
* 其中有一行代码是`MenuRegistry.appendMenuItem(MenuId.CommandPalette, ...)`, 也就是说每注册一次action，在条件允许的情况下（`f1`被定义了）都会往`MenuId.CommandPalette`注册一条。而`CommandPalette`就是下图这个功能（可以通过`ctrl+shift+P`快捷键调出来）：
![alt text](/assets/vscode-analysis/menu/menu2.png)
* 这里就能看出来，`MenuRegistry`里的注册信息，除了可以为「右键菜单」的渲染服务，也可以为了任何具有「列表性质」的UI服务。所以这个`MenuRegistry`的泛用性极其广泛。