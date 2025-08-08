### 字段中间件

> warning **警告** 本章仅适用于代码优先的方法。

字段中间件允许你在字段被解析之前或之后运行任意代码。可以使用字段中间件转换字段的结果、验证字段的参数，甚至检查字段级别的角色（例如，访问某个字段时需要执行中间件函数）。

你可以将多个中间件函数连接到一个字段。在这种情况下，它们将沿着链式结构依次调用，前一个中间件决定是否调用下一个中间件。`middleware` 数组中中间件函数的顺序非常重要。第一个解析器是“最外层”，因此它最先执行，最后结束（类似于 `graphql-middleware` 包）。第二个解析器是“次外层”，因此它第二个执行，倒数第二个结束。

#### 入门

让我们从创建一个简单的中间件开始，它会在字段值返回给客户端之前将其记录下来：

```typescript
import { FieldMiddleware, MiddlewareContext, NextFn } from '@nestjs/graphql';

const loggerMiddleware: FieldMiddleware = async (
  ctx: MiddlewareContext,
  next: NextFn,
) => {
  const value = await next();
  console.log(value);
  return value;
};
```

> info **提示** `MiddlewareContext` 是一个对象，它包含通常由 GraphQL 解析器函数接收的相同参数（`{{ '{' }} source, args, context, info {{ '}' }}`），而 `NextFn` 是一个函数，它允许你执行堆栈中的下一个中间件（绑定到该字段）或实际的字段解析器。

> warning **警告** 字段中间件函数不能注入依赖项，也无法访问 Nest 的 DI 容器，因为它们的设计非常轻量，不应该执行任何可能耗时的操作（如从数据库检索数据）。如果你需要调用外部服务/查询数据源，应该在一个绑定到根查询/变更处理器的守卫/拦截器中执行，并将其分配给 `context` 对象，你可以在字段中间件内访问该对象（具体来说，是从 `MiddlewareContext` 对象中访问）。

请注意，字段中间件必须符合 `FieldMiddleware` 接口。在上面的示例中，我们首先运行 `next()` 函数（执行实际的字段解析器并返回字段值），然后将该值记录到终端。此外，中间件函数返回的值会完全覆盖之前的值，而我们不想进行任何更改，因此我们只是返回了原始值。

有了这些，我们可以在 `@Field()` 装饰器中直接注册我们的中间件，如下所示：

```typescript
@ObjectType()
export class Recipe {
  @Field({ middleware: [loggerMiddleware] })
  title: string;
}
```

现在，每当我们请求 `Recipe` 对象类型的 `title` 字段时，原始字段的值将被记录到控制台。

> info **提示** 要了解如何使用[扩展](/graphql/extensions)功能实现字段级别的权限系统，请查看此[部分](/graphql/extensions#using-custom-metadata)。

> warning **警告** 字段中间件只能应用于 `ObjectType` 类。更多细节，请查看此[问题](https://github.com/nestjs/graphql/issues/2446)。

此外，如上所述，我们可以在中间件函数内部控制字段的值。为了演示，我们来将食谱标题（如果存在）大写：

```typescript
const value = await next();
return value?.toUpperCase();
```

在这种情况下，每次请求字段时，标题将自动转为大写。

同样，你可以将字段中间件绑定到自定义字段解析器（使用 `@ResolveField()` 装饰器标注的方法），如下所示：

```typescript
@ResolveField(() => String, { middleware: [loggerMiddleware] })
title() {
  return 'Placeholder';
}
```

> warning **警告** 如果在字段解析器级别启用了增强器（[了解更多](/graphql/other-features#execute-enhancers-at-the-field-resolver-level)），字段中间件函数将在绑定到该方法的任何拦截器、守卫等之前运行（但在查询或变更处理器注册的根级别增强器之后）。

#### 全局字段中间件

除了将中间件直接绑定到特定字段外，你还可以全局注册一个或多个中间件函数。在这种情况下，它们将自动连接到你的所有对象类型的字段上。

```typescript
GraphQLModule.forRoot({
  autoSchemaFile: 'schema.gql',
  buildSchemaOptions: {
    fieldMiddleware: [loggerMiddleware],
  },
}),
```

> info **提示** 全局注册的字段中间件函数将在本地注册的中间件（直接绑定到特定字段的中间件）之前执行。