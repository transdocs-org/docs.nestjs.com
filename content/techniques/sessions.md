### 会话

**HTTP 会话** 提供了一种在多个请求之间存储用户信息的方法，这在 [MVC](/techniques/mvc) 应用中特别有用。

#### 与 Express 配合使用（默认）

首先安装 [所需包](https://github.com/expressjs/session)（以及 TypeScript 用户所需的类型定义）：

```shell
$ npm i express-session
$ npm i -D @types/express-session
```

安装完成后，将 `express-session` 中间件作为全局中间件应用（例如，在你的 `main.ts` 文件中）。

```typescript
import * as session from 'express-session';
// 在初始化文件中的某处
app.use(
  session({
    secret: 'my-secret',
    resave: false,
    saveUninitialized: false,
  }),
);
```

> warning **注意** 默认的服务器端会话存储并不是为生产环境设计的。它在大多数情况下会导致内存泄漏，且无法扩展到单个进程以外，仅适用于调试和开发。更多信息请参阅 [官方仓库](https://github.com/expressjs/session)。

`secret` 用于签名会话 ID cookie。它可以是一个字符串（用于单个密钥），也可以是一个包含多个密钥的数组。如果提供了一个密钥数组，则只有第一个元素将用于签名会话 ID cookie，而在验证请求中的签名时会考虑所有元素。密钥本身不应该容易被人类解析，最好是一组随机字符。

启用 `resave` 选项会强制将会话保存回会话存储中，即使该会话在请求期间未被修改。默认值为 `true`，但使用默认值已被弃用，因为未来的默认值会发生变化。

同样，启用 `saveUninitialized` 选项会强制一个“未初始化”的会话被保存到存储中。当会话是新的但未被修改时，它被认为是未初始化的。选择 `false` 有助于实现登录会话、减少服务器存储使用量，或遵守在设置 cookie 前需要获得许可的法律。选择 `false` 还有助于解决客户端在没有会话的情况下发出多个并行请求时的竞态条件问题（[来源](https://github.com/expressjs/session#saveuninitialized)）。

你可以向 `session` 中间件传递其他几个选项，详细信息请参阅 [API 文档](https://github.com/expressjs/session#options)。

> info **提示** 请注意，`secure: true` 是一个推荐选项。但是，它要求网站启用 HTTPS，即安全 cookie 需要 HTTPS。如果设置了 `secure` 且你通过 HTTP 访问你的网站，则 cookie 将不会被设置。如果你的 Node.js 被代理服务器托管并使用 `secure: true`，你需要在 Express 中设置 `"trust proxy"`。

完成以上配置后，你现在可以在路由处理程序中设置和读取会话值，如下所示：

```typescript
@Get()
findAll(@Req() request: Request) {
  request.session.visits = request.session.visits ? request.session.visits + 1 : 1;
}
```

> info **提示** `@Req()` 装饰器来自 `@nestjs/common` 包，而 `Request` 来自 `express` 包。

或者，你可以使用 `@Session()` 装饰器从请求中提取会话对象，如下所示：

```typescript
@Get()
findAll(@Session() session: Record<string, any>) {
  session.visits = session.visits ? session.visits + 1 : 1;
}
```

> info **提示** `@Session()` 装饰器来自 `@nestjs/common` 包。

#### 与 Fastify 配合使用

首先安装所需的包：

```shell
$ npm i @fastify/secure-session
```

安装完成后，注册 `fastify-secure-session` 插件：

```typescript
import secureSession from '@fastify/secure-session';

// 在初始化文件中的某处
const app = await NestFactory.create<NestFastifyApplication>(
  AppModule,
  new FastifyAdapter(),
);
await app.register(secureSession, {
  secret: 'averylogphrasebiggerthanthirtytwochars',
  salt: 'mq9hDxBVDbspDR6n',
});
```

> info **提示** 你还可以预生成一个密钥（[查看说明](https://github.com/fastify/fastify-secure-session)）或使用 [密钥轮换](https://github.com/fastify/fastify-secure-session#using-keys-with-key-rotation)。

有关可用选项的更多信息，请参阅 [官方仓库](https://github.com/fastify/fastify-secure-session)。

完成以上配置后，你现在可以在路由处理程序中设置和读取会话值，如下所示：

```typescript
@Get()
findAll(@Req() request: FastifyRequest) {
  const visits = request.session.get('visits');
  request.session.set('visits', visits ? visits + 1 : 1);
}
```

或者，你可以使用 `@Session()` 装饰器从请求中提取会话对象，如下所示：

```typescript
@Get()
findAll(@Session() session: secureSession.Session) {
  const visits = session.get('visits');
  session.set('visits', visits ? visits + 1 : 1);
}
```

> info **提示** `@Session()` 装饰器来自 `@nestjs/common` 包，而 `secureSession.Session` 来自 `@fastify/secure-session` 包（导入语句：`import * as secureSession from '@fastify/secure-session'`）。