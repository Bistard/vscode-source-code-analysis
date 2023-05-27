# Configuration System
整个VSCode的配置系统还是比较复杂繁琐的，代码量很多，我自己没有chatGPT4.0的帮忙的话，读起来够呛。

> 阅读前置知识：`IJSONSchema`。

我大概可以讲整个配置系统分成以下几个大块：
1. `ConfigurationRegistry` - class
2. `ConfigurationService` - microservice (common)

# `IJSONSchema` and `IConfigurationPropertySchema`
`IJSONSchema`是一个用于描述 JSON 数据格式的标准，schema可以理解成a set of rules，是用来对传进来的JSON形式的data进行约束的。可以参照下面案例：
#### Data Example
```ts
interface IPerson {
    name: string;
    age: number;
}
```
#### The Schema Example
```ts
let personSchema: IJSONSchema = {
    type: 'object',
    properties: {
        name: {
            type: 'string',
            minLength: 1
        },
        age: {
            type: 'number',
            minimum: 0
        }
    },
    required: ['name', 'age']
};
```
#### Valid Data Example
```ts
let userData = {
    "name": "John Doe",
    "age": 30,
    "email": "john@example.com",
    "address": {
        "street": "123 Main St",
        "city": "Springfield",
        "country": "USA"
    },
    "hobbies": ["Reading", "Traveling", "Swimming"]
}
```
> `IConfigurationPropertySchema`也是同理，只不过继承了`IJSONSchema`并添加了一些新的字段。


# ConfigurationRegistry
VSCode通过`ConfigurationRegistry`来注册default configuration （特指`IConfigurationPropertySchema`）。真正可以进行修改配置的微服务在VSCode中叫做`ConfigurationService`。

## `IConfigurationNode`
* 每一个node都包含一组configuraion schema：`IConfigurationPropertySchema`（`properties`字段）。每一个schema都是一条我们常说的配置。
> 比如`font size: 12px`就可以理解成一条配置。
* 每一个node还可以construct多层node：通过`allOf`字段。
* `IConfigurationNode`属于是通过`ConfigurationRegistry`注册配置时所用的基本单位了。

## Class Fields
- `configurationProperties`
  - it is used to store all the registered configuration properties, each of which includes a default value. The other fields (`applicationSettings`, `machineSettings`, `machineOverridableSettings`, etc.) are categorized collections of these properties, grouped based on their scope.
  - `ConfigurationRegistry`提供了一个API叫做`getConfigurationProperties`用来返回这个字段，其他部件想要访问配置就可以调用该API。
- `configurationDefaultsOverrides`
  - It's used to store default configuration values that are meant to override the original default values of certain configuration properties. So, when a configuration property is accessed, the system first checks if there's an override default value for it in `configurationDefaultsOverrides`. If there is, this value is used instead of the original default value.
  > This allows extensions or other parts of the system to change the default behavior of certain features without changing the code that implements these features.
- `configurationContributors`
  - 这玩意就是一个`IConfigurationNode[]`类型。因为register和deregister配置的时候都是以`IConfigurationNode`为单位的，这个field就是用来追踪哪些`IConfigurationNode`被注册了而已。相关的method叫做`getConfigrations`，在整个VSCode的repository中出现的次数很少，应该不是很重要。

# ConfigurationService (common)
- 储存了两个重要的字段：`configuration`和`defaultConfiguration`。
