### 订阅

除了使用查询来获取数据和使用变更来修改数据外，GraphQL 规范还支持第三种操作类型，称为 `subscription`。GraphQL 订阅是一种将数据从服务器推送到客户端的方式，客户端可以选择监听来自服务器的实时消息。订阅类似于查询，因为它们都指定了一组要传递给客户端的字段，但不同的是，订阅不会立即返回单个答案，而是打开一个通道，每当服务器上发生特定事件时，都会将结果发送给客户端。

订阅的一个常见使用场景是通知客户端有关特定事件，例如新对象的创建、字段更新等（[了解更多](https://www.apollographql.com/docs/react/data/subscriptions)）。

#### 使用 Apollo 驱动启用订阅

要启用订阅，请将 `installSubscriptionHandlers` 属性设置为 `true`。

```typescript
GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  installSubscriptionHandlers: true,
}),
```

> warning **警告** `installSubscriptionHandlers` 配置选项已从 Apollo Server 的最新版本中移除，并且在本包中也将很快被弃用。默认情况下，`installSubscriptionHandlers` 将回退使用 `subscriptions-transport-ws`（[了解更多](https://github.com/apollographql/subscriptions-transport-ws)），但我们强烈建议改用 `graphql-ws`（[了解更多](https://github.com/enisdenjo/graphql-ws)）库。

要改用 `graphql-ws` 包，请使用以下配置：

```typescript
GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  subscriptions: {
    'graphql-ws': true
  },
}),
```

> info **提示** 你也可以同时使用两个包（`subscriptions-transport-ws` 和 `graphql-ws`），例如用于向后兼容。

#### 代码优先

要使用代码优先方法创建订阅，我们使用 `@Subscription()` 装饰器（从 `@nestjs/graphql` 包导出）和 `graphql-subscriptions` 包中的 `PubSub` 类，该类提供了一个简单的 **发布/订阅 API**。

以下订阅处理程序通过调用 `PubSub#asyncIterableIterator` 来**订阅**一个事件。此方法接受一个参数，即 `triggerName`，它对应于一个事件主题名称。

```typescript
const pubSub = new PubSub();

@Resolver(() => Author)
export class AuthorResolver {
  // ...
  @Subscription(() => Comment)
  commentAdded() {
    return pubSub.asyncIterableIterator('commentAdded');
  }
}
```

> info **提示** 所有装饰器都从 `@nestjs/graphql` 包导出，而 `PubSub` 类从 `graphql-subscriptions` 包导出。

> warning **注意** `PubSub` 是一个暴露简单 `publish` 和 `subscribe` API 的类。[了解更多](https://www.apollographql.com/docs/graphql-subscriptions/setup.html)。请注意，Apollo 文档警告默认实现不适合生产环境（[了解更多](https://github.com/apollographql/graphql-subscriptions#getting-started-with-your-first-subscription)）。生产应用程序应使用由外部存储支持的 `PubSub` 实现（[了解更多](https://github.com/apollographql/graphql-subscriptions#pubsub-implementations)）。

这将生成以下 GraphQL 模式的部分 SDL：

```graphql
type Subscription {
  commentAdded(): Comment!
}
```

请注意，根据定义，订阅返回一个对象，该对象只有一个顶级属性，其键是订阅的名称。该名称要么从订阅处理程序方法的名称继承（例如上面的 `commentAdded`），要么通过将带有键 `name` 的选项作为 `@Subscription()` 装饰器的第二个参数显式提供，如下所示。

```typescript
@Subscription(() => Comment, {
  name: 'commentAdded',
})
subscribeToCommentAdded() {
  return pubSub.asyncIterableIterator('commentAdded');
}
```

此构造生成的 SDL 与前面的代码示例相同，但允许我们将方法名与订阅解耦。

#### 发布

现在，要发布事件，我们使用 `PubSub#publish` 方法。这通常在变更中使用，当对象图的某部分发生更改时触发客户端更新。例如：

```typescript
@@filename(posts/posts.resolver)
@Mutation(() => Comment)
async addComment(
  @Args('postId', { type: () => Int }) postId: number,
  @Args('comment', { type: () => Comment }) comment: CommentInput,
) {
  const newComment = this.commentsService.addComment({ id: postId, comment });
  pubSub.publish('commentAdded', { commentAdded: newComment });
  return newComment;
}
```

`PubSub#publish` 方法将 `triggerName`（再次，将其视为事件主题名称）作为第一个参数，将事件负载作为第二个参数。如前所述，订阅按定义返回一个值，该值具有形状。再次查看我们 `commentAdded` 订阅生成的 SDL：

```graphql
type Subscription {
  commentAdded(): Comment!
}
```

这告诉我们订阅必须返回一个对象，其顶级属性名为 `commentAdded`，其值为 `Comment` 对象。需要注意的重要一点是，`PubSub#publish` 方法发出的事件负载的形状必须与订阅预期返回值的形状相对应。因此，在我们上面的示例中，`pubSub.publish('commentAdded', {{ '{' }} commentAdded: newComment {{ '}' }})` 语句发布了一个 `commentAdded` 事件，具有正确形状的负载。如果这些形状不匹配，你的订阅将在 GraphQL 验证阶段失败。

#### 过滤订阅

要过滤特定事件，请将 `filter` 属性设置为一个过滤函数。该函数的行为类似于传递给数组 `filter` 的函数。它接受两个参数：`payload` 包含事件负载（由事件发布者发送），`variables` 接收订阅请求期间传递的任何参数。它返回一个布尔值，决定是否应将此事件发布给客户端监听者。

```typescript
@Subscription(() => Comment, {
  filter: (payload, variables) =>
    payload.commentAdded.title === variables.title,
})
commentAdded(@Args('title') title: string) {
  return pubSub.asyncIterableIterator('commentAdded');
}
```

#### 变更订阅负载

要变更发布的事件负载，请将 `resolve` 属性设置为一个函数。该函数接收事件负载（由事件发布者发送），并返回适当的值。

```typescript
@Subscription(() => Comment, {
  resolve: value => value,
})
commentAdded() {
  return pubSub.asyncIterableIterator('commentAdded');
}
```

> warning **注意** 如果你使用 `resolve` 选项，应返回未包装的负载（例如，在我们的示例中直接返回 `newComment` 对象，而不是 `{{ '{' }} commentAdded: newComment {{ '}' }}` 对象）。

如果你需要访问注入的提供者（例如，使用外部服务验证数据），请使用以下结构。

```typescript
@Subscription(() => Comment, {
  resolve(this: AuthorResolver, value) {
    // "this" 指向 "AuthorResolver" 的实例
    return value;
  }
})
commentAdded() {
  return pubSub.asyncIterableIterator('commentAdded');
}
```

相同的结构适用于过滤器：

```typescript
@Subscription(() => Comment, {
  filter(this: AuthorResolver, payload, variables) {
    // "this" 指向 "AuthorResolver" 的实例
    return payload.commentAdded.title === variables.title;
  }
})
commentAdded() {
  return pubSub.asyncIterableIterator('commentAdded');
}
```

#### 模式优先

要在 Nest 中创建等效的订阅，我们将使用 `@Subscription()` 装饰器。

```typescript
const pubSub = new PubSub();

@Resolver('Author')
export class AuthorResolver {
  // ...
  @Subscription()
  commentAdded() {
    return pubSub.asyncIterableIterator('commentAdded');
  }
}
```

要根据上下文和参数过滤特定事件，请设置 `filter` 属性。

```typescript
@Subscription('commentAdded', {
  filter: (payload, variables) =>
    payload.commentAdded.title === variables.title,
})
commentAdded() {
  return pubSub.asyncIterableIterator('commentAdded');
}
```

要变更发布的负载，我们可以使用 `resolve` 函数。

```typescript
@Subscription('commentAdded', {
  resolve: value => value,
})
commentAdded() {
  return pubSub.asyncIterableIterator('commentAdded');
}
```

如果你需要访问注入的提供者（例如，使用外部服务验证数据），请使用以下结构：

```typescript
@Subscription('commentAdded', {
  resolve(this: AuthorResolver, value) {
    // "this" 指向 "AuthorResolver" 的实例
    return value;
  }
})
commentAdded() {
  return pubSub.asyncIterableIterator('commentAdded');
}
```

相同的结构适用于过滤器：

```typescript
@Subscription('commentAdded', {
  filter(this: AuthorResolver, payload, variables) {
    // "this" 指向 "AuthorResolver" 的实例
    return payload.commentAdded.title === variables.title;
  }
})
commentAdded() {
  return pubSub.asyncIterableIterator('commentAdded');
}
```

最后一步是更新类型定义文件。

```graphql
type Author {
  id: Int!
  firstName: String
  lastName: String
  posts: [Post]
}

type Post {
  id: Int!
  title: String
  votes: Int
}

type Query {
  author(id: Int!): Author
}

type Comment {
  id: String
  content: String
}

type Subscription {
  commentAdded(title: String!): Comment
}
```

通过此操作，我们创建了一个名为 `commentAdded(title: String!): Comment` 的订阅。你可以在此处找到完整的示例实现：[链接](https://github.com/nestjs/nest/blob/master/sample/12-graphql-schema-first)。

#### PubSub

我们在上面实例化了一个本地 `PubSub` 实例。首选方法是将 `PubSub` 定义为 [提供者](/fundamentals/custom-providers)，并通过构造函数注入它（使用 `@Inject()` 装饰器）。这允许我们在整个应用程序中重用该实例。例如，定义一个提供者如下，然后在需要时注入 `'PUB_SUB'`。

```typescript
{
  provide: 'PUB_SUB',
  useValue: new PubSub(),
}
```

#### 自定义订阅服务器

要自定义订阅服务器（例如更改路径），请使用 `subscriptions` 选项属性。

```typescript
GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  subscriptions: {
    'subscriptions-transport-ws': {
      path: '/graphql'
    },
  }
}),
```

如果你使用 `graphql-ws` 包进行订阅，请将 `subscriptions-transport-ws` 键替换为 `graphql-ws`，如下所示：

```typescript
GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  subscriptions: {
    'graphql-ws': {
      path: '/graphql'
    },
  }
}),
```

#### WebSocket 上的身份验证

检查用户是否已通过身份验证可以在 `onConnect` 回调函数中完成，你可以在 `subscriptions` 选项中指定该函数。

`onConnect` 将接收作为第一个参数的 `connectionParams`，该参数传递给 `SubscriptionClient`（[了解更多](https://www.apollographql.com/docs/react/data/subscriptions/#5-authenticate-over-websocket-optional)）。

```typescript
GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  subscriptions: {
    'subscriptions-transport-ws': {
      onConnect: (connectionParams) => {
        const authToken = connectionParams.authToken;
        if (!isValid(authToken)) {
          throw new Error('Token is not valid');
        }
        // 从 token 中提取用户信息
        const user = parseToken(authToken);
        // 返回用户信息以稍后添加到上下文中
        return { user };
      },
    }
  },
  context: ({ connection }) => {
    // connection.context 将等于 "onConnect" 回调返回的内容
  },
}),
```

在此示例中，`authToken` 仅在客户端首次建立连接时发送一次。
使用此连接进行的所有订阅都将具有相同的 `authToken`，从而具有相同的用户信息。

> warning **注意** `subscriptions-transport-ws` 中存在一个 bug，允许连接跳过 `onConnect` 阶段（[了解更多](https://github.com/apollographql/subscriptions-transport-ws/issues/349)）。你不应假设在用户启动订阅时调用了 `onConnect`，并始终检查 `context` 是否已填充。

如果你使用的是 `graphql-ws` 包，则 `onConnect` 回调的签名将略有不同：

```typescript
GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  subscriptions: {
    'graphql-ws': {
      onConnect: (context: Context<any>) => {
        const { connectionParams, extra } = context;
        // 用户验证将与上面的示例相同
        // 当使用 graphql-ws 时，应将额外的上下文值存储在 extra 字段中
        extra.user = { user: {} };
      },
    },
  },
  context: ({ extra }) => {
    // 现在你可以通过 extra 字段访问你的额外上下文值
  },
});
```

#### 使用 Mercurius 驱动启用订阅

要启用订阅，请将 `subscription` 属性设置为 `true`。

```typescript
GraphQLModule.forRoot<MercuriusDriverConfig>({
  driver: MercuriusDriver,
  subscription: true,
}),
```

> info **提示** 你还可以传递选项对象以设置自定义发射器、验证传入连接等。[了解更多](https://github.com/mercurius-js/mercurius/blob/master/docs/api/options.md#plugin-options)（参见 `subscription`）。

#### 代码优先

要使用代码优先方法创建订阅，我们使用 `@Subscription()` 装饰器（从 `@nestjs/graphql` 包导出）和 `mercurius` 包中的 `PubSub` 类，该类提供了一个简单的 **发布/订阅 API**。

以下订阅处理程序通过调用 `PubSub#asyncIterableIterator` 来**订阅**一个事件。此方法接受一个参数，即 `triggerName`，它对应于一个事件主题名称。

```typescript
@Resolver(() => Author)
export class AuthorResolver {
  // ...
  @Subscription(() => Comment)
  commentAdded(@Context('pubsub') pubSub: PubSub) {
    return pubSub.subscribe('commentAdded');
  }
}
```

> info **提示** 上面示例中使用的所有装饰器都从 `@nestjs/graphql` 包导出，而 `PubSub` 类从 `mercurius` 包导出。

> warning **注意** `PubSub` 是一个暴露简单 `publish` 和 `subscribe` API 的类。查看[此部分](https://github.com/mercurius-js/mercurius/blob/master/docs/subscriptions.md#subscriptions-with-custom-pubsub)了解如何注册自定义 `PubSub` 类。

这将生成以下 GraphQL 模式的部分 SDL：

```graphql
type Subscription {
  commentAdded(): Comment!
}
```

请注意，根据定义，订阅返回一个对象，该对象只有一个顶级属性，其键是订阅的名称。该名称要么从订阅处理程序方法的名称继承（例如上面的 `commentAdded`），要么通过将带有键 `name` 的选项作为 `@Subscription()` 装饰器的第二个参数显式提供，如下所示。

```typescript
@Subscription(() => Comment, {
  name: 'commentAdded',
})
subscribeToCommentAdded(@Context('pubsub') pubSub: PubSub) {
  return pubSub.subscribe('commentAdded');
}
```

此构造生成的 SDL 与前面的代码示例相同，但允许我们将方法名与订阅解耦。

#### 发布

现在，要发布事件，我们使用 `PubSub#publish` 方法。这通常在变更中使用，当对象图的某部分发生更改时触发客户端更新。例如：

```typescript
@@filename(posts/posts.resolver)
@Mutation(() => Comment)
async addComment(
  @Args('postId', { type: () => Int }) postId: number,
  @Args('comment', { type: () => Comment }) comment: CommentInput,
  @Context('pubsub') pubSub: PubSub,
) {
  const newComment = this.commentsService.addComment({ id: postId, comment });
  await pubSub.publish({
    topic: 'commentAdded',
    payload: {
      commentAdded: newComment
    }
  });
  return newComment;
}
```

如前所述，订阅按定义返回一个值，该值具有形状。再次查看我们 `commentAdded` 订阅生成的 SDL：

```graphql
type Subscription {
  commentAdded(): Comment!
}
```

这告诉我们订阅必须返回一个对象，其顶级属性名为 `commentAdded`，其值为 `Comment` 对象。需要注意的重要一点是，`PubSub#publish` 方法发出的事件负载的形状必须与订阅预期返回值的形状相对应。因此，在我们上面的示例中，`pubSub.publish({{ '{' }} topic: 'commentAdded', payload: {{ '{' }} commentAdded: newComment {{ '}' }} {{ '}' }})` 语句发布了一个 `commentAdded` 事件，具有正确形状的负载。如果这些形状不匹配，你的订阅将在 GraphQL 验证阶段失败。

#### 过滤订阅

要过滤特定事件，请将 `filter` 属性设置为一个过滤函数。该函数的行为类似于传递给数组 `filter` 的函数。它接受两个参数：`payload` 包含事件负载（由事件发布者发送），`variables` 接收订阅请求期间传递的任何参数。它返回一个布尔值，决定是否应将此事件发布给客户端监听者。

```typescript
@Subscription(() => Comment, {
  filter: (payload, variables) =>
    payload.commentAdded.title === variables.title,
})
commentAdded(@Args('title') title: string, @Context('pubsub') pubSub: PubSub) {
  return pubSub.subscribe('commentAdded');
}
```

如果你需要访问注入的提供者（例如，使用外部服务验证数据），请使用以下结构。

```typescript
@Subscription(() => Comment, {
  filter(this: AuthorResolver, payload, variables) {
    // "this" 指向 "AuthorResolver" 的实例
    return payload.commentAdded.title === variables.title;
  }
})
commentAdded(@Args('title') title: string, @Context('pubsub') pubSub: PubSub) {
  return pubSub.subscribe('commentAdded');
}
```

#### 模式优先

要在 Nest 中创建等效的订阅，我们将使用 `@Subscription()` 装饰器。

```typescript
const pubSub = new PubSub();

@Resolver('Author')
export class AuthorResolver {
  // ...
  @Subscription()
  commentAdded(@Context('pubsub') pubSub: PubSub) {
    return pubSub.subscribe('commentAdded');
  }
}
```

要根据上下文和参数过滤特定事件，请设置 `filter` 属性。

```typescript
@Subscription('commentAdded', {
  filter: (payload, variables) =>
    payload.commentAdded.title === variables.title,
})
commentAdded(@Context('pubsub') pubSub: PubSub) {
  return pubSub.subscribe('commentAdded');
}
```

如果你需要访问注入的提供者（例如，使用外部服务验证数据），请使用以下结构：

```typescript
@Subscription('commentAdded', {
  filter(this: AuthorResolver, payload, variables) {
    // "this" 指向 "AuthorResolver" 的实例
    return payload.commentAdded.title === variables.title;
  }
})
commentAdded(@Context('pubsub') pubSub: PubSub) {
  return pubSub.subscribe('commentAdded');
}
```

最后一步是更新类型定义文件。

```graphql
type Author {
  id: Int!
  firstName: String
  lastName: String
  posts: [Post]
}

type Post {
  id: Int!
  title: String
  votes: Int
}

type Query {
  author(id: Int!): Author
}

type Comment {
  id: String
  content: String
}

type Subscription {
  commentAdded(title: String!): Comment
}
```

通过此操作，我们创建了一个名为 `commentAdded(title: String!): Comment` 的订阅。

#### PubSub

在上面的示例中，我们使用了默认的 `PubSub` 发射器（[mqemitter](https://github.com/mcollina/mqemitter)）。首选方法（用于生产环境）是使用 `mqemitter-redis`。或者，可以提供一个自定义的 `PubSub` 实现（[了解更多](https://github.com/mercurius-js/mercurius/blob/master/docs/subscriptions.md)）。

```typescript
GraphQLModule.forRoot<MercuriusDriverConfig>({
  driver: MercuriusDriver,
  subscription: {
    emitter: require('mqemitter-redis')({
      port: 6579,
      host: '127.0.0.1',
    }),
  },
});
```

#### WebSocket 上的身份验证

检查用户是否已通过身份验证可以在 `verifyClient` 回调函数中完成，你可以在 `subscription` 选项中指定该函数。

`verifyClient` 将接收 `info` 对象作为第一个参数，你可以使用它来检索请求的 headers。

```typescript
GraphQLModule.forRoot<MercuriusDriverConfig>({
  driver: MercuriusDriver,
  subscription: {
    verifyClient: (info, next) => {
      const authorization = info.req.headers?.authorization as string;
      if (!authorization?.startsWith('Bearer ')) {
        return next(false);
      }
      next(true);
    },
  }
}),
```