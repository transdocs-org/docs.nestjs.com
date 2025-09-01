### SWC

[SWC](https://swc.rs/)（Speedy Web Compiler）是一个基于 Rust 的可扩展平台，可用于编译和打包。使用 Nest CLI 配合 SWC 是显著加快开发流程的一个简单有效的方式。

> info **提示** SWC 的速度大约是默认 TypeScript 编译器的 **20 倍**。

#### 安装

首先，安装一些必要的包：

```bash
$ npm i --save-dev @swc/cli @swc/core
```

#### 快速开始

安装完成后，你可以通过 Nest CLI 使用 `swc` 构建器，如下所示：

```bash
$ nest start -b swc
# 或者 nest start --builder swc
```

> info **提示** 如果你的仓库是一个单体仓库（monorepo），请查看[此部分](/recipes/swc#monorepo)。

除了传递 `-b` 参数，你也可以直接在 `nest-cli.json` 文件中将 `compilerOptions.builder` 属性设置为 `"swc"`，如下所示：

```json
{
  "compilerOptions": {
    "builder": "swc"
  }
}
```

要自定义构建器的行为，你可以传递一个包含 `type`（值为 `"swc"`）和 `options` 的对象，如下所示：

```json
{
  "compilerOptions": {
    "builder": {
      "type": "swc",
      "options": {
        "swcrcPath": "infrastructure/.swcrc"
      }
    }
  }
}
```

要以监听模式运行应用程序，请使用以下命令：

```bash
$ nest start -b swc -w
# 或者 nest start --builder swc --watch
```

#### 类型检查

SWC 本身不进行类型检查（与默认的 TypeScript 编译器不同），因此要启用类型检查，你需要使用 `--type-check` 标志：

```bash
$ nest start -b swc --type-check
```

该命令会指示 Nest CLI 在运行 SWC 的同时以 `noEmit` 模式运行 `tsc`，从而异步执行类型检查。同样，你也可以在 `nest-cli.json` 文件中将 `compilerOptions.typeCheck` 属性设置为 `true` 来代替 `--type-check` 标志，如下所示：

```json
{
  "compilerOptions": {
    "builder": "swc",
    "typeCheck": true
  }
}
```

#### CLI 插件（SWC）

`--type-check` 标志将自动执行 **NestJS CLI 插件** 并生成一个序列化的元数据文件，应用程序在运行时可以加载该文件。

#### SWC 配置

SWC 构建器已经预配置以满足 NestJS 应用程序的需求。不过，你可以在项目根目录创建 `.swcrc` 文件并按需调整配置选项来自定义配置。

```json
{
  "$schema": "https://swc.rs/schema.json",
  "sourceMaps": true,
  "jsc": {
    "parser": {
      "syntax": "typescript",
      "decorators": true,
      "dynamicImport": true
    },
    "baseUrl": "./"
  },
  "minify": false
}
```

#### 单体仓库（Monorepo）

如果你的仓库是单体仓库（monorepo），则不能使用 `swc` 构建器，而需要配置 `webpack` 使用 `swc-loader`。

首先，安装所需的包：

```bash
$ npm i --save-dev swc-loader
```

安装完成后，在应用程序的根目录创建一个 `webpack.config.js` 文件，并添加以下内容：

```js
const swcDefaultConfig = require('@nestjs/cli/lib/compiler/defaults/swc-defaults').swcDefaultsFactory().swcOptions;

module.exports = {
  module: {
    rules: [
      {
        test: /\.ts$/,
        exclude: /node_modules/,
        use: {
          loader: 'swc-loader',
          options: swcDefaultConfig,
        },
      },
    ],
  },
};
```

#### 单体仓库与 CLI 插件（Monorepo and CLI plugins）

现在，如果你使用了 CLI 插件，`swc-loader` 不会自动加载它们。你需要手动创建一个文件来加载这些插件。为此，在 `main.ts` 文件附近创建一个 `generate-metadata.ts` 文件，并添加以下内容：

```ts
import { PluginMetadataGenerator } from '@nestjs/cli/lib/compiler/plugins/plugin-metadata-generator';
import { ReadonlyVisitor } from '@nestjs/swagger/dist/plugin';

const generator = new PluginMetadataGenerator();
generator.generate({
  visitors: [new ReadonlyVisitor({ introspectComments: true, pathToSource: __dirname })],
  outputDir: __dirname,
  watch: true,
  tsconfigPath: 'apps/<name>/tsconfig.app.json',
});
```

> info **提示** 此示例中我们使用了 `@nestjs/swagger` 插件，但你可以使用任意你喜欢的插件。

`generate()` 方法接受以下选项：

|                    |                                                                                                |
| ------------------ | ---------------------------------------------------------------------------------------------- |
| `watch`            | 是否监听项目变化。                                                                             |
| `tsconfigPath`     | `tsconfig.json` 文件的路径，相对于当前工作目录 (`process.cwd()`)。                              |
| `outputDir`        | 元数据文件保存的目录路径。                                                                     |
| `visitors`         | 用于生成元数据的访问器数组。                                                                   |
| `filename`         | 元数据文件的名称，默认为 `metadata.ts`。                                                       |
| `printDiagnostics` | 是否将诊断信息打印到控制台，默认为 `true`。                                                    |

最后，你可以在另一个终端窗口中运行以下命令来执行 `generate-metadata` 脚本：

```bash
$ npx ts-node src/generate-metadata.ts
# 或 npx ts-node apps/{YOUR_APP}/src/generate-metadata.ts
```

#### 常见问题

如果你在项目中使用 TypeORM/MikroORM 或其他 ORM，可能会遇到循环导入问题。SWC 无法很好地处理 **循环导入**，所以你需要使用以下变通方法：

```typescript
@Entity()
export class User {
  @OneToOne(() => Profile, (profile) => profile.user)
  profile: Relation<Profile>; // <--- 注意这里使用了 "Relation<>" 类型而非直接使用 "Profile"
}
```

> info **提示** `Relation` 类型来自 `typeorm` 包。

这样可以防止属性类型被保存到转译后的代码中，从而避免循环依赖问题。

如果你的 ORM 没有提供类似的解决方法，你可以自己定义一个包装类型：

```typescript
/**
 * 包装类型，用于绕过 ESM 模块的循环依赖问题
 * 该问题由反射元数据保存属性类型引起。
 */
export type WrapperType<T> = T; // WrapperType === Relation
```

对于项目中的所有 [循环依赖注入](/fundamentals/circular-dependency)，你也需要使用上面定义的包装类型：

```typescript
@Injectable()
export class UsersService {
  constructor(
    @Inject(forwardRef(() => ProfileService))
    private readonly profileService: WrapperType<ProfileService>,
  ) {};
}
```

### Jest + SWC

要将 SWC 与 Jest 一起使用，你需要安装以下包：

```bash
$ npm i --save-dev jest @swc/core @swc/jest
```

安装完成后，根据你的配置更新 `package.json` 或 `jest.config.js` 文件内容如下：

```json
{
  "jest": {
    "transform": {
      "^.+\\.(t|j)s?$": ["@swc/jest"]
    }
  }
}
```

此外，你还需要在 `.swcrc` 文件中添加以下 `transform` 配置项：`legacyDecorator` 和 `decoratorMetadata`：

```json
{
  "$schema": "https://swc.rs/schema.json",
  "sourceMaps": true,
  "jsc": {
    "parser": {
      "syntax": "typescript",
      "decorators": true,
      "dynamicImport": true
    },
    "transform": {
      "legacyDecorator": true,
      "decoratorMetadata": true
    },
    "baseUrl": "./"
  },
  "minify": false
}
```

如果你在项目中使用了 NestJS CLI 插件，则需要手动运行 `PluginMetadataGenerator`。请转到[此部分](/recipes/swc#monorepo-and-cli-plugins)了解更多信息。

### Vitest

[Vitest](https://vitest.dev/) 是一个快速轻量的测试运行器，专为与 Vite 一起工作而设计。它提供了一个现代、快速且易于使用的测试解决方案，可与 NestJS 项目集成。

#### 安装

首先，安装所需的包：

```bash
$ npm i --save-dev vitest unplugin-swc @swc/core @vitest/coverage-v8
```

#### 配置

在项目根目录创建一个 `vitest.config.ts` 文件，并添加以下内容：

```ts
import swc from 'unplugin-swc';
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    globals: true,
    root: './',
  },
  plugins: [
    // 这是必需的，以使用 SWC 构建测试文件
    swc.vite({
      // 显式设置模块类型，避免从 `.swcrc` 配置文件继承此值
      module: { type: 'es6' },
    }),
  ],
  resolve: {
    alias: {
      // 确保 Vitest 正确解析 TypeScript 路径别名
      'src': resolve(__dirname, './src'),
    },
  },
});
```

该配置文件设置了 Vitest 的环境和根目录以及 SWC 插件。你还可以为端到端测试创建一个单独的配置文件，并添加一个 `include` 字段指定测试路径的正则表达式：

```ts
import swc from 'unplugin-swc';
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    include: ['**/*.e2e-spec.ts'],
    globals: true,
    root: './',
  },
  plugins: [swc.vite()],
});
```

此外，你可以设置 `alias` 选项以支持测试中的 TypeScript 路径：

```ts
import swc from 'unplugin-swc';
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    include: ['**/*.e2e-spec.ts'],
    globals: true,
    alias: {
      '@src': './src',
      '@test': './test',
    },
    root: './',
  },
  resolve: {
    alias: {
      '@src': './src',
      '@test': './test',
    },
  },
  plugins: [swc.vite()],
});
```

#### 路径别名（Path aliases）

与 Jest 不同，Vitest 不会自动解析类似 `src/` 的 TypeScript 路径别名。这可能导致测试期间出现依赖解析错误。为了解决这个问题，请在 `vitest.config.ts` 文件中添加以下 `resolve.alias` 配置：

```ts
import { resolve } from 'path';

export default defineConfig({
  resolve: {
    alias: {
      'src': resolve(__dirname, './src'),
    },
  },
});
```

这样可以确保 Vitest 正确解析模块导入，防止因依赖缺失导致的错误。

#### 更新 E2E 测试中的导入

将 E2E 测试中使用 `import * as request from 'supertest'` 的导入语句改为 `import request from 'supertest'`。这是因为 Vitest 与 Vite 一起打包时，期望 `supertest` 使用默认导入。在特定设置下，使用命名空间导入可能会导致问题。

最后，更新 `package.json` 中的测试脚本如下：

```json
{
  "scripts": {
    "test": "vitest run",
    "test:watch": "vitest",
    "test:cov": "vitest run --coverage",
    "test:debug": "vitest --inspect-brk --inspect --logHeapUsage --threads=false",
    "test:e2e": "vitest run --config ./vitest.config.e2e.ts"
  }
}
```

这些脚本配置了 Vitest 的运行测试、监听变化、生成覆盖率报告以及调试功能。`test:e2e` 脚本专门用于使用自定义配置文件运行端到端测试。

通过此设置，你现在可以在 NestJS 项目中享受 Vitest 带来的优势，包括更快的测试执行速度和更现代化的测试体验。

> info **提示** 你可以在[此仓库](https://github.com/TrilonIO/nest-vitest)中查看一个完整的工作示例。