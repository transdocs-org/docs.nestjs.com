### 简介

Nest（NestJS）是一个用于构建高效、可扩展的 [Node.js](https://nodejs.org/) 服务端应用程序的框架。它使用渐进式 JavaScript，基于 [TypeScript](http://www.typescriptlang.org/) 构建并完全支持 TypeScript（同时也允许开发者使用纯 JavaScript 进行开发），并融合了 OOP（面向对象编程）、FP（函数式编程）和 FRP（函数响应式编程）的元素。

在底层，Nest 使用了像 [Express](https://expressjs.com/)（默认）这样强大的 HTTP 服务框架，并且可以选择性地配置使用 [Fastify](https://github.com/fastify/fastify)。

Nest 在这些常见的 Node.js 框架（Express/Fastify）之上提供了一层抽象，但也直接向开发者暴露了这些框架的 API。这使得开发者可以自由使用底层平台提供的大量第三方模块。

#### 设计理念

近年来，得益于 Node.js 的发展，JavaScript 已成为前端和后端应用程序的“通用语言”。这也催生了许多优秀的项目，例如 [Angular](https://angular.dev/)、[React](https://github.com/facebook/react) 和 [Vue](https://github.com/vuejs/vue)，它们提高了开发者的生产力，并帮助创建快速、可测试、可扩展的前端应用。然而，尽管 Node.js（服务端 JavaScript）领域存在大量出色的库、辅助工具和框架，但没有一个能有效解决 **架构** 这一主要问题。

Nest 提供了一种开箱即用的应用程序架构，使开发者和团队能够创建高度可测试、可扩展、松耦合且易于维护的应用程序。该架构深受 Angular 的启发。

#### 安装

要开始使用，你可以使用 [Nest CLI](/cli/overview) 创建项目脚手架，或者 [克隆一个启动项目](#alternatives)（两者效果相同）。

使用 Nest CLI 创建项目脚手架，请运行以下命令。这将创建一个新的项目目录，并在其中填充初始的核心 Nest 文件和支持模块，为你的项目创建一个常规的结构基础。我们建议首次使用的用户使用 **Nest CLI** 来创建新项目。我们将在 [入门指南](first-steps) 中继续采用这种方式。

```bash
$ npm i -g @nestjs/cli
$ nest new project-name
```

> info **提示** 要创建一个具有更严格功能集的 TypeScript 项目，请在 `nest new` 命令后添加 `--strict` 标志。

#### 其他安装方式

或者，你可以使用 **Git** 安装 TypeScript 启动项目：

```bash
$ git clone https://github.com/nestjs/typescript-starter.git project
$ cd project
$ npm install
$ npm run start
```

> info **提示** 如果你想在不保留 Git 历史记录的情况下克隆仓库，可以使用 [degit](https://github.com/Rich-Harris/degit)。

打开浏览器，访问 [`http://localhost:3000/`](http://localhost:3000/)。

如果你想安装 JavaScript 版本的启动项目，只需将上面命令中的地址替换为 `javascript-starter.git`。

你也可以通过手动安装核心和支持包来从零开始构建一个新项目。请注意，你需要自己设置项目的基础模板文件。至少需要以下依赖项：`@nestjs/core`、`@nestjs/common`、`rxjs` 和 `reflect-metadata`。查看这篇简短的文章了解如何从零开始创建完整项目：[5 steps to create a bare minimum NestJS app from scratch!](https://dev.to/micalevisk/5-steps-to-create-a-bare-minimum-nestjs-app-from-scratch-5c3b)