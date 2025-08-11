### 变更操作（Mutations）

大多数关于 GraphQL 的讨论都集中在数据获取上，但一个完整的数据平台同样需要一种修改服务端数据的方式。在 REST 中，任何请求都有可能在服务端产生副作用，但最佳实践建议我们不要在 GET 请求中修改数据。GraphQL 的情况与此类似 —— 技术上来说，任何查询都可以被实现为引起数据写入。然而，与 REST 一样，推荐遵守如下约定：任何会引起数据写入的操作应明确通过变更操作（Mutation）发送（更多内容请参阅[这里](https://graphql.org/learn/queries/#mutations)）。

官方 [Apollo](https://www.apollographql.com/docs/graphql-tools/generate-schema.html) 文档中使用了一个 `upvotePost()` 变更操作的示例。该变更操作实现了一个方法，用于增加帖子的 `votes` 属性值。要在 Nest 中创建等效的变更操作，我们将使用 `@Mutation()` 装饰器。

#### 代码优先（Code first）

让我们在上一节中使用的 `AuthorResolver` 中再添加一个方法（参见 [解析器（resolvers）](/graphql/resolvers)）：

```typescript
@Mutation(() => Post)
async upvotePost(@Args({ name: 'postId', type: () => Int }) postId: number) {
  return this.postsService.upvoteById({ id: postId });
}
```

> info **提示** 所有装饰器（如 `@Resolver`、`@ResolveField`、`@Args` 等）都从 `@nestjs/graphql` 包中导出。

这将生成如下 SDL 格式的 GraphQL schema 部分：

```graphql
type Mutation {
  upvotePost(postId: Int!): Post
}
```

`upvotePost()` 方法将 `postId`（`Int`）作为参数，并返回更新后的 `Post` 实体。由于 [解析器（resolvers）](/graphql/resolvers) 一节中解释的原因，我们必须显式地设置预期类型。

如果变更操作需要以对象作为参数，我们可以创建一个**输入类型（input type）**。输入类型是一种特殊类型的对象类型，可以作为参数传入（更多内容请参阅[这里](https://graphql.org/learn/schema/#input-types)）。要声明输入类型，请使用 `@InputType()` 装饰器。

```typescript
import { InputType, Field } from '@nestjs/graphql';

@InputType()
export class UpvotePostInput {
  @Field()
  postId: number;
}
```

> info **提示** `@InputType()` 装饰器接收一个选项对象作为参数，因此你可以例如指定输入类型的描述。请注意，由于 TypeScript 的元数据反射系统的限制，你必须使用 `@Field` 装饰器手动指定类型，或者使用 [CLI 插件](/graphql/cli-plugin)。

然后我们可以在解析器类中使用该类型：

```typescript
@Mutation(() => Post)
async upvotePost(
  @Args('upvotePostData') upvotePostData: UpvotePostInput,
) {}
```

#### 模式优先（Schema first）

让我们扩展上一节中使用的 `AuthorResolver`（参见 [解析器（resolvers）](/graphql/resolvers)）：

```typescript
@Mutation()
async upvotePost(@Args('postId') postId: number) {
  return this.postsService.upvoteById({ id: postId });
}
```

请注意，我们假设上面的业务逻辑已经移至 `PostsService` 类中（查询帖子并增加其 `votes` 属性值）。`PostsService` 类中的逻辑可以根据需要简单或复杂。本示例的主要目的是展示解析器如何与其他提供者进行交互。

最后一步是将我们的变更操作添加到现有的类型定义中：

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

type Mutation {
  upvotePost(postId: Int!): Post
}
```

现在，我们的 GraphQL API 已经可以调用 `upvotePost(postId: Int!): Post` 这个变更操作了。