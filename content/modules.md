### 模块

模块是一个使用 `@Module()` 装饰器标注的类。该装饰器提供了一些元数据，**Nest** 使用这些元数据来高效地组织和管理应用程序的结构。

<figure><img class="illustrative-image" src="/assets/Modules_1.png" /></figure>

每个 Nest 应用程序至少有一个模块，即 **根模块**，它作为 Nest 构建 **应用程序图谱（application graph）** 的起点。这个图谱是 Nest 内部用来解析模块和提供者之间关系和依赖的结构。虽然小型应用程序可能只包含一个根模块，但这通常不是最佳实践。我们 **强烈推荐** 使用模块来组织组件。对于大多数应用程序来说，你可能会有多个模块，每个模块封装一组紧密相关的 **功能（capabilities）**。

`@Module()` 装饰器接收一个对象参数，该对象包含以下属性，用于描述该模块：

|               |                                                                                                                                                                                                          |
| ------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `providers`   | 由 Nest 注入器实例化的提供者，这些提供者至少可以在当前模块中共享                                                                                                                                         |
| `controllers` | 当前模块中定义的一组控制器，这些控制器需要被实例化                                                                                                                                                       |
| `imports`     | 导入的模块列表，这些模块导出了当前模块所需的提供者                                                                                                                                                       |
| `exports`     | 当前模块提供的提供者子集，这些提供者应可供导入当前模块的其他模块使用。你可以使用提供者本身或其令牌（`provide` 值）                                                                                          |

默认情况下，模块 **封装** 提供者，这意味着你只能注入属于当前模块或从其他导入模块中显式导出的提供者。一个模块导出的提供者本质上就是该模块的公共接口或 API。

#### 功能模块（Feature modules）

在我们的示例中，`CatsController` 和 `CatsService` 是紧密相关的，并服务于相同的应用程序领域。将它们分组到一个功能模块中是合理的。功能模块将与特定功能相关的代码组织在一起，有助于保持清晰的边界和更好的组织结构。这在应用程序或团队规模扩大时尤其重要，并且符合 [SOLID](https://en.wikipedia.org/wiki/SOLID) 原则。

接下来，我们将创建 `CatsModule` 来演示如何将控制器和服务进行分组。

```typescript
@@filename(cats/cats.module)
import { Module } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
export class CatsModule {}
```

> info **提示** 要使用 CLI 创建模块，只需执行 `$ nest g module cats` 命令。

上面，我们在 `cats.module.ts` 文件中定义了 `CatsModule`，并将所有与此模块相关的文件移动到了 `cats` 目录中。最后一步是将该模块导入到根模块中（即在 `app.module.ts` 文件中定义的 `AppModule`）。

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { CatsModule } from './cats/cats.module';

@Module({
  imports: [CatsModule],
})
export class AppModule {}
```

现在我们的目录结构如下所示：

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
      <div class="item">cats.module.ts</div>
      <div class="item">cats.service.ts</div>
    </div>
    <div class="item">app.module.ts</div>
    <div class="item">main.ts</div>
  </div>
</div>

#### 共享模块（Shared modules）

在 Nest 中，模块默认是 **单例（singleton）**，因此你可以轻松地在多个模块之间共享任意提供者的相同实例。

<figure><img class="illustrative-image" src="/assets/Shared_Module_1.png" /></figure>

每个模块都自动是一个 **共享模块（shared module）**。一旦创建，它可以被任何模块复用。假设我们希望在多个模块之间共享 `CatsService` 的同一个实例。为此，我们需要将 `CatsService` 提供者添加到模块的 `exports` 数组中，如下所示：

```typescript
@@filename(cats.module)
import { Module } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
  exports: [CatsService]
})
export class CatsModule {}
```

现在，任何导入 `CatsModule` 的模块都可以访问 `CatsService`，并且它与其他导入该模块的模块共享同一个实例。

如果我们直接在每个需要 `CatsService` 的模块中注册它，虽然也能工作，但会导致每个模块都获得一个独立的 `CatsService` 实例。这会增加内存消耗，并可能导致意外行为，例如当服务维护内部状态时出现状态不一致的问题。

通过将 `CatsService` 封装在 `CatsModule` 中并导出它，我们确保了所有导入 `CatsModule` 的模块共享同一个 `CatsService` 实例。这不仅减少了内存消耗，还使行为更加可预测，因为所有模块共享同一个实例，从而更容易管理共享状态或资源。这是 NestJS 等框架中模块化和依赖注入的关键优势之一——允许服务在整个应用程序中高效共享。

<app-banner-devtools></app-banner-devtools>

#### 模块重新导出（Module re-exporting）

如上所述，模块可以导出其内部的提供者。此外，它们还可以重新导出它们导入的模块。在下面的示例中，`CommonModule` 被导入到 `CoreModule` 中并从其中导出，使其可供导入 `CoreModule` 的其他模块使用。

```typescript
@Module({
  imports: [CommonModule],
  exports: [CommonModule],
})
export class CoreModule {}
```

#### 依赖注入（Dependency injection）

模块类本身也可以 **注入** 提供者（例如用于配置目的）：

```typescript
@@filename(cats.module)
import { Module } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
export class CatsModule {
  constructor(private catsService: CatsService) {}
}
@@switch
import { Module, Dependencies } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
@Dependencies(CatsService)
export class CatsModule {
  constructor(catsService) {
    this.catsService = catsService;
  }
}
```

然而，由于 [循环依赖（circular dependency）](/fundamentals/circular-dependency) 的原因，模块类本身不能作为提供者被注入。

#### 全局模块（Global modules）

如果你必须在多个地方导入相同的模块集，这会变得很繁琐。与 Nest 不同，[Angular](https://angular.dev) 的 `providers` 是在全局范围内注册的。一旦定义，它们就可以在任何地方使用。而 Nest 则将提供者封装在模块作用域中。除非先导入封装该提供者的模块，否则你无法在其他地方使用该模块的提供者。

当你希望提供一组应该在任何地方都能直接使用的提供者（例如工具函数、数据库连接等）时，可以使用 `@Global()` 装饰器将模块标记为 **全局模块（global）**。

```typescript
import { Module, Global } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Global()
@Module({
  controllers: [CatsController],
  providers: [CatsService],
  exports: [CatsService],
})
export class CatsModule {}
```

`@Global()` 装饰器使模块成为全局作用域的模块。全局模块应该 **只注册一次**，通常由根模块或核心模块注册。在上面的示例中，`CatsService` 提供者将成为全局可用的，希望注入该服务的模块无需在 `imports` 数组中导入 `CatsModule`。

> info **提示** 不推荐将所有模块都设置为全局模块。虽然全局模块可以减少样板代码，但通常更好的做法是使用 `imports` 数组以可控和清晰的方式向其他模块暴露模块的 API。这种方法提供了更好的结构和可维护性，确保只有模块中必要的部分被共享，同时避免了不相关部分之间的不必要的耦合。

#### 动态模块（Dynamic modules）

Nest 中的动态模块允许你在运行时配置模块。当你需要提供一个灵活、可定制的模块，其提供者可以根据某些选项或配置创建时，动态模块特别有用。以下是 **动态模块** 的工作原理简介。

```typescript
@@filename()
import { Module, DynamicModule } from '@nestjs/common';
import { createDatabaseProviders } from './database.providers';
import { Connection } from './connection.provider';

