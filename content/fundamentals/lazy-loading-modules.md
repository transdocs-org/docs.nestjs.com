### 模块的懒加载

默认情况下，模块是**立即加载**（eagerly loaded）的，这意味着一旦应用程序加载，所有模块都会随之加载，无论它们是否立即需要。对于大多数应用程序来说，这没有问题，但对于运行在**无服务器环境**（serverless environment）中的应用程序或 Worker 来说，这可能会成为瓶颈，因为启动延迟（“冷启动”）非常关键。

懒加载可以通过仅加载特定无服务器函数调用所需的模块来减少初始化时间。此外，你还可以在无服务器函数“预热”后异步加载其他模块，以进一步加快后续调用的初始化时间（延迟模块注册）。

> info **提示** 如果你熟悉 **[Angular](https://angular.dev/)** 框架，你可能以前见过“[懒加载模块](https://angular.dev/guide/ngmodules/lazy-loading#lazy-loading-basics)”这个术语。请注意，这种技术在 Nest 中是**功能不同的**，因此请将其视为一个完全不同的功能，只是共享了类似的命名约定。

> warning **警告** 请注意，懒加载模块和其服务中的 [生命周期钩子方法](https://docs.nestjs.com/fundamentals/lifecycle-events) 不会被调用。

#### 快速开始

为了按需加载模块，Nest 提供了 `LazyModuleLoader` 类，可以通过常规方式注入到类中：

```typescript
@@filename(cats.service)
@Injectable()
export class CatsService {
  constructor(private lazyModuleLoader: LazyModuleLoader) {}
}
@@switch
@Injectable()
@Dependencies(LazyModuleLoader)
export class CatsService {
  constructor(lazyModuleLoader) {
    this.lazyModuleLoader = lazyModuleLoader;
  }
}
```

> info **提示** `LazyModuleLoader` 类从 `@nestjs/core` 包中导入。

或者，你也可以在应用程序引导文件（`main.ts`）中获取 `LazyModuleLoader` 提供者的引用，如下所示：

```typescript
// "app" 表示一个 Nest 应用程序实例
const lazyModuleLoader = app.get(LazyModuleLoader);
```

有了这个，你现在可以使用以下结构加载任何模块：

```typescript
const { LazyModule } = await import('./lazy.module');
const moduleRef = await this.lazyModuleLoader.load(() => LazyModule);
```

> info **提示** “懒加载”的模块在首次调用 `LazyModuleLoader#load` 方法时会被**缓存**。这意味着，每次后续尝试加载 `LazyModule` 都会**非常快**，并返回缓存的实例，而不是重新加载模块。
>
> ```bash
> 加载 "LazyModule" 第 1 次
> time: 2.379ms
> 加载 "LazyModule" 第 2 次
> time: 0.294ms
> 加载 "LazyModule" 第 3 次
> time: 0.303ms
> ```
>
> 此外，“懒加载”的模块与应用程序启动时立即加载的模块以及稍后注册的其他懒加载模块共享相同的模块图。

其中 `lazy.module.ts` 是导出一个**普通 Nest 模块**的 TypeScript 文件（无需额外更改）。

`LazyModuleLoader#load` 方法返回 [模块引用](/fundamentals/module-ref)（即 `LazyModule` 的模块引用），它允许你遍历内部提供者列表，并使用其注入令牌作为查找键来获取任意提供者的引用。

例如，假设我们有一个 `LazyModule`，其定义如下：

```typescript
@Module({
  providers: [LazyService],
  exports: [LazyService],
})
export class LazyModule {}
```

> info **提示** 懒加载模块不能注册为**全局模块**，因为这没有意义（因为它们是按需懒加载的，而此时所有静态注册的模块已经实例化）。同样，已注册的**全局增强器**（守卫/拦截器等）**也不会正常工作**。

有了这个，我们可以获得 `LazyService` 提供者的引用，如下所示：

```typescript
const { LazyModule } = await import('./lazy.module');
const moduleRef = await this.lazyModuleLoader.load(() => LazyModule);

const { LazyService } = await import('./lazy.service');
const lazyService = moduleRef.get(LazyService);
```

> warning **警告** 如果你使用的是 **Webpack**，请确保更新你的 `tsconfig.json` 文件 —— 将 `compilerOptions.module` 设置为 `"esnext"` 并添加 `compilerOptions.moduleResolution` 属性，值为 `"node"`：
>
> ```json
> {
>   "compilerOptions": {
>     "module": "esnext",
>     "moduleResolution": "node",
>     ...
>   }
> }
> ```
>
> 设置好这些选项后，你可以利用 Webpack 的 [代码分割](https://webpack.js.org/guides/code-splitting/) 功能。

#### 懒加载控制器、网关和解析器

由于 Nest 中的控制器（或 GraphQL 应用程序中的解析器）代表一组路由/路径/主题（或查询/变更），你**无法**使用 `LazyModuleLoader` 类来懒加载它们。

> error **警告** 在懒加载模块中注册的控制器、[解析器](/graphql/resolvers) 和 [网关](/websockets/gateways) 将不会按预期工作。同样，你不能按需注册中间件函数（通过实现 `MiddlewareConsumer` 接口）。

例如，假设你正在构建一个 REST API（HTTP 应用程序），底层使用 Fastify 驱动（使用 `@nestjs/platform-fastify` 包）。Fastify 不允许在应用程序准备好/成功监听消息之后注册路由。这意味着，即使我们分析了模块控制器中注册的路由映射，所有懒加载的路由也无法访问，因为没有方法在运行时注册它们。

同样，我们作为 `@nestjs/microservices` 包的一部分提供的某些传输策略（包括 Kafka、gRPC 或 RabbitMQ）要求在建立连接之前订阅/监听特定主题/频道。一旦你的应用程序开始监听消息，框架将无法订阅/监听新的主题。

最后，启用代码优先方法的 `@nestjs/graphql` 包会根据元数据动态生成 GraphQL schema。这意味着它要求所有类在之前就已加载。否则，将无法生成正确有效的 schema。

#### 常见使用场景

最常见的场景是，当你希望根据输入参数（路由路径、日期、查询参数等）触发不同的服务（不同逻辑）时，你的 Worker、定时任务、Lambda 或无服务器函数、Webhook 中会使用懒加载模块。另一方面，对于单体应用程序来说，懒加载模块可能没有太多意义，因为启动时间通常无关紧要。