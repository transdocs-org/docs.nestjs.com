### 第一步

在这系列文章中，你将学习 Nest 的**核心基础**。为了熟悉 Nest 应用程序的基本构建模块，我们将构建一个基本的 CRUD 应用程序，涵盖许多入门级别的功能。

#### 语言

我们钟爱 [TypeScript](https://www.typescriptlang.org/)，但更重要的是，我们热爱 [Node.js](https://nodejs.org/en/)。这就是为什么 Nest 同时兼容 TypeScript 和纯 JavaScript。Nest 利用了最新的语言特性，因此要在 vanilla JavaScript 中使用它，我们需要一个 [Babel](https://babeljs.io/) 编译器。

我们的示例主要使用 TypeScript，但你可以随时将代码片段切换为 vanilla JavaScript 语法（只需点击每个代码块右上角的语言切换按钮）。

#### 前提条件

请确保你的操作系统上已安装 [Node.js](https://nodejs.org)（版本 >= 20）。

#### 设置

使用 [Nest CLI](/cli/overview) 创建新项目非常简单。安装 [npm](https://www.npmjs.com/) 后，你可以在操作系统终端中运行以下命令来创建一个新的 Nest 项目：

```bash
$ npm i -g @nestjs/cli
$ nest new project-name
```

> info **提示** 要使用 TypeScript 的 [更严格](https://www.typescriptlang.org/tsconfig#strict) 特性集创建新项目，请向 `nest new` 命令传递 `--strict` 标志。

将创建名为 `project-name` 的目录，安装 node 模块和其他一些样板文件，并创建 `src/` 目录并填充几个核心文件。

<div class="file-tree">
  <div class="item">src</div>
  <div class="children">
    <div class="item">app.controller.spec.ts</div>
    <div class="item">app.controller.ts</div>
    <div class="item">app.module.ts</div>
    <div class="item">app.service.ts</div>
    <div class="item">main.ts</div>
  </div>
</div>

以下是这些核心文件的简要概述：

|                          |                                                                                                                     |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------- |
| `app.controller.ts`      | 一个具有单个路由的基本控制器。                                                                                      |
| `app.controller.spec.ts` | 控制器的单元测试。                                                                                                  |
| `app.module.ts`          | 应用程序的根模块。                                                                                                  |
| `app.service.ts`         | 一个具有单个方法的基本服务。                                                                                        |
| `main.ts`                | 应用程序的入口文件，使用核心函数 `NestFactory` 创建一个 Nest 应用实例。                                               |

`main.ts` 包含一个异步函数，它将**启动**我们的应用程序：

```typescript
@@filename(main)

import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
@@switch
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```

要创建一个 Nest 应用实例，我们使用核心的 `NestFactory` 类。`NestFactory` 暴露了一些静态方法，允许创建应用程序实例。`create()` 方法返回一个应用程序对象，该对象实现了 `INestApplication` 接口。这个对象提供了一组方法，这些方法将在接下来的章节中描述。在上面的 `main.ts` 示例中，我们简单地启动了 HTTP 监听器，让应用程序等待传入的 HTTP 请求。

请注意，使用 Nest CLI 生成的项目结构鼓励开发人员遵循将每个模块保留在其专用目录中的约定。

> info **提示** 默认情况下，如果在创建应用程序时发生任何错误，你的应用程序将退出并返回代码 `1`。如果你想让它抛出错误而不是退出，请禁用 `abortOnError` 选项（例如，`NestFactory.create(AppModule, {{ '{' }} abortOnError: false {{ '}' }})`）。

<app-banner-courses></app-banner-courses>

#### 平台

Nest 旨在成为一个平台无关的框架。平台独立性使得开发人员可以跨多种类型的应用程序重用逻辑组件。从技术上讲，一旦创建了适配器，Nest 可以与任何 Node HTTP 框架一起工作。目前开箱即用支持两种 HTTP 平台：[express](https://expressjs.com/) 和 [fastify](https://www.fastify.io/)。你可以选择最适合你需求的平台。

|                    |                                                                                                                                                                                                                                                                                                                                    |
| ------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `platform-express` | [Express](https://expressjs.com/) 是一个知名的极简 Node Web 框架。它是一个经过实战检验的生产就绪库，社区已经实现了大量资源。默认情况下使用 `@nestjs/platform-express` 包。许多用户使用 Express 就已经足够，无需任何操作即可启用它。 |
| `platform-fastify` | [Fastify](https://www.fastify.io/) 是一个高性能、低开销的框架，专注于提供最大效率和速度。阅读[这里](/techniques/performance)了解如何使用它。                                                                                                                                  |

无论使用哪种平台，它都会暴露自己的应用程序接口。它们分别被称为 `NestExpressApplication` 和 `NestFastifyApplication`。

当你将类型传递给 `NestFactory.create()` 方法时，如下面的示例所示，`app` 对象将仅具有该特定平台可用的方法。不过请注意，除非你确实需要访问底层平台 API，否则不需要指定类型。

```typescript
const app = await NestFactory.create<NestExpressApplication>(AppModule);
```

#### 运行应用程序

安装过程完成后，你可以在操作系统命令提示符中运行以下命令，以启动应用程序并监听传入的 HTTP 请求：

```bash
$ npm run start
```

> info **提示** 为了加快开发过程（构建速度提高 20 倍），你可以通过向 `start` 脚本传递 `-b swc` 标志来使用 [SWC 构建器](/recipes/swc)，如下所示：`npm run start -- -b swc`。

此命令将启动应用程序，HTTP 服务器将监听 `src/main.ts` 文件中定义的端口。一旦应用程序运行起来，打开浏览器并访问 `http://localhost:3000/`。你应该会看到 `Hello World!` 消息。

要监听文件更改，你可以运行以下命令来启动应用程序：

```bash
$ npm run start:dev
```

该命令将监视你的文件，自动重新编译并重新加载服务器。

#### 代码检查和格式化

[CLI](/cli/overview) 提供了最佳实践，以在大规模开发中构建可靠的开发流程。因此，生成的 Nest 项目预装了代码**检查器**和**格式化工具**（分别是 [eslint](https://eslint.org/) 和 [prettier](https://prettier.io/)）。

> info **提示** 不确定格式化工具和检查器之间的区别？[这里](https://prettier.io/docs/en/comparison.html) 了解它们的不同。

为了确保最大稳定性和可扩展性，我们使用了基础的 [`eslint`](https://www.npmjs.com/package/eslint) 和 [`prettier`](https://www.npmjs.com/package/prettier) CLI 包。这种设置设计上允许与官方扩展的 IDE 无缝集成。

对于不涉及 IDE 的无头环境（持续集成、Git 钩子等），Nest 项目提供了现成的 `npm` 脚本。

```bash
# 使用 eslint 进行代码检查和自动修复
$ npm run lint

# 使用 prettier 进行格式化
$ npm run format
```