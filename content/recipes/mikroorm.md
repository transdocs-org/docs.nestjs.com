### MikroORM

本指南旨在帮助用户在 Nest 中快速上手 MikroORM。MikroORM 是基于 Data Mapper、Unit of Work 和 Identity Map 模式构建的 TypeScript ORM，是 TypeORM 的一个优秀替代品，从 TypeORM 迁移过来也非常容易。MikroORM 的完整文档请参见 [这里](https://mikro-orm.io/docs)。

> info **提示** `@mikro-orm/nestjs` 是一个第三方包，不由 NestJS 核心团队维护。如发现该库存在问题，请在 [对应仓库](https://github.com/mikro-orm/nestjs) 提交问题。

#### 安装

集成 MikroORM 到 Nest 的最简单方式是通过 [`@mikro-orm/nestjs` 模块](https://github.com/mikro-orm/nestjs)。只需将其与 Nest、MikroORM 及底层驱动一起安装：

```bash
$ npm i @mikro-orm/core @mikro-orm/nestjs @mikro-orm/sqlite
```

MikroORM 同样支持 `postgres`、`sqlite` 和 `mongo`。所有驱动的详细信息请参见 [官方文档](https://mikro-orm.io/docs/usage-with-sql/)。

安装完成后，我们可以将 `MikroOrmModule` 导入到根模块 `AppModule` 中。

```typescript
import { SqliteDriver } from '@mikro-orm/sqlite';

@Module({
  imports: [
    MikroOrmModule.forRoot({
      entities: ['./dist/entities'],
      entitiesTs: ['./src/entities'],
      dbName: 'my-db-name.sqlite3',
      driver: SqliteDriver,
    }),
  ],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

`forRoot()` 方法接受的配置对象与 MikroORM 包中的 `init()` 方法相同。完整配置文档请参见 [此页面](https://mikro-orm.io/docs/configuration)。

我们也可以通过创建配置文件 `mikro-orm.config.ts` 来 [配置 CLI](https://mikro-orm.io/docs/installation#setting-up-the-commandline-tool)，然后调用 `forRoot()` 而不传递任何参数：

```typescript
@Module({
  imports: [
    MikroOrmModule.forRoot(),
  ],
  ...
})
export class AppModule {}
```

但是，如果你使用了基于树摇（tree shaking）的构建工具，这将不起作用。在这种情况下，建议显式提供配置：

```typescript
import config from './mikro-orm.config'; // 你的 ORM 配置

@Module({
  imports: [
    MikroOrmModule.forRoot(config),
  ],
  ...
})
export class AppModule {}
```

之后，`EntityManager` 将在整个项目中可用（无需在其他模块中导入）：

```ts
// 从你的驱动包或 `@mikro-orm/knex` 中导入
import { EntityManager, MikroORM } from '@mikro-orm/sqlite';

@Injectable()
export class MyService {
  constructor(
    private readonly orm: MikroORM,
    private readonly em: EntityManager,
  ) {}
}
```

> info **提示** 注意，`EntityManager` 是从 `@mikro-orm/driver` 包中导入的，其中 `driver` 是 `mysql`、`sqlite`、`postgres` 或你正在使用的驱动。如果你已安装 `@mikro-orm/knex`，也可以从那里导入 `EntityManager`。

#### 仓库（Repositories）

MikroORM 支持仓库设计模式。对于每个实体，我们都可以创建一个仓库。关于仓库的完整文档请参见 [这里](https://mikro-orm.io/docs/repositories)。要定义当前作用域中应注册哪些仓库，可以使用 `forFeature()` 方法。例如：

> info **提示** 你不应通过 `forFeature()` 注册基础实体，因为这些实体没有对应的仓库。但基础实体需要包含在 `forRoot()` 方法（或整体 ORM 配置）的实体列表中。

```typescript
// photo.module.ts
@Module({
  imports: [MikroOrmModule.forFeature([Photo])],
  providers: [PhotoService],
  controllers: [PhotoController],
})
export class PhotoModule {}
```

然后将其导入根模块 `AppModule`：

```typescript
// app.module.ts
@Module({
  imports: [MikroOrmModule.forRoot(...), PhotoModule],
})
export class AppModule {}
```

这样，我们就可以使用 `@InjectRepository()` 装饰器将 `PhotoRepository` 注入到 `PhotoService` 中：

```typescript
@Injectable()
export class PhotoService {
  constructor(
    @InjectRepository(Photo)
    private readonly photoRepository: EntityRepository<Photo>,
  ) {}
}
```

#### 使用自定义仓库

使用自定义仓库时，不再需要 `@InjectRepository()` 装饰器，因为 Nest 的依赖注入是基于类引用的。

```ts
// `./author.entity.ts`
@Entity({ repository: () => AuthorRepository })
export class Author {
  // 用于 `em.getRepository()` 的类型推断
  [EntityRepositoryType]?: AuthorRepository;
}

// `./author.repository.ts`
export class AuthorRepository extends EntityRepository<Author> {
  // 自定义方法...
}
```

由于自定义仓库的名称与 `getRepositoryToken()` 返回的名称相同，因此不再需要 `@InjectRepository()` 装饰器：

```ts
@Injectable()
export class MyService {
  constructor(private readonly repo: AuthorRepository) {}
}
```

#### 自动加载实体

手动将实体添加到连接配置的 `entities` 数组中可能会很繁琐。此外，从根模块引用实体会破坏应用的领域边界，并导致实现细节的泄露。为了解决这个问题，可以使用静态通配路径。

但请注意，Webpack 不支持通配路径，因此如果你是在单体仓库中构建应用，就无法使用它们。为了解决这个问题，可以使用替代方案：将配置对象（传入 `forRoot()` 方法的对象）中的 `autoLoadEntities` 属性设置为 `true`，如下所示：

```ts
@Module({
  imports: [
    MikroOrmModule.forRoot({
      ...
      autoLoadEntities: true,
    }),
  ],
})
export class AppModule {}
```

启用该选项后，所有通过 `forFeature()` 方法注册的实体都会自动添加到配置对象的 `entities` 数组中。

> info **提示** 注意，未通过 `forFeature()` 方法注册、仅通过关系引用的实体不会被 `autoLoadEntities` 包含。

> info **提示** 使用 `autoLoadEntities` 对 MikroORM CLI 没有效果 —— 对于 CLI，我们仍需要配置文件中完整的实体列表。不过，CLI 支持通配路径，因为它不会经过 Webpack。

#### 序列化

> warning **注意** MikroORM 会将每个实体关系包装在 `Reference<T>` 或 `Collection<T>` 对象中，以提供更好的类型安全性。这将导致 [Nest 内置的序列化器](/techniques/serialization) 无法识别这些被包装的关系。换句话说，如果你从 HTTP 或 WebSocket 处理器返回 MikroORM 实体，它们的所有关系将不会被序列化。

幸运的是，MikroORM 提供了一个 [序列化 API](https://mikro-orm.io/docs/serializing)，可以替代 `ClassSerializerInterceptor` 使用：

```typescript
@Entity()
export class Book {
  @Property({ hidden: true }) // 等效于 class-transformer 的 `@Exclude`
  hiddenField = Date.now();

  @Property({ persist: false }) // 类似于 class-transformer 的 `@Expose()`。只存在于内存中，并会被序列化。
  count?: number;

  @ManyToOne({
    serializer: (value) => value.name,
    serializedName: 'authorName',
  }) // 等效于 class-transformer 的 `@Transform()`
  author: Author;
}
```

#### 队列中的请求作用域处理器

如 [文档](https://mikro-orm.io/docs/identity-map) 所述，我们需要为每个请求维护一个干净的状态。这通过中间件注册的 `RequestContext` 辅助类自动完成。

但中间件仅在常规 HTTP 请求处理器中执行，如果我们需要在请求处理器之外执行请求作用域的方法怎么办？例如队列处理器或定时任务。

这时可以使用 `@CreateRequestContext()` 装饰器。它要求你先将 `MikroORM` 实例注入到当前上下文中，它将用于为你创建上下文。在底层，该装饰器会为你的方法注册一个新的请求上下文，并在该上下文中执行它。

```ts
@Injectable()
export class MyService {
  constructor(private readonly orm: MikroORM) {}

  @CreateRequestContext()
  async doSomething() {
    // 此代码将在一个新的上下文中执行
  }
}
```

> warning **注意** 正如其名，该装饰器始终会创建新的上下文，而 `@EnsureRequestContext` 则仅在当前没有上下文时才创建。

#### 测试

`@mikro-orm/nestjs` 包暴露了 `getRepositoryToken()` 函数，该函数根据给定实体返回准备好的 token，用于模拟仓库。

```typescript
@Module({
  providers: [
    PhotoService,
    {
      // 如果你使用的是自定义仓库：`provide: PhotoRepository`
      provide: getRepositoryToken(Photo),
      useValue: mockedRepository,
    },
  ],
})
export class PhotoModule {}
```

#### 示例

一个使用 NestJS 与 MikroORM 的完整示例，请参见 [这里](https://github.com/mikro-orm/nestjs-realworld-example-app)。