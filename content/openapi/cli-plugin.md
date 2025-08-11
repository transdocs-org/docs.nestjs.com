### CLI 插件

[TypeScript](https://www.typescriptlang.org/docs/handbook/decorators.html) 的元数据反射系统存在一些限制，使得我们无法判断类中包含哪些属性，或者某个属性是必需的还是可选的。然而，其中一些限制可以在编译时解决。Nest 提供了一个插件，通过增强 TypeScript 的编译过程来减少所需的样板代码。

> info **提示** 此插件是**可选的**。如果您愿意，也可以手动声明所有装饰器，或仅在需要的地方声明特定装饰器。

#### 概述

Swagger 插件将自动：

- 为所有 DTO 属性添加 `@ApiProperty` 装饰器，除非使用了 `@ApiHideProperty`
- 根据问号判断 `required` 属性（例如 `name?: string` 将设置 `required: false`）
- 根据类型设置 `type` 或 `enum` 属性（支持数组）
- 根据默认值设置 `default` 属性
- 根据 `class-validator` 装饰器设置若干验证规则（如果 `classValidatorShim` 设置为 `true`）
- 为每个端点添加响应装饰器，并指定正确的状态码和响应模型 (`type`)
- 根据注释生成属性和端点的描述（如果 `introspectComments` 设置为 `true`）
- 根据注释生成属性的示例值（如果 `introspectComments` 设置为 `true`）

请注意，您的文件名**必须包含**以下后缀之一：`['.dto.ts', '.entity.ts']`（例如 `create-user.dto.ts`），插件才会对其进行分析。

如果您使用了不同的后缀，可以通过设置 `dtoFileNameSuffix` 选项来自定义插件行为（见下文）。

以前，如果您希望在 Swagger UI 中提供交互体验，您需要重复编写大量代码来告诉包如何在规范中声明您的模型/组件。例如，您可以如下定义一个简单的 `CreateUserDto` 类：

```typescript
export class CreateUserDto {
  @ApiProperty()
  email: string;

  @ApiProperty()
  password: string;

  @ApiProperty({ enum: RoleEnum, default: [], isArray: true })
  roles: RoleEnum[] = [];

  @ApiProperty({ required: false, default: true })
  isEnabled?: boolean = true;
}
```

虽然在中型项目中这并不是一个严重问题，但一旦项目规模扩大，这种写法会变得冗长且难以维护。

通过[启用 Swagger 插件](/openapi/cli-plugin#using-the-cli-plugin)，上述类定义可以简化为：

```typescript
export class CreateUserDto {
  email: string;
  password: string;
  roles: RoleEnum[] = [];
  isEnabled?: boolean = true;
}
```

> info **注意** Swagger 插件会根据 TypeScript 类型和 `class-validator` 装饰器自动推导出 `@ApiProperty()` 注解，从而帮助您更清晰地描述 API。然而，运行时的验证仍然由 `class-validator` 装饰器处理。因此，仍然需要使用 `IsEmail()`、`IsNumber()` 等验证器。

因此，如果您希望依赖自动注解生成文档，同时又希望进行运行时验证，则 `class-validator` 装饰器仍然是必需的。

> info **提示** 当在 DTO 中使用[映射类型工具](https://docs.nestjs.com/openapi/mapped-types)（如 `PartialType`）时，请从 `@nestjs/swagger` 而不是 `@nestjs/mapped-types` 导入它们，以便插件能够识别模式。

插件基于**抽象语法树（AST）**动态添加适当的装饰器，因此您无需在代码中手动添加 `@ApiProperty` 装饰器。

> info **提示** 插件将自动生成缺失的 Swagger 属性，但如果您需要覆盖它们，只需通过 `@ApiProperty()` 显式设置即可。

#### 注释解析

启用注释解析功能后，CLI 插件将根据注释生成属性的描述和示例值。

例如，以下 `roles` 属性：

```typescript
/**
 * A list of user's roles
 * @example ['admin']
 */
@ApiProperty({
  description: `A list of user's roles`,
  example: ['admin'],
})
roles: RoleEnum[] = [];
```

您必须重复编写描述和示例值。启用 `introspectComments` 后，CLI 插件可以从注释中提取这些信息，并自动为属性生成描述（和示例，如果定义了的话）。现在，上述属性可以简化为：

```typescript
/**
 * A list of user's roles
 * @example ['admin']
 */
roles: RoleEnum[] = [];
```

插件提供了 `dtoKeyOfComment` 和 `controllerKeyOfComment` 选项，用于自定义如何为 `ApiProperty` 和 `ApiOperation` 装饰器赋值。请参见以下示例：

```typescript
export class SomeController {
  /**
   * Create some resource
   */
  @Post()
  create() {}
}
```

这等价于：

```typescript
@ApiOperation({ summary: "Create some resource" })
```

> info **提示** 对于模型，同样的逻辑适用，但使用的是 `ApiProperty` 装饰器。

对于控制器，您不仅可以提供摘要，还可以提供描述（备注）、标签（如 `@deprecated`）和响应示例，如下所示：

```ts
/**
 * Create a new cat
 *
 * @remarks This operation allows you to create a new cat.
 *
 * @deprecated
 * @throws {500} Something went wrong.
 * @throws {400} Bad Request.
 */
@Post()
async create(): Promise<Cat> {}
```

#### 使用 CLI 插件

要启用插件，请打开 `nest-cli.json`（如果您使用的是 [Nest CLI](/cli/overview)），并添加以下 `plugins` 配置：

```javascript
{
  "collection": "@nestjs/schematics",
  "sourceRoot": "src",
  "compilerOptions": {
    "plugins": ["@nestjs/swagger"]
  }
}
```

您可以通过 `options` 属性来自定义插件行为。

```javascript
{
  "collection": "@nestjs/schematics",
  "sourceRoot": "src",
  "compilerOptions": {
    "plugins": [
      {
        "name": "@nestjs/swagger",
        "options": {
          "classValidatorShim": false,
          "introspectComments": true,
          "skipAutoHttpCode": true
        }
      }
    ]
  }
}
```

`options` 属性需满足以下接口：

```typescript
export interface PluginOptions {
  dtoFileNameSuffix?: string[];
  controllerFileNameSuffix?: string[];
  classValidatorShim?: boolean;
  dtoKeyOfComment?: string;
  controllerKeyOfComment?: string;
  introspectComments?: boolean;
  skipAutoHttpCode?: boolean;
  esmCompatible?: boolean;
}
```

<table>
  <tr>
    <th>选项</th>
    <th>默认值</th>
    <th>描述</th>
  </tr>
  <tr>
    <td><code>dtoFileNameSuffix</code></td>
    <td><code>['.dto.ts', '.entity.ts']</code></td>
    <td>DTO（数据传输对象）文件的后缀</td>
  </tr>
  <tr>
    <td><code>controllerFileNameSuffix</code></td>
    <td><code>.controller.ts</code></td>
    <td>控制器文件的后缀</td>
  </tr>
  <tr>
    <td><code>classValidatorShim</code></td>
    <td><code>true</code></td>
    <td>如果为 true，模块将复用 <code>class-validator</code> 的验证装饰器（例如 <code>@Max(10)</code> 会在 schema 定义中添加 <code>max: 10</code>）</td>
  </tr>
  <tr>
    <td><code>dtoKeyOfComment</code></td>
    <td><code>'description'</code></td>
    <td>设置 <code>ApiProperty</code> 上的注释文本对应的属性键</td>
  </tr>
  <tr>
    <td><code>controllerKeyOfComment</code></td>
    <td><code>'summary'</code></td>
    <td>设置 <code>ApiOperation</code> 上的注释文本对应的属性键</td>
  </tr>
  <tr>
    <td><code>introspectComments</code></td>
    <td><code>false</code></td>
    <td>如果为 true，插件将根据注释生成属性的描述和示例值</td>
  </tr>
  <tr>
    <td><code>skipAutoHttpCode</code></td>
    <td><code>false</code></td>
    <td>禁用在控制器中自动添加 <code>@HttpCode()</code></td>
  </tr>
  <tr>
    <td><code>esmCompatible</code></td>
    <td><code>false</code></td>
    <td>如果为 true，解决使用 ESM（<code>&#123; "type": "module" &#125;</code>）时遇到的语法错误</td>
  </tr>
</table>

每次更新插件选项后，请确保删除 `/dist` 文件夹并重新构建您的应用。

如果您未使用 CLI，而是使用了自定义的 `webpack` 配置，可以将此插件与 `ts-loader` 一起使用：

```javascript
getCustomTransformers: (program: any) => ({
  before: [require('@nestjs/swagger/plugin').before({}, program)]
}),
```

#### SWC 构建器

对于标准项目（非 monorepo），若要在使用 SWC 构建器时使用 CLI 插件，您需要启用类型检查，如[此处](/recipes/swc#type-checking)所述：

```bash
$ nest start -b swc --type-check
```

对于 monorepo 项目，请参考[此处](/recipes/swc#monorepo-and-cli-plugins)的说明：

```bash
$ npx ts-node src/generate-metadata.ts
# 或 npx ts-node apps/{YOUR_APP}/src/generate-metadata.ts
```

现在，必须通过 `SwaggerModule#loadPluginMetadata` 方法加载序列化的元数据文件，如下所示：

```typescript
import metadata from './metadata'; // <-- 由 "PluginMetadataGenerator" 自动生成的文件

await SwaggerModule.loadPluginMetadata(metadata); // <-- 此处
const document = SwaggerModule.createDocument(app, config);
```

#### 与 `ts-jest` 集成（端到端测试）

在运行端到端测试时，`ts-jest` 会在内存中即时编译源代码文件。这意味着它不会使用 Nest CLI 的编译器，也不会应用任何插件或 AST 转换。

要启用插件，请在您的 e2e 测试目录中创建以下文件：

```javascript
const transformer = require('@nestjs/swagger/plugin');

module.exports.name = 'nestjs-swagger-transformer';
// 每次更改以下配置时都应该修改版本号，否则 jest 将无法检测到更改
module.exports.version = 1;

module.exports.factory = (cs) => {
  return transformer.before(
    {
      // @nestjs/swagger/plugin 的选项（可以留空）
    },
    cs.program, // 对于旧版 Jest（<= v27），使用 "cs.tsCompiler.program"
  );
};
```

完成后，在您的 `jest` 配置文件中导入 AST 转换器。默认情况下（在启动项目中），e2e 测试的配置文件位于 `test` 文件夹下，名为 `jest-e2e.json`。

如果您使用的是 `jest@<29`，请使用以下配置：

```json
{
  ... // 其他配置
  "globals": {
    "ts-jest": {
      "astTransformers": {
        "before": ["<path to the file created above>"]
      }
    }
  }
}
```

如果您使用的是 `jest@^29`，请使用以下配置（旧方式已被弃用）：

```json
{
  ... // 其他配置
  "transform": {
    "^.+\\.(t|j)s$": [
      "ts-jest",
      {
        "astTransformers": {
          "before": ["<path to the file created above>"]
        }
      }
    ]
  }
}
```

#### `jest` 故障排查（端到端测试）

如果 `jest` 似乎未应用您的配置更改，可能是 Jest 已经**缓存**了构建结果。要应用新的配置，您需要清除 Jest 的缓存目录。

要清除缓存目录，请在 NestJS 项目文件夹中运行以下命令：

```bash
$ npx jest --clearCache
```

如果自动清除缓存失败，您也可以手动删除缓存文件夹：

```bash
# 查找 jest 缓存目录（通常是 /tmp/jest_rs）
# 在 NestJS 项目根目录下运行以下命令
$ npx jest --showConfig | grep cache
# 示例输出：
#   "cache": true,
#   "cacheDirectory": "/tmp/jest_rs"

# 删除或清空 Jest 缓存目录
$ rm -rf  <cacheDirectory value>
# 示例：
# rm -rf /tmp/jest_rs
```