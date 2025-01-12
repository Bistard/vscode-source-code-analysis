---
layout: post
title: "[CN] VSCode系列「系统篇」：配置系统2 - JSON Schema"
categories: [VSCode, Systems]
---


# `IJSONSchema` 接口

> 参考 [JSON Schema 官方网站](https://json-schema.org/)

`IJSONSchema` 是 Visual Studio Code 中用于描述 JSON 数据格式的标准接口。它提供了一套规则来约束 JSON 数据的结构和内容。可以将 `schema` 理解为一组约束 JSON 数据格式的规则，这些规则被用于定义和验证传入的 JSON 数据是否符合预期格式。以下是一个简单的示例。

### 数据示例
```typescript
interface IPerson {
    name: string;
    age: number;
}
```

### `schema` 示例
```typescript
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

### 有效数据示例
```typescript
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

在 VSCode 中，`IJSONSchema` 接口的定义可以在 `vscode/src/vs/base/common/jsonSchema.ts` 文件中找到。以下是部分代码：

```typescript
export interface IJSONSchema {
    id?: string;
    $id?: string;
    $schema?: string;
    type?: JSONSchemaType | JSONSchemaType[];
    title?: string;
    default?: any;
    definitions?: IJSONSchemaMap;
    description?: string;
    properties?: IJSONSchemaMap;
    patternProperties?: IJSONSchemaMap;
    additionalProperties?: boolean | IJSONSchema;
    minProperties?: number;
    maxProperties?: number;
    dependencies?: IJSONSchemaMap | { [prop: string]: string[]; };
    items?: IJSONSchema | IJSONSchema[];
    minItems?: number;
    maxItems?: number;
    uniqueItems?: boolean;
    additionalItems?: boolean | IJSONSchema;
    pattern?: string;
    minLength?: number;
    maxLength?: number;
    minimum?: number;
    maximum?: number;
    exclusiveMinimum?: boolean | number;
    exclusiveMaximum?: boolean | number;
    multipleOf?: number;
    required?: string[];
    $ref?: string;
    anyOf?: IJSONSchema[];
    allOf?: IJSONSchema[];
    oneOf?: IJSONSchema[];
    not?: IJSONSchema;
}
```

`JSONSchema` 接口的定义比较复杂，因为它整合了所有类型的属性。不同类型的数据会对应不同的属性约束：

- 当 `type: 'number'` 时，`minimum` 和 `maximum` 属性用于限制数值的范围。
- 当 `type: 'string'` 时，`minLength` 和 `maxLength` 属性用于限制字符串的长度。
- 当 `type: 'array'` 时，`minItems` 和 `maxItems` 属性用于限制数组项的数量。

由于没有使用复杂的类型系统，VSCode 的 `IJSONSchema` 将所有的 schema 字段都放在一起，而不是基于类型进行分离。这种方式虽然使得接口定义较为简单，但也导致接口字段较为冗长。

### VSCode的验证过程
> 在typescript中光定义一个Interface当然是无法起到数据验证功能的，因为TypeScript其中的类型在编译期间会进行类型擦除，而runtime时则没有任何runtime手段进行验证数据是否符合schema。
> 因此VSCode也会有相关的runtime代码来检查runtime读取的json数据是否匹配对应的schema。

不过在 `IJSONSchema` 接口中，虽然定义了大量的字段以方便开发者使用统一格式进行数据描述，但具体的runtime的验证过程在VSCode中我并没有找到哪里被明确规定。

我猜测实际的验证是由不同的组件各自负责处理的，这意味着 `IJSONSchema` 并没有提供一个集中化的验证过程，而是依赖于具体实现进行验证。

> 如果在你的项目中，我个人是建议弄一套集中化的验证过程，而不是让各个系统自己负责手动监测。除非有非常特殊的需求/性能敏感的场景。

我唯一能找到的相关验证文件是：
1. `src\vs\workbench\services\preferences\common\preferencesValidation.ts`
貌似是用来验证preference configuration时会检查`IJSONSchema`和给定的数据是否匹配。
