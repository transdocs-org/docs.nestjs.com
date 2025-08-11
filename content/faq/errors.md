### 常见错误

在使用 NestJS 开发过程中，您在学习框架时可能会遇到各种错误。

#### “Cannot resolve dependency” 错误

> info **提示** 请查看 [NestJS Devtools](/devtools/overview#investigating-the-cannot-resolve-dependency-error)，它可以帮助您轻松解决“Cannot resolve dependency”错误。

最常见的错误信息之一是 Nest 无法解析提供者（provider）的依赖关系。错误信息通常如下所示：

```bash
Nest can't resolve dependencies of the <provider> (?). Please make sure that the argument <unknown_token> at index [<index>] is available in the <module> context.

Potential solutions:
- Is <module> a valid NestJS module?
- If <unknown_token> is a provider, is it part of the current <module>?
- If <unknown_token> is exported from a separate @Module, is that module imported within <module>?
  @Module({
    imports: [ /* the Module containing <unknown_token> */ ]
  })
```

导致此错误最常见的原因是模块的 `providers` 数组中没有 `<provider>`。请确保该提供者确实位于 `providers` 数组中，并遵循 [标准的 NestJS 提供者实践](/fundamentals/custom-providers#di-fundamentals)。

还有一些常见陷阱需要注意。例如，将提供者错误地放在 `imports` 数组中。在这种情况下，错误信息中的 `<module>` 位置将显示提供者的名称。

如果您在开发过程中遇到此错误，请查看错误信息中提到的模块，并检查其 `providers`。对于 `providers` 数组中的每个提供者，请确保该模块能够访问所有依赖项。通常，`providers` 在“功能模块（Feature Module）”和“根模块（Root Module）”中重复定义，这意味着 Nest 将尝试实例化两次提供者。更合理的做法是将包含 `<provider>` 的模块添加到“根模块”的 `imports` 数组中。

如果上述 `<unknown_token>` 是 `dependency`，则可能表示存在文件间的循环导入。这与下面的 [循环依赖](/faq/common-errors#circular-dependency-error) 不同，因为这种情况下只是两个文件相互导入，而不是构造函数中相互依赖。一个常见的情况是：模块文件声明了一个 token 并导入了一个提供者，而该提供者又从模块文件导入了该 token 常量。如果您使用了 barrel 文件，请确保这些导入不会导致循环导入。

如果上述 `<unknown_token>` 是 `Object`，则表示您正在使用类型/接口进行注入，但没有提供正确的提供者 token。要解决此问题，请确保：

1. 您导入的是类引用，或者使用 `@Inject()` 装饰器配合自定义 token。请参阅 [自定义提供者页面](/fundamentals/custom-providers)，以及
2. 对于基于类的提供者，请使用常规 `import` 导入具体类，而不是仅使用 `import type ...` 语法导入类型。[TypeScript 3.8+ 支持](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-8.html#type-only-imports-and-export)

此外，请确保没有在提供者自身中注入自身，因为 NestJS 不允许自我注入。当发生这种情况时，`<unknown_token>` 很可能等于 `<provider>`。

<app-banner-devtools></app-banner-devtools>

如果您使用的是 **单体仓库（monorepo）**，您可能会遇到与上述相同的错误，但 `<unknown_token>` 是名为 `ModuleRef` 的核心提供者：

```bash
Nest can't resolve dependencies of the <provider> (?).
Please make sure that the argument ModuleRef at index [<index>] is available in the <module> context.
...
```

这通常是因为您的项目加载了两个 `@nestjs/core` 模块，例如：

```text
.
├── package.json
├── apps
│   └── api
│       └── node_modules
│           └── @nestjs/bull
│               └── node_modules
│                   └── @nestjs/core
└── node_modules
    ├── (其他包)
    └── @nestjs/core
```

解决方案：

- 对于 **Yarn** Workspaces，使用 [nohoist 功能](https://classic.yarnpkg.com/blog/2018/02/15/nohoist) 防止 `@nestjs/core` 被提升。
- 对于 **pnpm** Workspaces，在其他模块中将 `@nestjs/core` 设置为 peerDependencies，并在应用的 package.json 中设置 `"dependenciesMeta": {{ '{' }}"other-module-name": {{ '{' }}"injected": true &#125;&#125;`。详见：[dependenciesmetainjected](https://pnpm.io/package_json#dependenciesmetainjected)

#### “Circular dependency” 错误

有时您会发现应用程序中难以避免 [循环依赖](https://docs.nestjs.com/fundamentals/circular-dependency)。您需要采取一些措施来帮助 Nest 解决这些问题。循环依赖引发的错误看起来如下：

```bash
Nest cannot create the <module> instance.
The module at index [<index>] of the <module> "imports" array is undefined.

Potential causes:
- A circular dependency between modules. Use forwardRef() to avoid it. Read more: https://docs.nestjs.com/fundamentals/circular-dependency
- The module at index [<index>] is of type "undefined". Check your import statements and the type of the module.

Scope [<module_import_chain>]
# 示例链: AppModule -> FooModule
```

循环依赖可能来自提供者之间相互依赖，也可能来自 TypeScript 文件之间因常量（如模块文件导出的常量）而相互依赖。在后一种情况下，建议为常量创建单独的文件。在前一种情况下，请参考循环依赖指南，并确保模块 **和** 提供者都使用 `forwardRef` 标记。

#### 调试依赖错误

除了手动验证依赖项是否正确外，从 Nest 8.1.0 开始，您可以将环境变量 `NEST_DEBUG` 设置为一个解析为真值的字符串，在 Nest 解析应用程序所有依赖项时获得额外的日志信息。

<figure><img src="/assets/injector_logs.png" /></figure>

在上图中，黄色字符串是正在注入的宿主类，蓝色字符串是注入依赖项的名称或注入 token，紫色字符串是正在搜索依赖项的模块。通过这些信息，您可以通常追踪依赖解析过程，了解问题出在哪里以及为何出现依赖注入问题。

#### “File change detected” 无限循环

使用 TypeScript 4.9 及以上版本的 Windows 用户可能会遇到此问题。
当您尝试在监视模式下运行应用程序时（例如 `npm run start:dev`），会看到日志消息无限循环：

```bash
XX:XX:XX AM - File change detected. Starting incremental compilation...
XX:XX:XX AM - Found 0 errors. Watching for file changes.
```

当您使用 NestJS CLI 以监视模式启动应用程序时，实际上是通过调用 `tsc --watch` 实现的。从 TypeScript 4.9 开始，[检测文件更改的新策略](https://devblogs.microsoft.com/typescript/announcing-typescript-4-9/#file-watching-now-uses-file-system-events) 被采用，这可能是问题的根源。

要解决此问题，需要在 `tsconfig.json` 文件中 `compilerOptions` 后添加如下设置：

```json
  "watchOptions": {
    "watchFile": "fixedPollingInterval"
  }
```

这将告诉 TypeScript 使用轮询方式检查文件更改，而不是使用新的默认方式（文件系统事件），后者在某些机器上可能导致问题。
您可以在 [TypeScript 文档](https://www.typescriptlang.org/tsconfig#watch-watchDirectory) 中阅读更多关于 `"watchFile"` 选项的信息。