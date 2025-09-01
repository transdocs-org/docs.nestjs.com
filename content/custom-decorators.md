### 自定义路由装饰器

Nest 是围绕一个称为 **装饰器（Decorators）** 的语言特性构建的。装饰器在许多常用的编程语言中是一个众所周知的概念，但在 JavaScript 世界中它们仍然相对较新。为了更好地理解装饰器的工作原理，我们建议阅读 [这篇文章](https://medium.com/google-developers/exploring-es7-decorators-76ecb65fb841)。以下是一个简单的定义：

<blockquote class="external">
  ES2016 装饰器是一个返回函数的表达式，它可以接受目标对象、属性名和属性描述符作为参数。
  你可以通过在装饰器前面加上 <code>@</code> 字符，并将其放在你试图装饰的内容的最上方来使用它。
  装饰器可以用于类、方法或属性。
</blockquote>

#### 参数装饰器

Nest 提供了一组非常有用的 **参数装饰器（param decorators）**，你可以将其与 HTTP 路由处理器一起使用。以下列出了 Nest 提供的装饰器及其对应的 Express（或 Fastify）对象：

<table>
  <tbody>
    <tr>
      <td><code>@Request(), @Req()</code></td>
      <td><code>req</code></td>
    </tr>
    <tr>
      <td><code>@Response(), @Res()</code></td>
      <td><code>res</code></td>
    </tr>
    <tr>
      <td><code>@Next()</code></td>
      <td><code>next</code></td>
    </tr>
    <tr>
      <td><code>@Session()</code></td>
      <td><code>req.session</code></td>
    </tr>
    <tr>
      <td><code>@Param(param?: string)</code></td>
      <td><code>req.params</code> / <code>req.params[param]</code></td>
    </tr>
    <tr>
      <td><code>@Body(param?: string)</code></td>
      <td><code>req.body</code> / <code>req.body[param]</code></td>
    </tr>
    <tr>
      <td><code>@Query(param?: string)</code></td>
      <td><code>req.query</code> / <code>req.query[param]</code></td>
    </tr>
    <tr>
      <td><code>@Headers(param?: string)</code></td>
      <td><code>req.headers</code> / <code>req.headers[param]</code></td>
    </tr>
    <tr>
      <td><code>@Ip()</code></td>
      <td><code>req.ip</code></td>
    </tr>
    <tr>
      <td><code>@HostParam()</code></td>
      <td><code>req.hosts</code></td>
    </tr>
  </tbody>
</table>

此外，你还可以创建自己的 **自定义装饰器（custom decorators）**。为什么这很有用？

在 Node.js 的世界中，常见的做法是将属性附加到 **请求（request）** 对象上。然后你通常需要在每个路由处理函数中手动提取这些属性，例如：

```typescript
const user = req.user;
```

为了使你的代码更具可读性和透明性，你可以创建一个 `@User()` 装饰器，并在所有控制器中复用它。

```typescript
@@filename(user.decorator)
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const User = createParamDecorator(
  (data: unknown, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    return request.user;
  },
);
```

然后，你可以在任何需要的地方轻松使用它。

```typescript
@@filename()
@Get()
async findOne(@User() user: UserEntity) {
  console.log(user);
}
@@switch
@Get()
@Bind(User())
async findOne(user) {
  console.log(user);
}
```

#### 传递数据

当你的装饰器行为依赖于某些条件时，你可以使用 `data` 参数将参数传递给装饰器的工厂函数。一个典型用例是创建一个根据键从请求对象中提取属性的自定义装饰器。例如，假设我们的 <a href="techniques/authentication#implementing-passport-strategies">身份验证层</a> 验证请求并将用户实体附加到请求对象上。经过身份验证的请求中的用户实体可能如下所示：

```json
{
  "id": 101,
  "firstName": "Alan",
  "lastName": "Turing",
  "email": "alan@email.com",
  "roles": ["admin"]
}
```

现在我们定义一个装饰器，它接受一个属性名作为键，并返回对应的值（如果不存在该属性，或者 `user` 对象尚未创建，则返回 `undefined`）。

```typescript
@@filename(user.decorator)
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const User = createParamDecorator(
  (data: string, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    const user = request.user;

    return data ? user?.[data] : user;
  },
);
@@switch
import { createParamDecorator } from '@nestjs/common';

export const User = createParamDecorator((data, ctx) => {
  const request = ctx.switchToHttp().getRequest();
  const user = request.user;

  return data ? user && user[data] : user;
});
```

以下是如何在控制器中通过 `@User()` 装饰器访问特定属性的示例：

```typescript
@@filename()
@Get()
async findOne(@User('firstName') firstName: string) {
  console.log(`Hello ${firstName}`);
}
@@switch
@Get()
@Bind(User('firstName'))
async findOne(firstName) {
  console.log(`Hello ${firstName}`);
}
```

你可以使用相同的装饰器并传入不同的键来访问不同的属性。如果 `user` 对象结构复杂或嵌套较深，这种做法可以让你更轻松地实现可读性更强的请求处理逻辑。

> info **提示** 对于 TypeScript 用户，请注意 `createParamDecorator<T>()` 是泛型函数。这意味着你可以显式地强制类型安全，例如：`createParamDecorator<string>((data, ctx) => ...)`。或者，你也可以在工厂函数中指定参数类型，例如：`createParamDecorator((data: string, ctx) => ...)`。如果你两者都不指定，`data` 的类型将是 `any`。

#### 与管道一起使用

Nest 处理自定义参数装饰器的方式与内置装饰器（如 `@Body()`、`@Param()` 和 `@Query()`）相同。这意味着管道也会对使用自定义装饰器标注的参数进行处理（在我们的例子中是 `user` 参数）。此外，你还可以直接将管道应用到自定义装饰器上：

```typescript
@@filename()
@Get()
async findOne(
  @User(new ValidationPipe({ validateCustomDecorators: true }))
  user: UserEntity,
) {
  console.log(user);
}
@@switch
@Get()
@Bind(User(new ValidationPipe({ validateCustomDecorators: true })))
async findOne(user) {
  console.log(user);
}
```

> info **提示** 注意：`validateCustomDecorators` 选项必须设置为 `true`。默认情况下，`ValidationPipe` 不会对使用自定义装饰器标注的参数进行验证。

#### 装饰器组合

Nest 提供了一个辅助方法来组合多个装饰器。例如，假设你想将所有与身份验证相关的装饰器合并为一个装饰器。可以使用以下方式实现：

```typescript
@@filename(auth.decorator)
import { applyDecorators } from '@nestjs/common';

export function Auth(...roles: Role[]) {
  return applyDecorators(
    SetMetadata('roles', roles),
    UseGuards(AuthGuard, RolesGuard),
    ApiBearerAuth(),
    ApiUnauthorizedResponse({ description: 'Unauthorized' }),
  );
}
@@switch
import { applyDecorators } from '@nestjs/common';

export function Auth(...roles) {
  return applyDecorators(
    SetMetadata('roles', roles),
    UseGuards(AuthGuard, RolesGuard),
    ApiBearerAuth(),
    ApiUnauthorizedResponse({ description: 'Unauthorized' }),
  );
}
```

然后你可以像这样使用这个自定义的 `@Auth()` 装饰器：

```typescript
@Get('users')
@Auth('admin')
findAllUsers() {}
```

这样只需一次声明，就能应用全部四个装饰器。

> warning **警告** 来自 `@nestjs/swagger` 包的 `@ApiHideProperty()` 装饰器不可组合，与 `applyDecorators` 函数一起使用时不会正常工作。