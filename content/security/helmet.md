### Helmet

[Helmet](https://github.com/helmetjs/helmet) 可以通过适当设置 HTTP 头来帮助保护你的应用程序免受一些众所周知的 Web 漏洞的攻击。通常来说，Helmet 只是一组小型中间件函数的集合，这些函数用于设置与安全相关的 HTTP 头（阅读[更多](https://github.com/helmetjs/helmet#how-it-works)）。

> info **提示** 注意，应用 `helmet` 作为全局中间件或注册它时，必须位于其他调用 `app.use()` 或可能调用 `app.use()` 的设置函数之前。这是由于底层平台（如 Express 或 Fastify）的工作方式决定的，中间件/路由的定义顺序非常重要。如果你在定义某个路由之后使用像 `helmet` 或 `cors` 这样的中间件，那么该中间件将不会作用于该路由，而只会作用于其后定义的路由。

#### 与 Express 一起使用（默认）

首先安装所需的包：

```bash
$ npm i --save helmet
```

安装完成后，将其作为全局中间件应用：

```typescript
import helmet from 'helmet';
// 在你的初始化文件中的某个位置
app.use(helmet());
```

> warning **警告** 当使用 `helmet`、`@apollo/server`（4.x）以及 [Apollo Sandbox](https://docs.nestjs.com/graphql/quick-start#apollo-sandbox) 时，Apollo Sandbox 上可能会出现 [CSP](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/CSP) 问题。要解决此问题，请按如下方式配置 CSP：
>
> ```typescript
> app.use(helmet({
>   crossOriginEmbedderPolicy: false,
>   contentSecurityPolicy: {
>     directives: {
>       imgSrc: [`'self'`, 'data:', 'apollo-server-landing-page.cdn.apollographql.com'],
>       scriptSrc: [`'self'`, `https: 'unsafe-inline'`],
>       manifestSrc: [`'self'`, 'apollo-server-landing-page.cdn.apollographql.com'],
>       frameSrc: [`'self'`, 'sandbox.embed.apollographql.com'],
>     },
>   },
> }));

#### 与 Fastify 一起使用

如果你使用的是 `FastifyAdapter`，请安装 [@fastify/helmet](https://github.com/fastify/fastify-helmet) 包：

```bash
$ npm i --save @fastify/helmet
```

[fastify-helmet](https://github.com/fastify/fastify-helmet) 不应作为中间件使用，而应作为 [Fastify 插件](https://www.fastify.io/docs/latest/Reference/Plugins/) 使用，即通过 `app.register()` 方法：

```typescript
import helmet from '@fastify/helmet'
// 在你的初始化文件中的某个位置
await app.register(helmet)
```

> warning **警告** 当使用 `apollo-server-fastify` 和 `@fastify/helmet` 时，在 GraphQL Playground 中可能会出现 [CSP](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/CSP) 问题。要解决此冲突，请按如下方式配置 CSP：
>
> ```typescript
> await app.register(fastifyHelmet, {
>    contentSecurityPolicy: {
>      directives: {
>        defaultSrc: [`'self'`, 'unpkg.com'],
>        styleSrc: [
>          `'self'`,
>          `'unsafe-inline'`,
>          'cdn.jsdelivr.net',
>          'fonts.googleapis.com',
>          'unpkg.com',
>        ],
>        fontSrc: [`'self'`, 'fonts.gstatic.com', 'data:'],
>        imgSrc: [`'self'`, 'data:', 'cdn.jsdelivr.net'],
>        scriptSrc: [
>          `'self'`,
>          `https: 'unsafe-inline'`,
>          `cdn.jsdelivr.net`,
>          `'unsafe-eval'`,
>        ],
>      },
>    },
>  });
>
> // 如果你完全不打算使用 CSP，可以使用以下配置：
> await app.register(fastifyHelmet, {
>   contentSecurityPolicy: false,
> });
> ```
