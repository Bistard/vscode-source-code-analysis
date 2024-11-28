---
layout: post
title:  "[CN] VSCode系列「系统篇」：配置系统"
categories: [VSCode, Systems]
---

# 配置系统 - Configuration System

> 丑话说前面, 整个配置系统, 我私认为属于VSCode众多大系统中top10甚至top5的复杂系统了. 
>
> 业务本身复杂繁琐，代码量也很多，有些部分代码我自己没有chatGPT4.0的帮忙的话，读起来够呛。
>
> 在我读了基本框架之后, 感觉这个系统是随着项目开发, 一步一步添加新的功能, 最终成为了现在这个样子. 很多地方的namings和设计<u>并不直观</u>, 可以看到各种各样的设计模式/代码习惯.
>
> VSCode的整个配置系统包含了众多功能, 除了最容易联想到的“读取本地 (native) 配置文件”之外, 还会涉及到:
>
> - 跨进程的读写配置问题, 
> - 如何注册配置, 不同进程如何注册配置, 注册的配置要如何管理, 如何统一管理所有的配置:
>     - 如何分类, 如何注册默认配置, 如何标记哪些是override, 如何识别来自插件的配置等无数繁琐的细节.
> - 如何读取non-native的配置文件 (remote相关代码)
> - etc...
>
> 然而上面列出来的我基本都不太会涉及到 (doge
>
> 因为我学习的过程属于是挑着读的, 算不上很系统的阅读, 我自己暂时用不到的部分, 我自然就没动力去读了.
>
> * 比如在`ConfigurationService`中, 常见的一些`PolicyConfiguration`, `WorkspaceConfiguration`, `RemoteUserConfiguration`, `FolderConfiguration`等等我都略过去了, 因为这些功能跟VSCode这个产品本身有点息息相关. 各位在设计自己的配置系统的时候可以自行选择如何设计, 不一定非要参考VSCode.
> * 而像一些通用的设计: 比如主进程/渲染进程之间如何通信配置信息, 如何注册默认配置, 如何注册用户配置, 如何将这些配置整合起来等等, 这些我认为是一些很实用的设计, 也方便各位用来参考和学习.
>
> 我是边写边读的, 所以写的很硬核很枯燥很底层, 并没有用通俗的语言来进行大量的概括和总结, 基本上就是陈述代码是什么, 见谅. (或许后续我可能会写一个概括或者总结之类的)

我大概可以讲整个配置系统分成以下几个比较重要的组件：

1. `ConfigurationRegistry` 类 - 软件中各个子部件如何注册配置
2. `ConfigurationService` 主进程微服务
3. `WorkspaceService` 渲染进程微服务

## 1. 如何注册配置

> 前置知识: `IJSONSchema`和什么是`Registry`.

在开始之前我得简单介绍一些小接口.

### `IConfigurationPropertySchema`接口

* `IConfigurationPropertySchema`继承了`IJSONSchema`并添加了一些新的字段。
* 每一个`IConfigurationPropertySchema`都可以理解成我们通常语境下的：一条配置。
    * 比如`font size: 12px`就可以理解成一条配置。

### `IConfigurationNode`接口

* 而每一个node都包含一组`IConfigurationPropertySchema`（位于`properties`字段）。所以可以把node理解成一整个group of schemas，一套配置。
    * 每一个node还可以嵌套多层node：通过`allOf`字段。
* `IConfigurationNode`属于是`ConfigurationRegistry`注册默认配置时所用的基本单位了，可以参考`ConfigurationRegistry`的一系列APIs，入参都需要该接口。

### `ConfigurationRegistry`类

* VSCode通过`ConfigurationRegistry`来注册configurations（特指`IConfigurationNode`）。
    * **注意**，这里存的configurations要理解成存的是default configurations，因为并不能通过该registry来进行configuration的updation。

- 真正可以进行修改配置的微服务在VSCode中叫做`ConfigurationService`。

#### Class Fields - 类字段

- `configurationProperties`

    - it is used to store all the registered configuration properties, each of which includes a default value. The other fields (`applicationSettings`, `machineSettings`, `machineOverridableSettings`, etc.) are categorized collections of these properties, grouped based on their scope.
    - `ConfigurationRegistry`提供了一个API叫做`getConfigurationProperties`就是用来返回该字段的，其他部件想要访问配置就可以调用该API，这个API使用率比较高。

- `configurationDefaultsOverrides`

    - It's used to store default configuration values that are meant to override the original default values of certain configuration properties. So, when a configuration property is accessed, the system first checks if there's an override default value for it in `configurationDefaultsOverrides`. If there is, this value is used instead of the original default value.

    > This allows extensions or other parts of the system to change the default behavior of certain features without changing the code that implements these features.

- `configurationContributors`

    - 这玩意就是一个`IConfigurationNode[]`类型。因为register和deregister配置的时候都是以`IConfigurationNode`为单位的，这个field就是用来追踪哪些`IConfigurationNode`被注册了而已，register的时候push，deregister的时候splice掉。相关的API叫做`getConfigurations`，在整个VSCode的repository中出现的次数很少，应该不是很重要。

## 2. `ConfigurationService`主进程微服务

- 该类创建于整个程序的非常初期，初始创建于**主进程**内，位于`CodeMain`类（虽然创建于主进程, 不过相关文件存放在了`common`文件夹下）。

- `ConfiguraionService`的代码量很少，都被其他的类分担了。

- 该微服务**不支持**`updateValue`的API，意味着你是没法主动进行配置修改的，所以想要修改文件, 要么reload要么通过修改源配置文件（JSON文件）。

    - 之后提到的`WorkspaceService`支持`updateValue` API。

- 涉及到了三个比较重要的类: `DefaultConfiguration`和`UserSettings`和`Configuration`（`IPolicyConfiguration`在我写的时候没看懂，目前也不影响我去理解配置系统）。

    - 对于外部程序而言，所有的配置信息都是从`Configuration`类中获取。

- 下面是该微服务的constructor，可以作为参考简单了解一下构造顺序：

    ```ts
    export class ConfigurationService extends Disposable implements IConfigurationService, IDisposable {
    
    declare readonly _serviceBrand: undefined;
    
    private configuration: Configuration;
    private readonly defaultConfiguration: DefaultConfiguration;
    private readonly policyConfiguration: IPolicyConfiguration;
    private readonly userConfiguration: UserSettings;
    private readonly reloadConfigurationScheduler: RunOnceScheduler;
    
    private readonly _onDidChangeConfiguration: Emitter<IConfigurationChangeEvent> = this._register(new Emitter<IConfigurationChangeEvent>());
    readonly onDidChangeConfiguration: Event<IConfigurationChangeEvent> = this._onDidChangeConfiguration.event;
    
    constructor(
        private readonly settingsResource: URI,
        fileService: IFileService,
        policyService: IPolicyService,
        logService: ILogService,
    ) {
        super();
        this.defaultConfiguration = this._register(new DefaultConfiguration());
        this.policyConfiguration = policyService instanceof NullPolicyService ? new NullPolicyConfiguration() : this._register(new PolicyConfiguration(this.defaultConfiguration, policyService, logService));
        this.userConfiguration = this._register(new UserSettings(this.settingsResource, undefined, extUriBiasedIgnorePathCase, fileService));
        this.configuration = new Configuration(this.defaultConfiguration.configurationModel, this.policyConfiguration.configurationModel, new ConfigurationModel(), new ConfigurationModel());
    
        this.reloadConfigurationScheduler = this._register(new RunOnceScheduler(() => this.reloadConfiguration(), 50));
        this._register(this.defaultConfiguration.onDidChangeConfiguration(({ defaults, properties }) => this.onDidDefaultConfigurationChange(defaults, properties)));
        this._register(this.policyConfiguration.onDidChangeConfiguration(model => this.onDidPolicyConfigurationChange(model)));
        this._register(this.userConfiguration.onDidChange(() => this.reloadConfigurationScheduler.schedule()));
    }
    
    async initialize(): Promise<void> {
        const [defaultModel, policyModel, userModel] = await Promise.all([this.defaultConfiguration.initialize(), this.policyConfiguration.initialize(), this.userConfiguration.loadConfiguration()]);
        this.configuration = new Configuration(defaultModel, policyModel, new ConfigurationModel(), userModel);
    }
    
    // ...
    ```

> 在介绍这三个类之前，需要介绍一下它们三都涉及到了一个基础类叫做`ConfigurationModel`。

#### `ConfigurationModel` 类

- 该类中通过一个object类型来储存对应的configuration。

- 想要往model中add value的话，需要一对参数key/value。key的格式是被设定好的：

    - key必须是一个string类型，而且会以`.`为分割标识。
    - 比如key可以是`vscode`，也可以是`vscode.workspace.language`。
    - 假设我想用上述两个key分别赋值value为: `{ size: 5, name: 'hello world' }` 和 `{ lang: 'C++', restricted: true }` 的话，实际储存在model中的数据会如下储存：

    ```ts
    {
       'vscode': {
          size: 5,
          name: 'hello world',
          'workspace': {
             'language': {
                lang: 'C++',
                restricted: true,
             }
          }
       }
    }
    ```

#### `DefaultConfiguration` 类

- 位于`vscode\src\vs\platform\configuration\common\configurations.ts`.
- `reset` API
    - 该API会从`ConfigurationRegistry.getConfigurationProperties()`获得所有的默认配置，同时也会创建一个`ConfigurationModel`，并将这些默认配置复制到新创建的model中。
- `initialize` API
    - 先调用`reset`，然后监听`ConfigurationRegistry.onDidUpdateConfiguration`事件，并同时更新自己的model。

#### `UserSettings` 类

- 该类需要一个URI指向settingResource，通过内嵌的一个叫`ConfigurationModelParser`从URI读取配置文件，由于一开始读的是源文本，parser需要对其原始数据进行parsing。

    > 之所以这里需要手动parsing，是因为要利用`ConfigurationRegistry`中的schema进行validation，确保读取的文件里的配置的格式都是正确的。
    > 如果有任何字段不正确，读进内存的时候都会被filter掉。

- 读取完后，数据也会被包裹在`ConfigurationModel`中。


#### `Configuration`类

可以先简单看下该类的开头几行（其实我觉得这玩意改名成类似于`ConfigurationCollection`或者`AllConfiguration`更好）：

```ts
export class Configuration {

    private _workspaceConsolidatedConfiguration: ConfigurationModel | null = null;
    private _foldersConsolidatedConfigurations = new ResourceMap<ConfigurationModel>();

    constructor(
        private _defaultConfiguration: ConfigurationModel,
        private _policyConfiguration: ConfigurationModel,
        private _applicationConfiguration: ConfigurationModel,
        private _localUserConfiguration: ConfigurationModel,
        private _remoteUserConfiguration: ConfigurationModel = new ConfigurationModel(),
        private _workspaceConfiguration: ConfigurationModel = new ConfigurationModel(),
        private _folderConfigurations: ResourceMap<ConfigurationModel> = new ResourceMap<ConfigurationModel>(),
        private _memoryConfiguration: ConfigurationModel = new ConfigurationModel(),
        private _memoryConfigurationByResource: ResourceMap<ConfigurationModel> = new ResourceMap<ConfigurationModel>()
    ) {
    }
   // ...
```

- 在`ConfigurationService.initialized()`调用的时候，会将`DefaultConfiguration`和`UserSettings`里的两个`ConfigurationModel`都传进到`Configuration`中。
- `Configuration`类基本上可以理解成对于所有的model基于`DefaultConfiguration`的一种整合（所有其他配置会被merge到`DefaultConfiguration`对应的model中）。
- 代码量虽然挺多，不过主要都是各种各样的整合。

## 3. `WorkspaceService`渲染进程微服务

* <u>该类只出现在渲染进程</u>, 分别是`web.main.ts`和`desktop.main.ts`中.
* 虽然叫做`WorkspaceService`, 不过本质上就是渲染进程版本的`ConfigurationService`.
* 和`ConfigurationService`不同的是, 该服务支持`updateValue` API. 具体见`ConfigurationEditing`类.

### `ConfigurationCache`类

* 在渲染进程中, 如果想要读写配置文件, 不管该文件是native还是non-native (remote), 都必然涉及到异步操作, 这也是为什么它的接口都是Promise.

* 它的接口:

    ```ts
    export type ConfigurationKey = { type: 'defaults' | 'user' | 'workspaces' | 'folder'; key: string; };
    
    export interface IConfigurationCache {
        needsCaching(resource: URI): boolean;
        read(key: ConfigurationKey): Promise<string>;
        write(key: ConfigurationKey, content: string): Promise<void>;
        remove(key: ConfigurationKey): Promise<void>;
    }
    ```

* 一个很简单的通用类, 会为每个non-native的配置文件提供`read`, `write`, `remove `APIs. 

    * 只能对整个文件进行整体操作 (`write`的第二参数`content`就是整个文件的内容).
    * 对于每一个non-native配置文件, 都配对一个单独的queue来管理读写, 来确保读写的顺序正确性.
    * 比如`RemoteUserConfiguration`, `WorkspaceConfiguration`, `FolderConfiguration`都用到了这个类.
    * 而`DefaultConfiguration`和`UserConfiguration`基本没用到, 因为这两个配置信息一般都属于native.

* <u>该类只出现在渲染进程</u>, 分别是`web.main.ts`和`desktop.main.ts`中.

    * 在`desktop.main.ts`中, 它的构造:

        ```ts
        const configurationCache = new ConfigurationCache([Schemas.file, Schemas.vscodeUserData] /* Cache all non native resources */, /* ... */);
        const workspaceService = new WorkspaceService({ remoteAuthority: environmentService.remoteAuthority, configurationCache }, /* ... */);
        ```

### `DefaultConfiguration`类

* 位于`vscode\src\vs\workbench\services\configuration\browser\configuration.ts`.
* 继承了`\common\`文件夹下的`DefaultConfiguration`, 除此之外并没有特别值得注意的改变.

### `UserConfiguration`类

* 这个类也是处于`browser`文件夹下, 基本上就是`common`文件夹下的那个`UserSettings`的wrapper.
* 除此之外, 可能需要注意的点是当reload的时候, 内嵌的`UserSettings`会被换成一个`FileServiceBasedConfiguration`. 具体这个是干嘛的我略过去了.

### `Configuration`类

* 渲染进程用了`Configuration`类和主进程是同一个类, 所以这里就不多赘述了.

### `ConfigurationEditing`类 - 如何编辑configuration

* `WorkspaceService`通过`updateValue`来通过key和value来更新具体的配置. 其中核心函数为`writeConfigurationValue`.
    * 不支持修改默认配置.
    * `writeConfigurationValue`里会初始化一个`ConfigurationEditing`并调用其API.
* 该类只有一个API叫做`writeConfiguration`.
