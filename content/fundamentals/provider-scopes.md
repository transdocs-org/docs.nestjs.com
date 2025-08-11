### 注入作用域

对于来自不同编程语言背景的开发者来说，可能会感到意外的是，在 Nest 中几乎所有内容都是在传入请求之间共享的。我们拥有数据库连接池、具有全局状态的单例服务等等。请记住，Node.js 并不遵循每个请求都由单独线程处理的“请求/响应多线程无状态模型”。因此，使用单例实例对我们的应用程序来说是完全 **安全的**。

不过，也有一些边缘情况，请求生命周期可能是我们期望的行为，例如 GraphQL 应用程序中的按请求缓存、请求追踪和多租户架构。注入作用域提供了一种机制来实现我们期望的提供者生命周期行为。

#### 提供者作用域

一个提供者可以具有以下任意作用域：

<table>
  <tr>
    <td><code>DEFAULT</code></td>
    <td>提供者的单个实例在整个应用程序中共享。该实例的生命周期直接绑定到应用程序生命周期。一旦应用程序启动完成，所有单例提供者都会被实例化。默认使用单例作用域。</td>
  </tr>
  <tr>
    <td><code>REQUEST</code></td>
    <td>为每个传入的<strong>请求</strong>创建提供者的新实例。请求处理完成后，该实例将被垃圾回收。</td>
  </tr>
  <tr>
    <td><code>TRANSIENT</code></td>
    <td>瞬态提供者不会在消费者之间共享。每个注入瞬态提供者的消费者都将获得一个全新、专用的实例。</td>
  </tr>
</table>

> info **提示** 对于大多数使用场景，**推荐**使用单例作用域。跨消费者和跨请求共享提供者意味着实例可以被缓存，并且其初始化仅在应用程序启动期间发生一次。

#### 使用方式

通过向 `@Injectable()` 装饰器选项对象传递 `scope` 属性来指定注入作用域：

```typescript
import { Injectable, Scope } from '@nestjs/common';

@Injectable({ scope: Scope.REQUEST })
export class CatsService {}
```

同样，对于[自定义提供者](/fundamentals/custom-providers)，在提供者注册的长格式中设置 `scope` 属性：

```typescript
{
  provide: 'CACHE_MANAGER',
  useClass: CacheManager,
  scope: Scope.TRANSIENT,
}
```

> info **提示** 从 `@nestjs/common` 导入 `Scope` 枚举

默认使用单例作用域，不需要显式声明。如果你确实希望将提供者声明为单例作用域，请将 `scope` 属性设置为 `Scope.DEFAULT`。

