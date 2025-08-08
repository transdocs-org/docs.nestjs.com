### CLI 插件

> warning **警告** 本章节仅适用于代码优先（code first）方法。

TypeScript 的元数据反射系统有几个限制，例如无法确定一个类包含哪些属性，或无法识别某个属性是可选还是必需。然而，其中一些限制可以在编译时解决。Nest 提供了一个插件，可以增强 TypeScript 的编译过程，从而减少所需的样板代码。

> info **提示** 此插件是**可选的**。如果你愿意，可以手动声明所有装饰器，或者仅在需要的地方声明特定装饰器。

#### 概述

GraphQL 插件将自动：

- 为所有输入对象、对象类型和参数类的属性添加 `@Field` 装饰器，除非使用了 `@HideField`
- 根据问号设置 `nullable` 属性（例如 `name?: string` 将设置 `nullable: true`）
- 根据类型设置 `type` 属性（支持数组）
- 如果启用了 `introspectComments`，则根据注释生成属性描述

请注意，你的文件名 **必须** 包含以下后缀之一，以便被插件分析：`['.input.ts', '.args.ts', '.entity.ts', '.model.ts']`（例如 `author.entity.ts`）。如果你使用了不同的后缀，可以通过设置 `typeFileNameSuffix` 选项来调整插件行为（见下文）。

根据我们目前所学的内容，你必须重复编写大量代码，以便让包知道你的类型在 GraphQL 中应如何声明。例如，你可以定义一个简单的 `Author` 类如下：

```typescript
@@filename(authors/models/author.model)
@ObjectType()
export class Author {
  @Field(type => ID)
  id: number;

  @Field({ nullable: true })
  firstName?: string;

  @Field({ nullable: true })
  lastName?: string;

  @Field(type => [Post])
  posts: Post[];
}
```

虽然在中等规模的项目中问题不大，但一旦拥有大量类，这种方式就会变得冗长且难以维护。

启用 GraphQL 插件后，上述类定义可以简化为：

```typescript
@@filename(authors/models/author.model)
@ObjectType()
export class Author {
  @Field(type => ID)
  id: number;
  firstName?: string;
  lastName?: string;
  posts: Post[];
}
```

插件会根据 **抽象语法树（Abstract Syntax Tree）** 动态添加适当的装饰器。因此，你无需再在代码中到处使用 `@Field` 装饰器。

> info **提示** 插件会自动生成缺失的 GraphQL 属性，如果需要覆盖它们，只需通过 `@Field()` 显式设置即可。

#### 注释分析

启用注释分析功能后，CLI 插件将基于注释为字段生成描述。

例如，给定一个 `roles` 属性：

```typescript
/**
 * A list of user's roles
 */
@Field(() => [String], {
  description: `A list of user's roles`
})
roles: string[];
```

你必须重复书写描述内容。启用 `introspectComments` 后，CLI 插件可以提取这些注释并自动为属性提供描述。现在，上述字段可以简化为：

```typescript
/**
 * A list of user's roles
 */
roles: string[];
```

#### 使用 CLI 插件

要启用插件，请打开 `nest-cli.json` 文件（如果你使用的是 [Nest CLI](/cli/overview)），并添加以下 `plugins` 配置：

```javascript
{
  "collection": "@nestjs/schematics",
  "sourceRoot": "src",
  "compilerOptions": {
    "plugins": ["@nestjs/graphql"]
  }
}
```

你可以使用 `options` 属性来自定义插件行为。

```javascript
{
  "collection": "@nestjs/schematics",
  "sourceRoot": "src",
  "compilerOptions": {
    "plugins": [
      {
        "name": "@nestjs/graphql",
        "options": {
          "typeFileNameSuffix": [".input.ts", ".args.ts"],
          "introspectComments": true
        }
      }
    ]
  }
}
```

`options` 属性必须符合以下接口：

```typescript
export interface PluginOptions {
  typeFileNameSuffix?: string[];
  introspectComments?: boolean;
}
```

<table>
  <tr>
    <th>选项</th>
    <th>默认值</th>
    <th>描述</th>
  </tr>
  <tr>
    <td><code>typeFileNameSuffix</code></td>
    <td><code>['.input.ts', '.args.ts', '.entity.ts', '.model.ts']</code></td>
    <td>GraphQL 类型文件的后缀</td>
  </tr>
  <tr>
    <td><code>introspectComments</code></td>
    <td><code>false</code></td>
    <td>如果为 true，插件将根据注释生成属性描述</td>
  </tr>
</table>

如果你不使用 CLI，而是使用自定义的 `webpack` 配置，可以将此插件与 `ts-loader` 结合使用：

```javascript
getCustomTransformers: (program: any) => ({
  before: [require('@nestjs/graphql/plugin').before({}, program)]
}),
```

#### SWC 构建器

对于标准设置（非 monorepo），若要在 SWC 构建器中使用 CLI 插件，你需要启用类型检查，如 [此处](/recipes/swc#type-checking) 所述。

```bash
$ nest start -b swc --type-check
```

对于 monorepo 设置，请参考 [此处](/recipes/swc#monorepo-and-cli-plugins) 的说明。

```bash
$ npx ts-node src/generate-metadata.ts
# 或 npx ts-node apps/{YOUR_APP}/src/generate-metadata.ts
```

现在，必须由 `GraphQLModule` 方法加载生成的元数据文件，如下所示：

```typescript
import metadata from './metadata'; // <-- 由 "PluginMetadataGenerator" 自动生成的文件

GraphQLModule.forRoot<...>({
  ..., // 其他选项
  metadata,
}),
```

#### 与 `ts-jest` 的集成（端到端测试）

当启用此插件运行端到端测试时，你可能会遇到编译 schema 的问题。例如，最常见的错误之一是：

```json
Object type <name> must define one or more fields.
```

这是因为 `jest` 的配置中没有引入 `@nestjs/graphql/plugin` 插件。

要解决此问题，请在你的 e2e 测试目录中创建以下文件：

```javascript
const transformer = require('@nestjs/graphql/plugin');

module.exports.name = 'nestjs-graphql-transformer';
// 每次修改以下配置时都应更改版本号，否则 jest 将无法检测到更改
module.exports.version = 1;

module.exports.factory = (cs) => {
  return transformer.before(
    {
      // @nestjs/graphql/plugin 的选项（可以为空）
    },
    cs.program, // 对于旧版本的 Jest（<= v27），使用 cs.tsCompiler.program
  );
};
```

创建该文件后，在你的 `jest` 配置文件中导入 AST 转换器。默认情况下（在起始项目中），e2e 测试的配置文件位于 `test` 文件夹下，名为 `jest-e2e.json`。

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

如果你使用的是 `jest@^29`，请使用以下代码片段，因为之前的用法已被弃用。

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