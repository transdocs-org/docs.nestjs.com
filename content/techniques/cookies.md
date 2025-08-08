### Cookies

一个 **HTTP cookie** 是由用户浏览器存储的一小段数据。Cookie 被设计为网站用来记住有状态信息的一种可靠机制。当用户再次访问网站时，该 cookie 会随着请求自动发送。

#### 与 Express 配合使用（默认）

首先安装 [必需的包](https://github.com/expressjs/cookie-parser)（TypeScript 用户还需要安装其类型定义）：

```shell
$ npm i cookie-parser
$ npm i -D @types/cookie-parser
```

安装完成后，将 `cookie-parser` 中间件作为全局中间件应用（例如，在你的 `main.ts` 文件中）。

```typescript
import * as cookieParser from 'cookie-parser';
// 在你的初始化文件中的某个位置
app.use(cookieParser());
```

你可以向 `cookieParser` 中间件传递几个选项：

- `secret` 一个字符串或数组，用于签名 cookie。这是可选的，如果不指定，则不会解析签名的 cookie。如果提供了字符串，就将其用作密钥。如果提供了数组，则会按顺序尝试使用每个密钥来解签 cookie。
- `options` 一个传递给 `cookie.parse` 的第二个参数的对象。更多信息请查看 [cookie](https://www.npmjs.org/package/cookie)。

中间件将解析请求中的 `Cookie` 头，并将 cookie 数据作为属性 `req.cookies` 暴露出来，如果提供了密钥，则还会作为属性 `req.signedCookies` 暴露出来。这些属性是 cookie 名称到 cookie 值的键值对。

当提供了密钥时，该模块会解签并验证任何签名的 cookie 值，并将这些键值对从 `req.cookies` 移动到 `req.signedCookies`。签名的 cookie 是其值以 `s:` 为前缀的 cookie。签名验证失败的签名 cookie 将具有值 `false`，而不是被篡改的值。

完成此设置后，你现在可以在路由处理程序中读取 cookie，如下所示：

```typescript
@Get()
findAll(@Req() request: Request) {
  console.log(request.cookies); // 或 "request.cookies['cookieKey']"
  // 或 console.log(request.signedCookies);
}
```

> info **提示** `@Req()` 装饰器是从 `@nestjs/common` 导入的，而 `Request` 是从 `express` 包导入的。

要将 cookie 附加到传出响应中，请使用 `Response#cookie()` 方法：

```typescript
@Get()
findAll(@Res({ passthrough: true }) response: Response) {
  response.cookie('key', 'value')
}
```

> warning **警告** 如果你想将响应处理逻辑交给框架处理，请记住将 `passthrough` 选项设置为 `true`，如上所示。了解更多 [点击此处](/controllers#library-specific-approach)。

> info **提示** `@Res()` 装饰器是从 `@nestjs/common` 导入的，而 `Response` 是从 `express` 包导入的。

#### 与 Fastify 配合使用

首先安装必需的包：

```shell
$ npm i @fastify/cookie
```

安装完成后，注册 `@fastify/cookie` 插件：

```typescript
import fastifyCookie from '@fastify/cookie';

// 在你的初始化文件中的某个位置
const app = await NestFactory.create<NestFastifyApplication>(AppModule, new FastifyAdapter());
await app.register(fastifyCookie, {
  secret: 'my-secret', // 用于 cookie 签名
});
```

完成此设置后，你现在可以在路由处理程序中读取 cookie，如下所示：

```typescript
@Get()
findAll(@Req() request: FastifyRequest) {
  console.log(request.cookies); // 或 "request.cookies['cookieKey']"
}
```

> info **提示** `@Req()` 装饰器是从 `@nestjs/common` 导入的，而 `FastifyRequest` 是从 `fastify` 包导入的。

要将 cookie 附加到传出响应中，请使用 `FastifyReply#setCookie()` 方法：

```typescript
@Get()
findAll(@Res({ passthrough: true }) response: FastifyReply) {
  response.setCookie('key', 'value')
}
```

要了解有关 `FastifyReply#setCookie()` 方法的更多信息，请查看 [此页面](https://github.com/fastify/fastify-cookie#sending)。

> warning **警告** 如果你想将响应处理逻辑交给框架处理，请记住将 `passthrough` 选项设置为 `true`，如上所示。了解更多 [点击此处](/controllers#library-specific-approach)。

> info **提示** `@Res()` 装饰器是从 `@nestjs/common` 导入的，而 `FastifyReply` 是从 `fastify` 包导入的。

#### 创建自定义装饰器（跨平台）

为了提供一种方便、声明式的方式来访问传入的 cookie，我们可以创建一个[自定义装饰器](/custom-decorators)。

```typescript
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const Cookies = createParamDecorator((data: string, ctx: ExecutionContext) => {
  const request = ctx.switchToHttp().getRequest();
  return data ? request.cookies?.[data] : request.cookies;
});
```

`@Cookies()` 装饰器将从 `req.cookies` 对象中提取所有 cookie 或指定名称的 cookie，并将装饰的参数填充为该值。

完成此设置后，我们现在可以在路由处理程序中使用该装饰器，如下所示：

```typescript
@Get()
findAll(@Cookies('name') name: string) {}
```