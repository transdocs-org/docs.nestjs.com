### 动态模块

[模块章节](/modules) 介绍了 Nest 模块的基础知识，并包含对 [动态模块](https://docs.nestjs.com/modules#dynamic-modules) 的简要介绍。本章节将深入探讨动态模块的相关内容。完成本章后，你应该能够很好地理解它们是什么，以及何时和如何使用它们。

#### 简介

文档的 **概览** 部分中的大多数应用程序代码示例都使用了常规的、或称静态的模块。模块定义了一组组件，如 [提供者](/providers) 和 [控制器](/controllers)，这些组件共同构成了应用程序的模块化部分。它们为这些组件提供了一个执行上下文或作用域。例如，在模块中定义的提供者对于模块的其他成员是可见的，而无需导出它们。当某个提供者需要在模块外部可见时，首先需要从其宿主模块中导出，然后在消费模块中导入。

让我们通过一个熟悉的例子来理解这个过程。

首先，我们定义一个 `UsersModule` 来提供和导出一个 `UsersService`。`UsersModule` 是 `UsersService` 的**宿主**模块。

```typescript
import { Module } from '@nestjs/common';
import { UsersService } from './users.service';

@Module({
  providers: [UsersService],
  exports: [UsersService],
})
export class UsersModule {}
```

接下来，我们定义一个 `AuthModule`，它导入 `UsersModule`，使 `UsersModule` 中导出的提供者在 `AuthModule` 内部可用：

```typescript
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { UsersModule } from '../users/users.module';

@Module({
  imports: [UsersModule],
  providers: [AuthService],
  exports: [AuthService],
})
export class AuthModule {}
```

这些结构允许我们在例如 `AuthModule` 中托管的 `AuthService` 中注入 `UsersService`：

```typescript
import { Injectable } from '@nestjs/common';
import { UsersService } from '../users/users.service';

@Injectable()
export class AuthService {
  constructor(private usersService: UsersService) {}
  /*
    使用 this.usersService 的实现
  */
}
```

我们将这种模块绑定方式称为 **静态** 模块绑定。所有 Nest 需要用来连接模块的信息已经在宿主模块和消费模块中声明。让我们拆解一下在这个过程中发生了什么。Nest 通过以下方式使 `UsersService` 在 `AuthModule` 中可用：

1. 实例化 `UsersModule`，包括递归地导入 `UsersModule` 所消费的其他模块，并递归地解析任何依赖项（参见 [自定义提供者](https://docs.nestjs.com/fundamentals/custom-providers)）。
2. 实例化 `AuthModule`，并将 `UsersModule` 中导出的提供者提供给 `AuthModule` 中的组件（就像它们是在 `AuthModule` 中声明的一样）。
3. 在 `AuthService` 中注入一个 `UsersService` 的实例。

#### 动态模块的使用场景

在静态模块绑定中，消费模块**无法**影响宿主模块中提供者的配置方式。为什么这很重要？考虑这样一个情况：我们有一个通用模块，需要在不同的使用场景下表现不同。这类似于许多系统中的“插件”概念，即一个通用功能在被消费者使用之前需要一些配置。

在 Nest 中的一个很好的例子是 **配置模块**。许多应用程序发现通过使用配置模块来外部化配置细节是有用的。这使得在不同部署中动态更改应用程序设置变得容易：例如，开发人员使用的开发数据库、用于测试/预发布环境的预发布数据库等。通过将配置参数的管理委托给配置模块，应用程序的源代码可以保持与配置参数的独立。

挑战在于，配置模块本身由于是通用的（类似于“插件”），需要由其消费模块进行自定义。这就是 _动态模块_ 发挥作用的地方。利用动态模块的功能，我们可以使我们的配置模块 **动态化**，以便消费模块可以使用一个 API 来控制该模块在导入时如何被自定义。

换句话说，动态模块提供了一个 API，用于将一个模块导入到另一个模块中，并在导入时自定义该模块的属性和行为，而不是使用我们到目前为止看到的静态绑定。

<app-banner-devtools></app-banner-devtools>

#### 配置模块示例

我们将使用 [配置章节](https://docs.nestjs.com/techniques/configuration#service) 中示例代码的基本版本来演示本节内容。本章结束时的完整版本可以在 [这里](https://github.com/nestjs/nest/tree/master/sample/25-dynamic-modules) 找到。

我们的需求是让 `ConfigModule` 接受一个 `options` 对象以进行自定义。这是我们想要支持的功能。基本示例硬编码了 `.env` 文件的位置为项目根目录。假设我们想让这个位置可配置，这样你就可以将你的 `.env` 文件管理在任意选择的文件夹中。例如，假设你想将各种 `.env` 文件存储在项目根目录下的一个名为 `config` 的文件夹中（即与 `src` 同级的文件夹）。你希望在不同项目中使用 `ConfigModule` 时能够选择不同的文件夹。

动态模块使我们能够将参数传递到被导入的模块中，从而改变其行为。让我们看看它是如何工作的。从消费模块的视角出发，先从最终目标的写法入手，然后逐步回溯。首先，我们快速回顾一下 _静态_ 导入 `ConfigModule` 的示例（即一种无法影响导入模块行为的方法）。请注意 `@Module()` 装饰器中的 `imports` 数组：

```typescript
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { ConfigModule } from './config/config.module';

@Module({
  imports: [ConfigModule],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

现在让我们考虑一个传递配置对象的 _动态模块_ 导入会是什么样子。比较这两个示例中 `imports` 数组的差异：

```typescript
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { ConfigModule } from './config/config.module';

@Module({
  imports: [ConfigModule.register({ folder: './config' })],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

让我们看看上面动态示例中发生了什么。有哪些关键部分？

1. `ConfigModule` 是一个普通类，因此我们可以推断它必须有一个名为 `register()` 的 **静态方法**。我们知道它是静态的，因为我们是在 `ConfigModule` 类上调用它，而不是在类的实例上调用。注意：这个方法，我们将很快创建，可以有任意名称，但按照惯例，我们应该将其命名为 `forRoot()` 或 `register()`。
2. `register()` 方法由我们定义，因此我们可以接受任何我们喜欢的输入参数。在这种情况下，我们将接受一个带有合适属性的简单 `options` 对象，这是常见的情况。
3. 我们可以推断 `register()` 方法必须返回某种类似于 `module` 的东西，因为它的返回值出现在熟悉的 `imports` 列表中，我们到目前为止看到的是模块的列表。

实际上，我们的 `register()` 方法将返回一个 `DynamicModule`。动态模块只不过是一个在运行时创建的模块，它具有与静态模块完全相同的属性，外加一个名为 `module` 的附加属性。我们快速回顾一下一个静态模块声明的示例，特别注意装饰器中传入的模块选项：

```typescript
@Module({
  imports: [DogsModule],
  controllers: [CatsController],
  providers: [CatsService],
  exports: [CatsService]
})
```

动态模块必须返回一个具有完全相同接口的对象，外加一个名为 `module` 的附加属性。`module` 属性用作模块的名称，应与模块类的名称相同，如下例所示。

> info **提示** 对于动态模块，模块选项对象的所有属性都是可选的，**除了** `module`。

那静态的 `register()` 方法呢？我们现在可以看到，它的职责是返回一个具有 `DynamicModule` 接口的对象。当我们调用它时，我们实际上是在向 `imports` 列表提供一个模块，类似于我们通过列出模块类名在静态情况下所做的那样。换句话说，动态模块 API 只是返回一个模块，但与在 `@Module` 装饰器中固定属性不同，我们以编程方式指定它们。

还有一些细节需要补充，以帮助我们完整地理解：

1. 我们现在可以声明，`@Module()` 装饰器的 `imports` 属性不仅可以接受模块类名（例如 `imports: [UsersModule]`），还可以接受一个 **返回** 动态模块的函数（例如 `imports: [ConfigModule.register(...)]`）。
2. 动态模块本身也可以导入其他模块。我们在这个例子中不会这样做，但如果动态模块依赖于其他模块中的提供者，你可以使用可选的 `imports` 属性导入它们。再次说明，这与你使用 `@Module()` 装饰器声明静态模块元数据的方式完全相同。

有了这些理解，我们现在可以看看我们的动态 `ConfigModule` 声明应该是什么样子。让我们尝试一下。

```typescript
import { DynamicModule, Module } from '@nestjs/common';
import { ConfigService } from './config.service';

@Module({})
export class ConfigModule {
  static register(): DynamicModule {
    return {
      module: ConfigModule,
      providers: [ConfigService],
      exports: [ConfigService],
    };
  }
}
```

现在应该清楚这些部分是如何关联在一起的了。调用 `ConfigModule.register(...)` 返回一个 `DynamicModule` 对象，其属性本质上与我们到目前为止通过 `@Module()` 装饰器提供的元数据相同。

> info **提示** 从 `@nestjs/common` 导入 `DynamicModule`。

然而，我们的动态模块还不是很有趣，因为我们还没有引入任何可以**配置**它的能力，正如我们之前所说的那样。接下来我们将解决这个问题。

#### 模块配置

定制 `ConfigModule` 行为的显而易见的解决方案是在静态 `register()` 方法中传入一个 `options` 对象，正如我们之前猜测的那样。让我们再次看一下消费模块的 `imports` 属性：

```typescript
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { ConfigModule } from './config/config.module';

@Module({
  imports: [ConfigModule.register({ folder: './config' })],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

这很好地处理了向动态模块传递一个 `options` 对象。那我们如何在 `ConfigModule` 中使用这个 `options` 对象呢？让我们花一分钟考虑一下。我们知道我们的 `ConfigModule` 基本上是一个宿主模块，用于提供和导出一个可注入的服务 `ConfigService`，供其他提供者使用。实际上，是我们的 `ConfigService` 需要读取 `options` 对象来定制其行为。我们假设现在知道如何将 `options` 从 `register()` 方法传递到 `ConfigService` 中。有了这个假设，我们可以对服务进行一些更改，以根据 `options` 对象的属性来定制其行为。（**注意**：目前，由于我们尚未实际确定如何传递它，我们将先对 `options` 进行硬编码。我们稍后会修复这个问题）。

```typescript
import { Injectable } from '@nestjs/common';
import * as dotenv from 'dotenv';
import * as fs from 'fs';
import * as path from 'path';
import { EnvConfig } from './interfaces';

@Injectable()
export class ConfigService {
  private readonly envConfig: EnvConfig;

  constructor() {
    const options = { folder: './config' };

    const filePath = `${process.env.NODE_ENV || 'development'}.env`;
    const envFile = path.resolve(__dirname, '../../', options.folder, filePath);
    this.envConfig = dotenv.parse(fs.readFileSync(envFile));
  }

  get(key: string): string {
    return this.envConfig[key];
  }
}
```

现在我们的 `ConfigService` 知道如何在 `options` 指定的文件夹中找到 `.env` 文件。

我们剩下的任务是将 `register()` 步骤中的 `options` 对象注入到我们的 `ConfigService` 中。当然，我们将使用 _依赖注入_ 来完成这项工作。这是一个关键点，请确保你理解它。我们的 `ConfigModule` 提供了 `ConfigService`。`ConfigService` 反过来依赖于仅在运行时提供的 `options` 对象。因此，在运行时，我们需要首先将 `options` 对象绑定到 Nest IoC 容器，然后让 Nest 将其注入到我们的 `ConfigService` 中。记住，在 **自定义提供者** 章节中提到过，提供者可以 [包含任何值](https://docs.nestjs.com/fundamentals/custom-providers#non-service-based-providers)，不仅仅是服务，因此我们可以使用依赖注入来处理一个简单的 `options` 对象。

让我们先处理将 `options` 对象绑定到 IoC 容器。我们在静态 `register()` 方法中完成此操作。记住，我们正在动态构造一个模块，而模块的属性之一是其提供者列表。因此，我们需要做的是将我们的 `options` 对象定义为一个提供者。这将使其可以注入到 `ConfigService` 中，我们将在下一步中利用这一点。在下面的代码中，请注意 `providers` 数组：

```typescript
import { DynamicModule, Module } from '@nestjs/common';
import { ConfigService } from './config.service';

@Module({})
export class ConfigModule {
  static register(options: Record<string, any>): DynamicModule {
    return {
      module: ConfigModule,
      providers: [
        {
          provide: 'CONFIG_OPTIONS',
          useValue: options,
        },
        ConfigService,
      ],
      exports: [ConfigService],
    };
  }
}
```

现在我们可以通过将 `'CONFIG_OPTIONS'` 提供者注入到 `ConfigService` 中来完成整个过程。回想一下，当我们使用非类令牌定义提供者时，需要使用 `@Inject()` 装饰器 [如这里所述](https://docs.nestjs.com/fundamentals/custom-providers#non-class-based-provider-tokens)。

```typescript
import * as dotenv from 'dotenv';
import * as fs from 'fs';
import * as path from 'path';
import { Injectable, Inject } from '@nestjs/common';
import { EnvConfig } from './interfaces';

@Injectable()
export class ConfigService {
  private readonly envConfig: EnvConfig;

  constructor(@Inject('CONFIG_OPTIONS') private options: Record<string, any>) {
    const filePath = `${process.env.NODE_ENV || 'development'}.env`;
    const envFile = path.resolve(__dirname, '../../', options.folder, filePath);
    this.envConfig = dotenv.parse(fs.readFileSync(envFile));
  }

  get(key: string): string {
    return this.envConfig[key];
  }
}
```

最后一点说明：为了简单起见，我们在上面使用了基于字符串的注入令牌（`'CONFIG_OPTIONS'`），但最佳实践是将其定义为常量（或 `Symbol`）并单独保存在一个文件中，然后导入该文件。例如：

```typescript
export const CONFIG_OPTIONS = 'CONFIG_OPTIONS';
```

#### 示例

本章代码的完整示例可以 [在这里](https://github.com/nestjs/nest/tree/master/sample/25-dynamic-modules) 找到。

#### 社区指南

你可能在一些 `@nestjs/` 包中看到过 `forRoot`、`register` 和 `forFeature` 等方法的使用，并想知道这些方法之间的区别。虽然没有硬性规定，但 `@nestjs/` 包尽量遵循以下指南：

在创建模块时：

- 使用 `register`，你期望用一个特定的配置来配置一个动态模块，仅由调用模块使用。例如，在 Nest 的 `@nestjs/axios` 中：`HttpModule.register({{ '{' }} baseUrl: 'someUrl' {{ '}' }})`。如果在另一个模块中使用 `HttpModule.register({{ '{' }} baseUrl: 'somewhere else' {{ '}' }})`，它将具有不同的配置。你可以根据需要在任意多个模块中这样做。

- 使用 `forRoot`，你期望一次配置一个动态模块，并在多个地方重用该配置（尽管可能是抽象掉的）。这就是为什么你有一个 `GraphQLModule.forRoot()`，一个 `TypeOrmModule.forRoot()` 等等。

- 使用 `forFeature`，你期望使用动态模块的 `forRoot` 配置，但需要根据调用模块的需求修改一些特定配置（例如，该模块应该访问哪些仓库，或者日志记录器应该使用的上下文）。

所有这些方法通常都有其 `async` 对应版本，如 `registerAsync`、`forRootAsync` 和 `forFeatureAsync`，它们的含义相同，但同时也使用 Nest 的依赖注入来进行配置。

#### 可配置模块构建器

手动创建高度可配置的动态模块（尤其是那些暴露 `async` 方法（如 `registerAsync`、`forRootAsync` 等）的模块）非常复杂，尤其是对于新手来说。为此，Nest 提供了 `ConfigurableModuleBuilder` 类，它简化了这一过程，让你只需几行代码就能构建一个模块“蓝图”。

例如，让我们使用上面的示例（`ConfigModule`）并将其转换为使用 `ConfigurableModuleBuilder`。在开始之前，请确保我们创建一个专用接口来表示我们的 `ConfigModule` 所需的选项。

```typescript
export interface ConfigModuleOptions {
  folder: string;
}
```

在此基础上，创建一个新文件（与现有的 `config.module.ts` 文件一起），并将其命名为 `config.module-definition.ts`。在这个文件中，我们将使用 `ConfigurableModuleBuilder` 来构建 `ConfigModule` 的定义。

```typescript
@@filename(config.module-definition)
import { ConfigurableModuleBuilder } from '@nestjs/common';
import { ConfigModuleOptions } from './interfaces/config-module-options.interface';

export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN } =
  new ConfigurableModuleBuilder<ConfigModuleOptions>().build();
@@switch
import { ConfigurableModuleBuilder } from '@nestjs/common';

export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN } =
  new ConfigurableModuleBuilder().build();
```

现在让我们打开 `config.module.ts` 文件，并修改其实现以利用自动生成的 `ConfigurableModuleClass`：

```typescript
import { Module } from '@nestjs/common';
import { ConfigService } from './config.service';
import { ConfigurableModuleClass } from './config.module-definition';

@Module({
  providers: [ConfigService],
  exports: [ConfigService],
})
export class ConfigModule extends ConfigurableModuleClass {}
```

继承 `ConfigurableModuleClass` 意味着 `ConfigModule` 现在不仅提供 `register` 方法（如之前自定义实现那样），还提供 `registerAsync` 方法，这允许消费者异步配置该模块，例如，通过提供异步工厂：

```typescript
@Module({
  imports: [
    ConfigModule.register({ folder: './config' }),
    // 或者替代方式：
    // ConfigModule.registerAsync({
    //   useFactory: () => {
    //     return {
    //       folder: './config',
    //     }
    //   },
    //   inject: [...任何额外依赖...]
    // }),
  ],
})
export class AppModule {}
```

`registerAsync` 方法接受以下对象作为参数：

```typescript
{
  /**
   * 解析为将被实例化为提供者的类的注入令牌。
   * 该类必须实现相应的接口。
   */
  useClass?: Type<
    ConfigurableModuleOptionsFactory<ModuleOptions, FactoryClassMethodKey>
  >;
  /**
   * 返回配置（或解析为配置的 Promise）以配置模块的函数。
   */
  useFactory?: (...args: any[]) => Promise<ModuleOptions> | ModuleOptions;
  /**
   * 工厂可能注入的依赖项。
   */
  inject?: FactoryProvider['inject'];
  /**
   * 解析为现有提供者的注入令牌。该提供者必须实现相应的接口。
   */
  useExisting?: Type<
    ConfigurableModuleOptionsFactory<ModuleOptions, FactoryClassMethodKey>
  >;
}
```

让我们逐一解释上述属性：

- `useFactory` - 返回配置对象的函数。它可以是同步的或异步的。要将依赖项注入到工厂函数中，请使用 `inject` 属性。我们在上面的示例中使用了这个变体。
- `inject` - 将被注入到工厂函数中的依赖项数组。依赖项的顺序必须与工厂函数参数的顺序一致。
- `useClass` - 将被实例化为提供者的类。该类必须实现相应的接口。通常，这是一个提供 `create()` 方法返回配置对象的类。有关此内容的更多信息，请参见下方的 [自定义方法键](/fundamentals/dynamic-modules#custom-method-key) 部分。
- `useExisting` - `useClass` 的一个变体，允许你使用一个已注册的提供者而不是指示 Nest 创建一个新实例。当你想使用模块中已注册的提供者时非常有用。请注意，该类必须实现与 `useClass` 中使用的相同的接口（除非你覆盖默认方法名，请参见下方的 [自定义方法键](/fundamentals/dynamic-modules#custom-method-key) 部分）。

始终选择上述选项中的一个（`useFactory`、`useClass` 或 `useExisting`），因为它们是互斥的。

最后，让我们更新 `ConfigService` 类，以注入生成的模块选项提供者，而不是我们迄今为止使用的 `'CONFIG_OPTIONS'`。

```typescript
@Injectable()
export class ConfigService {
  constructor(@Inject(MODULE_OPTIONS_TOKEN) private options: ConfigModuleOptions) { ... }
}
```

#### 自定义方法键

`ConfigurableModuleClass` 默认提供 `register` 及其对应的 `registerAsync` 方法。要使用不同的方法名，请使用 `ConfigurableModuleBuilder#setClassMethodName` 方法，如下所示：

```typescript
@@filename(config.module-definition)
export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN } =
  new ConfigurableModuleBuilder<ConfigModuleOptions>().setClassMethodName('forRoot').build();
@@switch
export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN } =
  new ConfigurableModuleBuilder().setClassMethodName('forRoot').build();
```

此构造将指示 `ConfigurableModuleBuilder` 生成一个类，该类暴露 `forRoot` 和 `forRootAsync`。示例：

```typescript
@Module({
  imports: [
    ConfigModule.forRoot({ folder: './config' }), // <-- 注意使用 "forRoot" 而不是 "register"
    // 或者替代方式：
    // ConfigModule.forRootAsync({
    //   useFactory: () => {
    //     return {
    //       folder: './config',
    //     }
    //   },
    //   inject: [...任何额外依赖...]
    // }),
  ],
})
export class AppModule {}
```

#### 自定义选项工厂类

由于 `registerAsync` 方法（或 `forRootAsync` 或任何其他名称，取决于配置）允许消费者传递一个解析为模块配置的提供者定义，库的消费者可以潜在地提供一个类来构建配置对象。

```typescript
@Module({
  imports: [
    ConfigModule.registerAsync({
      useClass: ConfigModuleOptionsFactory,
    }),
  ],
})
export class AppModule {}
```

默认情况下，此类必须提供一个 `create()` 方法，该方法返回一个模块配置对象。但是，如果你的库遵循不同的命名约定，你可以更改此行为，并指示 `ConfigurableModuleBuilder` 期望一个不同的方法，例如 `createConfigOptions`，使用 `ConfigurableModuleBuilder#setFactoryMethodName` 方法：

```typescript
@@filename(config.module-definition)
export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN } =
  new ConfigurableModuleBuilder<ConfigModuleOptions>().setFactoryMethodName('createConfigOptions').build();
@@switch
export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN } =
  new ConfigurableModuleBuilder().setFactoryMethodName('createConfigOptions').build();
```

现在，`ConfigModuleOptionsFactory` 类必须暴露 `createConfigOptions` 方法（而不是 `create`）：

```typescript
@Module({
  imports: [
    ConfigModule.registerAsync({
      useClass: ConfigModuleOptionsFactory, // <-- 此类必须提供 "createConfigOptions" 方法
    }),
  ],
})
export class AppModule {}
```

#### 额外选项

有些边缘情况中，你的模块可能需要接受一些额外的选项，这些选项决定了它的行为方式（例如 `isGlobal` 标志，或者简称为 `global`），但这些选项不应包含在 `MODULE_OPTIONS_TOKEN` 提供者中（因为它们与模块中注册的服务/提供者无关，例如 `ConfigService` 不需要知道其宿主模块是否注册为全局模块）。

在这种情况下，可以使用 `ConfigurableModuleBuilder#setExtras` 方法。请参见以下示例：

```typescript
export const { ConfigurableModuleClass, MODULE_OPTIONS_TOKEN } =
  new ConfigurableModuleBuilder<ConfigModuleOptions>()
    .setExtras(
      {
        isGlobal: true,
      },
      (definition, extras) => ({
        ...definition,
        global: extras.isGlobal,
      }),
    )
    .build();
```

在上面的示例中，传递给 `setExtras` 方法的第一个参数是一个包含“额外”属性默认值的对象。第二个参数是一个函数，它接受一个自动生成的模块定义（包含 `provider`、`exports` 等）和一个表示额外属性的 `extras` 对象（由消费者指定或默认值）。该函数的返回值是修改后的模块定义。在这个特定示例中，我们获取 `extras.isGlobal` 属性并将其赋值给模块定义的 `global` 属性（这反过来决定了模块是否为全局模块，更多信息请参见 [动态模块](/modules#dynamic-modules)）。

现在在消费此模块时，可以传入额外的 `isGlobal` 标志，如下所示：

```typescript
@Module({
  imports: [
    ConfigModule.register({
      isGlobal: true,
      folder: './config',
    }),
  ],
})
export class AppModule {}
```

然而，由于 `isGlobal` 被声明为“额外”属性，它将不会出现在 `MODULE_OPTIONS_TOKEN` 提供者中：

```typescript
@Injectable()
export class ConfigService {
  constructor(
    @Inject(MODULE_OPTIONS_TOKEN) private options: ConfigModuleOptions,
  ) {
    // "options" 对象将不包含 "isGlobal" 属性
    // ...
  }
}
```

#### 扩展自动生成的方法

如果需要，可以扩展自动生成的静态方法（`register`、`registerAsync` 等），如下所示：

```typescript
import { Module } from '@nestjs/common';
import { ConfigService } from './config.service';
import {
  ConfigurableModuleClass,
  ASYNC_OPTIONS_TYPE,
  OPTIONS_TYPE,
} from './config.module-definition';

@Module({
  providers: [ConfigService],
  exports: [ConfigService],
})
export class ConfigModule extends ConfigurableModuleClass {
  static register(options: typeof OPTIONS_TYPE): DynamicModule {
    return {
      // 你的自定义逻辑
      ...super.register(options),
    };
  }

  static registerAsync(options: typeof ASYNC_OPTIONS_TYPE): DynamicModule {
    return {
      // 你的自定义逻辑
      ...super.registerAsync(options),
    };
  }
}
```

请注意使用了 `OPTIONS_TYPE` 和 `ASYNC_OPTIONS_TYPE` 类型，这些类型必须从模块定义文件中导出：

```typescript
export const {
  ConfigurableModuleClass,
  MODULE_OPTIONS_TOKEN,
  OPTIONS_TYPE,
  ASYNC_OPTIONS_TYPE,
} = new ConfigurableModuleBuilder<ConfigModuleOptions>().build();
```