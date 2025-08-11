### HTTP 适配器

有时，你可能希望在 Nest 应用上下文内部或外部访问底层的 HTTP 服务器。

每个原生（平台相关）HTTP 服务器/库（例如 Express 和 Fastify）实例都被封装在一个 **适配器（adapter）** 中。该适配器被注册为一个全局可用的提供者，可以从应用上下文中获取，也可以注入到其他提供者中。

#### 外部应用上下文策略

要从应用上下文外部获取 `HttpAdapter` 的引用，请调用 `getHttpAdapter()` 方法。

```typescript
@@filename()
const app = await NestFactory.create(AppModule);
const httpAdapter = app.getHttpAdapter();
```

#### 作为可注入对象

要从应用上下文内部获取 `HttpAdapterHost` 的引用，请使用与其他现有提供者相同的注入技术（例如，使用构造函数注入）。

```typescript
@@filename()
export class CatsService {
  constructor(private adapterHost: HttpAdapterHost) {}
}
@@switch
@Dependencies(HttpAdapterHost)
export class CatsService {
  constructor(adapterHost) {
    this.adapterHost = adapterHost;
  }
}
```

> info **提示** `HttpAdapterHost` 是从 `@nestjs/core` 包中导入的。

`HttpAdapterHost` **不是** 实际的 `HttpAdapter`。要获取实际的 `HttpAdapter` 实例，请访问 `httpAdapter` 属性。

```typescript
const adapterHost = app.get(HttpAdapterHost);
const httpAdapter = adapterHost.httpAdapter;
```

`httpAdapter` 是由底层框架使用的 HTTP 适配器的实际实例。它可能是 `ExpressAdapter` 或 `FastifyAdapter` 的实例（这两个类都继承自 `AbstractHttpAdapter`）。

适配器对象暴露了若干有用的方法来与 HTTP 服务器进行交互。然而，如果你希望直接访问库的实例（例如 Express 实例），请调用 `getInstance()` 方法。

```typescript
const instance = httpAdapter.getInstance();
```

#### 监听事件

当服务器开始监听传入请求时，如果你想执行某个操作，可以订阅 `listen$` 流，如下所示：

```typescript
this.httpAdapterHost.listen$.subscribe(() =>
  console.log('HTTP server is listening'),
);
```

此外，`HttpAdapterHost` 还提供了一个 `listening` 布尔属性，用于指示服务器是否正在运行并监听中：

```typescript
if (this.httpAdapterHost.listening) {
  console.log('HTTP server is listening');
}
```