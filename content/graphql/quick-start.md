## 利用 TypeScript 与 GraphQL 的强大功能

[GraphQL](https://graphql.org/) 是一种用于 API 的强大查询语言，也是一种运行时，用来利用你已有的数据满足这些查询。它是一种优雅的解决方案，解决了 REST API 中常见的许多问题。作为背景知识，我们建议阅读 GraphQL 与 REST 的[对比](https://www.apollographql.com/blog/graphql-vs-rest)。将 GraphQL 与 [TypeScript](https://www.typescriptlang.org/) 结合使用，可以让你在 GraphQL 查询中实现更好的类型安全，从而获得端到端的类型支持。

在本章节中，我们假设你对 GraphQL 有基本了解，并将重点介绍如何使用内置的 `@nestjs/graphql` 模块。`GraphQLModule` 可以配置为使用 [Apollo](https://www.apollographql.com/) 服务器（使用 `@nestjs/apollo` 驱动程序）和 [Mercurius](https://github.com/mercurius-js/mercurius)（使用 `@nestjs/mercurius` 驱动程序）。我们为这些经过验证的 GraphQL 包提供官方集成，以便简单地在 Nest 中使用 GraphQL（更多集成请参见[这里](https://docs.nestjs.com/graphql/quick-start#third-party-integrations)）。

你也可以构建自己的专用驱动程序（更多内容请参见[这里](/graphql/other-features#creating-a-custom-driver)）。

#### 安装

首先安装所需的包：

```bash
# 用于 Express 和 Apollo（默认）
$ npm i @nestjs/graphql @nestjs/apollo @apollo/server@^4.12.2 graphql

# 用于 Fastify 和 Apollo
# npm i @nestjs/graphql @nestjs/apollo @apollo/server@^4.12.2 @as-integrations/fastify graphql

# 用于 Fastify 和 Mercurius
# npm i @nestjs/graphql @nestjs/mercurius graphql mercurius
```

> warning **警告** `@nestjs/graphql@>=9` 和 `@nestjs/apollo^10` 包兼容 **Apollo v3**（详见 Apollo Server 3 的[迁移指南](https://www.apollographql.com/docs/apollo-server/migration/)），而 `@nestjs/graphql@^8` 仅支持 **Apollo v2**（例如 `apollo-server-express@2.x.x` 包）。

#### 概览

Nest 提供了两种构建 GraphQL 应用程序的方法：**代码优先（code first）** 和 **模式优先（schema first）**。你可以选择最适合你的方式。本 GraphQL 章节的大部分内容都分为两个主要部分：如果你采用 **代码优先** 方法，请参考第一部分；如果采用 **模式优先** 方法，请参考第二部分。

在 **代码优先** 方法中，你可以使用装饰器和 TypeScript 类来生成相应的 GraphQL 模式。如果你倾向于仅使用 TypeScript 并避免在不同语言语法之间切换，这种方法非常有用。

在 **模式优先** 方法中，源数据是 GraphQL SDL（Schema Definition Language）文件。SDL 是一种与语言无关的方式，用于在不同平台之间共享模式文件。Nest 会根据 GraphQL 模式自动生成你的 TypeScript 定义（使用类或接口），从而减少冗余样板代码的编写。

<app-banner-courses-graphql-cf></app-banner-courses-graphql-cf>

#### 开始使用 GraphQL & TypeScript

> info **提示** 在接下来的章节中，我们将集成 `@nestjs/apollo` 包。如果你想使用 `mercurius` 包，请跳转到[此部分](/graphql/quick-start#mercurius-integration)。

安装完包后，我们可以导入 `GraphQLModule` 并使用 `forRoot()` 静态方法进行配置。

```typescript
@@filename()
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';
import { ApolloDriver, ApolloDriverConfig } from '@nestjs/apollo';

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloDriverConfig>({
      driver: ApolloDriver,
    }),
  ],
})
export class AppModule {}
```

> info **提示** 对于 `mercurius` 集成，你应该使用 `MercuriusDriver` 和 `MercuriusDriverConfig`。两者都导自 `@nestjs/mercurius` 包。

`forRoot()` 方法接受一个选项对象作为参数。这些选项将被传递给底层驱动实例（有关可用设置的更多内容，请参见：[Apollo](https://www.apollographql.com/docs/apollo-server/api/apollo-server) 和 [Mercurius](https://github.com/mercurius-js/mercurius/blob/master/docs/api/options.md#plugin-options)）。例如，如果你想禁用 `playground` 并关闭 `debug` 模式（针对 Apollo），请传递以下选项：

```typescript
@@filename()
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';
import { ApolloDriver, ApolloDriverConfig } from '@nestjs/apollo';

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloDriverConfig>({
      driver: ApolloDriver,
      playground: false,
    }),
  ],
})
export class AppModule {}
```

在这种情况下，这些选项将被转发给 `ApolloServer` 构造函数。

#### GraphQL Playground

Playground 是一个图形化、交互式的浏览器内 GraphQL IDE，默认情况下与 GraphQL 服务器位于相同的 URL 上。要访问 Playground，你需要配置并运行一个基本的 GraphQL 服务器。现在你可以安装并构建[此处](https://github.com/nestjs/nest/tree/master/sample/23-graphql-code-first)的工作示例来查看它。或者，如果你正在跟随这些代码示例操作，在完成[解析器章节](/graphql/resolvers-map)中的步骤后，你可以访问 Playground。

一旦完成这些设置，并且应用程序在后台运行中，你可以打开浏览器并导航到 `http://localhost:3000/graphql`（主机和端口可能因你的配置而异）。你将看到 GraphQL Playground，如下图所示。

<figure>
  <img src="/assets/playground.png" alt="" />
</figure>

> info **注意** `@nestjs/mercurius` 集成不包含内置的 GraphQL Playground。你可以使用 [GraphiQL](https://github.com/graphql/graphiql)（设置 `graphiql: true`）。

> warning **警告** 更新（2025/04/14）：默认的 Apollo Playground 已被弃用，并将在下一个主要版本中移除。你可以改用 [GraphiQL](https://github.com/graphql/graphiql)，只需在 `GraphQLModule` 配置中设置 `graphiql: true`，如下所示：
>
> ```typescript
> GraphQLModule.forRoot<ApolloDriverConfig>({
>   driver: ApolloDriver,
>   graphiql: true,
> }),
> ```
>
> 如果你的应用程序使用了[订阅](/graphql/subscriptions)，请确保使用 `graphql-ws`，因为 GraphiQL 不支持 `subscriptions-transport-ws`。

#### 代码优先（Code first）

在 **代码优先** 方法中，你可以使用装饰器和 TypeScript 类来生成相应的 GraphQL 模式。

要使用代码优先方法，请首先向选项对象中添加 `autoSchemaFile` 属性：

```typescript
GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  autoSchemaFile: join(process.cwd(), 'src/schema.gql'),
}),
```

`autoSchemaFile` 属性值是你自动生成的模式文件的保存路径。或者，你也可以选择在内存中动态生成模式。要启用此功能，请将 `autoSchemaFile` 设置为 `true`：

```typescript
GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  autoSchemaFile: true,
}),
```

默认情况下，生成的模式中的类型将按照模块中定义的顺序排列。要按字母顺序对模式进行排序，请将 `sortSchema` 属性设置为 `true`：

```typescript
GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  autoSchemaFile: join(process.cwd(), 'src/schema.gql'),
  sortSchema: true,
}),
```

#### 示例

一个完整的代码优先示例可以在[这里](https://github.com/nestjs/nest/tree/master/sample/23-graphql-code-first)找到。

#### 模式优先（Schema first）

要使用模式优先方法，请首先向选项对象中添加 `typePaths` 属性。`typePaths` 属性指定了 `GraphQLModule` 应该在哪些位置查找你编写的 GraphQL SDL 模式定义文件。这些文件将在内存中合并，这允许你将模式拆分为多个文件，并将它们放在与其解析器相近的位置。

```typescript
GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  typePaths: ['./**/*.graphql'],
}),
```

通常，你还希望拥有与 GraphQL SDL 类型对应的 TypeScript 定义（类和接口）。手动创建这些 TypeScript 定义是冗余且繁琐的。这会导致我们失去单一的真实数据源——每次在 SDL 中进行更改，都需要同步调整 TypeScript 定义。为了解决这个问题，`@nestjs/graphql` 包可以**自动从抽象语法树**（[AST](https://en.wikipedia.org/wiki/Abstract_syntax_tree)）生成 TypeScript 定义。要启用此功能，请在配置 `GraphQLModule` 时添加 `definitions` 选项属性。

```typescript
GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  typePaths: ['./**/*.graphql'],
  definitions: {
    path: join(process.cwd(), 'src/graphql.ts'),
  },
}),
```

`definitions` 对象的 `path` 属性指定了生成的 TypeScript 输出文件的保存路径。默认情况下，所有生成的 TypeScript 类型都将作为接口生成。如果你想生成类，请将 `outputAs` 属性设置为 `'class'`。

```typescript
GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  typePaths: ['./**/*.graphql'],
  definitions: {
    path: join(process.cwd(), 'src/graphql.ts'),
    outputAs: 'class',
  },
}),
```

上述方法会在每次应用程序启动时动态生成 TypeScript 定义。或者，你可以编写一个简单的脚本来按需生成这些定义。例如，假设我们创建了以下脚本 `generate-typings.ts`：

```typescript
import { GraphQLDefinitionsFactory } from '@nestjs/graphql';
import { join } from 'path';

const definitionsFactory = new GraphQLDefinitionsFactory();
definitionsFactory.generate({
  typePaths: ['./src/**/*.graphql'],
  path: join(process.cwd(), 'src/graphql.ts'),
  outputAs: 'class',
});
```

现在你可以按需运行此脚本：

```bash
$ ts-node generate-typings
```

> info **提示** 你可以提前编译脚本（例如使用 `tsc`），然后使用 `node` 来执行它。

要启用脚本的监听模式（即在任何 `.graphql` 文件更改时自动重新生成类型定义），请将 `watch` 选项传递给 `generate()` 方法。

```typescript
definitionsFactory.generate({
  typePaths: ['./src/**/*.graphql'],
  path: join(process.cwd(), 'src/graphql.ts'),
  outputAs: 'class',
  watch: true,
});
```

要为每个对象类型自动生成 `__typename` 字段，请启用 `emitTypenameField` 选项：

```typescript
definitionsFactory.generate({
  // ...
  emitTypenameField: true,
});
```

要将解析器（查询、变更、订阅）生成为不带参数的普通字段，请启用 `skipResolverArgs` 选项：

```typescript
definitionsFactory.generate({
  // ...
  skipResolverArgs: true,
});
```

要将枚举生成为 TypeScript 联合类型而不是常规的 TypeScript 枚举，请将 `enumsAsTypes` 选项设置为 `true`：

```typescript
definitionsFactory.generate({
  // ...
  enumsAsTypes: true,
});
```

#### Apollo Sandbox

如果你想使用 [Apollo Sandbox](https://www.apollographql.com/blog/announcement/platform/apollo-sandbox-an-open-graphql-ide-for-local-development/) 作为本地开发的 GraphQL IDE，而不是 `graphql-playground`，请使用以下配置：

```typescript
import { ApolloDriver, ApolloDriverConfig } from '@nestjs/apollo';
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';
import { ApolloServerPluginLandingPageLocalDefault } from '@apollo/server/plugin/landingPage/default';

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloDriverConfig>({
      driver: ApolloDriver,
      playground: false,
      plugins: [ApolloServerPluginLandingPageLocalDefault()],
    }),
  ],
})
export class AppModule {}
```

#### 示例

一个完整的模式优先示例可以在[这里](https://github.com/nestjs/nest/tree/master/sample/12-graphql-schema-first)找到。

#### 访问生成的模式

在某些情况下（例如端到端测试），你可能希望获取生成的模式对象的引用。在端到端测试中，你可以使用 `graphql` 对象运行查询，而无需使用任何 HTTP 监听器。

你可以使用 `GraphQLSchemaHost` 类来访问生成的模式（无论是代码优先还是模式优先方法）：

```typescript
const { schema } = app.get(GraphQLSchemaHost);
```

> info **提示** 你必须在应用程序初始化之后（即 `app.listen()` 或 `app.init()` 方法触发 `onModuleInit` 钩子之后）调用 `GraphQLSchemaHost#schema` 的 getter。

#### 异步配置

当你需要异步而不是静态地传递模块选项时，请使用 `forRootAsync()` 方法。与大多数动态模块一样，Nest 提供了几种处理异步配置的技术。

其中一种技术是使用工厂函数：

```typescript
 GraphQLModule.forRootAsync<ApolloDriverConfig>({
  driver: ApolloDriver,
  useFactory: () => ({
    typePaths: ['./**/*.graphql'],
  }),
}),
```

与其他工厂提供者一样，我们的工厂函数可以是 <a href="https://docs.nestjs.com/fundamentals/custom-providers#factory-providers-usefactory">异步的</a>，并且可以通过 `inject` 注入依赖项。

```typescript
GraphQLModule.forRootAsync<ApolloDriverConfig>({
  driver: ApolloDriver,
  imports: [ConfigModule],
  useFactory: async (configService: ConfigService) => ({
    typePaths: configService.get<string>('GRAPHQL_TYPE_PATHS'),
  }),
  inject: [ConfigService],
}),
```

另外，你也可以使用类而不是工厂来配置 `GraphQLModule`，如下所示：

```typescript
GraphQLModule.forRootAsync<ApolloDriverConfig>({
  driver: ApolloDriver,
  useClass: GqlConfigService,
}),
```

上面的代码在 `GraphQLModule` 中实例化了 `GqlConfigService`，并使用它来创建选项对象。请注意，在这个例子中，`GqlConfigService` 必须实现 `GqlOptionsFactory` 接口，如下所示。`GraphQLModule` 将调用所提供的类实例的 `createGqlOptions()` 方法。

```typescript
@Injectable()
class GqlConfigService implements GqlOptionsFactory {
  createGqlOptions(): ApolloDriverConfig {
    return {
      typePaths: ['./**/*.graphql'],
    };
  }
}
```

如果你想重用现有的选项提供者，而不是在 `GraphQLModule` 中创建私有副本，请使用 `useExisting` 语法。

```typescript
GraphQLModule.forRootAsync<ApolloDriverConfig>({
  imports: [ConfigModule],
  useExisting: ConfigService,
}),
```

#### Mercurius 集成

除了使用 Apollo，Fastify 用户（更多信息请参见[这里](/techniques/performance)）还可以选择使用 `@nestjs/mercurius` 驱动程序。

```typescript
@@filename()
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';
import { MercuriusDriver, MercuriusDriverConfig } from '@nestjs/mercurius';

@Module({
  imports: [
    GraphQLModule.forRoot<MercuriusDriverConfig>({
      driver: MercuriusDriver,
      graphiql: true,
    }),
  ],
})
export class AppModule {}
```

> info **提示** 应用程序运行后，打开浏览器并访问 `http://localhost:3000/graphiql`，你将看到 [GraphQL IDE](https://github.com/graphql/graphiql)。

`forRoot()` 方法接受一个选项对象作为参数。这些选项将被传递给底层驱动实例。更多可用设置请参见[这里](https://github.com/mercurius-js/mercurius/blob/master/docs/api/options.md#plugin-options)。

#### 多个端点

`@nestjs/graphql` 模块的另一个有用功能是能够同时提供多个端点。这使你可以决定哪些模块应该包含在哪些端点中。默认情况下，`GraphQL` 会在整个应用程序中搜索解析器。若要限制扫描范围，仅扫描一部分模块，请使用 `include` 属性。

```typescript
GraphQLModule.forRoot({
  include: [CatsModule],
}),
```

> warning **警告** 如果你在单个应用程序中使用 `@apollo/server` 和 `@as-integrations/fastify` 包并配置了多个 GraphQL 端点，请确保在 `GraphQLModule` 配置中启用 `disableHealthCheck` 设置。

#### 第三方集成

- [GraphQL Yoga](https://github.com/dotansimha/graphql-yoga)

#### 示例

一个完整的工作示例可以在[这里](https://github.com/nestjs/nest/tree/master/sample/33-graphql-mercurius)找到。