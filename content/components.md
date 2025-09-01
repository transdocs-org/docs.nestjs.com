### 提供者（Providers）

提供者是 Nest 中的核心概念。许多基础的 Nest 类，如服务（services）、仓库（repositories）、工厂（factories）和辅助类（helpers），都可以被视作提供者。提供者的关键特性是它可以作为依赖项被**注入**，从而允许对象之间建立各种关系。这些对象之间的“绑定”工作主要由 Nest 运行时系统来处理。

<figure><img class="illustrative-image" src="/assets/Components_1.png" /></figure>

在上一章中，我们创建了一个简单的 `CatsController`。控制器应负责处理 HTTP 请求，并将更复杂的任务委托给**提供者**。提供者是使用 NestJS 模块中 `providers` 字段声明的普通 JavaScript 类。更多细节请参考“模块”章节。

> info **提示** 由于 Nest 允许你以面向对象的方式设计和组织依赖关系，我们强烈建议遵循 [SOLID 原则](https://zh.wikipedia.org/wiki/SOLID)。

#### 服务（Services）

让我们从创建一个简单的 `CatsService` 开始。该服务将处理数据的存储和检索，并被 `CatsController` 使用。由于它在管理应用程序逻辑中的作用，它是一个非常适合被定义为提供者的类。

```typescript
@@filename(cats.service)
import { Injectable } from '@nestjs/common';
import { Cat } from './interfaces/cat.interface';

@Injectable()
export class CatsService {
  private readonly cats: Cat[] = [];

  create(cat: Cat) {
    this.cats.push(cat);
  }

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

  create(cat) {
    this.cats.push(cat);
  }

  findAll() {
    return this.cats;
  }
}
```

> info **提示** 要使用 CLI 创建服务，只需执行 `$ nest g service cats` 命令。

我们的 `CatsService` 是一个具有一个属性和两个方法的简单类。这里的关键添加是 `@Injectable()` 装饰器。该装饰器为类附加元数据，表明 `CatsService` 是一个可以由 Nest [IoC](https://zh.wikipedia.org/wiki/%E6%8E%A7%E5%88%B6%E5%8F%8D%E8%BD%AC) 容器管理的类。

此外，本例中还使用了一个 `Cat` 接口，它可能如下所示：

```typescript
@@filename(interfaces/cat.interface)
export interface Cat {
  name: string;
  age: number;
  breed: string;
}
```

现在我们已经有了一个用于检索猫咪的服务类，让我们在 `CatsController` 中使用它：

```typescript
@@filename(cats.controller)
import { Controller, Get, Post, Body } from '@nestjs/common';
import { CreateCatDto } from './dto/create-cat.dto';
import { CatsService } from './cats.service';
import { Cat } from './interfaces/cat.interface';

@Controller('cats')
export class CatsController {
  constructor(private catsService: CatsService) {}

  @Post()
  async create(@Body() createCatDto: CreateCatDto) {
    this.catsService.create(createCatDto);
  }

  @Get()
  async findAll(): Promise<Cat[]> {
    return this.catsService.findAll();
  }
}
@@switch
import { Controller, Get, Post, Body, Bind, Dependencies } from '@nestjs/common';
import { CatsService } from './cats.service';

@Controller('cats')
@Dependencies(CatsService)
export class CatsController {
  constructor(catsService) {
    this.catsService = catsService;
  }

  @Post()
  @Bind(Body())
  async create(createCatDto) {
    this.catsService.create(createCatDto);
  }

  @Get()
  async findAll() {
    return this.catsService.findAll();
  }
}
```

`CatsService` 是通过类构造函数**注入**的。请注意使用了 `private` 关键字，这个简写语法允许我们在同一行中声明并初始化 `catsService` 成员，从而简化流程。

#### 依赖注入（Dependency injection）

Nest 建立在一个强大的设计模式——**依赖注入**之上。我们强烈建议阅读官方 [Angular 文档](https://angular.cn/guide/di) 中关于此概念的精彩文章。

在 Nest 中，得益于 TypeScript 的能力，管理依赖项非常简单，因为它们是根据类型解析的。在下面的例子中，Nest 将通过创建并返回 `CatsService` 的实例（或者在单例的情况下，如果已在其他地方请求过，则返回现有实例）来解析 `catsService`。然后，该依赖项会被注入到你的控制器构造函数中（或分配给指定的属性）：

```typescript
constructor(private catsService: CatsService) {}
```

#### 作用域（Scopes）

提供者通常具有与应用程序生命周期一致的生命周期（“作用域”）。当应用程序启动时，每个依赖项都必须被解析，这意味着每个提供者都会被实例化。同样，当应用程序关闭时，所有提供者都会被销毁。不过，也可以将提供者设置为**请求作用域**，这意味着它的生命周期与特定请求相关，而不是与应用程序生命周期相关。你可以在 [注入作用域](/zh-cn/fundamentals/injection-scopes) 章节中了解更多相关内容。

<app-banner-courses></app-banner-courses>

#### 自定义提供者（Custom providers）

Nest 自带一个内置的控制反转（“IoC”）容器，用于管理提供者之间的关系。这个功能是依赖注入的基础，但它的功能远比我们目前讨论的要强大得多。你可以通过多种方式定义一个提供者：可以使用普通值、类，以及异步或同步工厂。有关定义提供者的更多示例，请参阅 [依赖注入](/zh-cn/fundamentals/dependency-injection) 章节。

#### 可选提供者（Optional providers）

有时你可能会遇到一些不需要总是被解析的依赖项。例如，你的类可能依赖于一个**配置对象**，但如果未提供，则应使用默认值。在这种情况下，该依赖项被视为可选的，缺少该配置提供者不应导致错误。

要将一个提供者标记为可选，可以在构造函数签名中使用 `@Optional()` 装饰器。

```typescript
import { Injectable, Optional, Inject } from '@nestjs/common';

@Injectable()
export class HttpService<T> {
  constructor(@Optional() @Inject('HTTP_OPTIONS') private httpClient: T) {}
}
```

在上面的例子中，我们使用了一个自定义提供者，这就是为什么我们使用了 `HTTP_OPTIONS` 自定义**令牌**。前面的例子演示了基于构造函数的注入方式，其中依赖项是通过构造函数中的类来表示的。有关自定义提供者及其关联令牌的更多信息，请参阅 [自定义提供者](/zh-cn/fundamentals/custom-providers) 章节。

#### 基于属性的注入（Property-based injection）

到目前为止我们使用的注入技术称为基于构造函数的注入，即通过构造函数方法注入提供者。在某些特定情况下，**基于属性的注入**可能更有用。例如，如果你的顶层类依赖于一个或多个提供者，那么在子类中通过 `super()` 传递所有依赖可能会变得繁琐。为了避免这种情况，你可以在属性级别直接使用 `@Inject()` 装饰器。

```typescript
import { Injectable, Inject } from '@nestjs/common';

@Injectable()
export class HttpService<T> {
  @Inject('HTTP_OPTIONS')
  private readonly httpClient: T;
}
```

> warning **警告** 如果你的类不继承其他类，通常最好使用**基于构造函数的注入**。构造函数能清楚地说明需要哪些依赖项，提供更好的可见性，并使代码更容易理解，相比之下，使用 `@Inject` 注解的类属性则不具备这些优势。

#### 提供者注册（Provider registration）

现在我们已经定义了一个提供者（`CatsService`）和一个消费者（`CatsController`），我们需要将服务注册到 Nest 中，以便它能够处理注入。这可以通过编辑模块文件（`app.module.ts`）并在 `@Module()` 装饰器的 `providers` 数组中添加该服务来实现。

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

现在 Nest 将能够解析 `CatsController` 类的依赖项。

此时，我们的目录结构应如下所示：

<div class="file-tree">
<div class="item">src</div>
<div class="children">
<div class="item">cats</div>
<div class="children">
<div class="item">dto</div>
<div class="children">
<div class="item">create-cat.dto.ts</div>
</div>
<div class="item">interfaces</div>
<div class="children">
<div class="item">cat.interface.ts</div>
</div>
<div class="item">cats.controller.ts</div>
<div class="item">cats.service.ts</div>
</div>
<div class="item">app.module.ts</div>
<div class="item">main.ts</div>
</div>
</div>

#### 手动实例化（Manual instantiation）

到目前为止，我们已经介绍了 Nest 如何自动处理大多数依赖项解析的细节。然而，在某些情况下，你可能需要绕过内置的依赖注入系统，手动检索或实例化提供者。以下简要介绍两种这样的技术：

- 要检索现有实例或动态实例化提供者，可以使用 [模块引用](https://docs.nestjs.cn/fundamentals/module-ref)。
- 要在 `bootstrap()` 函数中获取提供者（例如用于独立应用程序或在引导期间使用配置服务），请参阅 [独立应用程序](https://docs.nestjs.cn/standalone-applications)。