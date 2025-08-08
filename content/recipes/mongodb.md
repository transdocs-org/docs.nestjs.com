### MongoDB (Mongoose)

> **警告** 在本文中，你将学习如何使用自定义组件从头开始基于 **Mongoose** 包创建一个 `DatabaseModule`。因此，本解决方案包含大量你可以通过使用现成的 `@nestjs/mongoose` 包来省略的冗余代码。了解更多，请参见[这里](/techniques/mongodb)。

[Mongoose](https://mongoosejs.com) 是最受欢迎的 [MongoDB](https://www.mongodb.org/) 对象建模工具。

#### 入门

要开始使用这个库，我们首先需要安装所有必需的依赖项：

```typescript
$ npm install --save mongoose
```

第一步是使用 `connect()` 函数与数据库建立连接。`connect()` 函数返回一个 `Promise`，因此我们需要创建一个 [异步提供者](/fundamentals/async-components)。

```typescript
@@filename(database.providers)
import * as mongoose from 'mongoose';

export const databaseProviders = [
  {
    provide: 'DATABASE_CONNECTION',
    useFactory: (): Promise<typeof mongoose> =>
      mongoose.connect('mongodb://localhost/nest'),
  },
];
@@switch
import * as mongoose from 'mongoose';

export const databaseProviders = [
  {
    provide: 'DATABASE_CONNECTION',
    useFactory: () => mongoose.connect('mongodb://localhost/nest'),
  },
];
```

> info **提示** 遵循最佳实践，我们在一个带有 `*.providers.ts` 后缀的单独文件中声明了自定义提供者。

然后，我们需要导出这些提供者，以使它们对应用程序的其余部分**可用**。

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

现在我们可以使用 `@Inject()` 装饰器注入 `Connection` 对象。每个依赖于 `Connection` 异步提供者的类都将等待 `Promise` 被解析。

#### 模型注入

在 Mongoose 中，所有内容都源自 [Schema](https://mongoosejs.com/docs/guide.html)。让我们定义 `CatSchema`：

```typescript
@@filename(schemas/cat.schema)
import * as mongoose from 'mongoose';

export const CatSchema = new mongoose.Schema({
  name: String,
  age: Number,
  breed: String,
});
```

`CatsSchema` 属于 `cats` 目录。该目录代表 `CatsModule`。

现在是时候创建一个 **Model** 提供者了：

```typescript
@@filename(cats.providers)
import { Connection } from 'mongoose';
import { CatSchema } from './schemas/cat.schema';

export const catsProviders = [
  {
    provide: 'CAT_MODEL',
    useFactory: (connection: Connection) => connection.model('Cat', CatSchema),
    inject: ['DATABASE_CONNECTION'],
  },
];
@@switch
import { CatSchema } from './schemas/cat.schema';

export const catsProviders = [
  {
    provide: 'CAT_MODEL',
    useFactory: (connection) => connection.model('Cat', CatSchema),
    inject: ['DATABASE_CONNECTION'],
  },
];
```

> warning **警告** 在实际应用中，应避免使用**魔术字符串**。`CAT_MODEL` 和 `DATABASE_CONNECTION` 都应该保存在单独的 `constants.ts` 文件中。

现在我们可以使用 `@Inject()` 装饰器将 `CAT_MODEL` 注入到 `CatsService` 中：

```typescript
@@filename(cats.service)
import { Model } from 'mongoose';
import { Injectable, Inject } from '@nestjs/common';
import { Cat } from './interfaces/cat.interface';
import { CreateCatDto } from './dto/create-cat.dto';

@Injectable()
export class CatsService {
  constructor(
    @Inject('CAT_MODEL')
    private catModel: Model<Cat>,
  ) {}

  async create(createCatDto: CreateCatDto): Promise<Cat> {
    const createdCat = new this.catModel(createCatDto);
    return createdCat.save();
  }

  async findAll(): Promise<Cat[]> {
    return this.catModel.find().exec();
  }
}
@@switch
import { Injectable, Dependencies } from '@nestjs/common';

@Injectable()
@Dependencies('CAT_MODEL')
export class CatsService {
  constructor(catModel) {
    this.catModel = catModel;
  }

  async create(createCatDto) {
    const createdCat = new this.catModel(createCatDto);
    return createdCat.save();
  }

  async findAll() {
    return this.catModel.find().exec();
  }
}
```

在上面的示例中我们使用了 `Cat` 接口。该接口扩展了 mongoose 包中的 `Document`：

```typescript
import { Document } from 'mongoose';

export interface Cat extends Document {
  readonly name: string;
  readonly age: number;
  readonly breed: string;
}
```

数据库连接是**异步**的，但 Nest 使这个过程对最终用户完全透明。`CatModel` 类在等待数据库连接，而 `CatsService` 则延迟到模型准备就绪后才可用。当每个类都被实例化后，整个应用程序才能启动。

以下是最终的 `CatsModule`：

```typescript
@@filename(cats.module)
import { Module } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';
import { catsProviders } from './cats.providers';
import { DatabaseModule } from '../database/database.module';

@Module({
  imports: [DatabaseModule],
  controllers: [CatsController],
  providers: [
    CatsService,
    ...catsProviders,
  ],
})
export class CatsModule {}
```

> info **提示** 不要忘记将 `CatsModule` 导入到根模块 `AppModule` 中。

#### 示例

一个完整的工作示例请参见[这里](https://github.com/nestjs/nest/tree/master/sample/14-mongoose-base)。