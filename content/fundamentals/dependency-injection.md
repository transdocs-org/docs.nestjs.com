### 自定义提供者

在之前的章节中，我们探讨了 **依赖注入 (DI)** 的各种方面，以及它在 Nest 中的使用方式。其中一个例子是 [基于构造函数](https://docs.nestjs.com/providers#dependency-injection) 的依赖注入，用于将实例（通常是服务提供者）注入到类中。了解到依赖注入是以一种基础方式内建在 Nest 核心中的，这并不令人意外。到目前为止，我们只探索了一种主要模式。随着你的应用程序变得越来越复杂，你可能需要利用 DI 系统的全部功能，因此让我们更详细地探讨这些功能。

#### DI 基础知识

依赖注入是一种 [控制反转 (IoC)](https://en.wikipedia.org/wiki/Inversion_of_control) 技术，在这种技术中，你将依赖项的实例化委托给 IoC 容器（在我们的案例中是 NestJS 运行时系统），而不是在代码中以命令式方式完成。让我们看一下 [Providers 章节](https://docs.nestjs.com/providers) 中的一个示例来理解这一点。

首先，我们定义了一个提供者。`@Injectable()` 装饰器将 `CatsService` 类标记为一个提供者。

```typescript
@@filename(cats.service)
import { Injectable } from '@nestjs/common';
import { Cat } from './interfaces/cat.interface';

@Injectable()
export class CatsService {
  private readonly cats: Cat[] = [];

  findAll(): Cat[] {
    return this.cats;
  }
}
@@switch
import { Injectable } from '@nestjs/common';

@Injectable()
export class CatsService {
  constructor() {
    this.cats = [];
  }

  findAll() {
    return this.cats;
  }
}
```

然后我们请求 Nest 将提供者注入到我们的控制器类中：

```typescript
@@filename(cats.controller)
import { Controller, Get } from '@nestjs/common';
import { CatsService } from './cats.service';
import { Cat } from './interfaces/cat.interface';

@Controller('cats')
export class CatsController {
  constructor(private catsService: CatsService) {}

  @Get()
  async findAll(): Promise<Cat[]> {
    return this.catsService.findAll();
  }
}
@@switch
import { Controller, Get, Bind, Dependencies } from '@nestjs/common';
import { CatsService } from './cats.service';

@Controller('cats')
@Dependencies(CatsService)
export class CatsController {
  constructor(catsService) {
    this.catsService = catsService;
  }

  @Get()
  async findAll() {
    return this.catsService.findAll();
  }
}
```

最后，我们将提供者注册到 Nest IoC 容器中：

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { CatsController } from './cats/cats.controller';
import { CatsService } from './cats/cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
export class AppModule {}
```

是什么机制在幕后使这一切正常工作？整个过程包含三个关键步骤：

1. 在 `cats.service.ts` 中，`@Injectable()` 装饰器声明 `CatsService` 类可以由 Nest IoC 容器管理。
2. 在 `cats.controller.ts` 中，`CatsController` 使用构造函数注入声明了对 `CatsService` token 的依赖：

```typescript
  constructor(private catsService: CatsService)
```

3. 在 `app.module.ts` 中，我们将 token `CatsService` 与 `cats.service.ts` 文件中的 `CatsService` 类关联起来。我们将在下面 <a href="/fundamentals/custom-providers#standard-providers">标准提供者</a> 章节中看到这种关联（也称为 _注册_）是如何发生的。

当 Nest IoC 容器实例化 `CatsController` 时，它首先会查找任何依赖项\*。当发现 `CatsService` 依赖项时，它会对 `CatsService` token 执行查找操作，根据注册步骤（如上第 3 步），该操作将返回 `CatsService` 类。假设作用域为 `SINGLETON`（默认行为），Nest 将会创建 `CatsService` 实例、缓存它并返回它，或者如果已经缓存了实例，则直接返回该现有实例。

\*为了说明问题，这个解释有点简化。我们忽略了一个重要的方面，即分析代码中依赖项的过程非常复杂，并且发生在应用程序启动期间。一个关键特性是依赖项分析（或“创建依赖图”）是**可传递的**。在上面的示例中，如果 `CatsService` 本身还有依赖项，那些依赖项也将被解析。依赖图确保依赖项以正确的顺序解析——基本上是“自底向上”。这种机制减轻了开发人员管理复杂依赖图的负担。

<app-banner-courses></app-banner-courses>

#### 标准提供者

让我们更仔细地看一下 `@Module()` 装饰器。在 `app.module` 中，我们声明了：

```typescript
@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
```

`providers` 属性接受一个 `providers` 数组。到目前为止，我们通过类名列表提供这些提供者。实际上，`providers: [CatsService]` 的语法是更完整语法的简写形式：

```typescript
providers: [
  {
    provide: CatsService,
    useClass: CatsService,
  },
];
```

现在我们看到了这种显式构造，我们可以理解注册过程。在这里，我们明确地将 token `CatsService` 与类 `CatsService` 关联起来。简写语法只是为最常见的用例提供了一种便利方式，其中 token 用于通过同名类请求实例。

#### 自定义提供者

当你的需求超出标准提供者所能提供的功能时会发生什么？以下是几个例子：

- 你希望创建一个自定义实例，而不是让 Nest 实例化（或返回缓存的）一个类
- 你希望在第二个依赖项中重用现有类
- 你希望在测试中用模拟版本替换一个类

Nest 允许你定义自定义提供者来处理这些情况。它提供了几种定义自定义提供者的方式。让我们逐一了解它们。

> info **提示** 如果你在依赖项解析方面遇到问题，可以设置 `NEST_DEBUG` 环境变量，并在启动期间获得额外的依赖项解析日志。

#### 值提供者：`useValue`

`useValue` 语法适用于注入常量值、将外部库放入 Nest 容器，或使用模拟对象替换真实实现。假设你想强制 Nest 在测试时使用一个模拟的 `CatsService`。

```typescript
import { CatsService } from './cats.service';

const mockCatsService = {
  /* 模拟实现
  ...
  */
};

@Module({
  imports: [CatsModule],
  providers: [
    {
      provide: CatsService,
      useValue: mockCatsService,
    },
  ],
})
export class AppModule {}
```

在此示例中，`CatsService` token 将解析为 `mockCatsService` 模拟对象。`useValue` 需要一个值 —— 在这种情况下，是一个具有与替换的 `CatsService` 类相同接口的字面对象。由于 TypeScript 的 [结构类型](https://www.typescriptlang.org/docs/handbook/type-compatibility.html)，你可以使用任何具有兼容接口的对象，包括字面对象或使用 `new` 实例化的类实例。

#### 非基于类的提供者 token

到目前为止，我们一直使用类名作为提供者的 token（在 `providers` 数组中列出的提供者中的 `provide` 属性的值）。这与 [构造函数注入](https://docs.nestjs.com/providers#dependency-injection) 标准模式相匹配，其中 token 也是类名。（如果这个概念还不清楚，请回到 <a href="/fundamentals/custom-providers#di-fundamentals">DI 基础知识</a> 章节复习 token 的相关内容）。有时，我们可能希望使用字符串或符号作为 DI token，以获得更大的灵活性。例如：

```typescript
import { connection } from './connection';

@Module({
  providers: [
    {
      provide: 'CONNECTION',
      useValue: connection,
    },
  ],
})
export class AppModule {}
```

在此示例中，我们将一个字符串 token（`'CONNECTION'`）与一个从外部文件导入的预定义 `connection` 对象关联起来。

> warning **注意** 除了使用字符串作为 token 值之外，你还可以使用 JavaScript [符号](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Symbol) 或 TypeScript [枚举](https://www.typescriptlang.org/docs/handbook/enums.html)。

我们之前已经了解了如何使用标准的 [构造函数注入](https://docs.nestjs.com/providers#dependency-injection) 模式注入提供者。此模式**要求**依赖项必须使用类名声明。而 `'CONNECTION'` 自定义提供者使用的是字符串 token。让我们看看如何注入这样的提供者。为此，我们使用 `@Inject()` 装饰器。这个装饰器接受一个参数 —— token。

```typescript
@@filename()
@Injectable()
export class CatsRepository {
  constructor(@Inject('CONNECTION') connection: Connection) {}
}
@@switch
@Injectable()
@Dependencies('CONNECTION')
export class CatsRepository {
  constructor(connection) {}
}
```

> info **提示** `@Inject()` 装饰器从 `@nestjs/common` 包中导入。

虽然在上面的示例中我们直接使用了字符串 `'CONNECTION'` 来说明问题，但为了代码组织的清晰性，最佳实践是在单独的文件（如 `constants.ts`）中定义 token。将它们视为符号或枚举，这些符号或枚举在单独的文件中定义并在需要时导入。

#### 类提供者：`useClass`

`useClass` 语法允许你动态确定一个 token 应该解析到哪个类。例如，假设我们有一个抽象（或默认）的 `ConfigService` 类。根据当前环境，我们希望 Nest 提供不同的配置服务实现。以下代码实现了这种策略。

```typescript
const configServiceProvider = {
  provide: ConfigService,
  useClass:
    process.env.NODE_ENV === 'development'
      ? DevelopmentConfigService
      : ProductionConfigService,
};

@Module({
  providers: [configServiceProvider],
})
export class AppModule {}
```

让我们看一下这段代码中的几个细节。你会注意到我们首先用一个字面对象定义了 `configServiceProvider`，然后将其传递给模块装饰器的 `providers` 属性。这只是代码组织的一种方式，但功能上与本章到目前为止使用的示例是等价的。

此外，我们使用了 `ConfigService` 类名作为我们的 token。对于任何依赖 `ConfigService` 的类，Nest 将注入提供的类（`DevelopmentConfigService` 或 `ProductionConfigService`）的实例，覆盖任何可能在其他地方声明的默认实现（例如，用 `@Injectable()` 装饰器声明的 `ConfigService`）。

#### 工厂提供者：`useFactory`

`useFactory` 语法允许动态创建提供者。实际的提供者将由工厂函数返回的值提供。工厂函数可以简单也可以复杂。一个简单的工厂可能不依赖任何其他提供者。一个更复杂的工厂可以自己注入它需要计算结果的其他提供者。对于后一种情况，工厂提供者语法有一对相关机制：

1. 工厂函数可以接受（可选的）参数。
2. （可选的）`inject` 属性接受一个提供者数组，Nest 会在实例化过程中解析这些提供者并作为参数传递给工厂函数。此外，这些提供者可以标记为可选。这两个列表应该相关联：Nest 会将 `inject` 列表中的实例按顺序作为参数传递给工厂函数。下面的示例演示了这一点。

```typescript
@@filename()
const connectionProvider = {
  provide: 'CONNECTION',
  useFactory: (optionsProvider: MyOptionsProvider, optionalProvider?: string) => {
    const options = optionsProvider.get();
    return new DatabaseConnection(options);
  },
  inject: [MyOptionsProvider, { token: 'SomeOptionalProvider', optional: true }],
  //       \______________/             \__________________/
  //        此提供者                    具有此 token 的提供者
  //        是必需的。                  可能解析为 `undefined`。
};

@Module({
  providers: [
    connectionProvider,
    MyOptionsProvider, // 基于类的提供者
    // { provide: 'SomeOptionalProvider', useValue: 'anything' },
  ],
})
export class AppModule {}
@@switch
const connectionProvider = {
  provide: 'CONNECTION',
  useFactory: (optionsProvider, optionalProvider) => {
    const options = optionsProvider.get();
    return new DatabaseConnection(options);
  },
  inject: [MyOptionsProvider, { token: 'SomeOptionalProvider', optional: true }],
  //       \______________/            \__________________/
  //        此提供者                   具有此 token 的提供者
  //        是必需的。                 可能解析为 `undefined`。
};

@Module({
  providers: [
    connectionProvider,
    MyOptionsProvider, // 基于类的提供者
    // { provide: 'SomeOptionalProvider', useValue: 'anything' },
  ],
})
export class AppModule {}
```

#### 别名提供者：`useExisting`

`useExisting` 语法允许你为现有提供者创建别名。这将创建两种访问同一提供者的方式。在下面的示例中，基于字符串的 token `'AliasedLoggerService'` 是基于类的 token `LoggerService` 的别名。假设我们有两个不同的依赖项，一个依赖 `'AliasedLoggerService'`，另一个依赖 `LoggerService`。如果这两个依赖项都指定了 `SINGLETON` 作用域，则它们都将解析为同一个实例。

```typescript
@Injectable()
class LoggerService {
  /* 实现细节 */
}

const loggerAliasProvider = {
  provide: 'AliasedLoggerService',
  useExisting: LoggerService,
};

@Module({
  providers: [LoggerService, loggerAliasProvider],
})
export class AppModule {}
```

#### 非服务型提供者

虽然提供者通常提供服务，但它们的用途并不限于此。提供者可以提供**任何**值。例如，提供者可以基于当前环境提供一个配置对象数组，如下所示：

```typescript
const configFactory = {
  provide: 'CONFIG',
  useFactory: () => {
    return process.env.NODE_ENV === 'development' ? devConfig : prodConfig;
  },
};

@Module({
  providers: [configFactory],
})
export class AppModule {}
```

#### 导出自定义提供者

像任何提供者一样，自定义提供者的作用域限定在其声明模块。要使其对其他模块可见，必须导出它。要导出自定义提供者，我们可以使用其 token 或完整的提供者对象。

以下示例展示使用 token 导出：

```typescript
@@filename()
const connectionFactory = {
  provide: 'CONNECTION',
  useFactory: (optionsProvider: OptionsProvider) => {
    const options = optionsProvider.get();
    return new DatabaseConnection(options);
  },
  inject: [OptionsProvider],
};

@Module({
  providers: [connectionFactory],
  exports: ['CONNECTION'],
})
export class AppModule {}
@@switch
const connectionFactory = {
  provide: 'CONNECTION',
  useFactory: (optionsProvider) => {
    const options = optionsProvider.get();
    return new DatabaseConnection(options);
  },
  inject: [OptionsProvider],
};

@Module({
  providers: [connectionFactory],
  exports: ['CONNECTION'],
})
export class AppModule {}
```

或者，使用完整的提供者对象导出：

```typescript
@@filename()
const connectionFactory = {
  provide: 'CONNECTION',
  useFactory: (optionsProvider: OptionsProvider) => {
    const options = optionsProvider.get();
    return new DatabaseConnection(options);
  },
  inject: [OptionsProvider],
};

@Module({
  providers: [connectionFactory],
  exports: [connectionFactory],
})
export class AppModule {}
@@switch
const connectionFactory = {
  provide: 'CONNECTION',
  useFactory: (optionsProvider) => {
    const options = optionsProvider.get();
    return new DatabaseConnection(options);
  },
  inject: [OptionsProvider],
};

@Module({
  providers: [connectionFactory],
  exports: [connectionFactory],
})
export class AppModule {}
```