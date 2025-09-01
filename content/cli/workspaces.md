### 工作区（Workspaces）

Nest 提供了两种组织代码的模式：

- **标准模式（standard mode）**：适用于构建专注于单个项目的应用程序，这些应用程序拥有自己的依赖和配置，且不需要优化模块共享或复杂构建。这是默认模式。
- **单体仓库模式（monorepo mode）**：这种模式将代码工件视为轻量级 **单体仓库（monorepo）** 的一部分，更适合开发团队和/或多项目环境。它自动化了部分构建流程，便于创建和组合模块化组件，促进代码复用，简化集成测试，方便共享项目范围内的工件（如 `eslint` 规则和其他配置策略），并且比 Git 子模块等替代方案更易于使用。单体仓库模式使用 **工作区（workspace）** 的概念，通过 `nest-cli.json` 文件来协调单体仓库中各组件之间的关系。

需要注意的是，Nest 的几乎所有功能都与你的代码组织模式无关。这种选择的 **唯一** 影响是项目如何组合以及构建工件如何生成。从 CLI 到核心模块再到附加模块，所有其他功能在两种模式下都是一样的。

此外，你可以在任何时候轻松地从 **标准模式** 切换到 **单体仓库模式**，因此你可以推迟这个决定，直到某种方法的优势变得更为明显。

#### 标准模式

当你运行 `nest new` 命令时，Nest 会使用内置的 schematics 创建一个新的 **项目（project）**。Nest 会执行以下操作：

