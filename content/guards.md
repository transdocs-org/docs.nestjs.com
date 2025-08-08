### 守卫（Guards）

守卫是一个使用 `@Injectable()` 装饰器标注的类，并实现了 `CanActivate` 接口。

<figure><img class="illustrative-image" src="/assets/Guards_1.png" /></figure>

守卫具有 **单一职责**。它根据运行时的某些条件（如权限、角色、访问控制列表等）决定是否由路由处理程序来处理给定的请求。这通常被称为 **授权（authorization）**。在传统的 Express 应用中，授权（以及其“近亲”——**身份验证（authentication）**，通常与授权协作）通常由 [中间件](/middleware) 来处理。对于身份验证来说，中间件是一个不错的选择，因为像令牌验证和向 `request` 对象附加属性这样的操作，并不与特定路由上下文（及其元数据）紧密相关。

但中间件本质上是“无知”的。它不知道在调用 `next()` 函数后会执行哪个处理程序。而 **守卫（Guards）** 可以访问 `ExecutionContext` 实例，因此可以确切地知道下一步将执行什么。它们的设计类似于异常过滤器、管道和拦截器，允许你在请求/响应周期的合适位置插入处理逻辑，并且以声明式的方式进行。这有助于保持代码的 DRY（不重复）和声明式风格。

> info **提示** 守卫在所有中间件之后执行，但在任何拦截器或管道之前执行。

#### 授权守卫（Authorization guard）

如前所述，**授权** 是守卫的一个理想用例，因为特定的路由应该仅在调用者（通常是经过身份验证的用户）具有足够权限时才可用。我们即将构建的 `AuthGuard` 假设用户已通过身份验证（因此，请求头中附带了一个令牌）。它将提取并验证该令牌，并利用提取的信息决定请求是否可以继续执行。

```typescript
@@filename(auth.guard)
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Observable } from 'rxjs';

@Injectable()
export class AuthGuard implements CanActivate {
  canActivate(
    context: ExecutionContext,
  ): boolean | Promise<boolean> | Observable<boolean> {
    const request = context.switchToHttp().getRequest();
    return validateRequest(request);
  }
}
@@switch
import { Injectable } from '@nestjs/common';

@Injectable()
export class AuthGuard {
  async canActivate(context) {
    const request = context.switchToHttp().getRequest();
    return validateRequest(request);
  }
}
```

> info **提示** 如果你想了解如何在应用程序中实现身份验证机制的真实示例，请访问 [本章节](/security/authentication)。同样，如需更复杂的授权示例，请查看 [此页面](/security/authorization)。

`validateRequest()` 函数中的逻辑可以根据需要设计得简单或复杂。这个示例的主要目的是展示守卫是如何融入请求/响应周期的。

每个守卫都必须实现一个 `canActivate()` 函数。该函数应返回一个布尔值，表示是否允许当前请求。它可以同步或异步（通过 `Promise` 或 `Observable`）返回结果。Nest 使用返回值来控制下一步操作：

- 如果返回 `true`，则请求将被处理。
- 如果返回 `false`，则 Nest 会拒绝该请求。

<app-banner-enterprise></app-banner-enterprise>

#### 执行上下文（Execution context）

