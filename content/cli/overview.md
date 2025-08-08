### 概览

[Nest CLI](https://github.com/nestjs/nest-cli) 是一个命令行接口工具，可帮助你初始化、开发和维护 Nest 应用程序。它可以通过多种方式提供帮助，包括项目脚手架、以开发模式运行项目、以及为生产环境构建和打包应用程序。它体现了最佳实践的架构模式，以鼓励构建结构良好的应用程序。

#### 安装

**注意**：在本指南中，我们使用 [npm](https://docs.npmjs.com/downloading-and-installing-node-js-and-npm) 来安装包，包括 Nest CLI。你也可以根据需要使用其他包管理器。使用 npm 时，你可以通过多种方式管理操作系统命令行解析 `nest` CLI 可执行文件路径的位置。这里我们描述使用 `-g` 参数全局安装 `nest` 可执行文件的方法。这提供了一定的便利性，也是文档中假设采用的方式。请注意，以全局方式安装 **任何** `npm` 包后，确保运行正确的版本将由用户自行负责。这也意味着如果你有多个项目，它们都将运行 **相同** 的 CLI 版本。一个合理的替代方案是使用内置于 `npm` CLI 的 [npx](https://github.com/npm/cli/blob/latest/docs/lib/content/commands/npx.md) 程序（或其他包管理器中的类似功能），以确保你运行的是受管理版本的 Nest CLI。我们建议你查阅 [npx 文档](https://github.com/npm/cli/blob/latest/docs/lib/content/commands/npx.md) 或咨询你的 DevOps 支持人员以获取更多信息。

使用 `npm install -g` 命令全局安装 CLI（有关全局安装的详细信息，请参见上面的**注意**）。

```bash
$ npm install -g @nestjs/cli
```

> info **提示** 或者，你可以使用此命令 `npx @nestjs/cli@latest` 而无需全局安装 CLI。

#### 基本工作流程

安装完成后，你可以通过 `nest` 可执行文件直接从操作系统命令行调用 CLI 命令。输入以下命令查看可用的 `nest` 命令：

```bash
$ nest --help
```

使用以下方式获取特定命令的帮助。将下面示例中的 `generate` 替换为你需要的命令（如 `new`、`add` 等），以获取该命令的详细帮助信息：

```bash
$ nest generate --help
```

要在开发模式下创建、构建并运行一个新的基础 Nest 项目，请进入你希望作为新项目父目录的文件夹，然后运行以下命令：

```bash
$ nest new my-nest-project
$ cd my-nest-project
$ npm run start:dev
```

在浏览器中打开 [http://localhost:3000](http://localhost:3000) 即可看到新应用程序的运行效果。当你修改任何源文件时，应用程序会自动重新编译并重新加载。

> info **提示** 我们建议使用 [SWC 构建器](/recipes/swc)，以获得更快的构建速度（比默认的 TypeScript 编译器快 10 倍）。

#### 项目结构

当你运行 `nest new` 时，Nest 会通过创建一个新文件夹并填充一组初始文件来生成一个样板应用程序结构。你可以继续使用这种默认结构，并根据本文档所述添加新组件。我们将 `nest new` 生成的项目结构称为 **标准模式**。Nest 还支持另一种用于管理多个项目和库的结构，称为 **单仓库模式（monorepo mode）**。

除了关于**构建**过程的一些具体考虑（本质上，单仓库模式简化了单仓库风格项目结构可能带来的构建复杂性），以及内置的 [库支持](/cli/libraries)，其余的 Nest 特性以及本文档内容均适用于标准模式和单仓库模式。事实上，你可以在未来任何时候轻松从标准模式切换到单仓库模式，因此在学习 Nest 的过程中，你可以安全地推迟这一决定。

你可以使用任意一种模式来管理多个项目。以下是两种模式的主要区别总结：

| 功能                                                         | 标准模式                                                       | 单仓库模式                                                  |
| -------------------------------------------------------------- | -------------------------------------------------------------- | ----------------------------------------------------------- |
| 多项目支持                                                     | 独立的文件系统结构                                              | 单一的文件系统结构                                            |
| `node_modules` 与 `package.json`                               | 各自独立                                                         | 单仓库内共享                                                  |
| 默认编译器                                                     | `tsc`                                                         | webpack                                                     |
| 编译器设置                                                     | 各自单独配置                                                     | 单仓库默认配置，可针对每个项目覆盖                             |
| 配置文件如 `eslint.config.mjs`、`.prettierrc` 等                | 各自单独配置                                                     | 单仓库内共享                                                  |
| `nest build` 和 `nest start` 命令                              | 默认作用于上下文中的（唯一）项目                                 | 默认作用于单仓库中的**默认项目**                              |
| 库支持                                                         | 需手动管理，通常通过 npm 打包                                   | 内置支持，包括路径管理和打包                                 |

有关更详细的信息，请阅读 [工作区](/cli/monorepo) 和 [库](/cli/libraries) 相关章节，以帮助你决定哪种模式更适合你。

<app-banner-courses></app-banner-courses>

#### CLI 命令语法

所有的 `nest` 命令都遵循相同的格式：

```bash
nest commandOrAlias requiredArg [optionalArg] [options]
```

例如：

```bash
$ nest new my-nest-project --dry-run
```

其中，`new` 是 _commandOrAlias_。`new` 命令有一个别名 `n`。`my-nest-project` 是 _requiredArg_。如果没有在命令行中提供 _requiredArg_，`nest` 将会提示你输入。此外，`--dry-run` 有一个等效的短格式 `-d`。考虑到这一点，以下命令等价于上面的命令：

```bash
$ nest n my-nest-project -d
```

大多数命令和部分选项都有别名。尝试运行 `nest new --help` 来查看这些选项和别名，以确认你对上述格式的理解。

#### 命令概览

对于以下任意命令，运行 `nest <command> --help` 以查看该命令的特定选项。

有关每个命令的详细描述，请参阅 [用法](/cli/usages)。

| 命令     | 别名 | 描述                                                                                      |
| -------- | ---- | ----------------------------------------------------------------------------------------- |
| `new`    | `n`  | 生成一个包含所有运行所需样板文件的新**标准模式**应用程序。                                   |
| `generate` | `g`  | 基于 schematics 生成和/或修改文件。                                                       |
| `build`  |      | 将应用程序或工作区编译到输出目录中。                                                      |
| `start`  |      | 编译并运行应用程序（或工作区中的默认项目）。                                                |
| `add`    |      | 导入一个被打包为 **nest library** 的库，并运行其安装 schematics。                           |
| `info`   | `i`  | 显示已安装的 Nest 包信息以及其他有用的系统信息。                                            |

#### 系统要求

Nest CLI 要求使用支持 [国际化支持（ICU）](https://nodejs.org/api/intl.html) 的 Node.js 可执行文件，比如从 [Node.js 官方页面](https://nodejs.org/en/download) 下载的官方可执行文件。如果你遇到与 ICU 相关的错误，请检查你的可执行文件是否符合此要求。

```bash
node -p process.versions.icu
```

如果该命令输出 `undefined`，则表示你的 Node.js 可执行文件不支持国际化。