### Nest CLI 和脚本

本节提供有关 `nest` 命令如何与编译器和脚本交互的更多背景信息，以帮助 DevOps 人员管理开发环境。

一个 Nest 应用程序是一个**标准的** TypeScript 应用程序，在执行之前需要编译为 JavaScript。完成编译步骤的方法有很多种，开发者/团队可以自由选择最适合自己的方式。考虑到这一点，Nest 提供了一组开箱即用的工具，旨在实现以下目标：

- 提供一个标准的构建/执行流程，可通过命令行使用，并且在合理默认配置下“即开即用”。
- 确保构建/执行流程是**开放的**，这样开发者可以直接访问底层工具，并使用其原生功能和选项进行自定义。
- 保持完全标准的 TypeScript/Node.js 框架特性，这样开发团队可以选择的任何外部工具都可以管理整个编译/部署/执行流程。

通过 `nest` 命令、本地安装的 TypeScript 编译器以及 `package.json` 脚本的组合实现了这一目标。我们将在下文中描述这些技术如何协同工作。这将帮助你理解构建/执行流程中每一步发生了什么，以及在必要时如何自定义这些行为。

#### Nest 可执行文件

`nest` 命令是一个操作系统级别的二进制文件（即，从操作系统命令行运行）。该命令实际上涵盖了以下三个不同的功能区域。我们建议你通过项目脚手架时自动生成的 `package.json` 脚本来运行构建（`nest build`）和执行（`nest start`）子命令（如果你想通过克隆仓库开始开发而不是运行 `nest new`，请参阅 [typescript starter](https://github.com/nestjs/typescript-starter)）。

#### 构建

`nest build` 是标准 `tsc` 编译器或 `swc` 编译器（适用于[标准项目](https://docs.nestjs.com/cli/overview#project-structure)）或使用 `ts-loader` 的 webpack 打包工具（适用于[单体仓库](https://docs.nestjs.com/cli/overview#project-structure)）的封装。它除了开箱即用地处理 `tsconfig-paths` 之外，不添加任何其他编译功能或步骤。它存在的原因是大多数开发者，特别是刚开始使用 Nest 的开发者，通常不需要调整编译器选项（例如 `tsconfig.json` 文件），而这些选项有时可能比较复杂。

更多细节请参阅 [nest build](https://docs.nestjs.com/cli/usages#nest-build) 文档。

#### 执行

`nest start` 简单地确保项目已经构建完成（等同于 `nest build`），然后以一种可移植、简便的方式调用 `node` 命令来执行编译后的应用程序。与构建过程一样，你可以根据需要自由地自定义此流程，无论是使用 `nest start` 命令及其选项，还是完全替换它。整个流程是一个标准的 TypeScript 应用程序构建和执行流程，你可以按此方式管理该流程。

更多细节请参阅 [nest start](https://docs.nestjs.com/cli/usages#nest-start) 文档。

#### 生成

顾名思义，`nest generate` 命令用于生成新的 Nest 项目或其中的组件。

#### 包脚本（Package scripts）

在操作系统命令行级别运行 `nest` 命令要求 `nest` 可执行文件已全局安装。这是 npm 的一项标准功能，不在 Nest 的直接控制范围内。这带来一个后果是全局安装的 `nest` 可执行文件**不会**作为项目的依赖项记录在 `package.json` 中。例如，两个不同的开发者可能运行的是不同版本的 `nest` 可执行文件。解决这个问题的标准方法是使用包脚本，这样你可以将构建和执行步骤中使用的工具作为开发依赖项进行管理。

当你运行 `nest new` 或克隆 [typescript starter](https://github.com/nestjs/typescript-starter) 时，Nest 会将 `build` 和 `start` 等命令填充到新项目的 `package.json` 脚本中。同时，它会将底层编译工具（如 `typescript`）安装为**开发依赖项**。

你可以使用如下命令运行构建和执行脚本：

```bash
$ npm run build
```

和

```bash
$ npm run start
```

这些命令使用 npm 的脚本运行功能，通过**本地安装的** `nest` 可执行文件执行 `nest build` 或 `nest start`。通过使用这些内置的包脚本，你可以完全管理 Nest CLI 命令的依赖关系\*。这意味着，通过遵循这种**推荐**的使用方式，你的组织中的所有成员都可以确保运行相同版本的命令。

\*这适用于 `build` 和 `start` 命令。`nest new` 和 `nest generate` 命令不属于构建/执行流程，因此它们在不同的上下文中运行，并且没有内置的 `package.json` 脚本。

对于大多数开发者/团队来说，建议使用包脚本来构建和执行他们的 Nest 项目。你可以通过其选项（`--path`、`--webpack`、`--webpackPath`）完全自定义这些脚本的行为，和/或根据需要自定义 `tsc` 或 webpack 编译器选项文件（例如 `tsconfig.json`）。你也可以自由运行完全自定义的构建流程来编译 TypeScript（甚至可以直接使用 `ts-node` 执行 TypeScript）。

#### 向后兼容性

由于 Nest 应用程序是纯 TypeScript 应用程序，因此旧版本的 Nest 构建/执行脚本将继续运行。你无需升级它们。你可以在准备就绪时选择使用新的 `nest build` 和 `nest start` 命令，或者继续运行以前或自定义的脚本。

#### 迁移

虽然你无需进行任何更改，但你可能希望迁移到使用新的 CLI 命令，而不是使用 `tsc-watch` 或 `ts-node` 等工具。在这种情况下，只需全局和本地安装最新版本的 `@nestjs/cli`：

```bash
$ npm install -g @nestjs/cli
$ cd  /some/project/root/folder
$ npm install -D @nestjs/cli
```

然后你可以将 `package.json` 中定义的 `scripts` 替换为以下内容：

```json
"build": "nest build",
"start": "nest start",
"start:dev": "nest start --watch",
"start:debug": "nest start --debug --watch"
```