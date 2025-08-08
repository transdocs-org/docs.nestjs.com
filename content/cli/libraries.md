### 库

许多应用程序需要解决相同的一般性问题，或者在多个不同上下文中重复使用一个模块化的组件。Nest 提供了几种方式来解决这个问题，但每种方式都在不同层级上工作，以帮助满足不同的架构和组织目标。

Nest 的 [模块](/modules) 对提供一个执行上下文非常有用，该上下文可以在单个应用程序内共享组件。模块也可以通过 [npm](https://npmjs.com) 打包，创建可重用的库，可在不同的项目中安装使用。这对于分发可配置、可重用的库非常有效，这些库可以被不同、松散关联或无关联的组织使用（例如通过分发/安装第三方库）。

在紧密组织的组内共享代码时（例如，在公司/项目边界内），可以采用一种更轻量级的方式来共享组件。单体仓库（Monorepo）的出现正是为了实现这一目标，在单体仓库中，**库** 提供了一种简单、轻量的方式来共享代码。在 Nest 单体仓库中，使用库可以轻松组装共享组件的应用程序。实际上，这鼓励了将单体应用分解为模块化组件，并促进模块化组件的开发和组合。

#### Nest 库

一个 Nest 库是一个 Nest 项目，它与应用程序的不同之处在于它不能独立运行。库必须被导入到一个包含它的应用程序中才能执行其代码。本节中描述的对库的内置支持**仅适用于单体仓库**（标准模式项目可以通过使用 npm 包实现类似功能）。

例如，一个组织可能开发了一个 `AuthModule`，用于通过实现管理所有内部应用程序的公司策略来进行身份验证。与其为每个应用程序单独构建该模块，或者将代码物理打包并通过 npm 安装到每个项目中，单体仓库可以将该模块定义为库。以这种方式组织后，所有使用该库模块的消费者都可以看到随着提交而更新的 `AuthModule` 最新版本。这对于协调组件开发和组装以及简化端到端测试具有显著优势。

#### 创建库

任何适合重用的功能都可以作为库来管理。决定什么应该作为库、什么应该作为应用程序的一部分，是一个架构设计决策。创建库不仅仅是简单地将代码从现有应用程序复制到新库中。当代码被打包为库时，必须将其与应用程序解耦。这可能需要更多的前期时间，并迫使你做出一些在耦合更紧密的代码中不会遇到的设计决策。但当该库可以用于在多个应用程序中实现更快速的应用程序组装时，这种额外的努力是值得的。

要开始创建库，请运行以下命令：

```bash
$ nest g library my-library
```

运行该命令时，`library` 生成器会提示你为库输入一个前缀（又名别名）：

```bash
What prefix would you like to use for the library (default: @app)?
```

这将在你的工作区中创建一个名为 `my-library` 的新项目。
库类型的项目，像应用程序类型的项目一样，是通过生成器生成到一个命名的文件夹中。库在单体仓库根目录下的 `libs` 文件夹中进行管理。Nest 会在第一次创建库时自动创建 `libs` 文件夹。

为库生成的文件与为应用程序生成的文件略有不同。以下是执行上述命令后 `libs` 文件夹的内容：

<div class="file-tree">
  <div class="item">libs</div>
  <div class="children">
    <div class="item">my-library</div>
    <div class="children">
      <div class="item">src</div>
      <div class="children">
        <div class="item">index.ts</div>
        <div class="item">my-library.module.ts</div>
        <div class="item">my-library.service.ts</div>
      </div>
      <div class="item">tsconfig.lib.json</div>
    </div>
  </div>
</div>

`nest-cli.json` 文件将在 `"projects"` 键下新增一个库的条目：

```javascript
...
{
    "my-library": {
      "type": "library",
      "root": "libs/my-library",
      "entryFile": "index",
      "sourceRoot": "libs/my-library/src",
      "compilerOptions": {
        "tsConfigPath": "libs/my-library/tsconfig.lib.json"
      }
}
...
```

`nest-cli.json` 中库与应用程序的元数据有两个不同之处：

- `"type"` 属性设置为 `"library"` 而不是 `"application"`
- `"entryFile"` 属性设置为 `"index"` 而不是 `"main"`

这些差异使构建过程能够正确处理库。例如，库通过 `index.js` 文件导出其功能。

与应用程序类型的项目一样，每个库都有自己的 `tsconfig.lib.json` 文件，它继承自根目录（整个单体仓库）的 `tsconfig.json` 文件。如有必要，你可以修改此文件以提供库特定的编译器选项。

你可以使用 CLI 命令构建库：

```bash
$ nest build my-library
```

#### 使用库

在自动生成的配置文件就位后，使用库非常直接。我们如何将 `my-library` 库中的 `MyLibraryService` 导入到 `my-project` 应用程序中？

首先，请注意使用库模块与使用任何其他 Nest 模块相同。单体仓库所做的就是以一种导入库和生成构建的方式管理路径。要使用 `MyLibraryService`，我们需要导入其声明模块。我们可以按如下方式修改 `my-project/src/app.module.ts` 来导入 `MyLibraryModule`。

```typescript
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { MyLibraryModule } from '@app/my-library';

@Module({
  imports: [MyLibraryModule],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

请注意上面我们在 ES 模块的 `import` 行中使用了 `@app` 的路径别名，这正是我们通过 `nest g library` 命令提供的 `prefix`。在底层，Nest 通过 tsconfig 路径映射来处理这个问题。当添加库时，Nest 会更新全局（单体仓库）`tsconfig.json` 文件中的 `"paths"` 键，如下所示：

```javascript
"paths": {
    "@app/my-library": [
        "libs/my-library/src"
    ],
    "@app/my-library/*": [
        "libs/my-library/src/*"
    ]
}
```

简而言之，单体仓库和库功能的结合使得将库模块包含到应用程序中变得简单直观。

同样的机制也支持构建和部署包含库的应用程序。一旦你导入了 `MyLibraryModule`，运行 `nest build` 将自动处理所有模块解析，并将应用程序及其所有库依赖打包，用于部署。单体仓库的默认编译器是 **webpack**，因此生成的分发文件是一个将所有转译后的 JavaScript 文件打包成一个文件的单一文件。你也可以切换到 `tsc`，具体操作请参见 <a href="https://docs.nestjs.com/cli/monorepo#global-compiler-options">此处</a>。