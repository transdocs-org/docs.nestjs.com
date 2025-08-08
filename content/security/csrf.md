### CSRF 保护

跨站请求伪造（CSRF 或 XSRF）是一种攻击类型，其中 **未经授权的** 命令会从一个受信任的用户发送到 Web 应用程序。为了帮助防止此类攻击，你可以使用 [csrf-csrf](https://github.com/Psifi-Solutions/csrf-csrf) 包。

#### 与 Express 配合使用（默认）

首先安装所需的包：

```bash
$ npm i csrf-csrf
```

> warning **警告** 如 [csrf-csrf 文档](https://github.com/Psifi-Solutions/csrf-csrf?tab=readme-ov-file#getting-started) 中所述，该中间件要求事先初始化 session 中间件或 `cookie-parser`。有关更多细节，请参考相关文档。

安装完成后，在全局中间件中注册 `csrf-csrf` 中间件。

```typescript
import { doubleCsrf } from 'csrf-csrf';
// ...
// 在你的初始化文件中的某个位置
const {
  invalidCsrfTokenError, // 此项仅作为便利提供，如果你打算创建自己的中间件，可以使用它。
  generateToken, // 在你的路由中使用它来生成 CSRF 哈希、令牌 Cookie 和令牌。
  validateRequest, // 如果你打算创建自己的中间件，这项也很方便。
  doubleCsrfProtection, // 这是默认的 CSRF 保护中间件。
} = doubleCsrf(doubleCsrfOptions);
app.use(doubleCsrfProtection);
```

#### 与 Fastify 配合使用

首先安装所需的包：

```bash
$ npm i --save @fastify/csrf-protection
```

安装完成后，按如下方式注册 `@fastify/csrf-protection` 插件：

```typescript
import fastifyCsrf from '@fastify/csrf-protection';
// ...
// 在你的初始化文件中，注册了某些存储插件之后
await app.register(fastifyCsrf);
```

> warning **警告** 如 `@fastify/csrf-protection` 文档 [中所述](https://github.com/fastify/csrf-protection#usage)，该插件要求首先初始化一个存储插件。请参考相关文档获取更多说明。