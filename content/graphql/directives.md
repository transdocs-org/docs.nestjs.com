### 指令（Directives）

指令可以附加到字段或片段包含上，并且可以按照服务器期望的任何方式影响查询的执行（更多内容请[点击此处](https://graphql.org/learn/queries/#directives)了解）。GraphQL 规范提供了以下默认指令：

- `@include(if: Boolean)` - 仅当参数为 `true` 时，才将该字段包含在结果中
- `@skip(if: Boolean)` - 如果参数为 `true`，则跳过该字段
- `@deprecated(reason: String)` - 使用消息将字段标记为已弃用

指令是一个以 `@` 字符开头的标识符，可以后接一组命名参数，这些参数可以出现在 GraphQL 查询和模式语言中的几乎任何元素之后。

#### 自定义指令

要告诉 Apollo/Mercurius 在遇到你的指令时应如何处理，你可以创建一个转换函数（transformer function）。该函数使用 `mapSchema` 函数来遍历 schema 中的各个位置（字段定义、类型定义等），并执行相应的转换。

```typescript
import { getDirective, MapperKind, mapSchema } from '@graphql-tools/utils';
import { defaultFieldResolver, GraphQLSchema } from 'graphql';

export function upperDirectiveTransformer(
  schema: GraphQLSchema,
  directiveName: string,
) {
  return mapSchema(schema, {
    [MapperKind.OBJECT_FIELD]: (fieldConfig) => {
      const upperDirective = getDirective(
        schema,
        fieldConfig,
        directiveName,
      )?.[0];

      if (upperDirective) {
        const { resolve = defaultFieldResolver } = fieldConfig;

        // 替换原始解析器函数，使其在调用原始解析器之后
        // 将结果转换为大写形式
        fieldConfig.resolve = async function (source, args, context, info) {
          const result = await resolve(source, args, context, info);
          if (typeof result === 'string') {
            return result.toUpperCase();
          }
          return result;
        };
        return fieldConfig;
      }
    },
  });
}
```

现在，在 `GraphQLModule#forRoot` 方法中使用 `transformSchema` 函数应用 `upperDirectiveTransformer` 转换函数：

```typescript
GraphQLModule.forRoot({
  // ...
  transformSchema: (schema) => upperDirectiveTransformer(schema, 'upper'),
});
```

一旦注册完成，就可以在 schema 中使用 `@upper` 指令。不过，应用指令的方式取决于你使用的方法（代码优先或模式优先）。

#### 代码优先（Code first）

在代码优先方法中，使用 `@Directive()` 装饰器来应用指令。

```typescript
@Directive('@upper')
@Field()
title: string;
```

> info **提示** `@Directive()` 装饰器是从 `@nestjs/graphql` 包中导出的。

指令可以应用于字段、字段解析器、输入类型和对象类型，以及查询、变更和订阅。下面是一个在查询处理器级别应用指令的示例：

```typescript
@Directive('@deprecated(reason: "此查询将在下一个版本中移除")')
@Query(() => Author, { name: 'author' })
async getAuthor(@Args({ name: 'id', type: () => Int }) id: number) {
  return this.authorsService.findOneById(id);
}
```

> warn **警告** 通过 `@Directive()` 装饰器应用的指令不会反映在生成的 schema 定义文件中。

最后，确保在 `GraphQLModule` 中声明指令，如下所示：

```typescript
GraphQLModule.forRoot({
  // ...,
  transformSchema: schema => upperDirectiveTransformer(schema, 'upper'),
  buildSchemaOptions: {
    directives: [
      new GraphQLDirective({
        name: 'upper',
        locations: [DirectiveLocation.FIELD_DEFINITION],
      }),
    ],
  },
}),
```

> info **提示** `GraphQLDirective` 和 `DirectiveLocation` 都是从 `graphql` 包中导出的。

#### 模式优先（Schema first）

在模式优先方法中，直接在 SDL 中应用指令。

```graphql
directive @upper on FIELD_DEFINITION

type Post {
  id: Int!
  title: String! @upper
  votes: Int
}
```