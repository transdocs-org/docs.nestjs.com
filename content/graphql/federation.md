### 联邦（Federation）

联邦（Federation）提供了一种将单体的 GraphQL 服务器拆分为多个独立微服务的方式。它由两个部分组成：一个网关和一个或多个联邦微服务。每个微服务拥有部分 schema，而网关会将这些 schema 合并成一个整体的 schema，供客户端使用。

引用 [Apollo 文档](https://blog.apollographql.com/apollo-federation-f260cf525d21)，联邦的设计遵循以下核心原则：

- 构建图应该是**声明式**的。通过联邦，你可以从 schema 内部声明式地组合图，而不是编写命令式的 schema stitching 代码。
- 代码应该按**关注点**而非类型来划分。通常没有一个团队能控制像 User 或 Product 这样的重要类型的全部方面，因此这些类型的定义应分布于不同的团队和代码库中，而不是集中管理。
- 图对客户端来说应该简单易用。联邦服务一起可以组成一个完整、以产品为中心的图，准确反映客户端的使用方式。
- 它仍然是**GraphQL**，仅使用语言规范中的合规特性。任何语言，不仅仅是 JavaScript，都可以实现联邦。

> warning **警告** 联邦目前不支持订阅（subscriptions）。

在接下来的章节中，我们将构建一个包含网关和两个联邦服务（用户服务和帖子服务）的示例应用。

#### 使用 Apollo 的联邦

首先安装所需的依赖项：

```bash
$ npm install --save @apollo/subgraph
```

#### 基于 Schema 的方式

“用户服务”提供了一个简单的 schema。注意 `@key` 指令：它告诉 Apollo 查询规划器，只要指定 `User` 的 `id` 就可以获取某个实例。同时注意我们对 `Query` 类型进行了 `extend` 扩展。

```graphql
type User @key(fields: "id") {
  id: ID!
  name: String!
}

extend type Query {
  getUser(id: ID!): User
}
```

解析器提供了一个名为 `resolveReference()` 的额外方法。当相关资源需要一个 User 实例时，Apollo 网关会触发该方法。我们稍后将在帖子服务中看到这个例子。请注意，该方法必须使用 `@ResolveReference()` 装饰器进行注解。

```typescript
import { Args, Query, Resolver, ResolveReference } from '@nestjs/graphql';
import { UsersService } from './users.service';

@Resolver('User')
export class UsersResolver {
  constructor(private usersService: UsersService) {}

  @Query()
  getUser(@Args('id') id: string) {
    return this.usersService.findById(id);
  }

  @ResolveReference()
  resolveReference(reference: { __typename: string; id: string }) {
    return this.usersService.findById(reference.id);
  }
}
```

最后，我们通过注册 `GraphQLModule` 并在配置对象中传入 `ApolloFederationDriver` 驱动来完成整个配置：

```typescript
import {
  ApolloFederationDriver,
  ApolloFederationDriverConfig,
} from '@nestjs/apollo';
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';
import { UsersResolver } from './users.resolver';

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloFederationDriverConfig>({
      driver: ApolloFederationDriver,
      typePaths: ['**/*.graphql'],
    }),
  ],
  providers: [UsersResolver],
})
export class AppModule {}
```

#### 基于代码优先的方式（Code first）

首先，在 `User` 实体上添加一些额外的装饰器。

```ts
import { Directive, Field, ID, ObjectType } from '@nestjs/graphql';

@ObjectType()
@Directive('@key(fields: "id")')
export class User {
  @Field(() => ID)
  id: number;

  @Field()
  name: string;
}
```

解析器提供了一个名为 `resolveReference()` 的额外方法。当相关资源需要一个 User 实例时，Apollo 网关会触发该方法。我们稍后将在帖子服务中看到这个例子。请注意，该方法必须使用 `@ResolveReference()` 装饰器进行注解。

```ts
import { Args, Query, Resolver, ResolveReference } from '@nestjs/graphql';
import { User } from './user.entity';
import { UsersService } from './users.service';

@Resolver(() => User)
export class UsersResolver {
  constructor(private usersService: UsersService) {}

  @Query(() => User)
  getUser(@Args('id') id: number): User {
    return this.usersService.findById(id);
  }

  @ResolveReference()
  resolveReference(reference: { __typename: string; id: number }): User {
    return this.usersService.findById(reference.id);
  }
}
```

最后，我们通过注册 `GraphQLModule` 并在配置对象中传入 `ApolloFederationDriver` 驱动来完成整个配置：

```typescript
import {
  ApolloFederationDriver,
  ApolloFederationDriverConfig,
} from '@nestjs/apollo';
import { Module } from '@nestjs/common';
import { UsersResolver } from './users.resolver';
import { UsersService } from './users.service'; // 本例中未包含

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloFederationDriverConfig>({
      driver: ApolloFederationDriver,
      autoSchemaFile: true,
    }),
  ],
  providers: [UsersResolver, UsersService],
})
export class AppModule {}
```

完整的代码示例可在 [这里](https://github.com/nestjs/nest/tree/master/sample/31-graphql-federation-code-first/users-application) 查看（代码优先模式）和 [这里](https://github.com/nestjs/nest/tree/master/sample/32-graphql-federation-schema-first/users-application)（Schema 优先模式）。

#### 联邦示例：Posts

帖子服务（Post service）通过 `getPosts` 查询提供聚合的帖子，同时扩展 `User` 类型，添加 `user.posts` 字段。

#### Schema First

“帖子服务”在其 schema 中通过 `extend` 关键字引用 `User` 类型。它还在 `User` 类型上声明了一个额外的属性 `posts`。注意 `@key` 指令用于匹配 User 实例，`@external` 指令表示 `id` 字段由其他服务管理。

```graphql
type Post @key(fields: "id") {
  id: ID!
  title: String!
  body: String!
  user: User
}

extend type User @key(fields: "id") {
  id: ID! @external
  posts: [Post]
}

extend type Query {
  getPosts: [Post]
}
```

在下面的示例中，`PostsResolver` 提供了 `getUser()` 方法，返回一个包含 `__typename` 和其他用于解析引用的属性（如 `id`）的对象。`__typename` 被 GraphQL 网关用于定位负责 `User` 类型的微服务并获取对应的实例。上面描述的“用户服务”将在执行 `resolveReference()` 方法时被调用。

```typescript
import { Query, Resolver, Parent, ResolveField } from '@nestjs/graphql';
import { PostsService } from './posts.service';
import { Post } from './posts.interfaces';

@Resolver('Post')
export class PostsResolver {
  constructor(private postsService: PostsService) {}

  @Query('getPosts')
  getPosts() {
    return this.postsService.findAll();
  }

  @ResolveField('user')
  getUser(@Parent() post: Post) {
    return { __typename: 'User', id: post.userId };
  }
}
```

最后，我们必须注册 `GraphQLModule`，这与我们在“用户服务”部分所做的类似。

```typescript
import {
  ApolloFederationDriver,
  ApolloFederationDriverConfig,
} from '@nestjs/apollo';
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';
import { PostsResolver } from './posts.resolver';

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloFederationDriverConfig>({
      driver: ApolloFederationDriver,
      typePaths: ['**/*.graphql'],
    }),
  ],
  providers: [PostsResolver],
})
export class AppModule {}
```

#### Code First

首先，我们需要声明一个代表 `User` 实体的类。虽然该实体本身存在于另一个服务中，但我们会在这里使用它（扩展其定义）。注意 `@extends` 和 `@external` 指令。

```ts
import { Directive, ObjectType, Field, ID } from '@nestjs/graphql';
import { Post } from './post.entity';

@ObjectType()
@Directive('@extends')
@Directive('@key(fields: "id")')
export class User {
  @Field(() => ID)
  @Directive('@external')
  id: number;

  @Field(() => [Post])
  posts?: Post[];
}
```

现在我们为 `User` 实体的扩展创建对应的解析器：

```ts
import { Parent, ResolveField, Resolver } from '@nestjs/graphql';
import { PostsService } from './posts.service';
import { Post } from './post.entity';
import { User } from './user.entity';

@Resolver(() => User)
export class UsersResolver {
  constructor(private readonly postsService: PostsService) {}

  @ResolveField(() => [Post])
  public posts(@Parent() user: User): Post[] {
    return this.postsService.forAuthor(user.id);
  }
}
```

我们还需要定义 `Post` 实体类：

```ts
import { Directive, Field, ID, Int, ObjectType } from '@nestjs/graphql';
import { User } from './user.entity';

@ObjectType()
@Directive('@key(fields: "id")')
export class Post {
  @Field(() => ID)
  id: number;

  @Field()
  title: string;

  @Field(() => Int)
  authorId: number;

  @Field(() => User)
  user?: User;
}
```

以及它的解析器：

```ts
import { Query, Args, ResolveField, Resolver, Parent } from '@nestjs/graphql';
import { PostsService } from './posts.service';
import { Post } from './post.entity';
import { User } from './user.entity';

@Resolver(() => Post)
export class PostsResolver {
  constructor(private readonly postsService: PostsService) {}

  @Query(() => Post)
  findPost(@Args('id') id: number): Post {
    return this.postsService.findOne(id);
  }

  @Query(() => [Post])
  getPosts(): Post[] {
    return this.postsService.all();
  }

  @ResolveField(() => User)
  user(@Parent() post: Post): any {
    return { __typename: 'User', id: post.authorId };
  }
}
```

最后，将它们整合到模块中。注意 schema 构建选项中我们指定了 `User` 是一个孤儿（外部）类型。

```ts
import {
  ApolloFederationDriver,
  ApolloFederationDriverConfig,
} from '@nestjs/apollo';
import { Module } from '@nestjs/common';
import { User } from './user.entity';
import { PostsResolvers } from './posts.resolvers';
import { UsersResolvers } from './users.resolvers';
import { PostsService } from './posts.service'; // 示例中未包含

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloFederationDriverConfig>({
      driver: ApolloFederationDriver,
      autoSchemaFile: true,
      buildSchemaOptions: {
        orphanedTypes: [User],
      },
    }),
  ],
  providers: [PostsResolver, UsersResolver, PostsService],
})
export class AppModule {}
```

完整的代码示例可在 [这里](https://github.com/nestjs/nest/tree/master/sample/31-graphql-federation-code-first/posts-application) 查看（代码优先模式）和 [这里](https://github.com/nestjs/nest/tree/master/sample/32-graphql-federation-schema-first/posts-application)（Schema 优先模式）。

#### 联邦示例：网关（Gateway）

首先安装所需的依赖项：

```bash
$ npm install --save @apollo/gateway
```

网关需要指定一组端点，并会自动发现对应的 schema。因此，网关服务的实现对于代码优先和 schema 优先方式是相同的。

```typescript
import { IntrospectAndCompose } from '@apollo/gateway';
import { ApolloGatewayDriver, ApolloGatewayDriverConfig } from '@nestjs/apollo';
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloGatewayDriverConfig>({
      driver: ApolloGatewayDriver,
      server: {
        // ... Apollo server options
        cors: true,
      },
      gateway: {
        supergraphSdl: new IntrospectAndCompose({
          subgraphs: [
            { name: 'users', url: 'http://user-service/graphql' },
            { name: 'posts', url: 'http://post-service/graphql' },
          ],
        }),
      },
    }),
  ],
})
export class AppModule {}
```

完整的代码示例可在 [这里](https://github.com/nestjs/nest/tree/master/sample/31-graphql-federation-code-first/gateway) 查看（代码优先模式）和 [这里](https://github.com/nestjs/nest/tree/master/sample/32-graphql-federation-schema-first/gateway)（schema 优先模式）。

#### 使用 Mercurius 的联邦

首先安装所需的依赖项：

```bash
$ npm install --save @apollo/subgraph @nestjs/mercurius
```

> info **注意** `@apollo/subgraph` 包用于构建子图 schema（如 `buildSubgraphSchema` 和 `printSubgraphSchema` 函数）。

#### Schema First

“用户服务”提供了一个简单的 schema。注意 `@key` 指令：它告诉 Mercurius 查询规划器，只要指定 `User` 的 `id` 就可以获取某个实例。同时注意我们对 `Query` 类型进行了 `extend` 扩展。

```graphql
type User @key(fields: "id") {
  id: ID!
  name: String!
}

extend type Query {
  getUser(id: ID!): User
}
```

解析器提供了一个名为 `resolveReference()` 的额外方法。当相关资源需要一个 User 实例时，Mercurius 网关会触发该方法。我们稍后将在帖子服务中看到这个例子。请注意，该方法必须使用 `@ResolveReference()` 装饰器进行注解。

```ts
import { Args, Query, Resolver, ResolveReference } from '@nestjs/graphql';
import { UsersService } from './users.service';

@Resolver('User')
export class UsersResolver {
  constructor(private usersService: UsersService) {}

  @Query()
  getUser(@Args('id') id: string) {
    return this.usersService.findById(id);
  }

  @ResolveReference()
  resolveReference(reference: { __typename: string; id: string }) {
    return this.usersService.findById(reference.id);
  }
}
```

最后，我们通过注册 `GraphQLModule` 并在配置对象中传入 `MercuriusFederationDriver` 驱动来完成整个配置：

```ts
import {
  MercuriusFederationDriver,
  MercuriusFederationDriverConfig,
} from '@nestjs/mercurius';
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';
import { UsersResolver } from './users.resolver';

@Module({
  imports: [
    GraphQLModule.forRoot<MercuriusFederationDriverConfig>({
      driver: MercuriusFederationDriver,
      typePaths: ['**/*.graphql'],
      federationMetadata: true,
    }),
  ],
  providers: [UsersResolver],
})
export class AppModule {}
```

#### Code First

首先，在 `User` 实体上添加一些额外的装饰器。

```ts
import { Directive, Field, ID, ObjectType } from '@nestjs/graphql';

@ObjectType()
@Directive('@key(fields: "id")')
export class User {
  @Field(() => ID)
  id: number;

  @Field()
  name: string;
}
```

解析器提供了一个名为 `resolveReference()` 的额外方法。当相关资源需要一个 User 实例时，Mercurius 网关会触发该方法。我们稍后将在帖子服务中看到这个例子。请注意，该方法必须使用 `@ResolveReference()` 装饰器进行注解。

```ts
import { Args, Query, Resolver, ResolveReference } from '@nestjs/graphql';
import { User } from './user.entity';
import { UsersService } from './users.service';

@Resolver(() => User)
export class UsersResolver {
  constructor(private usersService: UsersService) {}

  @Query(() => User)
  getUser(@Args('id') id: number): User {
    return this.usersService.findById(id);
  }

  @ResolveReference()
  resolveReference(reference: { __typename: string; id: number }): User {
    return this.usersService.findById(reference.id);
  }
}
```

最后，我们通过注册 `GraphQLModule` 并在配置对象中传入 `MercuriusFederationDriver` 驱动来完成整个配置：

```ts
import {
  MercuriusFederationDriver,
  MercuriusFederationDriverConfig,
} from '@nestjs/mercurius';
import { Module } from '@nestjs/common';
import { UsersResolver } from './users.resolver';
import { UsersService } from './users.service'; // 本例中未包含

@Module({
  imports: [
    GraphQLModule.forRoot<MercuriusFederationDriverConfig>({
      driver: MercuriusFederationDriver,
      autoSchemaFile: true,
      federationMetadata: true,
    }),
  ],
  providers: [UsersResolver, UsersService],
})
export class AppModule {}
```

#### 联邦示例：Posts

帖子服务（Post service）通过 `getPosts` 查询提供聚合的帖子，同时扩展 `User` 类型，添加 `user.posts` 字段。

#### Schema First

“帖子服务”在其 schema 中通过 `extend` 关键字引用 `User` 类型。它还在 `User` 类型上声明了一个额外的属性 `posts`。注意 `@key` 指令用于匹配 User 实例，`@external` 指令表示 `id` 字段由其他服务管理。

```graphql
type Post @key(fields: "id") {
  id: ID!
  title: String!
  body: String!
  user: User
}

extend type User @key(fields: "id") {
  id: ID! @external
  posts: [Post]
}

extend type Query {
  getPosts: [Post]
}
```

在下面的示例中，`PostsResolver` 提供了 `getUser()` 方法，返回一个包含 `__typename` 和其他用于解析引用的属性（如 `id`）的对象。`__typename` 被 GraphQL 网关用于定位负责 `User` 类型的微服务并获取对应的实例。上面描述的“用户服务”将在执行 `resolveReference()` 方法时被调用。

```ts
import { Query, Resolver, Parent, ResolveField } from '@nestjs/graphql';
import { PostsService } from './posts.service';
import { Post } from './posts.interfaces';

@Resolver('Post')
export class PostsResolver {
  constructor(private postsService: PostsService) {}

  @Query('getPosts')
  getPosts() {
    return this.postsService.findAll();
  }

  @ResolveField('user')
  getUser(@Parent() post: Post) {
    return { __typename: 'User', id: post.userId };
  }
}
```

最后，我们必须注册 `GraphQLModule`，这与我们在“用户服务”部分所做的类似。

```ts
import {
  MercuriusFederationDriver,
  MercuriusFederationDriverConfig,
} from '@nestjs/mercurius';
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';
import { PostsResolver } from './posts.resolver';

@Module({
  imports: [
    GraphQLModule.forRoot<MercuriusFederationDriverConfig>({
      driver: MercuriusFederationDriver,
      federationMetadata: true,
      typePaths: ['**/*.graphql'],
    }),
  ],
  providers: [PostsResolver],
})
export class AppModule {}
```

#### Code First

首先，我们需要声明一个代表 `User` 实体的类。虽然该实体本身存在于另一个服务中，但我们会在这里使用它（扩展其定义）。注意 `@extends` 和 `@external` 指令。

```ts
import { Directive, ObjectType, Field, ID } from '@nestjs/graphql';
import { Post } from './post.entity';

@ObjectType()
@Directive('@extends')
@Directive('@key(fields: "id")')
export class User {
  @Field(() => ID)
  @Directive('@external')
  id: number;

  @Field(() => [Post])
  posts?: Post[];
}
```

现在我们为 `User` 实体的扩展创建对应的解析器：

```ts
import { Parent, ResolveField, Resolver } from '@nestjs/graphql';
import { PostsService } from './posts.service';
import { Post } from './post.entity';
import { User } from './user.entity';

@Resolver(() => User)
export class UsersResolver {
  constructor(private readonly postsService: PostsService) {}

  @ResolveField(() => [Post])
  public posts(@Parent() user: User): Post[] {
    return this.postsService.forAuthor(user.id);
  }
}
```

我们还需要定义 `Post` 实体类：

```ts
import { Directive, Field, ID, Int, ObjectType } from '@nestjs/graphql';
import { User } from './user.entity';

@ObjectType()
@Directive('@key(fields: "id")')
export class Post {
  @Field(() => ID)
  id: number;

  @Field()
  title: string;

  @Field(() => Int)
  authorId: number;

  @Field(() => User)
  user?: User;
}
```

以及它的解析器：

```ts
import { Query, Args, ResolveField, Resolver, Parent } from '@nestjs/graphql';
import { PostsService } from './posts.service';
import { Post } from './post.entity';
import { User } from './user.entity';

@Resolver(() => Post)
export class PostsResolver {
  constructor(private readonly postsService: PostsService) {}

  @Query(() => Post)
  findPost(@Args('id') id: number): Post {
    return this.postsService.findOne(id);
  }

  @Query(() => [Post])
  getPosts(): Post[] {
    return this.postsService.all();
  }

  @ResolveField(() => User)
  user(@Parent() post: Post): any {
    return { __typename: 'User', id: post.authorId };
  }
}
```

最后，将它们整合到模块中。注意 schema 构建选项中我们指定了 `User` 是一个孤儿（外部）类型。

```ts
import {
  MercuriusFederationDriver,
  MercuriusFederationDriverConfig,
} from '@nestjs/mercurius';
import { Module } from '@nestjs/common';
import { User } from './user.entity';
import { PostsResolvers } from './posts.resolvers';
import { UsersResolvers } from './users.resolvers';
import { PostsService } from './posts.service'; // 示例中未包含

@Module({
  imports: [
    GraphQLModule.forRoot<MercuriusFederationDriverConfig>({
      driver: MercuriusFederationDriver,
      autoSchemaFile: true,
      federationMetadata: true,
      buildSchemaOptions: {
        orphanedTypes: [User],
      },
    }),
  ],
  providers: [PostsResolver, UsersResolver, PostsService],
})
export class AppModule {}
```

#### 联邦示例：网关（Gateway）

网关需要指定一组端点，并会自动发现对应的 schema。因此，网关服务的实现对于代码优先和 schema 优先方式是相同的。

```ts
import {
  MercuriusGatewayDriver,
  MercuriusGatewayDriverConfig,
} from '@nestjs/mercurius';
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';

@Module({
  imports: [
    GraphQLModule.forRoot<MercuriusGatewayDriverConfig>({
      driver: MercuriusGatewayDriver,
      gateway: {
        services: [
          { name: 'users', url: 'http://user-service/graphql' },
          { name: 'posts', url: 'http://post-service/graphql' },
        ],
      },
    }),
  ],
})
export class AppModule {}
```

### 联邦 2（Federation 2）

引用 [Apollo 文档](https://www.apollographql.com/docs/federation/federation-2/new-in-federation-2)，Federation 2 改进了原始的 Apollo 联邦（称为 Federation 1），并保持与大多数原始超图（supergraphs）的向后兼容性。

> warning **警告** Mercurius 尚未完全支持 Federation 2。你可以在 [此处](https://www.apollographql.com/docs/federation/supported-subgraphs#javascript--typescript) 查看支持 Federation 2 的库列表。

在接下来的章节中，我们将把之前的示例升级到 Federation 2。

#### 联邦示例：用户服务（Users）

Federation 2 的一个变化是实体不再有源子图（originating subgraph），因此我们不再需要扩展 `Query`。更多细节请参阅 [Apollo Federation 2 文档中的实体主题](https://www.apollographql.com/docs/federation/federation-2/new-in-federation-2#entities)。

#### Schema First

我们可以简单地从 schema 中移除 `extend` 关键字。

```graphql
type User @key(fields: "id") {
  id: ID!
  name: String!
}

type Query {
  getUser(id: ID!): User
}
```

#### Code First

要使用 Federation 2，我们需要在 `autoSchemaFile` 选项中指定联邦版本。

```ts
import {
  ApolloFederationDriver,
  ApolloFederationDriverConfig,
} from '@nestjs/apollo';
import { Module } from '@nestjs/common';
import { UsersResolver } from './users.resolver';
import { UsersService } from './users.service'; // 示例中未包含

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloFederationDriverConfig>({
      driver: ApolloFederationDriver,
      autoSchemaFile: {
        federation: 2,
      },
    }),
  ],
  providers: [UsersResolver, UsersService],
})
export class AppModule {}
```

#### 联邦示例：帖子服务（Posts）

由于相同的原因，我们不再需要扩展 `User` 和 `Query`。

#### Schema First

我们可以简单地从 schema 中移除 `extend` 和 `external` 指令。

```graphql
type Post @key(fields: "id") {
  id: ID!
  title: String!
  body: String!
  user: User
}

type User @key(fields: "id") {
  id: ID!
  posts: [Post]
}

type Query {
  getPosts: [Post]
}
```

#### Code First

由于我们不再扩展 `User` 实体，我们可以直接从 `User` 类中移除 `extends` 和 `external` 指令。

```ts
import { Directive, ObjectType, Field, ID } from '@nestjs/graphql';
import { Post } from './post.entity';

@ObjectType()
@Directive('@key(fields: "id")')
export class User {
  @Field(() => ID)
  id: number;

  @Field(() => [Post])
  posts?: Post[];
}
```

此外，与用户服务类似，我们需要在 `GraphQLModule` 中指定使用 Federation 2。

```ts
import {
  ApolloFederationDriver,
  ApolloFederationDriverConfig,
} from '@nestjs/apollo';
import { Module } from '@nestjs/common';
import { User } from './user.entity';
import { PostsResolvers } from './posts.resolvers';
import { UsersResolvers } from './users.resolvers';
import { PostsService } from './posts.service'; // 示例中未包含

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloFederationDriverConfig>({
      driver: ApolloFederationDriver,
      autoSchemaFile: {
        federation: 2,
      },
      buildSchemaOptions: {
        orphanedTypes: [User],
      },
    }),
  ],
  providers: [PostsResolver, UsersResolver, PostsService],
})
export class AppModule {}
```