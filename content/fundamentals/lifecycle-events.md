### 生命周期事件

一个 Nest 应用程序，以及每一个应用程序元素，都具有由 Nest 管理的生命周期。Nest 提供了 **生命周期钩子**，可以让你了解关键的生命周期事件，并在事件发生时执行相应的代码（例如在你的模块、提供者或控制器中运行注册的代码）。

#### 生命周期顺序

下图展示了从应用程序启动到 Node 进程退出期间的关键应用程序生命周期事件。我们可以将整个生命周期分为三个阶段：**初始化**、**运行中** 和 **终止中**。通过这个生命周期，你可以在模块和服务初始化时进行规划，管理活动连接，并在应用程序收到终止信号时优雅地关闭应用。

<figure><img class="illustrative-image" src="/assets/lifecycle-events.png" /></figure>

#### 生命周期事件

生命周期事件发生在应用程序启动和关闭过程中。在每个以下生命周期事件发生时，Nest 会在模块、提供者和控制器中调用已注册的生命周期钩子方法（**关闭钩子** 需要先启用，具体见[下文](#application-shutdown)）。如上图所示，Nest 也会调用适当的底层方法来开始监听连接和停止监听连接。

在下表中，`onModuleInit` 和 `onApplicationBootstrap` 仅在你显式调用 `app.init()` 或 `app.listen()` 时才会被触发。

在下表中，`onModuleDestroy`、`beforeApplicationShutdown` 和 `onApplicationShutdown` 仅在你显式调用 `app.close()` 或进程收到特殊系统信号（如 `SIGTERM`）并且你在应用程序启动时正确调用了 `enableShutdownHooks` 时才会被触发（见下文 **应用程序关闭** 部分）。

| 生命周期钩子方法              | 触发钩子方法调用的生命周期事件                                                                                                                                                                   |
| ---------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `onModuleInit()`             | 当宿主模块的依赖项已解析后调用。                                                                                                                                                                 |
| `onApplicationBootstrap()`   | 当所有模块初始化完成后、开始监听连接之前调用。                                                                                                                                                   |
| `onModuleDestroy()`\*        | 收到终止信号（例如 `SIGTERM`）后调用。                                                                                                                                                           |
| `beforeApplicationShutdown()`\* | 当所有 `onModuleDestroy()` 处理程序完成后（Promise 已解决或拒绝）；完成后（Promise 已解决或拒绝），所有现有连接将被关闭（调用 `app.close()`）。 |
| `onApplicationShutdown()`\*  | 在连接关闭后（`app.close()` 解决）调用。                                                                                                                                                         |

\* 对于这些事件，如果你没有显式调用 `app.close()`，则必须选择启用以使其响应系统信号（如 `SIGTERM`）。参见下文的 [应用程序关闭](#application-shutdown)。

> warning **警告** 上述生命周期钩子不会触发 **请求作用域** 类。请求作用域类不绑定到应用程序生命周期，其生命周期不可预测。它们只为每个请求创建，并在响应发送后自动被垃圾回收。

> info **提示** `onModuleInit()` 和 `onApplicationBootstrap()` 的执行顺序直接取决于模块导入的顺序，并会等待前一个钩子完成。

#### 使用方式

每个生命周期钩子都由一个接口表示。从技术上讲，接口是可选的，因为它们在 TypeScript 编译后就不存在了。不过，建议使用接口以获得强类型支持和编辑器工具提示。要注册一个生命周期钩子，只需实现相应的接口。例如，要在某个类（例如 Controller、Provider 或 Module）的模块初始化阶段注册一个方法，请实现 `OnModuleInit` 接口并提供 `onModuleInit()` 方法，如下所示：

```typescript
@@filename()
import { Injectable, OnModuleInit } from '@nestjs/common';

@Injectable()
export class UsersService implements OnModuleInit {
  onModuleInit() {
    console.log(`模块已初始化。`);
  }
}
@@switch
import { Injectable } from '@nestjs/common';

@Injectable()
export class UsersService {
  onModuleInit() {
    console.log(`模块已初始化。`);
  }
}
```

#### 异步初始化

`OnModuleInit` 和 `OnApplicationBootstrap` 钩子都允许你延迟应用程序的初始化过程（返回一个 `Promise`，或在方法体内使用 `async` 和 `await` 来等待异步方法完成）。

```typescript
@@filename()
async onModuleInit(): Promise<void> {
  await this.fetch();
}
@@switch
async onModuleInit() {
  await this.fetch();
}
```

#### 应用程序关闭

当处于终止阶段时（即显式调用 `app.close()` 或启用后收到系统信号如 SIGTERM 时），会调用 `onModuleDestroy()`、`beforeApplicationShutdown()` 和 `onApplicationShutdown()` 钩子。此功能通常与 [Kubernetes](https://kubernetes.io/) 一起用于管理容器生命周期，或者与 [Heroku](https://www.heroku.com/) 的 dynos 或类似服务一起使用。

关闭钩子监听器会消耗系统资源，因此默认情况下是禁用的。要使用关闭钩子，你**必须通过调用 `enableShutdownHooks()` 启用监听器**：

```typescript
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // 开始监听关闭钩子
  app.enableShutdownHooks();

  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```

> warning **警告** 由于 Windows 平台本身的限制，NestJS 对应用程序关闭钩子的支持有限。你可以预期 `SIGINT` 有效，`SIGBREAK` 以及一定程度上的 `SIGHUP` 也有效 —— [了解更多](https://nodejs.org/api/process.html#process_signal_events)。但 `SIGTERM` 在 Windows 上永远不会生效，因为任务管理器中终止进程是无条件的，“即应用程序无法检测或阻止它”。可以参考 libuv 的 [相关文档](https://docs.libuv.org/en/v1.x/signal.html) 了解 `SIGINT`、`SIGBREAK` 等在 Windows 上是如何处理的。另外，参见 Node.js 的 [进程信号事件文档](https://nodejs.org/api/process.html#process_signal_events)。

> info **提示** `enableShutdownHooks` 会通过启动监听器占用内存。在单个 Node 进程中运行多个 Nest 应用程序时（例如使用 Jest 并行运行测试时），Node 可能会报出过多监听器的警告。因此，默认情况下 `enableShutdownHooks` 是禁用的。当你在单个 Node 进程中运行多个实例时，请注意这种情况。

当应用程序收到终止信号时，它会依次调用已注册的 `onModuleDestroy()`、`beforeApplicationShutdown()` 和 `onApplicationShutdown()` 方法（顺序如上所述），并将相应的信号作为第一个参数传入。如果注册的函数等待异步调用（返回一个 Promise），Nest 将不会继续执行后续钩子，直到该 Promise 被解决或拒绝。

```typescript
@@filename()
@Injectable()
class UsersService implements OnApplicationShutdown {
  onApplicationShutdown(signal: string) {
    console.log(signal); // 例如 "SIGINT"
  }
}
@@switch
@Injectable()
class UsersService implements OnApplicationShutdown {
  onApplicationShutdown(signal) {
    console.log(signal); // 例如 "SIGINT"
  }
}
```

> info **提示** 调用 `app.close()` 不会终止 Node 进程，仅会触发 `onModuleDestroy()` 和 `onApplicationShutdown()` 钩子。因此，如果有某些定时器、长时间运行的后台任务等，进程不会自动终止。