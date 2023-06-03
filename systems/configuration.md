# 配置系统 - Configuration System
整个VSCode的配置系统算得上是复杂繁琐的，代码量也很多，有些部分代码我自己没有chatGPT4.0的帮忙的话，读起来够呛。

> 阅读前置知识：`IJSONSchema`， `Registry`。

我大概可以讲整个配置系统分成以下几个大块：
1. `ConfigurationRegistry` 类
2. `ConfigurationService` 主进程微服务
3. `WorkspaceService` 渲染进程微服务

## `IConfigurationPropertySchema`接口
* `IConfigurationPropertySchema`继承了`IJSONSchema`并添加了一些新的字段。
* 每一个`IConfigurationPropertySchema`都可以理解成我们通常语境下的：一条配置。
    * 比如`font size: 12px`就可以理解成一条配置。

## `IConfigurationNode`接口
* 而每一个node都包含一组`IConfigurationPropertySchema`（位于`properties`字段）。所以可以把node理解成一整个group of schemas，一套配置。
    * 每一个node还可以嵌套多层node：通过`allOf`字段。
* `IConfigurationNode`属于是`ConfigurationRegistry`注册默认配置时所用的基本单位了，可以参考`ConfigurationRegistry`的一系列APIs，入参都需要该接口。

## `ConfigurationRegistry`类
VSCode通过`ConfigurationRegistry`来注册configurations（特指`IConfigurationNode`）。注意，这里存的configurations要理解成default configurations，因为并不能通过该registry来进行updation。
- 真正可以进行修改配置的微服务在VSCode中叫做`ConfigurationService`。

### Class Fields - 类字段
- `configurationProperties`
  - it is used to store all the registered configuration properties, each of which includes a default value. The other fields (`applicationSettings`, `machineSettings`, `machineOverridableSettings`, etc.) are categorized collections of these properties, grouped based on their scope.
  - `ConfigurationRegistry`提供了一个API叫做`getConfigurationProperties`就是用来返回该字段的，其他部件想要访问配置就可以调用该API，这个API使用率比较高。
- `configurationDefaultsOverrides`
  - It's used to store default configuration values that are meant to override the original default values of certain configuration properties. So, when a configuration property is accessed, the system first checks if there's an override default value for it in `configurationDefaultsOverrides`. If there is, this value is used instead of the original default value.
  > This allows extensions or other parts of the system to change the default behavior of certain features without changing the code that implements these features.
- `configurationContributors`
  - 这玩意就是一个`IConfigurationNode[]`类型。因为register和deregister配置的时候都是以`IConfigurationNode`为单位的，这个field就是用来追踪哪些`IConfigurationNode`被注册了而已，register的时候push，deregister的时候splice掉。相关的API叫做`getConfigurations`，在整个VSCode的repository中出现的次数很少，应该不是很重要。

## `ConfigurationService`主进程微服务
- 该类创建于整个程序的非常初期，初始创建于**主进程**内，位于`CodeMain`类（不过相关文件存放在了`common`下）。
- `ConfiguraionService`的代码量很少，都被其他的类分担了。
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


## `WorkspaceService`渲染进程微服务

#### `ConfigurationEditing`类
