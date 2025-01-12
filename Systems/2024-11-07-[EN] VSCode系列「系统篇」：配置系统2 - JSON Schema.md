---
layout: post
title: "[EN] VSCode Series: Configuration System 2 - JSON Schema"
categories: [VSCode, Systems]
---

# `IJSONSchema` Interface

> Reference: [JSON Schema Official Website](https://json-schema.org/)

`IJSONSchema` is a standard interface in Visual Studio Code used to describe the structure and content of JSON data. It provides a set of rules to constrain the format of JSON data. A `schema` can be understood as a collection of rules that define and validate whether incoming JSON data conforms to the expected format. Below is a simple example.

### Data Example
```typescript
interface IPerson {
    name: string;
    age: number;
}
```

### `schema` Example
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

### Valid Data Example
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

In VSCode, the `IJSONSchema` interface definition can be found in the `vscode/src/vs/base/common/jsonSchema.ts` file. Here’s a snippet of the interface:

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

The `IJSONSchema` interface is complex because it consolidates attributes for all types. Different data types correspond to different constraints:

- When `type: 'number'`, attributes like `minimum` and `maximum` are used to limit the range of values.
- When `type: 'string'`, attributes like `minLength` and `maxLength` constrain the length of the string.
- When `type: 'array'`, attributes like `minItems` and `maxItems` specify the number of items.

Since it doesn’t use an elaborate type system, VSCode’s `IJSONSchema` combines all schema fields into a single structure rather than segregating them by type. While this approach simplifies the interface definition, it makes the interface more verbose.

### Validation Process in VSCode
> In TypeScript, defining an interface alone does not enable data validation, as TypeScript's type system is erased during compilation, leaving no runtime means to validate data against the schema.
> Therefore, VSCode includes runtime code to verify that JSON data conforms to the schema.

Although the `IJSONSchema` interface includes many fields to help developers describe data in a unified format, the specific runtime validation process is not clearly documented in VSCode.

I speculate that validation is handled by different components independently. This means `IJSONSchema` does not provide a centralized validation process but relies on specific implementations to perform the validation.

> In your own project, I recommend implementing a centralized validation process rather than having different systems handle validation manually—unless you have highly specific requirements or performance-sensitive scenarios.

The only related validation file I found is:
1. `src/vs/workbench/services/preferences/common/preferencesValidation.ts`
   It seems to validate `IJSONSchema` against provided data when checking preference configurations.