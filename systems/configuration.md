# 配置系统 - Configuration System
整个VSCode的配置系统算得上是复杂繁琐的，代码量也很多，我自己没有chatGPT4.0的帮忙的话，读起来够呛。

> 阅读前置知识：`IJSONSchema`， `Registry`。

我大概可以讲整个配置系统分成以下几个大块：
1. `ConfigurationRegistry` - class
2. `ConfigurationService` - microservice (common)

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

## `ConfigurationService`微服务 (common)
> 涉及到了三个比较重要的类: `DefaultConfiguration`和`UserSettings`和`Configuration`。在介绍这三个类之前，需要介绍一下它们三都涉及到了一个基础类叫做`ConfigurationModel`。

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

#### `Configuration`类
