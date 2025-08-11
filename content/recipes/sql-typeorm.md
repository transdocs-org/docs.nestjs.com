### SQL (TypeORM)

##### 本章仅适用于 TypeScript

> **警告** 在本文中，你将学习如何使用自定义提供者机制从头开始创建一个基于 **TypeORM** 包的 `DatabaseModule`。因此，这个解决方案包含了许多你可以通过现成的 `@nestjs/typeorm` 包来避免的额外步骤。要了解更多信息，请参见[此处](/techniques/sql)。

[TypeORM](https://github.com/typeorm/typeorm) 是 node.js 世界中最成熟的对象关系映射器（ORM）。由于它是使用 TypeScript 编写的，因此与 Nest 框架配合使用非常良好。

#### 入门

要开始使用这个库，我们需要安装所有必需的依赖项：

```bash
$ npm install --save typeorm mysql2
```

我们首先要做的一步是使用从 `typeorm` 包导入的 `new DataSource().initialize()` 类来建立与数据库的连接。`initialize()` 函数返回一个 `Promise`，因此我们必须创建一个 [异步提供者](/fundamentals/async-components)。

```typescript
@@filename(database.providers)
import { DataSource } from 'typeorm';

export const databaseProviders = [
  {
    provide: 'DATA_SOURCE',
    useFactory: async () => {
      const dataSource = new DataSource({
        type: 'mysql',
        host: 'localhost',
        port: 3306,
        username: 'root',
        password: 'root',
        database: 'test',
        entities: [
            __dirname + '/../**/*.entity{.ts,.js}',
        ],
        synchronize: true,
      });

      return dataSource.initialize();
    },
  },
];
```

> warning **警告** 不应在生产环境中使用 `synchronize: true`，否则可能导致生产数据丢失。

> info **提示** 遵循最佳实践，我们将自定义提供者声明在一个以 `*.providers.ts` 为后缀的独立文件中。

然后，我们需要导出这些提供者，以便整个应用程序可以访问它们。

```typescript
@@filename(database.module)
import { Module } from '@nestjs/common';
import { databaseProviders } from './database.providers';

@Module({
  providers: [...databaseProviders],
  exports: [...databaseProviders],
})
export class DatabaseModule {}
```

现在我们可以使用 `@Inject()` 装饰器注入 `DATA_SOURCE` 对象。每个依赖于 `DATA_SOURCE` 异步提供者的类都将等待 `Promise` 被解析。

#### 仓储模式

[TypeORM](https://github.com/typeorm/typeorm) 支持仓储设计模式，因此每个实体都有其对应的 Repository。这些仓储可以从数据库连接中获取。

但首先，我们需要至少一个实体。我们将复用官方文档中的 `Photo` 实体。

```typescript
@@filename(photo.entity)
import { Entity, Column, PrimaryGeneratedColumn } from 'typeorm';

@Entity()
export class Photo {
  @PrimaryGeneratedColumn()
  id: number;

  @Column({ length: 500 })
  name: string;

  @Column('text')
  description: string;

  @Column()
  filename: string;

  @Column('int')
  views: number;

  @Column()
  isPublished: boolean;
}
```

`Photo` 实体位于 `photo` 目录中。该目录代表 `PhotoModule`。现在，我们来创建一个 **Repository** 提供者：

```typescript
@@filename(photo.providers)
import { DataSource } from 'typeorm';
import { Photo } from './photo.entity';

export const photoProviders = [
  {
    provide: 'PHOTO_REPOSITORY',
    useFactory: (dataSource: DataSource) => dataSource.getRepository(Photo),
    inject: ['DATA_SOURCE'],
  },
];
```

> warning **警告** 在实际应用中应避免使用 **魔法字符串**。`PHOTO_REPOSITORY` 和 `DATA_SOURCE` 都应保存在单独的 `constants.ts` 文件中。

现在我们可以在 `PhotoService` 中通过 `@Inject()` 装饰器注入 `Repository<Photo>`：

```typescript
@@filename(photo.service)
import { Injectable, Inject } from '@nestjs/common';
import { Repository } from 'typeorm';
import { Photo } from './photo.entity';

@Injectable()
export class PhotoService {
  constructor(
    @Inject('PHOTO_REPOSITORY')
    private photoRepository: Repository<Photo>,
  ) {}

  async findAll(): Promise<Photo[]> {
    return this.photoRepository.find();
  }
}
```

数据库连接是 **异步的**，但 Nest 使这个过程对最终用户完全透明。`PhotoRepository` 会等待数据库连接，而 `PhotoService` 会延迟到仓储准备就绪后才使用。当所有类都被实例化后，整个应用程序就可以启动了。

下面是完整的 `PhotoModule`：

```typescript
@@filename(photo.module)
import { Module } from '@nestjs/common';
import { DatabaseModule } from '../database/database.module';
import { photoProviders } from './photo.providers';
import { PhotoService } from './photo.service';

@Module({
  imports: [DatabaseModule],
  providers: [
    ...photoProviders,
    PhotoService,
  ],
})
export class PhotoModule {}
```

> info **提示** 不要忘记将 `PhotoModule` 导入到根模块 `AppModule` 中。