@Module({
  providers: [Connection],
  exports: [Connection],
})
export class DatabaseModule {
  static forRoot(entities = [], options?): DynamicModule {
    const providers = createDatabaseProviders(options, entities);
    return {
      module: DatabaseModule,
      providers: providers,
      exports: providers,
    };
  }
}
@@switch
import { Module } from '@nestjs/common';
import { createDatabaseProviders } from './database.providers';
import { Connection } from './connection.provider';

@Module({
  providers: [Connection],
  exports: [Connection],
})
export class DatabaseModule {
  static forRoot(entities = [], options) {
    const providers = createDatabaseProviders(options, entities);
    return {
      module: DatabaseModule,
      providers: providers,
      exports: providers,
    };
  }
}
```

> info **提示** `forRoot()` 方法可以同步或异步（即通过 `Promise`）返回一个动态模块。

该模块默认在 `@Module()` 装饰器中定义了 `Connection` 提供者，但根据传递给 `forRoot()` 方法的 `entities` 和 `options` 对象，还可以导出一组提供者（例如仓库）。请注意，动态模块返回的属性 **扩展**（而不是覆盖）了在 `@Module()` 装饰器中定义的基础模块元数据。这就是为什么静态声明的 `Connection` 提供者和动态生成的仓库提供者都可以从模块中导出的原因。

如果你想将一个动态模块注册到全局作用域中，请将 `global` 属性设置为 `true`。

```typescript
{
  global: true,
  module: DatabaseModule,
  providers: providers,
  exports: providers,
}
```

> warning **警告** 如上所述，将所有模块都设置为全局模块 **不是一个好的设计决策**。

`DatabaseModule` 可以通过以下方式导入和配置：

```typescript
import { Module } from '@nestjs/common';
import { DatabaseModule } from './database/database.module';
import { User } from './users/entities/user.entity';

@Module({
  imports: [DatabaseModule.forRoot([User])],
})
export class AppModule {}
```

如果你想重新导出一个动态模块，可以在 `exports` 数组中省略 `forRoot()` 方法的调用：

```typescript
import { Module } from '@nestjs/common';
import { DatabaseModule } from './database/database.module';
import { User } from './users/entities/user.entity';

@Module({
  imports: [DatabaseModule.forRoot([User])],
  exports: [DatabaseModule],
})
export class AppModule {}
```

[动态模块](/fundamentals/dynamic-modules) 章节对该主题进行了更详细的介绍，并包含了一个 [工作示例](https://github.com/nestjs/nest/tree/master/sample/25-dynamic-modules)。

> info **提示** 在 [此章节](/fundamentals/dynamic-modules#configurable-module-builder) 中学习如何使用 `ConfigurableModuleBuilder` 构建高度可定制的动态模块。