> warning **注意** WebSocket 网关不应使用请求作用域的提供者，因为它们必须作为单例存在。每个网关封装了一个实际的 socket，不能多次实例化。此限制也适用于一些其他提供者，例如 [_Passport 策略_](../security/authentication#request-scoped-strategies) 或 _Cron 控制器_。

#### 控制器作用域

控制器也可以具有作用域，该作用域适用于该控制器中声明的所有请求方法处理器。与提供者作用域类似，控制器的作用域声明了其生命周期。对于请求作用域的控制器，每个传入请求都会创建一个新实例，并在请求处理完成后进行垃圾回收。

使用 `ControllerOptions` 对象的 `scope` 属性声明控制器作用域：

```typescript
@Controller({
  path: 'cats',
  scope: Scope.REQUEST,
})
export class CatsController {}
```

#### 作用域层次结构

`REQUEST` 作用域会在注入链中向上冒泡。依赖于请求作用域提供者的控制器本身也将成为请求作用域。

想象以下依赖关系图：`CatsController <- CatsService <- CatsRepository`。如果 `CatsService` 是请求作用域（而其他是默认单例），则 `CatsController` 将成为请求作用域，因为它依赖于注入的服务。未依赖的 `CatsRepository` 将保持单例作用域。

瞬态作用域的依赖项不遵循这种模式。如果单例作用域的 `DogsService` 注入了一个瞬态 `LoggerService` 提供者，它将获得一个全新的实例。然而，`DogsService` 仍将保持单例作用域，因此在任何地方注入它都不会解析为 `DogsService` 的新实例。如果这是期望的行为，则 `DogsService` 也必须显式标记为 `TRANSIENT`。

<app-banner-courses></app-banner-courses>

#### 请求提供者

在基于 HTTP 服务器的应用程序中（例如使用 `@nestjs/platform-express` 或 `@nestjs/platform-fastify`），当使用请求作用域的提供者时，您可能希望访问原始请求对象的引用。您可以注入 `REQUEST` 对象来实现这一点。

`REQUEST` 提供者本质上是请求作用域的，这意味着使用它时不需要显式指定 `REQUEST` 作用域。此外，即使尝试指定，也会被忽略。依赖请求作用域提供者的任何提供者都会自动采用请求作用域，并且此行为无法更改。

```typescript
import { Injectable, Scope, Inject } from '@nestjs/common';
import { REQUEST } from '@nestjs/core';
import { Request } from 'express';

@Injectable({ scope: Scope.REQUEST })
export class CatsService {
  constructor(@Inject(REQUEST) private request: Request) {}
}
```

由于底层平台/协议的差异，在微服务或 GraphQL 应用程序中访问传入请求的方式略有不同。在 [GraphQL](/graphql/quick-start) 应用程序中，请注入 `CONTEXT` 而不是 `REQUEST`：

```typescript
import { Injectable, Scope, Inject } from '@nestjs/common';
import { CONTEXT } from '@nestjs/graphql';

@Injectable({ scope: Scope.REQUEST })
export class CatsService {
  constructor(@Inject(CONTEXT) private context) {}
}
```

然后在 `GraphQLModule` 中配置您的 `context` 值以包含 `request` 作为其属性。

#### 请求发起者提供者（Inquirer provider）

如果您想获取提供者被构造的类，例如在日志或指标提供者中，可以注入 `INQUIRER` 令牌。

```typescript
import { Inject, Injectable, Scope } from '@nestjs/common';
import { INQUIRER } from '@nestjs/core';

@Injectable({ scope: Scope.TRANSIENT })
export class HelloService {
  constructor(@Inject(INQUIRER) private parentClass: object) {}

  sayHello(message: string) {
    console.log(`${this.parentClass?.constructor?.name}: ${message}`);
  }
}
```

然后按如下方式使用：

```typescript
import { Injectable } from '@nestjs/common';
import { HelloService } from './hello.service';

@Injectable()
export class AppService {
  constructor(private helloService: HelloService) {}

  getRoot(): string {
    this.helloService.sayHello('My name is getRoot');

    return 'Hello world!';
  }
}
```

在上面的示例中，当调用 `AppService#getRoot` 时，控制台将输出 `"AppService: My name is getRoot"`。

#### 性能

使用请求作用域的提供者会对应用程序性能产生影响。虽然 Nest 会尽可能缓存元数据，但每次请求仍必须创建类的实例。因此，这将减慢平均响应时间以及整体基准测试结果。除非提供者必须为请求作用域，否则强烈建议使用默认的单例作用域。

> info **提示** 虽然听起来可能令人担忧，但一个设计良好的使用请求作用域提供者的应用程序，延迟增加不应超过约 5%。

#### 持久提供者（Durable providers）

如上所述，请求作用域的提供者可能导致延迟增加，因为只要控制器实例中注入了一个请求作用域的提供者（或更深的依赖），该控制器也将变为请求作用域。这意味着必须为每个单独的请求重新创建（实例化）控制器（并在之后进行垃圾回收）。例如，对于 30k 个并行请求，将创建 30k 个临时控制器实例（及其请求作用域的提供者）。

如果一个通用提供者被大多数提供者依赖（例如数据库连接或日志服务），那么它会自动将所有这些提供者也转换为请求作用域的提供者。这在**多租户应用程序**中可能带来挑战，尤其是那些具有中心化的请求作用域“数据源”提供者的应用程序，该提供者从请求对象中获取 headers/token，并根据其值检索相应的数据库连接/模式（特定于该租户）。

例如，假设您有一个被 10 个不同客户交替使用的应用程序。每个客户都有其**专用的数据源**，并且您希望确保客户 A 永远无法访问客户 B 的数据库。一种实现方式是声明一个请求作用域的“数据源”提供者，根据请求对象确定“当前客户”并检索其对应的数据库。使用这种方法，您可以在几分钟内将应用程序转换为多租户应用程序。但这种方式的一个主要缺点是，由于大多数应用程序组件都依赖于“数据源”提供者，它们将隐式地变为“请求作用域”，因此您肯定会看到应用程序性能的下降。

但是，如果我们有更好的解决方案呢？既然我们只有 10 个客户，是否可以为每个客户创建 10 个独立的 [DI 子树](/fundamentals/module-ref#resolving-scoped-providers)（而不是每次请求都重新创建整棵树）？如果您的提供者不依赖于每个连续请求都唯一的一些属性（例如请求 UUID），而是有一些特定属性可以让我们聚合（分类）它们，那么就没有理由在每次传入请求时重新创建 DI 子树。

这正是 **持久提供者（durable providers）** 发挥作用的地方。

在开始将提供者标记为持久之前，我们必须首先注册一个**策略**，该策略告诉 Nest 哪些是“常见的请求属性”，并提供逻辑来对请求进行分组——将它们与对应的 DI 子树关联。

```typescript
import {
  HostComponentInfo,
  ContextId,
  ContextIdFactory,
  ContextIdStrategy,
} from '@nestjs/core';
import { Request } from 'express';

const tenants = new Map<string, ContextId>();

export class AggregateByTenantContextIdStrategy implements ContextIdStrategy {
  attach(contextId: ContextId, request: Request) {
    const tenantId = request.headers['x-tenant-id'] as string;
    let tenantSubTreeId: ContextId;

    if (tenants.has(tenantId)) {
      tenantSubTreeId = tenants.get(tenantId);
    } else {
      tenantSubTreeId = ContextIdFactory.create();
      tenants.set(tenantId, tenantSubTreeId);
    }

    // 如果树不是持久的，返回原始的 "contextId" 对象
    return (info: HostComponentInfo) =>
      info.isTreeDurable ? tenantSubTreeId : contextId;
  }
}
```

> info **提示** 与请求作用域类似，持久性会在注入链中向上冒泡。这意味着如果 A 依赖于标记为 `durable` 的 B，则 A 也会隐式变为持久（除非为 A 提供者显式将 `durable` 设置为 `false`）。

> warning **警告** 请注意，此策略对于具有大量租户的应用程序来说并不理想。

`attach` 方法返回的值指示 Nest 对于给定主机应使用什么上下文标识符。在此例中，我们指定当主机组件（例如请求作用域的控制器）被标记为 durable 时，应使用 `tenantSubTreeId` 而不是原始的自动生成的 `contextId` 对象（您可以在下面学习如何将提供者标记为 durable）。此外，在上面的示例中，**不会注册任何 payload**（payload = 表示“根”的 `REQUEST`/`CONTEXT` 提供者，即子树的父级）。

如果您想为持久树注册 payload，请改用以下构造：

```typescript
// AggregateByTenantContextIdStrategy#attach 方法的返回值：
return {
  resolve: (info: HostComponentInfo) =>
    info.isTreeDurable ? tenantSubTreeId : contextId,
  payload: { tenantId },
};
```

现在，当您使用 `@Inject(REQUEST)`/`@Inject(CONTEXT)` 注入 `REQUEST` 提供者（或 GraphQL 应用程序中的 `CONTEXT`）时，将注入 `payload` 对象（在此例中包含一个属性 `tenantId`）。

好了，有了这个策略，您可以在代码中的某个地方注册它（因为它适用于全局），例如，您可以将其放在 `main.ts` 文件中：

```typescript
ContextIdFactory.apply(new AggregateByTenantContextIdStrategy());
```

> info **提示** `ContextIdFactory` 类从 `@nestjs/core` 包中导入。

只要注册发生在任何请求到达您的应用程序之前，一切都会按预期工作。

最后，要将普通提供者转换为持久提供者，只需将 `durable` 标志设置为 `true` 并将其作用域更改为 `Scope.REQUEST`（如果注入链中已有 REQUEST 作用域则不需要）：

```typescript
import { Injectable, Scope } from '@nestjs/common';

@Injectable({ scope: Scope.REQUEST, durable: true })
export class CatsService {}
```

同样，对于[自定义提供者](/fundamentals/custom-providers)，在提供者注册的长格式中设置 `durable` 属性：

```typescript
{
  provide: 'foobar',
  useFactory: () => { ... },
  scope: Scope.REQUEST,
  durable: true,
}
```