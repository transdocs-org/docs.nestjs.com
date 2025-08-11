### 其他功能

在 GraphQL 的世界中，关于如何处理诸如 **认证** 或操作的 **副作用** 等问题有很多争论。我们应该在业务逻辑内部处理这些问题吗？我们是否应该使用高阶函数，通过授权逻辑增强查询和变更？或者我们是否应该使用 [schema 指令](https://www.apollographql.com/docs/apollo-server/schema/directives/)？这些问题没有一种放之四海而皆准的答案。

Nest 通过其跨平台特性如 [守卫（guards）](/guards) 和 [拦截器（interceptors）](/interceptors) 来帮助解决这些问题。其理念是减少冗余，并提供有助于创建结构良好、可读性强且一致的应用程序的工具。

#### 概述

你可以像在任何 RESTful 应用程序中一样，在 GraphQL 中使用标准的 [守卫（guards）](/guards)、[拦截器（interceptors）](/interceptors)、[异常过滤器（filters）](/exception-filters) 和 [管道（pipes）](/pipes)。此外，你还可以通过利用 [自定义装饰器（custom decorators）](/custom-decorators) 功能轻松创建自己的装饰器。让我们看一下一个 GraphQL 查询处理程序的示例。

```typescript
@Query('author')
@UseGuards(AuthGuard)
async getAuthor(@Args('id', ParseIntPipe) id: number) {
  return this.authorsService.findOneById(id);
}
```

如你所见，GraphQL 与 HTTP REST 处理程序一样，同样支持守卫和管道。因此，你可以将身份验证逻辑移至守卫中；你甚至可以在 REST 和 GraphQL API 接口之间重用相同的守卫类。同样，拦截器在两种类型的应用程序中也以相同方式工作：

```typescript
@Mutation()
@UseInterceptors(EventsInterceptor)
async upvotePost(@Args('postId') postId: number) {
  return this.postsService.upvoteById({ id: postId });
}
```

#### 执行上下文

由于 GraphQL 在传入请求中接收到不同类型的请求数据，守卫和拦截器接收到的 [执行上下文](https://docs.nestjs.com/fundamentals/execution-context) 在 GraphQL 与 REST 之间略有不同。GraphQL 解析器具有一组独特的参数：`root`、`args`、`context` 和 `info`。因此，守卫和拦截器必须将通用的 `ExecutionContext` 转换为 `GqlExecutionContext`。这非常简单：

```typescript
import { CanActivate, ExecutionContext, Injectable } from '@nestjs/common';
import { GqlExecutionContext } from '@nestjs/graphql';

@Injectable()
export class AuthGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const ctx = GqlExecutionContext.create(context);
    return true;
  }
}
```

通过 `GqlExecutionContext.create()` 返回的 GraphQL 上下文对象为每个 GraphQL 解析器参数（如 `getArgs()`、`getContext()` 等）暴露了一个 **get** 方法。一旦转换完成，我们就可以轻松地提取当前请求的任何 GraphQL 参数。

#### 异常过滤器

Nest 的标准 [异常过滤器（exception filters）](/exception-filters) 也兼容 GraphQL 应用程序。与 `ExecutionContext` 一样，GraphQL 应用程序应将 `ArgumentsHost` 对象转换为 `GqlArgumentsHost` 对象。

```typescript
@Catch(HttpException)
export class HttpExceptionFilter implements GqlExceptionFilter {
  catch(exception: HttpException, host: ArgumentsHost) {
    const gqlHost = GqlArgumentsHost.create(host);
    return exception;
  }
}
```

> info **提示** `GqlExceptionFilter` 和 `GqlArgumentsHost` 都从 `@nestjs/graphql` 包中导入。

请注意，与 REST 不同，你不能使用原生的 `response` 对象来生成响应。

#### 自定义装饰器

如前所述，[自定义装饰器（custom decorators）](/custom-decorators) 在 GraphQL 解析器中也按预期工作。

```typescript
export const User = createParamDecorator(
  (data: unknown, ctx: ExecutionContext) =>
    GqlExecutionContext.create(ctx).getContext().user,
);
```

如下所示使用 `@User()` 自定义装饰器：

```typescript
@Mutation()
async upvotePost(
  @User() user: UserEntity,
  @Args('postId') postId: number,
) {}
```

> info **提示** 在上面的示例中，我们假设 `user` 对象已分配给你的 GraphQL 应用程序的上下文。

#### 在字段解析器级别执行增强器

在 GraphQL 上下文中，Nest 不会在字段级别运行 **增强器**（拦截器、守卫和过滤器的通用名称）[参见此问题](https://github.com/nestjs/graphql/issues/320#issuecomment-511193229)：它们仅对顶级的 `@Query()` / `@Mutation()` 方法运行。你可以通过在 `GqlModuleOptions` 中设置 `fieldResolverEnhancers` 选项，指示 Nest 对使用 `@ResolveField()` 注解的方法执行拦截器、守卫或过滤器。根据需要传入一个包含 `'interceptors'`、`'guards'` 和/或 `'filters'` 的列表：

```typescript
GraphQLModule.forRoot({
  fieldResolverEnhancers: ['interceptors']
}),
```

> **警告** 当你返回大量记录并且字段解析器执行数千次时，为字段解析器启用增强器可能会导致性能问题。因此，当你启用 `fieldResolverEnhancers` 时，建议你跳过字段解析器不需要的增强器的执行。你可以使用以下辅助函数来实现：

```typescript
export function isResolvingGraphQLField(context: ExecutionContext): boolean {
  if (context.getType<GqlContextType>() === 'graphql') {
    const gqlContext = GqlExecutionContext.create(context);
    const info = gqlContext.getInfo();
    const parentType = info.parentType.name;
    return parentType !== 'Query' && parentType !== 'Mutation';
  }
  return false;
}
```

#### 创建自定义驱动

Nest 提供了两个官方驱动：`@nestjs/apollo` 和 `@nestjs/mercurius`，以及一个允许开发者构建新的 **自定义驱动** 的 API。使用自定义驱动，你可以集成任何 GraphQL 库或扩展现有集成，添加额外功能。

例如，要集成 `express-graphql` 包，你可以创建如下驱动类：

```typescript
import { AbstractGraphQLDriver, GqlModuleOptions } from '@nestjs/graphql';
import { graphqlHTTP } from 'express-graphql';

class ExpressGraphQLDriver extends AbstractGraphQLDriver {
  async start(options: GqlModuleOptions<any>): Promise<void> {
    options = await this.graphQlFactory.mergeWithSchema(options);

    const { httpAdapter } = this.httpAdapterHost;
    httpAdapter.use(
      '/graphql',
      graphqlHTTP({
        schema: options.schema,
        graphiql: true,
      }),
    );
  }

  async stop() {}
}
```

然后像这样使用它：

```typescript
GraphQLModule.forRoot({
  driver: ExpressGraphQLDriver,
});
```