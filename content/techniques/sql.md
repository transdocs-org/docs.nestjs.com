### 数据库

Nest 是数据库无关的，允许你轻松集成任何 SQL 或 NoSQL 数据库。你可以根据自己的偏好选择多种方式。在最通用的层面，将 Nest 连接到数据库只需加载适当的 Node.js 数据库驱动，就像你使用 [Express](https://expressjs.com/en/guide/database-integration.html) 或 Fastify 时一样。

你也可以直接使用任何通用的 Node.js 数据库集成 **库** 或 ORM，例如 [MikroORM](https://mikro-orm.io/)（参见 [MikroORM 食谱](/recipes/mikroorm)）、[Sequelize](https://sequelize.org/)（参见 [Sequelize 集成](/techniques/database#sequelize-integration)）、[Knex.js](https://knexjs.org/)（参见 [Knex.js 教程](https://dev.to/nestjs/build-a-nestjs-module-for-knex-js-or-other-resource-based-libraries-in-5-minutes-12an)）、[TypeORM](https://github.com/typeorm/typeorm) 和 [Prisma](https://www.github.com/prisma/prisma)（参见 [Prisma 食谱](/recipes/prisma)），以在更高层次的抽象上进行操作。

为了方便起见，Nest 提供了开箱即用的 TypeORM 和 Sequelize 集成，分别通过 `@nestjs/typeorm` 和 `@nestjs/sequelize` 包，我们将在本章中介绍它们，以及 `@nestjs/mongoose` 的集成，这将在 [本章](/techniques/mongodb) 中介绍。这些集成提供了额外的 NestJS 特性，如模型/仓库注入、可测试性和异步配置，以使访问你选择的数据库更加简单。

### TypeORM 集成

为了与 SQL 和 NoSQL 数据库集成，Nest 提供了 `@nestjs/typeorm` 包。[TypeORM](https://github.com/typeorm/typeorm) 是目前最成熟的 TypeScript ORM。由于它是用 TypeScript 编写的，因此与 Nest 框架集成良好。

要开始使用它，我们首先安装所需的依赖项。在本章中，我们将演示使用流行的 [MySQL](https://www.mysql.com/) 关系型数据库管理系统，但 TypeORM 支持许多关系型数据库，如 PostgreSQL、Oracle、Microsoft SQL Server、SQLite，甚至像 MongoDB 这样的 NoSQL 数据库。本章中介绍的过程对于 TypeORM 支持的任何数据库都是相同的。你只需安装所选数据库的客户端 API 库即可。

```bash
$ npm install --save @nestjs/typeorm typeorm mysql2
```

安装完成后，我们可以将 `TypeOrmModule` 导入到根模块 `AppModule` 中。

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';

@Module({
  imports: [
    TypeOrmModule.forRoot({
      type: 'mysql',
      host: 'localhost',
      port: 3306,
      username: 'root',
      password: 'root',
      database: 'test',
      entities: [],
      synchronize: true,
    }),
  ],
})
export class AppModule {}
```

> warning **警告** 不要在生产环境中设置 `synchronize: true`，否则可能导致生产数据丢失。

`forRoot()` 方法支持 [TypeORM](https://typeorm.io/data-source-options#common-data-source-options) 包中 `DataSource` 构造函数暴露的所有配置属性。此外，还有一些额外的配置属性如下所示。

<table>
  <tr>
    <td><code>retryAttempts</code></td>
    <td>连接数据库的尝试次数（默认值：<code>10</code>）</td>
  </tr>
  <tr>
    <td><code>retryDelay</code></td>
    <td>连接重试之间的延迟（毫秒）（默认值：<code>3000</code>）</td>
  </tr>
  <tr>
    <td><code>autoLoadEntities</code></td>
    <td>如果为 <code>true</code>，实体将自动加载（默认值：<code>false</code>）</td>
  </tr>
</table>

> info **提示** 点击 [这里](https://typeorm.io/data-source-options) 了解更多关于数据源选项的信息。

完成这些设置后，TypeORM 的 `DataSource` 和 `EntityManager` 对象将在整个项目中可用（无需导入任何模块），例如：

```typescript
@@filename(app.module)
import { DataSource } from 'typeorm';

@Module({
  imports: [TypeOrmModule.forRoot(), UsersModule],
})
export class AppModule {
  constructor(private dataSource: DataSource) {}
}
@@switch
import { DataSource } from 'typeorm';

@Dependencies(DataSource)
@Module({
  imports: [TypeOrmModule.forRoot(), UsersModule],
})
export class AppModule {
  constructor(dataSource) {
    this.dataSource = dataSource;
  }
}
```

#### 仓库模式

[TypeORM](https://github.com/typeorm/typeorm) 支持 **仓库设计模式**，因此每个实体都有自己的仓库。这些仓库可以从数据库数据源中获取。

为了继续示例，我们需要至少一个实体。让我们定义 `User` 实体。

```typescript
@@filename(user.entity)
import { Entity, Column, PrimaryGeneratedColumn } from 'typeorm';

@Entity()
export class User {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  firstName: string;

  @Column()
  lastName: string;

  @Column({ default: true })
  isActive: boolean;
}
```

> info **提示** 了解更多关于实体的信息，请参见 [TypeORM 文档](https://typeorm.io/#/entities)。

`User` 实体文件位于 `users` 目录中。该目录包含与 `UsersModule` 相关的所有文件。你可以决定将模型文件放在哪里，但建议在相应的模块目录中靠近其 **领域** 创建它们。

要开始使用 `User` 实体，我们需要通过将其插入到模块 `forRoot()` 方法选项中的 `entities` 数组中来让 TypeORM 知道它（除非你使用静态 glob 路径）：

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { User } from './users/user.entity';

@Module({
  imports: [
    TypeOrmModule.forRoot({
      type: 'mysql',
      host: 'localhost',
      port: 3306,
      username: 'root',
      password: 'root',
      database: 'test',
      entities: [User],
      synchronize: true,
    }),
  ],
})
export class AppModule {}
```

接下来，我们看一下 `UsersModule`：

```typescript
@@filename(users.module)
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { UsersService } from './users.service';
import { UsersController } from './users.controller';
import { User } from './user.entity';

@Module({
  imports: [TypeOrmModule.forFeature([User])],
  providers: [UsersService],
  controllers: [UsersController],
})
export class UsersModule {}
```

这个模块使用 `forFeature()` 方法定义在当前作用域中注册哪些仓库。这样，我们就可以使用 `@InjectRepository()` 装饰器将 `UsersRepository` 注入到 `UsersService` 中：

```typescript
@@filename(users.service)
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { User } from './user.entity';

@Injectable()
export class UsersService {
  constructor(
    @InjectRepository(User)
    private usersRepository: Repository<User>,
  ) {}

  findAll(): Promise<User[]> {
    return this.usersRepository.find();
  }

  findOne(id: number): Promise<User | null> {
    return this.usersRepository.findOneBy({ id });
  }

  async remove(id: number): Promise<void> {
    await this.usersRepository.delete(id);
  }
}
@@switch
import { Injectable, Dependencies } from '@nestjs/common';
import { getRepositoryToken } from '@nestjs/typeorm';
import { User } from './user.entity';

@Injectable()
@Dependencies(getRepositoryToken(User))
export class UsersService {
  constructor(usersRepository) {
    this.usersRepository = usersRepository;
  }

  findAll() {
    return this.usersRepository.find();
  }

  findOne(id) {
    return this.usersRepository.findOneBy({ id });
  }

  async remove(id) {
    await this.usersRepository.delete(id);
  }
}
```

> warning **注意** 不要忘记将 `UsersModule` 导入根模块 `AppModule`。

如果你想在导入 `TypeOrmModule.forFeature` 的模块之外使用仓库，你需要重新导出由它生成的提供者。
你可以通过导出整个模块来实现这一点，如下所示：

```typescript
@@filename(users.module)
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { User } from './user.entity';

@Module({
  imports: [TypeOrmModule.forFeature([User])],
  exports: [TypeOrmModule]
})
export class UsersModule {}
```

现在，如果我们把 `UsersModule` 导入 `UserHttpModule`，我们就可以在后者的提供者中使用 `@InjectRepository(User)`。

```typescript
@@filename(users-http.module)
import { Module } from '@nestjs/common';
import { UsersModule } from './users.module';
import { UsersService } from './users.service';
import { UsersController } from './users.controller';

@Module({
  imports: [UsersModule],
  providers: [UsersService],
  controllers: [UsersController]
})
export class UserHttpModule {}
```

#### 关系

关系是建立在两个或多个表之间的关联。关系基于每个表的公共字段，通常涉及主键和外键。

有三种类型的关系：

<table>
  <tr>
    <td><code>一对一</code></td>
    <td>主表中的每一行在外部表中只有一个且仅有一个关联行。使用 <code>@OneToOne()</code> 装饰器定义这种类型的关系。</td>
  </tr>
  <tr>
    <td><code>一对多 / 多对一</code></td>
    <td>主表中的每一行在外部表中有一行或多行相关行。使用 <code>@OneToMany()</code> 和 <code>@ManyToOne()</code> 装饰器定义这种类型的关系。</td>
  </tr>
  <tr>
    <td><code>多对多</code></td>
    <td>主表中的每一行在外部表中有多个相关行，外部表中的每条记录在主表中也有多个相关行。使用 <code>@ManyToMany()</code> 装饰器定义这种类型的关系。</td>
  </tr>
</table>

要在实体中定义关系，请使用相应的 **装饰器**。例如，要定义每个 `User` 可以拥有多个照片，请使用 `@OneToMany()` 装饰器。

```typescript
@@filename(user.entity)
import { Entity, Column, PrimaryGeneratedColumn, OneToMany } from 'typeorm';
import { Photo } from '../photos/photo.entity';

@Entity()
export class User {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  firstName: string;

  @Column()
  lastName: string;

  @Column({ default: true })
  isActive: boolean;

  @OneToMany(type => Photo, photo => photo.user)
  photos: Photo[];
}
```

> info **提示** 要了解更多关于 TypeORM 中的关系，请访问 [TypeORM 文档](https://typeorm.io/#/relations)。

#### 自动加载实体

手动将实体添加到数据源选项的 `entities` 数组中可能会很繁琐。此外，从根模块引用实体会破坏应用程序的领域边界，并导致实现细节泄露到应用程序的其他部分。为了解决这个问题，提供了一种替代解决方案。要自动加载实体，请将配置对象（传递给 `forRoot()` 方法）的 `autoLoadEntities` 属性设置为 `true`，如下所示：

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';

@Module({
  imports: [
    TypeOrmModule.forRoot({
      ...
      autoLoadEntities: true,
    }),
  ],
})
export class AppModule {}
```

指定该选项后，通过 `forFeature()` 方法注册的每个实体将自动添加到配置对象的 `entities` 数组中。

> warning **警告** 注意，那些没有通过 `forFeature()` 方法注册，而是仅通过关系从实体中引用的实体，不会通过 `autoLoadEntities` 设置包含在内。

#### 分离实体定义

你可以直接在模型中使用装饰器定义实体和其列。但有些人更喜欢使用 ["实体模式"](https://typeorm.io/#/separating-entity-definition) 在单独的文件中定义实体和其列。

```typescript
import { EntitySchema } from 'typeorm';
import { User } from './user.entity';

export const UserSchema = new EntitySchema<User>({
  name: 'User',
  target: User,
  columns: {
    id: {
      type: Number,
      primary: true,
      generated: true,
    },
    firstName: {
      type: String,
    },
    lastName: {
      type: String,
    },
    isActive: {
      type: Boolean,
      default: true,
    },
  },
  relations: {
    photos: {
      type: 'one-to-many',
      target: 'Photo', // PhotoSchema 的名称
    },
  },
});
```

> warning error **警告** 如果你提供了 `target` 选项，那么 `name` 选项的值必须与目标类的名称相同。
> 如果你不提供 `target`，你可以使用任何名称。

Nest 允许你在任何需要 `Entity` 的地方使用 `EntitySchema` 实例，例如：

```typescript
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { UserSchema } from './user.schema';
import { UsersController } from './users.controller';
import { UsersService } from './users.service';

@Module({
  imports: [TypeOrmModule.forFeature([UserSchema])],
  providers: [UsersService],
  controllers: [UsersController],
})
export class UsersModule {}
```

#### TypeORM 事务

数据库事务表示在数据库管理系统中对数据库执行的一组工作单元，并以一种连贯和可靠的方式处理，独立于其他事务。事务通常代表数据库中的任何更改（[了解更多](https://en.wikipedia.org/wiki/Database_transaction)）。

处理 [TypeORM 事务](https://typeorm.io/#/transactions) 有多种不同的策略。我们推荐使用 `QueryRunner` 类，因为它提供了对事务的完全控制。

首先，我们需要以正常方式将 `DataSource` 对象注入到一个类中：

```typescript
@Injectable()
export class UsersService {
  constructor(private dataSource: DataSource) {}
}
```

> info **提示** `DataSource` 类是从 `typeorm` 包中导入的。

现在，我们可以使用这个对象来创建一个事务。

```typescript
async createMany(users: User[]) {
  const queryRunner = this.dataSource.createQueryRunner();

  await queryRunner.connect();
  await queryRunner.startTransaction();
  try {
    await queryRunner.manager.save(users[0]);
    await queryRunner.manager.save(users[1]);

    await queryRunner.commitTransaction();
  } catch (err) {
    // 因为我们遇到了错误，所以回滚我们所做的更改
    await queryRunner.rollbackTransaction();
  } finally {
    // 你需要释放手动实例化的 queryRunner
    await queryRunner.release();
  }
}
```

> info **提示** 注意，`dataSource` 仅用于创建 `QueryRunner`。然而，要测试这个类将需要模拟整个 `DataSource` 对象（它暴露了几个方法）。因此，我们建议使用一个辅助工厂类（例如，`QueryRunnerFactory`）并定义一个包含有限方法的接口以维护事务。这种技术使得模拟这些方法变得非常简单。

<app-banner-devtools></app-banner-devtools>

或者，你可以使用带有 `DataSource` 对象的 `transaction` 方法的回调风格方法（[阅读更多](https://typeorm.io/#/transactions/creating-and-using-transactions)）。

```typescript
async createMany(users: User[]) {
  await this.dataSource.transaction(async manager => {
    await manager.save(users[0]);
    await manager.save(users[1]);
  });
}
```

#### 订阅者

使用 TypeORM 的 [订阅者](https://typeorm.io/#/listeners-and-subscribers/what-is-a-subscriber)，你可以监听特定实体事件。

```typescript
import {
  DataSource,
  EntitySubscriberInterface,
  EventSubscriber,
  InsertEvent,
} from 'typeorm';
import { User } from './user.entity';

@EventSubscriber()
export class UserSubscriber implements EntitySubscriberInterface<User> {
  constructor(dataSource: DataSource) {
    dataSource.subscribers.push(this);
  }

  listenTo() {
    return User;
  }

  beforeInsert(event: InsertEvent<User>) {
    console.log(`BEFORE USER INSERTED: `, event.entity);
  }
}
```

> error **警告** 事件订阅者不能是 [请求作用域](/fundamentals/injection-scopes) 的。

现在，将 `UserSubscriber` 类添加到 `providers` 数组中：

```typescript
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { User } from './user.entity';
import { UsersController } from './users.controller';
import { UsersService } from './users.service';
import { UserSubscriber } from './user.subscriber';

@Module({
  imports: [TypeOrmModule.forFeature([User])],
  providers: [UsersService, UserSubscriber],
  controllers: [UsersController],
})
export class UsersModule {}
```

> info **提示** 了解更多关于实体订阅者的信息，请参见 [这里](https://typeorm.io/#/listeners-and-subscribers/what-is-a-subscriber)。

#### 迁移

[迁移](https://typeorm.io/#/migrations) 提供了一种逐步更新数据库模式的方法，以使其与应用程序的数据模型保持同步，同时保留数据库中的现有数据。要生成、运行和回滚迁移，TypeORM 提供了一个专用的 [CLI](https://typeorm.io/#/migrations/creating-a-new-migration)。

迁移类与 Nest 应用程序的源代码是分开的。它们的生命周期由 TypeORM CLI 维护。因此，你无法利用依赖注入和其他特定于 Nest 的功能进行迁移。要了解更多关于迁移的信息，请参见 [TypeORM 文档](https://typeorm.io/#/migrations/creating-a-new-migration) 中的指南。

#### 多个数据库

有些项目需要多个数据库连接。这也可以通过此模块实现。要使用多个连接，首先创建连接。在这种情况下，数据源命名变得**必须**。

假设你有一个 `Album` 实体存储在其自己的数据库中。

```typescript
const defaultOptions = {
  type: 'postgres',
  port: 5432,
  username: 'user',
  password: 'password',
  database: 'db',
  synchronize: true,
};

@Module({
  imports: [
    TypeOrmModule.forRoot({
      ...defaultOptions,
      host: 'user_db_host',
      entities: [User],
    }),
    TypeOrmModule.forRoot({
      ...defaultOptions,
      name: 'albumsConnection',
      host: 'album_db_host',
      entities: [Album],
    }),
  ],
})
export class AppModule {}
```

> warning **注意** 如果你不为数据源设置 `name`，其名称将被设置为 `default`。请注意，你不应该有多个没有名称的连接，或具有相同名称的连接，否则它们将被覆盖。

> warning **注意** 如果你使用 `TypeOrmModule.forRootAsync`，你必须**同时**在 `useFactory` 之外设置数据源名称。例如：
>
> ```typescript
> TypeOrmModule.forRootAsync({
>   name: 'albumsConnection',
>   useFactory: ...,
>   inject: ...,
> }),
> ```
>
> 更多细节请参见 [此问题](https://github.com/nestjs/typeorm/issues/86)。

此时，你已经将 `User` 和 `Album` 实体注册到了它们各自的数据源中。有了这个设置，你需要告诉 `TypeOrmModule.forFeature()` 方法和 `@InjectRepository()` 装饰器应该使用哪个数据源。如果你不传递任何数据源名称，则会使用 `default` 数据源。

```typescript
@Module({
  imports: [
    TypeOrmModule.forFeature([User]),
    TypeOrmModule.forFeature([Album], 'albumsConnection'),
  ],
})
export class AppModule {}
```

你还可以注入给定数据源的 `DataSource` 或 `EntityManager`：

```typescript
@Injectable()
export class AlbumsService {
  constructor(
    @InjectDataSource('albumsConnection')
    private dataSource: DataSource,
    @InjectEntityManager('albumsConnection')
    private entityManager: EntityManager,
  ) {}
}
```

也可以将任何 `DataSource` 注入到提供者中：

```typescript
@Module({
  providers: [
    {
      provide: AlbumsService,
      useFactory: (albumsConnection: DataSource) => {
        return new AlbumsService(albumsConnection);
      },
      inject: [getDataSourceToken('albumsConnection')],
    },
  ],
})
export class AlbumsModule {}
```

#### 测试

在进行单元测试时，我们通常希望避免建立数据库连接，以保持测试套件的独立性并加快执行过程。但我们的类可能依赖于从数据源（连接）实例中提取的仓库。我们如何处理这种情况？解决方案是创建模拟仓库。为此，我们需要设置 [自定义提供者](/fundamentals/custom-providers)。每个注册的仓库都会自动由 `<EntityName>Repository` 令牌表示，其中 `EntityName` 是你的实体类的名称。

`@nestjs/typeorm` 包暴露了 `getRepositoryToken()` 函数，该函数返回一个基于给定实体准备好的令牌。

```typescript
@Module({
  providers: [
    UsersService,
    {
      provide: getRepositoryToken(User),
      useValue: mockRepository,
    },
  ],
})
export class UsersModule {}
```

现在，替代的 `mockRepository` 将被用作 `UsersRepository`。每当任何类使用 `@InjectRepository()` 装饰器请求 `UsersRepository` 时，Nest 将使用注册的 `mockRepository` 对象。

#### 异步配置

你可能希望异步而不是静态地传递仓库模块选项。在这种情况下，使用 `forRootAsync()` 方法，它提供了几种处理异步配置的方法。

一种方法是使用工厂函数：

```typescript
TypeOrmModule.forRootAsync({
  useFactory: () => ({
    type: 'mysql',
    host: 'localhost',
    port: 3306,
    username: 'root',
    password: 'root',
    database: 'test',
    entities: [],
    synchronize: true,
  }),
});
```

我们的工厂表现得像任何其他 [异步提供者](https://docs.nestjs.com/fundamentals/async-providers)（例如，它可以是 `async` 的，并且可以通过 `inject` 注入依赖项）。

```typescript
TypeOrmModule.forRootAsync({
  imports: [ConfigModule],
  useFactory: (configService: ConfigService) => ({
    type: 'mysql',
    host: configService.get('HOST'),
    port: +configService.get('PORT'),
    username: configService.get('USERNAME'),
    password: configService.get('PASSWORD'),
    database: configService.get('DATABASE'),
    entities: [],
    synchronize: true,
  }),
  inject: [ConfigService],
});
```

或者，你可以使用 `useClass` 语法：

```typescript
TypeOrmModule.forRootAsync({
  useClass: TypeOrmConfigService,
});
```

上述构造将在 `TypeOrmModule` 内实例化 `TypeOrmConfigService` 并使用它通过调用 `createTypeOrmOptions()` 来提供一个选项对象。请注意，这意味着 `TypeOrmConfigService` 必须实现 `TypeOrmOptionsFactory` 接口，如下所示：

```typescript
@Injectable()
export class TypeOrmConfigService implements TypeOrmOptionsFactory {
  createTypeOrmOptions(): TypeOrmModuleOptions {
    return {
      type: 'mysql',
      host: 'localhost',
      port: 3306,
      username: 'root',
      password: 'root',
      database: 'test',
      entities: [],
      synchronize: true,
    };
  }
}
```

为了防止在 `TypeOrmModule` 内部创建 `TypeOrmConfigService` 并使用从其他模块导入的提供者，你可以使用 `useExisting` 语法。

```typescript
TypeOrmModule.forRootAsync({
  imports: [ConfigModule],
  useExisting: ConfigService,
});
```

这种构造与 `useClass` 的工作方式相同，但有一个关键区别——`TypeOrmModule` 将查找导入的模块以重用现有的 `ConfigService`，而不是实例化一个新的。

> info **提示** 确保 `name` 属性与 `useFactory`、`useClass` 或 `useValue` 属性在同一个级别定义。这将允许 Nest 正确地在适当的注入令牌下注册数据源。

#### 自定义数据源工厂

结合使用 `useFactory`、`useClass` 或 `useExisting` 的异步配置，你可以选择性地指定一个 `dataSourceFactory` 函数，这将允许你提供自己的 TypeORM 数据源，而不是让 `TypeOrmModule` 创建数据源。

`dataSourceFactory` 接收在使用 `useFactory`、`useClass` 或 `useExisting` 进行异步配置期间配置的 TypeORM `DataSourceOptions`，并返回一个解析为 TypeORM `DataSource` 的 `Promise`。

```typescript
TypeOrmModule.forRootAsync({
  imports: [ConfigModule],
  inject: [ConfigService],
  // 使用 useFactory、useClass 或 useExisting
  // 配置 DataSourceOptions。
  useFactory: (configService: ConfigService) => ({
    type: 'mysql',
    host: configService.get('HOST'),
    port: +configService.get('PORT'),
    username: configService.get('USERNAME'),
    password: configService.get('PASSWORD'),
    database: configService.get('DATABASE'),
    entities: [],
    synchronize: true,
  }),
  // dataSource 接收配置的 DataSourceOptions
  // 并返回一个 Promise<DataSource>。
  dataSourceFactory: async (options) => {
    const dataSource = await new DataSource(options).initialize();
    return dataSource;
  },
});
```

> info **提示** `DataSource` 类是从 `typeorm` 包中导入的。

#### 示例

一个工作示例可在 [这里](https://github.com/nestjs/nest/tree/master/sample/05-sql-typeorm) 找到。

<app-banner-enterprise></app-banner-enterprise>

### Sequelize 集成

使用 TypeORM 的替代方案是使用 [Sequelize](https://sequelize.org/) ORM 和 `@nestjs/sequelize` 包。此外，我们利用 [sequelize-typescript](https://github.com/RobinBuschmann/sequelize-typescript) 包，它提供了一组额外的装饰器来声明性地定义实体。

要开始使用它，我们首先安装所需的依赖项。在本章中，我们将演示使用流行的 [MySQL](https://www.mysql.com/) 关系型数据库管理系统，但 Sequelize 支持许多关系型数据库，如 PostgreSQL、MySQL、Microsoft SQL Server、SQLite 和 MariaDB。本章中介绍的过程对于 Sequelize 支持的任何数据库都是一样的。你只需安装所选数据库的关联客户端 API 库即可。

```bash
$ npm install --save @nestjs/sequelize sequelize sequelize-typescript mysql2
$ npm install --save-dev @types/sequelize
```

安装完成后，我们可以将 `SequelizeModule` 导入到根模块 `AppModule` 中。

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { SequelizeModule } from '@nestjs/sequelize';

@Module({
  imports: [
    SequelizeModule.forRoot({
      dialect: 'mysql',
      host: 'localhost',
      port: 3306,
      username: 'root',
      password: 'root',
      database: 'test',
      models: [],
    }),
  ],
})
export class AppModule {}
```

`forRoot()` 方法支持 Sequelize 构造函数暴露的所有配置属性（[阅读更多](https://sequelize.org/docs/v6/getting-started/#connecting-to-a-database)）。此外，还有以下额外的配置属性。

<table>
  <tr>
    <td><code>retryAttempts</code></td>
    <td>连接数据库的尝试次数（默认值：<code>10</code>）</td>
  </tr>
  <tr>
    <td><code>retryDelay</code></td>
    <td>连接重试之间的延迟（毫秒）（默认值：<code>3000</code>）</td>
  </tr>
  <tr>
    <td><code>autoLoadModels</code></td>
    <td>如果为 <code>true</code>，模型将自动加载（默认值：<code>false</code>）</td>
  </tr>
  <tr>
    <td><code>keepConnectionAlive</code></td>
    <td>如果为 <code>true</code>，在应用程序关闭时连接不会关闭（默认值：<code>false</code>）</td>
  </tr>
  <tr>
    <td><code>synchronize</code></td>
    <td>如果为 <code>true</code>，自动加载的模型将被同步（默认值：<code>true</code>）</td>
  </tr>
</table>

完成这些设置后，`Sequelize` 对象将在整个项目中可用（无需导入任何模块），例如：

```typescript
@@filename(app.service)
import { Injectable } from '@nestjs/common';
import { Sequelize } from 'sequelize-typescript';

@Injectable()
export class AppService {
  constructor(private sequelize: Sequelize) {}
}
@@switch
import { Injectable } from '@nestjs/common';
import { Sequelize } from 'sequelize-typescript';

@Dependencies(Sequelize)
@Injectable()
export class AppService {
  constructor(sequelize) {
    this.sequelize = sequelize;
  }
}
```

#### 模型

Sequelize 实现了主动记录模式。使用此模式，你可以直接使用模型类与数据库交互。为了继续示例，我们需要至少一个模型。让我们定义 `User` 模型。

```typescript
@@filename(user.model)
import { Column, Model, Table } from 'sequelize-typescript';

@Table
export class User extends Model {
  @Column
  firstName: string;

  @Column
  lastName: string;

  @Column({ defaultValue: true })
  isActive: boolean;
}
```

> info **提示** 了解更多关于可用装饰器的信息，请参见 [这里](https://github.com/RobinBuschmann/sequelize-typescript#column)。

`User` 模型文件位于 `users` 目录中。该目录包含与 `UsersModule` 相关的所有文件。你可以决定将模型文件放在哪里，但建议在相应的模块目录中靠近其 **领域** 创建它们。

要开始使用 `User` 模型，我们需要通过将其插入到模块 `forRoot()` 方法选项中的 `models` 数组中来让 Sequelize 知道它：

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { SequelizeModule } from '@nestjs/sequelize';
import { User } from './users/user.model';

@Module({
  imports: [
    SequelizeModule.forRoot({
      dialect: 'mysql',
      host: 'localhost',
      port: 3306,
      username: 'root',
      password: 'root',
      database: 'test',
      models: [User],
    }),
  ],
})
export class AppModule {}
```

接下来，我们看一下 `UsersModule`：

```typescript
@@filename(users.module)
import { Module } from '@nestjs/common';
import { SequelizeModule } from '@nestjs/sequelize';
import { User } from './user.model';
import { UsersController } from './users.controller';
import { UsersService } from './users.service';

@Module({
  imports: [SequelizeModule.forFeature([User])],
  providers: [UsersService],
  controllers: [UsersController],
})
export class UsersModule {}
```

该模块使用 `forFeature()` 方法定义在当前作用域中注册哪些模型。这样，我们就可以使用 `@InjectModel()` 装饰器将 `UserModel` 注入到 `UsersService` 中：

```typescript
@@filename(users.service)
import { Injectable } from '@nestjs/common';
import { InjectModel } from '@nestjs/sequelize';
import { User } from './user.model';

@Injectable()
export class UsersService {
  constructor(
    @InjectModel(User)
    private userModel: typeof User,
  ) {}

  async findAll(): Promise<User[]> {
    return this.userModel.findAll();
  }

  findOne(id: string): Promise<User> {
    return this.userModel.findOne({
      where: {
        id,
      },
    });
  }

  async remove(id: string): Promise<void> {
    const user = await this.findOne(id);
    await user.destroy();
  }
}
@@switch
import { Injectable, Dependencies } from '@nestjs/common';
import { getModelToken } from '@nestjs/sequelize';
import { User } from './user.model';

@Injectable()
@Dependencies(getModelToken(User))
export class UsersService {
  constructor(usersRepository) {
    this.usersRepository = usersRepository;
  }

  async findAll() {
    return this.userModel.findAll();
  }

  findOne(id) {
    return this.userModel.findOne({
      where: {
        id,
      },
    });
  }

  async remove(id) {
    const user = await this.findOne(id);
    await user.destroy();
  }
}
```

> warning **注意** 不要忘记将 `UsersModule` 导入根模块 `AppModule`。

如果你想在导入 `SequelizeModule.forFeature` 的模块之外使用模型，你需要重新导出由它生成的提供者。
你可以通过导出整个模块来实现这一点，如下所示：

```typescript
@@filename(users.module)
import { Module } from '@nestjs/common';
import { SequelizeModule } from '@nestjs/sequelize';
import { User } from './user.entity';

@Module({
  imports: [SequelizeModule.forFeature([User])],
  exports: [SequelizeModule]
})
export class UsersModule {}
```

现在，如果我们把 `UsersModule` 导入 `UserHttpModule`，我们就可以在后者的提供者中使用 `@InjectModel(User)`。

```typescript
@@filename(users-http.module)
import { Module } from '@nestjs/common';
import { UsersModule } from './users.module';
import { UsersService } from './users.service';
import { UsersController } from './users.controller';

@Module({
  imports: [UsersModule],
  providers: [UsersService],
  controllers: [UsersController]
})
export class UserHttpModule {}
```

#### 关系

关系是建立在两个或多个表之间的关联。关系基于每个表的公共字段，通常涉及主键和外键。

有三种类型的关系：

<table>
  <tr>
    <td><code>一对一</code></td>
    <td>主表中的每一行在外部表中只有一个且仅有一个关联行</td>
  </tr>
  <tr>
    <td><code>一对多 / 多对一</code></td>
    <td>主表中的每一行在外部表中有一行或多行相关行</td>
  </tr>
  <tr>
    <td><code>多对多</code></td>
    <td>主表中的每一行在外部表中有多个相关行，外部表中的每条记录在主表中也有多个相关行</td>
  </tr>
</table>

要在模型中定义关系，请使用相应的 **装饰器**。例如，要定义每个 `User` 可以拥有多个照片，请使用 `@HasMany()` 装饰器。

```typescript
@@filename(user.model)
import { Column, Model, Table, HasMany } from 'sequelize-typescript';
import { Photo } from '../photos/photo.model';

@Table
export class User extends Model {
  @Column
  firstName: string;

  @Column
  lastName: string;

  @Column({ defaultValue: true })
  isActive: boolean;

  @HasMany(() => Photo)
  photos: Photo[];
}
```

> info **提示** 要了解更多关于 Sequelize 中的关联，请阅读 [这一章](https://github.com/RobinBuschmann/sequelize-typescript#model-association)。

#### 自动加载模型

手动将模型添加到连接选项的 `models` 数组中可能会很繁琐。此外，从根模块引用模型会破坏应用程序的领域边界，并导致实现细节泄露到应用程序的其他部分。为了解决这个问题，通过将配置对象的 `autoLoadModels` 和 `synchronize` 属性（传递给 `forRoot()` 方法）都设置为 `true` 来自动加载模型，如下所示：

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { SequelizeModule } from '@nestjs/sequelize';

@Module({
  imports: [
    SequelizeModule.forRoot({
      ...
      autoLoadModels: true,
      synchronize: true,
    }),
  ],
})
export class AppModule {}
```

指定该选项后，通过 `forFeature()` 方法注册的每个模型将自动添加到配置对象的 `models` 数组中。

> warning **警告** 注意，那些没有通过 `forFeature()` 方法注册，而是仅通过关联从模型中引用的模型不会被包含在内。

#### Sequelize 事务

数据库事务表示在数据库管理系统中对数据库执行的一组工作单元，并以一种连贯和可靠的方式处理，独立于其他事务。事务通常代表数据库中的任何更改（[了解更多](https://en.wikipedia.org/wiki/Database_transaction)）。

处理 [Sequelize 事务](https://sequelize.org/docs/v6/other-topics/transactions/) 有多种不同的策略。下面是一个管理事务（自动回调）的示例实现。

首先，我们需要以正常方式将 `Sequelize` 对象注入到一个类中：

```typescript
@Injectable()
export class UsersService {
  constructor(private sequelize: Sequelize) {}
}
```

> info **提示** `Sequelize` 类是从 `sequelize-typescript` 包中导入的。

现在，我们可以使用这个对象来创建一个事务。

```typescript
async createMany() {
  try {
    await this.sequelize.transaction(async t => {
      const transactionHost = { transaction: t };

      await this.userModel.create(
          { firstName: 'Abraham', lastName: 'Lincoln' },
          transactionHost,
      );
      await this.userModel.create(
          { firstName: 'John', lastName: 'Boothe' },
          transactionHost,
      );
    });
  } catch (err) {
    // 事务已被回滚
    // err 是传递给事务回调的承诺链所拒绝的错误
  }
}
```

> info **提示** 注意，`Sequelize` 实例仅用于启动事务。但是，要测试此类需要模拟整个 `Sequelize` 对象（它暴露了多个方法）。因此，我们建议使用一个辅助工厂类（例如 `TransactionRunner`）并定义一个包含有限方法的接口以维护事务。这种方法使得模拟这些方法变得非常简单。

#### 迁移

[迁移](https://sequelize.org/docs/v6/other-topics/migrations/) 提供了一种逐步更新数据库模式的方法，以使其与应用程序的数据模型保持同步，同时保留数据库中的现有数据。要生成、运行和回滚迁移，Sequelize 提供了一个专用的 [CLI](https://sequelize.org/docs/v6/other-topics/migrations/#installing-the-cli)。

迁移类与 Nest 应用程序的源代码是分开的。它们的生命周期由 Sequelize CLI 维护。因此，你无法利用依赖注入和其他特定于 Nest 的功能进行迁移。要了解更多关于迁移的信息，请参见 [Sequelize 文档](https://sequelize.org/docs/v6/other-topics/migrations/#installing-the-cli) 中的指南。

<app-banner-courses></app-banner-courses>

#### 多个数据库

有些项目需要多个数据库连接。这也可以通过此模块实现。要使用多个连接，首先创建连接。在这种情况下，连接命名变得**必须**。

假设你有一个 `Album` 实体存储在其自己的数据库中。

```typescript
const defaultOptions = {
  dialect: 'postgres',
  port: 5432,
  username: 'user',
  password: 'password',
  database: 'db',
  synchronize: true,
};

@Module({
  imports: [
    SequelizeModule.forRoot({
      ...defaultOptions,
      host: 'user_db_host',
      models: [User],
    }),
    SequelizeModule.forRoot({
      ...defaultOptions,
      name: 'albumsConnection',
      host: 'album_db_host',
      models: [Album],
    }),
  ],
})
export class AppModule {}
```

> warning **注意** 如果你不为连接设置 `name`，其名称将被设置为 `default`。请注意，你不应该有多个没有名称的连接，或者具有相同名称的连接，否则它们将被覆盖。

此时，你已经将 `User` 和 `Album` 模型注册到了它们各自对应的连接中。有了这个设置，你需要告诉 `SequelizeModule.forFeature()` 方法和 `@InjectModel()` 装饰器应该使用哪个连接。如果你不传递任何连接名称，则使用 `default` 连接。

```typescript
@Module({
  imports: [
    SequelizeModule.forFeature([User]),
    SequelizeModule.forFeature([Album], 'albumsConnection'),
  ],
})
export class AppModule {}
```

你还可以为给定的连接注入 `Sequelize` 实例：

```typescript
@Injectable()
export class AlbumsService {
  constructor(
    @InjectConnection('albumsConnection')
    private sequelize: Sequelize,
  ) {}
}
```

也可以将任何 `Sequelize` 实例注入到提供者中：

```typescript
@Module({
  providers: [
    {
      provide: AlbumsService,
      useFactory: (albumsSequelize: Sequelize) => {
        return new AlbumsService(albumsSequelize);
      },
      inject: [getDataSourceToken('albumsConnection')],
    },
  ],
})
export class AlbumsModule {}
```

#### 测试

在进行单元测试时，我们通常希望避免建立数据库连接，以保持测试套件的独立性并加快执行过程。但我们的类可能依赖于从连接实例中提取的模型。我们如何处理这种情况？解决方案是创建模拟模型。为此，我们需要设置 [自定义提供者](/fundamentals/custom-providers)。每个注册的模型都会自动由 `<ModelName>Model` 令牌表示，其中 `ModelName` 是你的模型类的名称。

`@nestjs/sequelize` 包暴露了 `getModelToken()` 函数，该函数返回一个基于给定模型准备好的令牌。

```typescript
@Module({
  providers: [
    UsersService,
    {
      provide: getModelToken(User),
      useValue: mockModel,
    },
  ],
})
export class UsersModule {}
```

现在，替代的 `mockModel` 将被用作 `UserModel`。每当任何类使用 `@InjectModel()` 装饰器请求 `UserModel` 时，Nest 将使用注册的 `mockModel` 对象。

#### 异步配置

你可能希望异步而不是静态地传递 `SequelizeModule` 选项。在这种情况下，使用 `forRootAsync()` 方法，它提供了几种处理异步配置的方法。

一种方法是使用工厂函数：

```typescript
SequelizeModule.forRootAsync({
  useFactory: () => ({
    dialect: 'mysql',
    host: 'localhost',
    port: 3306,
    username: 'root',
    password: 'root',
    database: 'test',
    models: [],
  }),
});
```

我们的工厂表现得像任何其他 [异步提供者](https://docs.nestjs.com/fundamentals/async-providers)（例如，它可以是 `async` 的，并且可以通过 `inject` 注入依赖项）。

```typescript
SequelizeModule.forRootAsync({
  imports: [ConfigModule],
  useFactory: (configService: ConfigService) => ({
    dialect: 'mysql',
    host: configService.get('HOST'),
    port: +configService.get('PORT'),
    username: configService.get('USERNAME'),
    password: configService.get('PASSWORD'),
    database: configService.get('DATABASE'),
    models: [],
  }),
  inject: [ConfigService],
});
```

或者，你可以使用 `useClass` 语法：

```typescript
SequelizeModule.forRootAsync({
  useClass: SequelizeConfigService,
});
```

上面的构造将在 `SequelizeModule` 内部实例化 `SequelizeConfigService` 并使用它通过调用 `createSequelizeOptions()` 来提供一个选项对象。请注意，这意味着 `SequelizeConfigService` 必须实现 `SequelizeOptionsFactory` 接口，如下所示：

```typescript
@Injectable()
class SequelizeConfigService implements SequelizeOptionsFactory {
  createSequelizeOptions(): SequelizeModuleOptions {
    return {
      dialect: 'mysql',
      host: 'localhost',
      port: 3306,
      username: 'root',
      password: 'root',
      database: 'test',
      models: [],
    };
  }
}
```

为了防止在 `SequelizeModule` 内部创建 `SequelizeConfigService` 并使用从其他模块导入的提供者，你可以使用 `useExisting` 语法。

```typescript
SequelizeModule.forRootAsync({
  imports: [ConfigModule],
  useExisting: ConfigService,
});
```

这种构造与 `useClass` 的工作方式相同，但有一个关键区别——`SequelizeModule` 将查找导入的模块以重用现有的 `ConfigService`，而不是实例化一个新的。

#### 示例

一个工作示例可在 [这里](https://github.com/nestjs/nest/tree/master/sample/07-sequelize) 找到。