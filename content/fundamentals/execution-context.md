### 执行上下文

Nest 提供了一些实用工具类，这些类可以帮助你编写适用于多个应用程序上下文（例如：基于 Nest HTTP 服务器、微服务和 WebSockets 的应用程序上下文）的应用程序。这些工具提供了当前执行上下文的信息，可以用于构建通用的 [守卫（guards）](/guards)、[异常过滤器（exception filters）](/exception-filters) 和 [拦截器（interceptors）](/interceptors)，它们可以广泛适用于控制器、方法和执行上下文。

本章我们将介绍其中的两个类：`ArgumentsHost` 和 `ExecutionContext`。

#### ArgumentsHost 类

`ArgumentsHost` 类提供了获取传递给处理函数参数的方法。它允许你选择合适的上下文（例如：HTTP、RPC（微服务）或 WebSockets）来获取参数。框架会在你需要访问它的地方提供一个 `ArgumentsHost` 实例，通常被引用为 `host` 参数。例如，[异常过滤器（exception filter）](https://docs.nestjs.com/exception-filters#arguments-host) 中的 `catch()` 方法就是通过传入一个 `ArgumentsHost` 实例来调用的。

`ArgumentsHost` 只是处理函数参数的一个抽象。例如，对于 HTTP 服务器应用（当使用 `@nestjs/platform-express` 时），`host` 对象封装了 Express 的 `[request, response, next]` 数组，其中 `request` 是请求对象，`response` 是响应对象，`next` 是控制应用程序请求-响应周期的函数。另一方面，对于 [GraphQL](/graphql/quick-start) 应用程序，`host` 对象则包含 `[root, args, context, info]` 数组。

#### 当前应用程序上下文

在构建通用的 [守卫（guards）](/guards)、[异常过滤器（exception filters）](/exception-filters) 和 [拦截器（interceptors）](/interceptors) 时，我们可能需要确定当前方法运行在哪种应用程序上下文中。可以使用 `ArgumentsHost` 的 `getType()` 方法来实现这一目的：

```typescript
if (host.getType() === 'http') {
  // 在常规 HTTP 请求（REST）上下文中执行特定操作
} else if (host.getType() === 'rpc') {
  // 在微服务请求上下文中执行特定操作
} else if (host.getType<GqlContextType>() === 'graphql') {
  // 在 GraphQL 请求上下文中执行特定操作
}
```

> info **提示** `GqlContextType` 从 `@nestjs/graphql` 包中导入。

有了应用程序类型信息后，我们就可以编写更加通用的组件，如下所示。

#### 主机处理函数参数

要获取传递给处理函数的参数数组，可以使用 host 对象的 `getArgs()` 方法：

```typescript
const [req, res, next] = host.getArgs();
```

你也可以通过索引使用 `getArgByIndex()` 方法来获取特定参数：

```typescript
const request = host.getArgByIndex(0);
const response = host.getArgByIndex(1);
```

在这些示例中，我们通过索引获取了请求和响应对象，但通常不建议这样做，因为它会使你的应用程序耦合到特定的执行上下文。相反，你可以通过使用 host 对象提供的工具方法切换到适合的上下文来使你的代码更加健壮和可重用。host 对象的上下文切换工具方法如下所示：

```typescript
/**
 * 切换到 RPC 上下文。
 */
switchToRpc(): RpcArgumentsHost;
/**
 * 切换到 HTTP 上下文。
 */
switchToHttp(): HttpArgumentsHost;
/**
 * 切换到 WebSockets 上下文。
 */
switchToWs(): WsArgumentsHost;
```

让我们使用 `switchToHttp()` 方法重写之前的示例。`host.switchToHttp()` 辅助调用返回一个适用于 HTTP 应用程序上下文的 `HttpArgumentsHost` 对象。`HttpArgumentsHost` 对象有两个有用的方法可以用来提取所需的对象。在这种情况下，我们还使用了 Express 类型断言来返回原生 Express 类型的对象：

```typescript
const ctx = host.switchToHttp();
const request = ctx.getRequest<Request>();
const response = ctx.getResponse<Response>();
```

类似地，`WsArgumentsHost` 和 `RpcArgumentsHost` 也提供了返回微服务和 WebSockets 上下文中适当对象的方法。以下是 `WsArgumentsHost` 的方法：

```typescript
export interface WsArgumentsHost {
  /**
   * 返回数据对象。
   */
  getData<T>(): T;
  /**
   * 返回客户端对象。
   */
  getClient<T>(): T;
}
```

以下是 `RpcArgumentsHost` 的方法：

```typescript
export interface RpcArgumentsHost {
  /**
   * 返回数据对象。
   */
  getData<T>(): T;

  /**
   * 返回上下文对象。
   */
  getContext<T>(): T;
}
```

#### ExecutionContext 类

`ExecutionContext` 继承自 `ArgumentsHost`，提供了当前执行过程的额外细节。和 `ArgumentsHost` 一样，Nest 在需要的地方提供了一个 `ExecutionContext` 实例，例如在 [守卫（guard）](https://docs.nestjs.com/guards#execution-context) 的 `canActivate()` 方法中，或在 [拦截器（interceptor）](https://docs.nestjs.com/interceptors#execution-context) 的 `intercept()` 方法中。它提供了以下方法：

```typescript
export interface ExecutionContext extends ArgumentsHost {
  /**
   * 返回当前处理程序所属控制器类的类型。
   */
  getClass<T>(): Type<T>;
  /**
   * 返回对下一个将要调用的处理程序（方法）的引用。
   */
  getHandler(): Function;
}
```

`getHandler()` 方法返回即将调用的处理程序的引用。`getClass()` 方法返回这个特定处理程序所属的控制器类的类型。例如，在 HTTP 上下文中，如果当前处理的请求是一个绑定到 `CatsController` 中 `create()` 方法的 `POST` 请求，那么 `getHandler()` 返回对 `create()` 方法的引用，而 `getClass()` 返回 `CatsController` **类**（而不是实例）。

```typescript
const methodKey = ctx.getHandler().name; // "create"
const className = ctx.getClass().name; // "CatsController"
```

能够访问当前类和处理程序方法的引用提供了极大的灵活性。最重要的是，它使我们有机会访问通过 `Reflector#createDecorator` 创建的装饰器或拦截器中内置的 `@SetMetadata()` 装饰器设置的元数据。我们将在下面详细讨论这一用例。

<app-banner-enterprise></app-banner-enterprise>

#### 反射与元数据

Nest 提供了通过使用 `Reflector#createDecorator` 方法创建的装饰器或内置的 `@SetMetadata()` 装饰器来将**自定义元数据**附加到路由处理程序的能力。在本节中，我们将比较这两种方法，并展示如何在守卫或拦截器中访问这些元数据。

要使用 `Reflector#createDecorator` 创建强类型装饰器，我们需要指定类型参数。例如，我们创建一个 `Roles` 装饰器，该装饰器接受一个字符串数组作为参数：

```ts
@@filename(roles.decorator)
import { Reflector } from '@nestjs/core';

export const Roles = Reflector.createDecorator<string[]>();
```

这里的 `Roles` 装饰器是一个接受类型为 `string[]` 的单个参数的函数。

现在，要使用这个装饰器，只需将其标注到处理程序上即可：

```typescript
@@filename(cats.controller)
@Post()
@Roles(['admin'])
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
@@switch
@Post()
@Roles(['admin'])
@Bind(Body())
async create(createCatDto) {
  this.catsService.create(createCatDto);
}
```

在这里，我们将 `Roles` 装饰器元数据附加到了 `create()` 方法上，表示只有具有 `admin` 角色的用户才能访问此路由。

要访问路由的角色（自定义元数据），我们将再次使用 `Reflector` 辅助类。`Reflector` 可以通过正常方式注入到类中：

```typescript
@@filename(roles.guard)
@Injectable()
export class RolesGuard {
  constructor(private reflector: Reflector) {}
}
@@switch
@Injectable()
@Dependencies(Reflector)
export class CatsService {
  constructor(reflector) {
    this.reflector = reflector;
  }
}
```

> info **提示** `Reflector` 类从 `@nestjs/core` 包中导入。

现在，要读取处理程序的元数据，使用 `get()` 方法：

```typescript
const roles = this.reflector.get(Roles, context.getHandler());
```

`Reflector#get` 方法允许我们通过传入两个参数轻松访问元数据：一个装饰器引用和一个**上下文**（装饰器目标）来检索元数据。在这个例子中，指定的**装饰器**是 `Roles`（请参考上面的 `roles.decorator.ts` 文件）。上下文由调用 `context.getHandler()` 提供，这将导致提取当前处理的路由处理程序的元数据。记住，`getHandler()` 给我们的是一个**对路由处理函数的引用**。

或者，我们可以通过在控制器级别应用元数据来组织我们的控制器，这样将影响控制器类中的所有路由。

```typescript
@@filename(cats.controller)
@Roles(['admin'])
@Controller('cats')
export class CatsController {}
@@switch
@Roles(['admin'])
@Controller('cats')
export class CatsController {}
```

在这种情况下，为了提取控制器元数据，我们传递 `context.getClass()` 作为第二个参数（将控制器类作为元数据提取的上下文），而不是 `context.getHandler()`：

```typescript
@@filename(roles.guard)
const roles = this.reflector.get(Roles, context.getClass());
```

鉴于你可以在多个级别提供元数据，你可能需要从多个上下文中提取并合并元数据。`Reflector` 类提供了两个用于此目的的工具方法。这些方法一次提取**控制器和方法的元数据**，并以不同的方式将它们合并。

考虑以下场景，你已在两个级别都提供了 `Roles` 元数据：

```typescript
@@filename(cats.controller)
@Roles(['user'])
@Controller('cats')
export class CatsController {
  @Post()
  @Roles(['admin'])
  async create(@Body() createCatDto: CreateCatDto) {
    this.catsService.create(createCatDto);
  }
}
@@switch
@Roles(['user'])
@Controller('cats')
export class CatsController {}
  @Post()
  @Roles(['admin'])
  @Bind(Body())
  async create(createCatDto) {
    this.catsService.create(createCatDto);
  }
}
```

如果你的意图是将 `'user'` 指定为默认角色，并选择性地覆盖某些方法，你可能会使用 `getAllAndOverride()` 方法：

```typescript
const roles = this.reflector.getAllAndOverride(Roles, [context.getHandler(), context.getClass()]);
```

运行在 `create()` 方法上下文中的带有上述元数据的守卫，将导致 `roles` 包含 `['admin']`。

要同时获取元数据并进行合并（此方法会合并数组和对象），请使用 `getAllAndMerge()` 方法：

```typescript
const roles = this.reflector.getAllAndMerge(Roles, [context.getHandler(), context.getClass()]);
```

这将导致 `roles` 包含 `['user', 'admin']`。

对于这两种合并方法，你将元数据键作为第一个参数传入，并将元数据目标上下文的数组（即调用 `getHandler()` 和/或 `getClass()` 方法）作为第二个参数传入。

#### 低级方法

如前所述，除了使用 `Reflector#createDecorator` 之外，你还可以使用内置的 `@SetMetadata()` 装饰器来将元数据附加到处理程序上。

```typescript
@@filename(cats.controller)
@Post()
@SetMetadata('roles', ['admin'])
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
@@switch
@Post()
@SetMetadata('roles', ['admin'])
@Bind(Body())
async create(createCatDto) {
  this.catsService.create(createCatDto);
}
```

> info **提示** `@SetMetadata()` 装饰器从 `@nestjs/common` 包中导入。

通过上述构造，我们将 `roles` 元数据（`roles` 是元数据键，`['admin']` 是关联的值）附加到了 `create()` 方法上。虽然这样可以工作，但直接在路由中使用 `@SetMetadata()` 并不是一个好习惯。相反，你可以创建自己的装饰器，如下所示：

```typescript
@@filename(roles.decorator)
import { SetMetadata } from '@nestjs/common';

export const Roles = (...roles: string[]) => SetMetadata('roles', roles);
@@switch
import { SetMetadata } from '@nestjs/common';

export const Roles = (...roles) => SetMetadata('roles', roles);
```

这种方法更加整洁和可读，并且在某种程度上类似于 `Reflector#createDecorator` 方法。不同之处在于，使用 `@SetMetadata` 你可以更灵活地控制元数据键和值，并且还可以创建接受多个参数的装饰器。

现在我们有了自定义的 `@Roles()` 装饰器，我们可以使用它来装饰 `create()` 方法：

```typescript
@@filename(cats.controller)
@Post()
@Roles('admin')
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
@@switch
@Post()
@Roles('admin')
@Bind(Body())
async create(createCatDto) {
  this.catsService.create(createCatDto);
}
```

要访问路由的角色（自定义元数据），我们将再次使用 `Reflector` 辅助类：

```typescript
@@filename(roles.guard)
@Injectable()
export class RolesGuard {
  constructor(private reflector: Reflector) {}
}
@@switch
@Injectable()
@Dependencies(Reflector)
export class CatsService {
  constructor(reflector) {
    this.reflector = reflector;
  }
}
```

> info **提示** `Reflector` 类从 `@nestjs/core` 包中导入。

现在，要读取处理程序的元数据，请使用 `get()` 方法：

```typescript
const roles = this.reflector.get<string[]>('roles', context.getHandler());
```

在这里，我们不是传递一个装饰器引用，而是将元数据**键**作为第一个参数传入（在我们的例子中是 `'roles'`）。其余部分与 `Reflector#createDecorator` 示例中的一样。