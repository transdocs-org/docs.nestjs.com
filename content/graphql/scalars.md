### 标量类型（Scalars）

GraphQL对象类型具有名称和字段，但最终这些字段必须解析为一些具体的数据。这就是标量类型的作用：它们表示查询的叶节点（可进一步阅读[此处](https://graphql.org/learn/schema/#scalar-types)）。GraphQL包含以下默认类型：`Int`、`Float`、`String`、`Boolean` 和 `ID`。除了这些内置类型之外，你可能还需要支持自定义的基本数据类型（例如 `Date`）。

#### 代码优先（Code first）

代码优先方式默认提供了五种标量类型，其中三个是现有GraphQL类型的简单别名。

- `ID`（`GraphQLID` 的别名）—— 表示唯一标识符，通常用于重新获取对象或作为缓存的键
- `Int`（`GraphQLInt` 的别名）—— 有符号的32位整数
- `Float`（`GraphQLFloat` 的别名）—— 有符号的双精度浮点值
- `GraphQLISODateTime` —— UTC时间的日期时间字符串（默认用于表示 `Date` 类型）
- `GraphQLTimestamp` —— 有符号整数，表示从UNIX纪元开始的毫秒数作为日期和时间

默认使用 `GraphQLISODateTime`（例如 `2019-12-03T09:54:33Z`）来表示 `Date` 类型。若要使用 `GraphQLTimestamp`，请将 `buildSchemaOptions` 对象的 `dateScalarMode` 设置为 `'timestamp'`，如下所示：

```typescript
GraphQLModule.forRoot({
  buildSchemaOptions: {
    dateScalarMode: 'timestamp',
  }
}),
```

同样，默认使用 `GraphQLFloat` 来表示 `number` 类型。如果要使用 `GraphQLInt`，请将 `buildSchemaOptions` 对象的 `numberScalarMode` 设置为 `'integer'`，如下所示：

```typescript
GraphQLModule.forRoot({
  buildSchemaOptions: {
    numberScalarMode: 'integer',
  }
}),
```

此外，你还可以创建自定义标量。

#### 覆盖默认标量（Override a default scalar）

要为 `Date` 标量创建自定义实现，只需创建一个新类。

```typescript
import { Scalar, CustomScalar } from '@nestjs/graphql';
import { Kind, ValueNode } from 'graphql';

@Scalar('Date', () => Date)
export class DateScalar implements CustomScalar<number, Date> {
  description = 'Date custom scalar type';

  parseValue(value: number): Date {
    return new Date(value); // 来自客户端的值
  }

  serialize(value: Date): number {
    return value.getTime(); // 发送给客户端的值
  }

  parseLiteral(ast: ValueNode): Date {
    if (ast.kind === Kind.INT) {
      return new Date(ast.value);
    }
    return null;
  }
}
```

完成此操作后，将 `DateScalar` 注册为提供者。

```typescript
@Module({
  providers: [DateScalar],
})
export class CommonModule {}
```

现在我们可以在类中使用 `Date` 类型。

```typescript
@Field()
creationDate: Date;
```

#### 导入自定义标量（Import a custom scalar）

要使用自定义标量，需导入并将其注册为解析器。我们以 `graphql-type-json` 包作为演示示例。这个npm包定义了一个 `JSON` 的GraphQL标量类型。

首先安装该包：

```bash
$ npm i --save graphql-type-json
```

安装完成后，我们将一个自定义解析器传递给 `forRoot()` 方法：

```typescript
import GraphQLJSON from 'graphql-type-json';

@Module({
  imports: [
    GraphQLModule.forRoot({
      resolvers: { JSON: GraphQLJSON },
    }),
  ],
})
export class AppModule {}
```

现在我们可以在类中使用 `JSON` 类型。

```typescript
@Field(() => GraphQLJSON)
info: JSON;
```

如需更多有用的标量，请查看 [graphql-scalars](https://www.npmjs.com/package/graphql-scalars) 包。

#### 创建自定义标量（Create a custom scalar）

要定义一个自定义标量，需创建一个新的 `GraphQLScalarType` 实例。我们将创建一个自定义的 `UUID` 标量。

```typescript
const regex = /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i;

function validate(uuid: unknown): string | never {
  if (typeof uuid !== 'string' || !regex.test(uuid)) {
    throw new Error('invalid uuid');
  }
  return uuid;
}

export const CustomUuidScalar = new GraphQLScalarType({
  name: 'UUID',
  description: '一个简单的UUID解析器',
  serialize: (value) => validate(value),
  parseValue: (value) => validate(value),
  parseLiteral: (ast) => validate(ast.value),
});
```

我们将一个自定义解析器传递给 `forRoot()` 方法：

```typescript
@Module({
  imports: [
    GraphQLModule.forRoot({
      resolvers: { UUID: CustomUuidScalar },
    }),
  ],
})
export class AppModule {}
```

现在我们可以在类中使用 `UUID` 类型。

```typescript
@Field(() => CustomUuidScalar)
uuid: string;
```

#### 模式优先（Schema first）

要定义一个自定义标量（进一步阅读标量[此处](https://www.apollographql.com/docs/graphql-tools/scalars.html)），请创建一个类型定义和一个专用解析器。此处（如官方文档所示），我们使用 `graphql-type-json` 包进行演示。这个npm包定义了一个 `JSON` 的GraphQL标量类型。

首先安装该包：

```bash
$ npm i --save graphql-type-json
```

安装完成后，我们将一个自定义解析器传递给 `forRoot()` 方法：

```typescript
import GraphQLJSON from 'graphql-type-json';

@Module({
  imports: [
    GraphQLModule.forRoot({
      typePaths: ['./**/*.graphql'],
      resolvers: { JSON: GraphQLJSON },
    }),
  ],
})
export class AppModule {}
```

现在我们可以在类型定义中使用 `JSON` 标量：

```graphql
scalar JSON

type Foo {
  field: JSON
}
```

另一种定义标量类型的方法是创建一个简单类。假设我们想使用 `Date` 类型来增强我们的schema。

```typescript
import { Scalar, CustomScalar } from '@nestjs/graphql';
import { Kind, ValueNode } from 'graphql';

@Scalar('Date')
export class DateScalar implements CustomScalar<number, Date> {
  description = 'Date custom scalar type';

  parseValue(value: number): Date {
    return new Date(value); // 来自客户端的值
  }

  serialize(value: Date): number {
    return value.getTime(); // 发送给客户端的值
  }

  parseLiteral(ast: ValueNode): Date {
    if (ast.kind === Kind.INT) {
      return new Date(ast.value);
    }
    return null;
  }
}
```

完成此操作后，将 `DateScalar` 注册为提供者。

```typescript
@Module({
  providers: [DateScalar],
})
export class CommonModule {}
```

现在我们可以在类型定义中使用 `Date` 标量。

```graphql
scalar Date
```

默认情况下，所有标量的生成TypeScript定义是 `any` —— 这并不是特别类型安全。  
但是，你可以在指定如何生成类型时配置Nest如何为你的自定义标量生成类型定义：

```typescript
import { GraphQLDefinitionsFactory } from '@nestjs/graphql';
import { join } from 'path';

const definitionsFactory = new GraphQLDefinitionsFactory();

definitionsFactory.generate({
  typePaths: ['./src/**/*.graphql'],
  path: join(process.cwd(), 'src/graphql.ts'),
  outputAs: 'class',
  defaultScalarType: 'unknown',
  customScalarTypeMapping: {
    DateTime: 'Date',
    BigNumber: '_BigNumber',
  },
  additionalHeader: "import _BigNumber from 'bignumber.js'",
});
```

> info **提示** 或者，你也可以使用类型引用，例如：`DateTime: Date`。在这种情况下，`GraphQLDefinitionsFactory` 将提取指定类型的 `name` 属性（`Date.name`）来生成TS定义。注意：对于非内置类型（自定义类型），需要添加导入语句。

现在，假设有以下GraphQL自定义标量类型：

```graphql
scalar DateTime
scalar BigNumber
scalar Payload
```

我们现在将在 `src/graphql.ts` 中看到如下生成的TypeScript定义：

```typescript
import _BigNumber from 'bignumber.js';

export type DateTime = Date;
export type BigNumber = _BigNumber;
export type Payload = unknown;
```

在此处，我们使用了 `customScalarTypeMapping` 属性来提供我们希望为自定义标量声明的类型的映射。我们还提供了 `additionalHeader` 属性以便为这些类型定义添加所需的导入语句。最后，我们添加了 `defaultScalarType` 为 `'unknown'`，这样所有未在 `customScalarTypeMapping` 中指定的自定义标量将被别名为 `unknown` 而不是 `any`（从TypeScript 3.0起，[官方推荐](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-0.html#new-unknown-top-type)使用 `unknown` 以增强类型安全）。

> info **提示** 注意我们从 `bignumber.js` 导入了 `_BigNumber`；这是为了避免[循环类型引用](https://github.com/Microsoft/TypeScript/issues/12525#issuecomment-263166239)。