1. 创建一个新文件夹，文件夹名对应你提供给 `nest new` 的 `name` 参数
2. 使用一个最小的 Nest 应用程序基础文件填充该文件夹。你可以在 [typescript-starter](https://github.com/nestjs/typescript-starter) 仓库中查看这些文件
3. 提供额外的配置文件，例如 `nest-cli.json`、`package.json` 和 `tsconfig.json`，用于配置和启用各种工具来编译、测试和运行你的应用程序

从这里开始，你可以修改初始文件，添加新组件，添加依赖项（例如 `npm install`），并按照本文档其余部分所述继续开发你的应用程序。

#### 单体仓库模式

要启用单体仓库模式，你需要从 _标准模式_ 的结构开始，并添加 **项目（projects）**。一个项目可以是一个完整的 **应用程序（application）**（使用 `nest generate app` 命令添加到工作区中），也可以是一个 **库（library）**（使用 `nest generate library` 命令添加到工作区中）。我们将在下文中讨论这些项目组件的详细信息。现在需要记住的关键点是，**将项目添加到现有标准模式结构中的操作** 会将其 **转换为单体仓库模式**。我们来看一个例子。

如果我们运行以下命令：

```bash
$ nest new my-project
```

我们构建了一个 _标准模式_ 的结构，其文件结构如下所示：

<div class="file-tree">
  <div class="item">node_modules</div>
  <div class="item">src</div>
  <div class="children">
    <div class="item">app.controller.ts</div>
    <div class="item">app.module.ts</div>
    <div class="item">app.service.ts</div>
    <div class="item">main.ts</div>
  </div>
  <div class="item">nest-cli.json</div>
  <div class="item">package.json</div>
  <div class="item">tsconfig.json</div>
  <div class="item">eslint.config.mjs</div>
</div>

我们可以按以下方式将其转换为单体仓库模式：

```bash
$ cd my-project
$ nest generate app my-app
```

此时，`nest` 会将现有结构转换为 **单体仓库模式（monorepo mode）**。这会带来一些重要的变化。新的文件结构如下所示：

<div class="file-tree">
  <div class="item">apps</div>
    <div class="children">
      <div class="item">my-app</div>
      <div class="children">
        <div class="item">src</div>
        <div class="children">
          <div class="item">app.controller.ts</div>
          <div class="item">app.module.ts</div>
          <div class="item">app.service.ts</div>
          <div class="item">main.ts</div>
        </div>
        <div class="item">tsconfig.app.json</div>
      </div>
      <div class="item">my-project</div>
      <div class="children">
        <div class="item">src</div>
        <div class="children">
          <div class="item">app.controller.ts</div>
          <div class="item">app.module.ts</div>
          <div class="item">app.service.ts</div>
          <div class="item">main.ts</div>
        </div>
        <div class="item">tsconfig.app.json</div>
      </div>
    </div>
  <div class="item">nest-cli.json</div>
  <div class="item">package.json</div>
  <div class="item">tsconfig.json</div>
  <div class="item">eslint.config.mjs</div>
</div>

`generate app` 的 schematics 重新组织了代码 —— 将每个 **应用程序项目** 移动到 `apps` 文件夹下，并在每个项目的根目录中添加了一个项目特定的 `tsconfig.app.json` 文件。我们原来的 `my-project` 应用程序现在成为了单体仓库的 **默认项目（default project）**，并与新添加的 `my-app` 并列，位于 `apps` 文件夹下。我们将在下文中介绍默认项目。

> error **警告**：将标准模式结构转换为单体仓库模式仅适用于遵循 Nest 标准项目结构的项目。在转换过程中，schematics 会尝试将 `src` 和 `test` 文件夹移动到 `apps` 文件夹下的项目文件夹中。如果项目未使用此结构，转换将失败或产生不可靠的结果。

#### 工作区项目

单体仓库使用 **工作区（workspace）** 的概念来管理其成员实体。工作区由 **项目（projects）** 组成。一个项目可以是：

- 一个 **应用程序（application）**：一个完整的 Nest 应用程序，包含一个用于启动应用程序的 `main.ts` 文件。除了编译和构建方面的考虑，工作区中的应用程序项目与 _标准模式_ 结构中的应用程序在功能上是完全相同的。
- 一个 **库（library）**：一种打包通用功能（模块、提供者、控制器等）的方式，可以在其他项目中使用。库不能独立运行，也没有 `main.ts` 文件。有关库的更多内容，请参阅 [此处](/cli/libraries)。

所有工作区都有一个 **默认项目（default project）**（通常应为应用程序类型项目）。它由 `nest-cli.json` 文件中的顶级 `"root"` 属性定义，该属性指向默认项目的根目录（详见下方的 [CLI 属性](/cli/monorepo#cli-properties)）。通常情况下，这就是你最初创建的标准模式应用程序，之后通过 `nest generate app` 转换为单体仓库。当你按照这些步骤操作时，该属性会自动填充。

当你未指定项目名称时，`nest` 命令（如 `nest build` 和 `nest start`）将使用默认项目。

例如，在上述单体仓库结构中运行：

```bash
$ nest start
```

将会启动 `my-project` 应用程序。要启动 `my-app`，我们应使用：

```bash
$ nest start my-app
```

#### 应用程序

应用程序类型的项目，或者我们通常简称为“应用程序（application）”，是你可以运行和部署的完整 Nest 应用程序。你可以使用 `nest generate app` 命令生成应用程序类型项目。

该命令会自动生成一个项目骨架，包括来自 [typescript starter](https://github.com/nestjs/typescript-starter) 的标准 `src` 和 `test` 文件夹。与标准模式不同的是，单体仓库中的应用程序项目不包含任何包依赖（`package.json`）或其他项目配置文件（如 `.prettierrc` 和 `eslint.config.mjs`）。相反，这些配置文件在单体仓库范围内共享使用。

不过，schematics 会在项目的根目录中生成一个项目特定的 `tsconfig.app.json` 文件。该配置文件会自动设置适当的构建选项，包括正确设置编译输出目录。该文件继承自顶级（单体仓库）的 `tsconfig.json` 文件，因此你可以统一管理全局设置，也可以在项目级别覆盖这些设置。

#### 库

如前所述，库类型项目，或简称为“库（library）”，是 Nest 组件的包，需要被组合到应用程序中才能运行。你可以使用 `nest generate library` 命令生成库类型项目。决定哪些内容应放入库中是一个架构设计决策。我们将在 [库](/cli/libraries) 章节中深入讨论库的使用。

#### CLI 属性

Nest 将组织、构建和部署标准模式和单体仓库结构项目所需的元数据保存在 `nest-cli.json` 文件中。当你添加项目时，Nest 会自动向该文件中添加内容并更新它，因此你通常不需要手动编辑其内容。然而，有些设置你可能希望手动修改，因此了解该文件的结构是有帮助的。

在运行上述步骤创建单体仓库后，我们的 `nest-cli.json` 文件内容如下：

```javascript
{
  "collection": "@nestjs/schematics",
  "sourceRoot": "apps/my-project/src",
  "monorepo": true,
  "root": "apps/my-project",
  "compilerOptions": {
    "webpack": true,
    "tsConfigPath": "apps/my-project/tsconfig.app.json"
  },
  "projects": {
    "my-project": {
      "type": "application",
      "root": "apps/my-project",
      "entryFile": "main",
      "sourceRoot": "apps/my-project/src",
      "compilerOptions": {
        "tsConfigPath": "apps/my-project/tsconfig.app.json"
      }
    },
    "my-app": {
      "type": "application",
      "root": "apps/my-app",
      "entryFile": "main",
      "sourceRoot": "apps/my-app/src",
      "compilerOptions": {
        "tsConfigPath": "apps/my-app/tsconfig.app.json"
      }
    }
  }
}
```

该文件分为几个部分：

- 一个全局部分，包含控制标准模式和单体仓库范围设置的顶层属性
- 一个顶层属性（`"projects"`），包含每个项目的元数据。该部分仅在单体仓库模式结构中存在

顶层属性如下：

- `"collection"`：指向用于生成组件的 schematics 集合；通常不应更改此值
- `"sourceRoot"`：指向标准模式结构中单个项目的源代码根目录，或单体仓库模式结构中 _默认项目_ 的源代码根目录
- `"compilerOptions"`：一个映射，键表示编译器选项，值表示该选项的设置；详见下方
- `"generateOptions"`：一个映射，键表示全局生成选项，值表示该选项的设置；详见下方
- `"monorepo"`：（仅限单体仓库）在单体仓库模式结构中，此值始终为 `true`
- `"root"`：（仅限单体仓库）指向 _默认项目_ 的项目根目录

#### 全局编译器选项

这些属性指定使用的编译器以及影响 **任何** 编译步骤的各种选项，无论该步骤是作为 `nest build` 或 `nest start` 的一部分，还是使用 `tsc` 或 webpack。

| 属性名称           | 属性值类型 | 描述                                                                                                                                                                                                                                                               |
| ------------------ | ---------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `webpack`          | 布尔值     | 如果为 `true`，则使用 [webpack 编译器](https://webpack.js.org/)。如果为 `false` 或未设置，则使用 `tsc`。在单体仓库模式中，默认值为 `true`（使用 webpack），在标准模式中，默认值为 `false`（使用 `tsc`）。（已弃用：请使用 `builder` 代替） |
| `tsConfigPath`     | 字符串     | （**仅限单体仓库**）指向包含 `tsconfig.json` 设置的文件。当 `nest build` 或 `nest start` 被调用且未指定 `project` 参数时（例如，构建或启动默认项目时）将使用该设置。                                                                 |
| `webpackConfigPath`| 字符串     | 指向 webpack 配置文件。如果未指定，Nest 会查找文件 `webpack.config.js`。详见下方。                                                                                                                                                                                    |
| `deleteOutDir`     | 布尔值     | 如果为 `true`，则在每次调用编译器时，都会先删除编译输出目录（该目录在 `tsconfig.json` 中配置，默认为 `./dist`）。                                                                                             |
| `assets`           | 数组       | 启用在每次编译步骤开始时自动分发非 TypeScript 资源（在 `--watch` 模式下的增量编译中不会发生资源分发）。详见下方。                                                                                              |
| `watchAssets`      | 布尔值     | 如果为 `true`，则在 watch 模式下运行，监视 **所有** 非 TypeScript 资源。（如需更细粒度地控制要监视的资源，请参见下方的 [资源（Assets）](cli/monorepo#assets) 部分）。                                               |
| `manualRestart`    | 布尔值     | 如果为 `true`，则启用快捷键 `rs` 手动重启服务器。默认值为 `false`。                                                                                                                                                                                                 |
| `builder`          | 字符串/对象| 指示 CLI 使用什么 `builder` 来编译项目（`tsc`、`swc` 或 `webpack`）。如需自定义 builder 的行为，可以传递一个包含两个属性的对象：`type`（`tsc`、`swc` 或 `webpack`）和 `options`。                              |
| `typeCheck`        | 布尔值     | 如果为 `true`，则为基于 SWC 的项目（当 `builder` 为 `swc` 时）启用类型检查。默认值为 `false`。                                                                                                                                                                         |

#### 全局生成选项

这些属性指定 `nest generate` 命令使用的默认生成选项。

| 属性名称 | 属性值类型 | 描述                                                                                                                                                                                                                                                                                                                                                                                                 |
| -------- | ---------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `spec`   | 布尔值/对象| 如果值为布尔值，`true` 表示默认启用 `spec` 生成，`false` 表示禁用。CLI 命令行中传递的标志会覆盖此设置，项目特定的 `generateOptions` 设置也会覆盖。如果值为对象，每个键代表一个 schematics 名称，布尔值决定是否为该特定 schematics 启用/禁用默认的 spec 生成。                                                                                             |
| `flat`   | 布尔值     | 如果为 `true`，所有生成命令将生成扁平结构                                                                                                                                                                                                                                                                                                                                                             |

以下示例使用布尔值指定所有项目的 `spec` 文件生成默认禁用：

```javascript
{
  "generateOptions": {
    "spec": false
  },
  ...
}
```

以下示例使用布尔值指定所有项目的文件生成默认为扁平结构：

```javascript
{
  "generateOptions": {
    "flat": true
  },
  ...
}
```

在以下示例中，仅对 `service` schematics（例如 `nest generate service...`）禁用 `spec` 文件生成：

```javascript
{
  "generateOptions": {
    "spec": {
      "service": false
    }
  },
  ...
}
```

> warning **警告**：当将 `spec` 指定为对象时，目前不支持自动处理 schematics 的别名。这意味着如果你将键指定为 `service: false`，并尝试通过别名 `s` 生成服务，则 `spec` 文件仍会被生成。为了确保正常命令名和别名都能正常工作，请同时指定正常命令名和别名，如下所示：
>
> ```javascript
> {
>   "generateOptions": {
>     "spec": {
>       "service": false,
>       "s": false
>     }
>   },
>   ...
> }
> ```

#### 项目特定的生成选项

除了提供全局生成选项外，你还可以指定项目特定的生成选项。项目特定的生成选项与全局生成选项格式完全相同，但直接指定在每个项目中。

项目特定的生成选项会覆盖全局生成选项。

```javascript
{
  "projects": {
    "cats-project": {
      "generateOptions": {
        "spec": {
          "service": false
        }
      },
      ...
    }
  },
  ...
}
```

> warning **警告**：生成选项的优先级顺序如下：CLI 命令行中指定的选项优先于项目特定选项，项目特定选项覆盖全局选项。

#### 指定的编译器

不同默认编译器的原因是，在较大项目中（例如，单体仓库中更常见的情况），webpack 在构建时间和生成单个包含所有项目组件的文件方面具有显著优势。如果你希望生成单独的文件，请将 `"webpack"` 设置为 `false`，这将使构建过程使用 `tsc`（或 `swc`）。

#### Webpack 选项

Webpack 配置文件可以包含标准的 [webpack 配置选项](https://webpack.js.org/configuration/)。例如，要告诉 webpack 打包 `node_modules`（默认情况下这些模块是被排除的），请在 `webpack.config.js` 文件中添加以下内容：

```javascript
module.exports = {
  externals: [],
};
```

由于 webpack 配置文件是一个 JavaScript 文件，你甚至可以导出一个函数，该函数接受默认选项并返回修改后的对象：

```javascript
module.exports = function (options) {
  return {
    ...options,
    externals: [],
  };
};
```

#### 资源（Assets）

TypeScript 编译会自动将编译输出（`.js` 和 `.d.ts` 文件）分发到指定的输出目录。将非 TypeScript 文件（如 `.graphql` 文件、`images`、`.html` 文件和其他资源）也进行分发有时也很方便。这使你可以将 `nest build`（以及任何初始编译步骤）视为一个轻量级的 **开发构建** 步骤，在此步骤中你可以编辑非 TypeScript 文件并进行迭代编译和测试。
这些资源应位于 `src` 文件夹中，否则它们不会被复制。

`assets` 键的值应为一个数组，指定要分发的文件。数组元素可以是简单的字符串，使用类似 `glob` 的文件规范，例如：

```typescript
"assets": ["**/*.graphql"],
"watchAssets": true,
```

如需更精细的控制，数组元素可以是包含以下键的对象：

- `"include"`：类似 `glob` 的文件规范，指定要分发的资源
- `"exclude"`：类似 `glob` 的文件规范，指定要从 `include` 列表中 **排除** 的资源
- `"outDir"`：字符串，指定资源应分发到的路径（相对于根目录）。默认值与编译器输出目录相同
- `"watchAssets"`：布尔值，如果为 `true`，则在 watch 模式下运行并监视指定资源

例如：

```typescript
"assets": [
  { "include": "**/*.graphql", "exclude": "**/omitted.graphql", "watchAssets": true },
]
```

> warning **警告**：在顶层 `compilerOptions` 属性中设置 `watchAssets` 会覆盖 `assets` 属性中的任何 `watchAssets` 设置。

#### 项目属性

该元素仅存在于单体仓库模式结构中。你通常不应编辑这些属性，因为它们由 Nest 用于在单体仓库中定位项目及其配置选项。