`canActivate()` 函数接受一个参数，即 `ExecutionContext` 实例。`ExecutionContext` 继承自 `ArgumentsHost`。我们在异常过滤器章节中曾见过 `ArgumentsHost`。在上面的示例中，我们只是使用了之前定义在 `ArgumentsHost` 上的辅助方法来获取对 `Request` 对象的引用。你可以回顾 [异常过滤器](https://docs.nestjs.com/exception-filters#arguments-host) 章节中的 **Arguments host** 部分了解更多信息。

通过继承 `ArgumentsHost`，`ExecutionContext` 还添加了一些新的辅助方法，这些方法提供了关于当前执行过程的更多细节。这些信息对于构建适用于广泛控制器、方法和执行上下文的通用守卫非常有帮助。了解更多关于 `ExecutionContext` 的内容请访问 [此处](/fundamentals/execution-context)。

#### 基于角色的身份验证（Role-based authentication）

让我们构建一个更实用的守卫，仅允许具有特定角色的用户访问资源。我们从一个基本的守卫模板开始，并在后续部分逐步完善它。目前，它允许所有请求通过：

```typescript
@@filename(roles.guard)
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Observable } from 'rxjs';

@Injectable()
export class RolesGuard implements CanActivate {
  canActivate(
    context: ExecutionContext,
  ): boolean | Promise<boolean> | Observable<boolean> {
    return true;
  }
}
@@switch
import { Injectable } from '@nestjs/common';

@Injectable()
export class RolesGuard {
  canActivate(context) {
    return true;
  }
}
```

#### 绑定守卫（Binding guards）

像管道和异常过滤器一样，守卫可以是 **控制器作用域**、**方法作用域** 或 **全局作用域**。下面，我们使用 `@UseGuards()` 装饰器设置一个控制器作用域的守卫。该装饰器可以接受一个参数，或者一个逗号分隔的参数列表。这使你可以轻松地通过一次声明应用适当的守卫集合。

```typescript
@@filename()
@Controller('cats')
@UseGuards(RolesGuard)
export class CatsController {}
```

> info **提示** `@UseGuards()` 装饰器从 `@nestjs/common` 包导入。

上面，我们传递了 `RolesGuard` 类（而不是实例），将实例化责任交给了框架，并启用了依赖注入。与管道和异常过滤器一样，我们也可以传入一个就地实例：

```typescript
@@filename()
@Controller('cats')
@UseGuards(new RolesGuard())
export class CatsController {}
```

上面的构造将守卫附加到该控制器声明的每个处理程序上。如果我们希望守卫仅应用于一个方法，可以在 **方法级别** 使用 `@UseGuards()` 装饰器。

要设置全局守卫，请使用 Nest 应用程序实例的 `useGlobalGuards()` 方法：

```typescript
@@filename()
const app = await NestFactory.create(AppModule);
app.useGlobalGuards(new RolesGuard());
```

> warning **注意** 对于混合应用程序，默认情况下 `useGlobalGuards()` 方法不会为网关和微服务设置守卫（有关如何更改此行为的信息，请参阅 [混合应用程序](/faq/hybrid-application)）。对于“标准”（非混合）微服务应用程序，`useGlobalGuards()` 会全局挂载守卫。

全局守卫应用于整个应用程序的每个控制器和每个路由处理程序。在依赖注入方面，从模块外部（如上面示例中使用 `useGlobalGuards()`）注册的全局守卫无法注入依赖项，因为这是在模块上下文之外进行的。为了解决这个问题，你可以使用以下构造直接从任何模块设置守卫：

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { APP_GUARD } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_GUARD,
      useClass: RolesGuard,
    },
  ],
})
export class AppModule {}
```

> info **提示** 当使用这种方式对守卫进行依赖注入时，请注意无论在哪个模块中使用此构造，守卫实际上都是全局的。应该在哪里使用？选择定义守卫的模块（如上面的 `RolesGuard`）。此外，`useClass` 不是处理自定义提供者注册的唯一方式。了解更多请访问 [此处](/fundamentals/custom-providers)。

#### 按处理程序设置角色（Setting roles per handler）

我们的 `RolesGuard` 已经生效，但它还不够智能。我们尚未利用守卫最重要的功能——[执行上下文](/fundamentals/execution-context)。它还不知道角色，也不知道每个处理程序允许哪些角色。例如，`CatsController` 的不同路由可能有不同的权限方案。有些可能只对管理员用户开放，而有些则对所有人开放。我们如何以灵活且可复用的方式将角色与路由匹配？

这就是 **自定义元数据（custom metadata）** 发挥作用的地方（了解更多请访问 [此处](https://docs.nestjs.com/fundamentals/execution-context#reflection-and-metadata)）。Nest 提供了通过 `Reflector.createDecorator` 静态方法创建的装饰器，或者内置的 `@SetMetadata()` 装饰器，将自定义 **元数据** 附加到路由处理程序上的能力。

例如，我们可以通过 `Reflector.createDecorator` 方法创建一个 `@Roles()` 装饰器，将元数据附加到处理程序上。`Reflector` 是框架内置并从 `@nestjs/core` 包导出的。

```ts
@@filename(roles.decorator)
import { Reflector } from '@nestjs/core';

