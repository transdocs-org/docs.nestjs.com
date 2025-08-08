### 联合类型（Unions）

联合类型与接口非常相似，但它们不能在类型之间定义任何公共字段（了解更多请参见[此处](https://graphql.org/learn/schema/#union-types)）。联合类型对于从单个字段返回互斥的数据类型非常有用。

#### 代码优先（Code first）

要定义一个 GraphQL 联合类型，我们必须先定义该联合所包含的类。根据 [Apollo 文档](https://www.apollographql.com/docs/apollo-server/schema/unions-interfaces/#union-type) 的示例，我们创建两个类：首先是 `Book`：

```typescript
import { Field, ObjectType } from '@nestjs/graphql';

@ObjectType()
export class Book {
  @Field()
  title: string;
}
```

然后是 `Author`：

```typescript
import { Field, ObjectType } from '@nestjs/graphql';

@ObjectType()
export class Author {
  @Field()
  name: string;
}
```

完成这些类定义后，使用 `@nestjs/graphql` 包导出的 `createUnionType` 函数注册 `ResultUnion` 联合类型：

```typescript
export const ResultUnion = createUnionType({
  name: 'ResultUnion',
  types: () => [Author, Book] as const,
});
```

> warning **警告** `createUnionType` 函数中 `types` 属性返回的数组应使用 `const` 断言。如果没有使用 `const` 断言，在编译时会生成错误的声明文件，并且在其他项目中使用时会导致错误。

现在我们可以在查询中引用 `ResultUnion`：

```typescript
@Query(() => [ResultUnion])
search(): Array<typeof ResultUnion> {
  return [new Author(), new Book()];
}
```

这将生成如下的 GraphQL 模式（SDL）部分：

```graphql
type Author {
  name: String!
}

type Book {
  title: String!
}

union ResultUnion = Author | Book

type Query {
  search: [ResultUnion!]!
}
```

该库生成的默认 `resolveType()` 函数将根据解析器方法返回的值提取类型。这意味着必须返回类实例，而不是普通的 JavaScript 对象字面量。

要提供自定义的 `resolveType()` 函数，请将 `resolveType` 属性添加到传递给 `createUnionType()` 函数的选项对象中，如下所示：

```typescript
export const ResultUnion = createUnionType({
  name: 'ResultUnion',
  types: () => [Author, Book] as const,
  resolveType(value) {
    if (value.name) {
      return Author;
    }
    if (value.title) {
      return Book;
    }
    return null;
  },
});
```

#### 模式优先（Schema first）

在模式优先的方式中，只需使用 SDL 定义一个 GraphQL 联合类型：

```graphql
type Author {
  name: String!
}

type Book {
  title: String!
}

union ResultUnion = Author | Book
```

然后，你可以使用类型生成功能（如 [快速入门](/graphql/quick-start) 章节中所示）生成相应的 TypeScript 定义：

```typescript
export class Author {
  name: string;
}

export class Book {
  title: string;
}

export type ResultUnion = Author | Book;
```

联合类型在解析器映射中需要额外的 `__resolveType` 字段来确定该联合类型应解析为何种类型。此外，注意 `ResultUnionResolver` 类必须在任意模块中注册为提供者。我们创建一个 `ResultUnionResolver` 类并定义 `__resolveType` 方法：

```typescript
@Resolver('ResultUnion')
export class ResultUnionResolver {
  @ResolveField()
  __resolveType(value) {
    if (value.name) {
      return 'Author';
    }
    if (value.title) {
      return 'Book';
    }
    return null;
  }
}
```

> info **提示** 所有装饰器均从 `@nestjs/graphql` 包中导出。

### 枚举（Enums）

枚举类型是一种特殊的标量类型，其取值被限制在一组特定的允许值中（了解更多请参见[此处](https://graphql.org/learn/schema/#enumeration-types)）。这使得你可以：

- 验证该类型的所有参数是否为允许的值之一
- 通过类型系统表明某个字段的取值始终是有限集合中的一个

#### 代码优先（Code first）

在代码优先方式中，你只需创建一个 TypeScript 枚举即可定义一个 GraphQL 枚举类型：

```typescript
export enum AllowedColor {
  RED,
  GREEN,
  BLUE,
}
```

完成定义后，使用 `@nestjs/graphql` 包导出的 `registerEnumType` 函数注册 `AllowedColor` 枚举：

```typescript
registerEnumType(AllowedColor, {
  name: 'AllowedColor',
});
```

现在你可以在类型中引用 `AllowedColor`：

```typescript
@Field(type => AllowedColor)
favoriteColor: AllowedColor;
```

这将生成如下的 GraphQL 模式（SDL）部分：

```graphql
enum AllowedColor {
  RED
  GREEN
  BLUE
}
```

要为枚举添加描述，请将 `description` 属性传递给 `registerEnumType()` 函数：

```typescript
registerEnumType(AllowedColor, {
  name: 'AllowedColor',
  description: '支持的颜色列表。',
});
```

要为枚举值添加描述，或标记某个值为已弃用，请使用 `valuesMap` 属性，如下所示：

```typescript
registerEnumType(AllowedColor, {
  name: 'AllowedColor',
  description: '支持的颜色列表。',
  valuesMap: {
    RED: {
      description: '默认颜色。',
    },
    BLUE: {
      deprecationReason: '太蓝了。',
    },
  },
});
```

这将生成如下的 GraphQL 模式（SDL）：

```graphql
"""
支持的颜色列表。
"""
enum AllowedColor {
  """
  默认颜色。
  """
  RED
  GREEN
  BLUE @deprecated(reason: "太蓝了。")
}
```

#### 模式优先（Schema first）

在模式优先方式中，只需使用 SDL 定义一个 GraphQL 枚举：

```graphql
enum AllowedColor {
  RED
  GREEN
  BLUE
}
```

然后你可以使用类型生成功能（如 [快速入门](/graphql/quick-start) 章节中所示）生成相应的 TypeScript 定义：

```typescript
export enum AllowedColor {
  RED
  GREEN
  BLUE
}
```

有时后端在内部使用的枚举值与公共 API 中的值不同。例如 API 中使用的是 `RED`，但在解析器中我们可能使用 `#f00`（了解更多请参见[此处](https://www.apollographql.com/docs/apollo-server/schema/scalars-enums/#internal-values)）。为此，为 `AllowedColor` 枚举声明一个解析器对象：

```typescript
export const allowedColorResolver: Record<keyof typeof AllowedColor, any> = {
  RED: '#f00',
};
```

> info **提示** 所有装饰器均从 `@nestjs/graphql` 包中导出。

然后将此解析器对象与 `GraphQLModule#forRoot()` 方法的 `resolvers` 属性一起使用，如下所示：

```typescript
GraphQLModule.forRoot({
  resolvers: {
    AllowedColor: allowedColorResolver,
  },
});
```