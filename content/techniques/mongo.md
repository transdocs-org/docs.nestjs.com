### Mongo

Nest 支持两种与 [MongoDB](https://www.mongodb.com/) 数据库集成的方法。你可以使用内置的 [TypeORM](https://github.com/typeorm/typeorm) 模块（该模块提供了 MongoDB 的连接器），也可以使用 [Mongoose](https://mongoosejs.com)，这是最流行的 MongoDB 对象建模工具。在本章中，我们将介绍后者，并使用专门的 `@nestjs/mongoose` 包。

首先，安装 [所需的依赖项](https://github.com/Automattic/mongoose)：

```bash
$ npm i @nestjs/mongoose mongoose
```

安装完成后，我们可以将 `MongooseModule` 导入根模块 `AppModule`。

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';

@Module({
  imports: [MongooseModule.forRoot('mongodb://localhost/nest')],
})
export class AppModule {}
```

`forRoot()` 方法接受与 Mongoose 包中的 `mongoose.connect()` 相同的配置对象，具体描述请参见 [此处](https://mongoosejs.com/docs/connections.html)。

#### 模型注入

在 Mongoose 中，一切都源自 [Schema](http://mongoosejs.com/docs/guide.html)。每个 schema 映射到一个 MongoDB 集合，并定义该集合中文档的结构。schema 用于定义 [Model](https://mongoosejs.com/docs/models.html)。模型负责从底层 MongoDB 数据库创建和读取文档。

可以使用 NestJS 装饰器或手动使用 Mongoose 创建 schema。使用装饰器创建 schema 可以大大减少样板代码并提高整体代码的可读性。

我们来定义 `CatSchema`：

```typescript
@@filename(schemas/cat.schema)
import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';
import { HydratedDocument } from 'mongoose';

export type CatDocument = HydratedDocument<Cat>;

@Schema()
export class Cat {
  @Prop()
  name: string;

  @Prop()
  age: number;

  @Prop()
  breed: string;
}

export const CatSchema = SchemaFactory.createForClass(Cat);
```

> info **提示** 注意，你也可以使用 `DefinitionsFactory` 类（来自 `nestjs/mongoose`）生成原始的 schema 定义。这允许你手动修改根据元数据生成的 schema 定义。这在某些边缘情况下非常有用，因为可能很难用装饰器表示所有内容。

`@Schema()` 装饰器将一个类标记为 schema 定义。它将我们的 `Cat` 类映射到一个同名的 MongoDB 集合（但末尾会加上一个 "s"），因此最终的 MongoDB 集合名称将是 `cats`。此装饰器接受一个可选参数，即 schema 选项对象。可以把它想象成你通常传递给 `mongoose.Schema` 构造函数的第二个参数（例如，`new mongoose.Schema(_, options)`）。有关可用的 schema 选项，请参见 [本章](https://mongoosejs.com/docs/guide.html#options)。

`@Prop()` 装饰器定义文档中的属性。例如，在上面的 schema 定义中，我们定义了三个属性：`name`、`age` 和 `breed`。由于 TypeScript 元数据（和反射）功能，这些属性的 [schema 类型](https://mongoosejs.com/docs/schematypes.html) 会自动推断出来。然而，在更复杂的场景中，当类型无法隐式推断时（例如数组或嵌套对象结构），必须显式指定类型，如下所示：

```typescript
@Prop([String])
tags: string[];
```

或者，`@Prop()` 装饰器接受一个选项对象参数（有关可用选项的更多信息，请参见 [此处](https://mongoosejs.com/docs/schematypes.html#schematype-options)）。通过这种方式，你可以指定属性是否为必需项、指定默认值，或将属性标记为不可变。例如：

```typescript
@Prop({ required: true })
name: string;
```

如果你想指定与另一个模型的关系，以便后续进行填充（populate），也可以使用 `@Prop()` 装饰器。例如，如果 `Cat` 有一个 `Owner`，它存储在名为 `owners` 的不同集合中，则属性应具有类型和引用。例如：

```typescript
import * as mongoose from 'mongoose';
import { Owner } from '../owners/schemas/owner.schema';

// 在类定义内部
@Prop({ type: mongoose.Schema.Types.ObjectId, ref: 'Owner' })
owner: Owner;
```

如果有多个所有者，则属性配置应如下所示：

```typescript
@Prop({ type: [{ type: mongoose.Schema.Types.ObjectId, ref: 'Owner' }] })
owners: Owner[];
```

如果你不打算总是填充对另一个集合的引用，则可以考虑使用 `mongoose.Types.ObjectId` 作为类型：

```typescript
@Prop({ type: { type: mongoose.Schema.Types.ObjectId, ref: 'Owner' } })
// 这确保该字段不会与填充的引用混淆
owner: mongoose.Types.ObjectId;
```

然后，当你需要选择性地填充它时，可以使用指定正确类型的仓库函数：

```typescript
import { Owner } from './schemas/owner.schema';

// 例如在服务或仓库内部
async findAllPopulated() {
  return this.catModel.find().populate<{ owner: Owner }>("owner");
}
```

> info **提示** 如果没有要填充的外部文档，类型可以是 `Owner | null`，这取决于你的 [Mongoose 配置](https://mongoosejs.com/docs/populate.html#doc-not-found)。或者，它可能会抛出错误，此时类型将是 `Owner`。

最后，**原始** 的 schema 定义也可以传递给装饰器。这在例如属性表示一个未定义为类的嵌套对象时非常有用。为此，可以使用 `@nestjs/mongoose` 包中的 `raw()` 函数，如下所示：

```typescript
@Prop(raw({
  firstName: { type: String },
  lastName: { type: String }
}))
details: Record<string, any>;
```

或者，如果你更喜欢 **不使用装饰器**，可以手动定义 schema。例如：

```typescript
export const CatSchema = new mongoose.Schema({
  name: String,
  age: Number,
  breed: String,
});
```

`cat.schema` 文件位于 `cats` 目录中的一个文件夹中，我们也在该目录中定义了 `CatsModule`。虽然你可以将 schema 文件存储在任何你喜欢的位置，但我们建议将其保存在与其相关的 **领域对象** 附近，即在适当的模块目录中。

让我们看一下 `CatsModule`：

```typescript
@@filename(cats.module)
import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';
import { Cat, CatSchema } from './schemas/cat.schema';

@Module({
  imports: [MongooseModule.forFeature([{ name: Cat.name, schema: CatSchema }])],
  controllers: [CatsController],
  providers: [CatsService],
})
export class CatsModule {}
```

`MongooseModule` 提供了 `forFeature()` 方法来配置模块，包括定义应在当前作用域中注册的模型。如果你想在另一个模块中使用这些模型，请将 `MongooseModule` 添加到 `CatsModule` 的 `exports` 部分，并在其他模块中导入 `CatsModule`。

一旦你注册了 schema，就可以使用 `@InjectModel()` 装饰器将 `Cat` 模型注入到 `CatsService` 中：

```typescript
@@filename(cats.service)
import { Model } from 'mongoose';
import { Injectable } from '@nestjs/common';
import { InjectModel } from '@nestjs/mongoose';
import { Cat } from './schemas/cat.schema';
import { CreateCatDto } from './dto/create-cat.dto';

@Injectable()
export class CatsService {
  constructor(@InjectModel(Cat.name) private catModel: Model<Cat>) {}

  async create(createCatDto: CreateCatDto): Promise<Cat> {
    const createdCat = new this.catModel(createCatDto);
    return createdCat.save();
  }

  async findAll(): Promise<Cat[]> {
    return this.catModel.find().exec();
  }
}
@@switch
import { Model } from 'mongoose';
import { Injectable, Dependencies } from '@nestjs/common';
import { getModelToken } from '@nestjs/mongoose';
import { Cat } from './schemas/cat.schema';

@Injectable()
@Dependencies(getModelToken(Cat.name))
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

#### 连接

有时你可能需要访问原生的 [Mongoose Connection](https://mongoosejs.com/docs/api.html#Connection) 对象。例如，你可能希望在连接对象上进行原生 API 调用。你可以使用 `@InjectConnection()` 装饰器注入 Mongoose Connection，如下所示：

```typescript
import { Injectable } from '@nestjs/common';
import { InjectConnection } from '@nestjs/mongoose';
import { Connection } from 'mongoose';

@Injectable()
export class CatsService {
  constructor(@InjectConnection() private connection: Connection) {}
}
```

#### 会话

要使用 Mongoose 启动一个会话，建议使用 `@InjectConnection` 注入数据库连接，而不是直接调用 `mongoose.startSession()`。这种方法可以更好地与 NestJS 的依赖注入系统集成，确保连接管理的正确性。

以下是如何启动会话的示例：

```typescript
import { InjectConnection } from '@nestjs/mongoose';
import { Connection } from 'mongoose';

@Injectable()
export class CatsService {
  constructor(@InjectConnection() private readonly connection: Connection) {}

  async startTransaction() {
    const session = await this.connection.startSession();
    session.startTransaction();
    // 你的事务逻辑
  }
}
```

在此示例中，使用 `@InjectConnection()` 将 Mongoose 连接注入到服务中。一旦连接被注入，你可以使用 `connection.startSession()` 开始一个新会话。该会话可用于管理数据库事务，确保多个查询之间的原子操作。启动会话后，请记得根据你的逻辑提交或中止事务。

#### 多数据库

某些项目需要多个数据库连接。这个模块也支持这一点。要使用多个连接，首先需要创建这些连接。在这种情况下，连接命名是 **必须的**。

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';

@Module({
  imports: [
    MongooseModule.forRoot('mongodb://localhost/test', {
      connectionName: 'cats',
    }),
    MongooseModule.forRoot('mongodb://localhost/users', {
      connectionName: 'users',
    }),
  ],
})
export class AppModule {}
```

> warning **注意** 请注意，你不应该拥有未命名的多个连接，或者具有相同名称的连接，否则它们将被覆盖。

有了这个设置，你需要告诉 `MongooseModule.forFeature()` 函数应该使用哪个连接。

```typescript
@Module({
  imports: [
    MongooseModule.forFeature([{ name: Cat.name, schema: CatSchema }], 'cats'),
  ],
})
export class CatsModule {}
```

你也可以注入给定连接的 `Connection`：

```typescript
import { Injectable } from '@nestjs/common';
import { InjectConnection } from '@nestjs/mongoose';
import { Connection } from 'mongoose';

@Injectable()
export class CatsService {
  constructor(@InjectConnection('cats') private connection: Connection) {}
}
```

要将给定的 `Connection` 注入到自定义提供者（例如工厂提供者）中，可以使用 `getConnectionToken()` 函数，并将连接名称作为参数传入。

```typescript
{
  provide: CatsService,
  useFactory: (catsConnection: Connection) => {
    return new CatsService(catsConnection);
  },
  inject: [getConnectionToken('cats')],
}
```

如果你只是想从命名数据库中注入模型，可以在 `@InjectModel()` 装饰器中使用连接名称作为第二个参数。

```typescript
@@filename(cats.service)
@Injectable()
export class CatsService {
  constructor(@InjectModel(Cat.name, 'cats') private catModel: Model<Cat>) {}
}
@@switch
@Injectable()
@Dependencies(getModelToken(Cat.name, 'cats'))
export class CatsService {
  constructor(catModel) {
    this.catModel = catModel;
  }
}
```

#### 钩子（中间件）

中间件（也称为 pre 和 post 钩子）是在执行异步函数期间传递控制权的函数。中间件是在 schema 级别上指定的，对于编写插件非常有用（[来源](https://mongoosejs.com/docs/middleware.html)）。在编译模型后调用 `pre()` 或 `post()` 在 Mongoose 中不起作用。要在模型注册 **之前** 注册钩子，请使用 `MongooseModule` 的 `forFeatureAsync()` 方法结合工厂提供者（即 `useFactory`）。通过这种技术，你可以访问 schema 对象，然后使用 `pre()` 或 `post()` 方法在该 schema 上注册钩子。请参见下面的示例：

```typescript
@Module({
  imports: [
    MongooseModule.forFeatureAsync([
      {
        name: Cat.name,
        useFactory: () => {
          const schema = CatsSchema;
          schema.pre('save', function () {
            console.log('Hello from pre save');
          });
          return schema;
        },
      },
    ]),
  ],
})
export class AppModule {}
```

与其他 [工厂提供者](https://docs.nestjs.com/fundamentals/custom-providers#factory-providers-usefactory) 一样，我们的工厂函数可以是 `async` 的，并且可以通过 `inject` 注入依赖项。

```typescript
@Module({
  imports: [
    MongooseModule.forFeatureAsync([
      {
        name: Cat.name,
        imports: [ConfigModule],
        useFactory: (configService: ConfigService) => {
          const schema = CatsSchema;
          schema.pre('save', function() {
            console.log(
              `${configService.get('APP_NAME')}: Hello from pre save`,
            ),
          });
          return schema;
        },
        inject: [ConfigService],
      },
    ]),
  ],
})
export class AppModule {}
```

#### 插件

要为给定的 schema 注册一个 [插件](https://mongoosejs.com/docs/plugins.html)，请使用 `forFeatureAsync()` 方法。

```typescript
@Module({
  imports: [
    MongooseModule.forFeatureAsync([
      {
        name: Cat.name,
        useFactory: () => {
          const schema = CatsSchema;
          schema.plugin(require('mongoose-autopopulate'));
          return schema;
        },
      },
    ]),
  ],
})
export class AppModule {}
```

要为所有 schema 一次性注册插件，请调用 `Connection` 对象的 `.plugin()` 方法。你应在模型创建之前访问连接；为此，请使用 `connectionFactory`：

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';

@Module({
  imports: [
    MongooseModule.forRoot('mongodb://localhost/test', {
      connectionFactory: (connection) => {
        connection.plugin(require('mongoose-autopopulate'));
        return connection;
      }
    }),
  ],
})
export class AppModule {}
```

#### 鉴别器

[鉴别器](https://mongoosejs.com/docs/discriminators.html) 是一种 schema 继承机制。它们允许你在同一个底层 MongoDB 集合上拥有具有重叠 schema 的多个模型。

假设你想在一个集合中跟踪不同类型的事件。每个事件都将有一个时间戳。

```typescript
@@filename(event.schema)
@Schema({ discriminatorKey: 'kind' })
export class Event {
  @Prop({
    type: String,
    required: true,
    enum: [ClickedLinkEvent.name, SignUpEvent.name],
  })
  kind: string;

  @Prop({ type: Date, required: true })
  time: Date;
}

export const EventSchema = SchemaFactory.createForClass(Event);
```

> info **提示** Mongoose 通过“鉴别器键”来区分不同的鉴别器模型，默认情况下是 `__t`。Mongoose 会在你的 schema 中添加一个名为 `__t` 的 String 路径，用于跟踪该文档是哪个鉴别器的实例。
> 你也可以使用 `discriminatorKey` 选项来定义鉴别路径。

`SignedUpEvent` 和 `ClickedLinkEvent` 的实例将与通用事件一起存储在同一个集合中。

现在，我们来定义 `ClickedLinkEvent` 类，如下所示：

```typescript
@@filename(click-link-event.schema)
@Schema()
export class ClickedLinkEvent {
  kind: string;
  time: Date;

  @Prop({ type: String, required: true })
  url: string;
}

export const ClickedLinkEventSchema = SchemaFactory.createForClass(ClickedLinkEvent);
```

以及 `SignUpEvent` 类：

```typescript
@@filename(sign-up-event.schema)
@Schema()
export class SignUpEvent {
  kind: string;
  time: Date;

  @Prop({ type: String, required: true })
  user: string;
}

export const SignUpEventSchema = SchemaFactory.createForClass(SignUpEvent);
```

有了这些定义后，使用 `discriminators` 选项为给定的 schema 注册一个鉴别器。这在 `MongooseModule.forFeature` 和 `MongooseModule.forFeatureAsync` 中都适用：

```typescript
@@filename(event.module)
import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';

@Module({
  imports: [
    MongooseModule.forFeature([
      {
        name: Event.name,
        schema: EventSchema,
        discriminators: [
          { name: ClickedLinkEvent.name, schema: ClickedLinkEventSchema },
          { name: SignUpEvent.name, schema: SignUpEventSchema },
        ],
      },
    ]),
  ]
})
export class EventsModule {}
```

#### 测试

在对应用程序进行单元测试时，我们通常希望避免任何数据库连接，以使测试套件更易于设置且执行更快。但我们的类可能依赖于从连接实例中提取的模型。我们如何解决这些类？解决方案是创建模拟模型。

为了简化这一过程，`@nestjs/mongoose` 包暴露了一个 `getModelToken()` 函数，该函数返回一个基于 token 名称的准备好的 [注入 token](https://docs.nestjs.com/fundamentals/custom-providers#di-fundamentals)。使用这个 token，你可以轻松地使用任何标准的 [自定义提供者](/fundamentals/custom-providers) 技术（包括 `useClass`、`useValue` 和 `useFactory`）来提供一个模拟实现。例如：

```typescript
@Module({
  providers: [
    CatsService,
    {
      provide: getModelToken(Cat.name),
      useValue: catModel,
    },
  ],
})
export class CatsModule {}
```

在这个例子中，当任何消费者使用 `@InjectModel()` 装饰器注入 `Model<Cat>` 时，都会提供一个硬编码的 `catModel`（对象实例）。

<app-banner-courses></app-banner-courses>

#### 异步配置

当你需要传递模块选项而不是静态配置时，请使用 `forRootAsync()` 方法。与大多数动态模块一样，Nest 提供了多种处理异步配置的技术。

一种技术是使用工厂函数：

```typescript
MongooseModule.forRootAsync({
  useFactory: () => ({
    uri: 'mongodb://localhost/nest',
  }),
});
```

与其他 [工厂提供者](https://docs.nestjs.com/fundamentals/custom-providers#factory-providers-usefactory) 一样，我们的工厂函数可以是 `async` 的，并且可以通过 `inject` 注入依赖项。

```typescript
MongooseModule.forRootAsync({
  imports: [ConfigModule],
  useFactory: async (configService: ConfigService) => ({
    uri: configService.get<string>('MONGODB_URI'),
  }),
  inject: [ConfigService],
});
```

或者，你可以使用类而不是工厂来配置 `MongooseModule`，如下所示：

```typescript
MongooseModule.forRootAsync({
  useClass: MongooseConfigService,
});
```

上面的构造会在 `MongooseModule` 内部实例化 `MongooseConfigService`，并使用它来创建所需的选项对象。注意，在此示例中，`MongooseConfigService` 必须实现 `MongooseOptionsFactory` 接口，如下所示。`MongooseModule` 将调用所提供的类实例上的 `createMongooseOptions()` 方法。

```typescript
@Injectable()
export class MongooseConfigService implements MongooseOptionsFactory {
  createMongooseOptions(): MongooseModuleOptions {
    return {
      uri: 'mongodb://localhost/nest',
    };
  }
}
```

如果你想重用现有的选项提供者而不是在 `MongooseModule` 内部创建私有副本，请使用 `useExisting` 语法。

```typescript
MongooseModule.forRootAsync({
  imports: [ConfigModule],
  useExisting: ConfigService,
});
```

#### 连接事件

你可以使用 `onConnectionCreate` 配置选项来监听 Mongoose [连接事件](https://mongoosejs.com/docs/connections.html#connection-events)。这允许你在建立连接时实现自定义逻辑。例如，你可以为 `connected`、`open`、`disconnected`、`reconnected` 和 `disconnecting` 事件注册事件监听器，如下所示：

```typescript
MongooseModule.forRoot('mongodb://localhost/test', {
  onConnectionCreate: (connection: Connection) => {
    connection.on('connected', () => console.log('connected'));
    connection.on('open', () => console.log('open'));
    connection.on('disconnected', () => console.log('disconnected'));
    connection.on('reconnected', () => console.log('reconnected'));
    connection.on('disconnecting', () => console.log('disconnecting'));

    return connection;
  },
}),
```

在此代码片段中，我们正在连接到 `mongodb://localhost/test` 的 MongoDB 数据库。`onConnectionCreate` 选项允许你设置特定的事件监听器来监控连接状态：

- `connected`: 当连接成功建立时触发。
- `open`: 当连接完全打开并准备好进行操作时触发。
- `disconnected`: 当连接丢失时调用。
- `reconnected`: 当断开连接后重新建立连接时调用。
- `disconnecting`: 当连接正在关闭过程中时发生。

你也可以将 `onConnectionCreate` 属性集成到使用 `MongooseModule.forRootAsync()` 创建的异步配置中：

```typescript
MongooseModule.forRootAsync({
  useFactory: () => ({
    uri: 'mongodb://localhost/test',
    onConnectionCreate: (connection: Connection) => {
      // 在此处注册事件监听器
      return connection;
    },
  }),
}),
```

这提供了一种灵活的方式来管理连接事件，使你能够有效地处理连接状态的变化。

#### 子文档

要在父文档中嵌套子文档，你可以如下定义你的 schema：

```typescript
@@filename(name.schema)
@Schema()
export class Name {
  @Prop()
  firstName: string;

  @Prop()
  lastName: string;
}

export const NameSchema = SchemaFactory.createForClass(Name);
```

然后在父 schema 中引用子文档：

```typescript
@@filename(person.schema)
@Schema()
export class Person {
  @Prop(NameSchema)
  name: Name;
}

export const PersonSchema = SchemaFactory.createForClass(Person);

export type PersonDocumentOverride = {
  name: Types.Subdocument<Types.ObjectId> & Name;
};

export type PersonDocument = HydratedDocument<Person, PersonDocumentOverride>;
```

如果你想包含多个子文档，可以使用子文档数组。重要的是要相应地覆盖属性的类型：

```typescript
@@filename(name.schema)
@Schema()
export class Person {
  @Prop([NameSchema])
  name: Name[];
}

export const PersonSchema = SchemaFactory.createForClass(Person);

export type PersonDocumentOverride = {
  name: Types.DocumentArray<Name>;
};

export type PersonDocument = HydratedDocument<Person, PersonDocumentOverride>;
```

#### 虚拟属性

在 Mongoose 中，**虚拟属性** 是存在于文档中的属性，但不会持久化到 MongoDB。它不会存储在数据库中，而是在每次访问时动态计算。虚拟属性通常用于派生或计算值，例如合并字段（例如，通过连接 `firstName` 和 `lastName` 创建 `fullName` 属性），或者创建依赖于文档中现有数据的属性。

```ts
class Person {
  @Prop()
  firstName: string;

  @Prop()
  lastName: string;

  @Virtual({
    get: function (this: Person) {
      return `${this.firstName} ${this.lastName}`;
    },
  })
  fullName: string;
}
```

> info **提示** `@Virtual()` 装饰器来自 `@nestjs/mongoose` 包。

在此示例中，`fullName` 虚拟属性是从 `firstName` 和 `lastName` 派生而来的。尽管在访问时它的行为像一个普通属性，但它永远不会保存到 MongoDB 文档中。

#### 示例

一个可用的示例请参见 [此处](https://github.com/nestjs/nest/tree/master/sample/06-mongoose)。