export const Roles = Reflector.createDecorator<string[]>();
```

这里的 `Roles` 装饰器是一个接受 `string[]` 类型参数的函数。

现在，要使用这个装饰器，我们只需将其标注在处理程序上即可：

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

我们已将 `Roles` 装饰器的元数据附加到 `create()` 方法上，表明只有具有 `admin` 角色的用户才能访问该路由。

或者，除了使用 `Reflector.createDecorator` 方法，我们还可以使用内置的 `@SetMetadata()` 装饰器。了解更多请访问 [此处](/fundamentals/execution-context#low-level-approach)。

#### 综合运用（Putting it all together）

现在我们回到 `RolesGuard`，将其与我们之前创建的元数据结合使用。目前，它在所有情况下都返回 `true`，允许所有请求继续执行。我们希望根据当前用户的角色与当前路由所需角色的比较结果来决定返回值。为了访问路由的角色（自定义元数据），我们将再次使用 `Reflector` 辅助类，如下所示：

```typescript
@@filename(roles.guard)
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { Roles } from './roles.decorator';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const roles = this.reflector.get(Roles, context.getHandler());
    if (!roles) {
      return true;
    }
    const request = context.switchToHttp().getRequest();
    const user = request.user;
    return matchRoles(roles, user.roles);
  }
}
@@switch
import { Injectable, Dependencies } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { Roles } from './roles.decorator';

@Injectable()
@Dependencies(Reflector)
export class RolesGuard {
  constructor(reflector) {
    this.reflector = reflector;
  }

  canActivate(context) {
    const roles = this.reflector.get(Roles, context.getHandler());
    if (!roles) {
      return true;
    }
    const request = context.switchToHttp().getRequest();
    const user = request.user;
    return matchRoles(roles, user.roles);
  }
}
```

> info **提示** 在 node.js 世界中，将已授权的用户附加到 `request` 对象上是一种常见做法。因此，在上面的示例代码中，我们假设 `request.user` 包含用户实例和允许的角色。在你的应用程序中，你可能在自定义 **身份验证守卫**（或中间件）中进行这种关联。有关此主题的更多信息，请查看 [本章](/security/authentication)。

> warning **警告** `matchRoles()` 函数中的逻辑可以根据需要设计得简单或复杂。本示例的主要目的是展示守卫如何融入请求/响应周期。

有关如何在上下文敏感的方式中使用 `Reflector` 的更多细节，请参阅 **执行上下文（Execution context）** 章节中的 <a href="https://docs.nestjs.com/fundamentals/execution-context#reflection-and-metadata">反射与元数据（Reflection and metadata）</a> 部分。

当一个权限不足的用户请求某个端点时，Nest 会自动返回如下响应：

```typescript
{
  "statusCode": 403,
  "message": "Forbidden resource",
  "error": "Forbidden"
}
```

请注意，幕后，当守卫返回 `false` 时，框架会抛出一个 `ForbiddenException`。如果你想返回不同的错误响应，你应该抛出自定义异常。例如：

```typescript
throw new UnauthorizedException();
```

守卫抛出的任何异常都将由 [异常层](/exception-filters)（全局异常过滤器和应用于当前上下文的任何异常过滤器）处理。

> info **提示** 如果你想了解如何实现授权的真实示例，请查看 [本章](/security/authorization)。