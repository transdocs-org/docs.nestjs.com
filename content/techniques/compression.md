### 压缩

压缩可以显著减小响应体的大小，从而提高 Web 应用的速度。

对于生产环境中的**高流量**网站，强烈建议将压缩任务从应用服务器卸载到反向代理（例如 Nginx）中处理。在这种情况下，你不应使用压缩中间件。

#### 与 Express 配合使用（默认方式）

使用 [compression](https://github.com/expressjs/compression) 中间件包来启用 gzip 压缩。

首先安装所需的包：

```bash
$ npm i --save compression
$ npm i --save-dev @types/compression
```

安装完成后，将压缩中间件作为全局中间件应用。

```typescript
import * as compression from 'compression';
// 在你的初始化文件中的某处
app.use(compression());
```

#### 与 Fastify 配合使用

如果使用 `FastifyAdapter`，请使用 [fastify-compress](https://github.com/fastify/fastify-compress)：

```bash
$ npm i --save @fastify/compress
```

安装完成后，将 `@fastify/compress` 中间件作为全局中间件应用。

> warning **警告** 请确保在创建应用程序时使用了 `NestFastifyApplication` 类型。否则，你将无法通过 `register` 方法应用压缩中间件。

```typescript
import { FastifyAdapter, NestFastifyApplication } from '@nestjs/platform-fastify';
import compression from '@fastify/compress';

// 在 bootstrap() 函数内部
const app = await NestFactory.create<NestFastifyApplication>(AppModule, new FastifyAdapter());
await app.register(compression);
```

默认情况下，`@fastify/compress` 将在 Node.js 版本 >= 11.7.0 且浏览器支持该编码时使用 Brotli 压缩。虽然 Brotli 在压缩率方面非常高效，但它也可能导致较高的 CPU 开销。Brotli 默认的最大压缩质量为 11，可以通过调整 `BROTLI_PARAM_QUALITY` 参数（范围从 0 到 11）来降低压缩质量以换取更快的压缩速度。以下是一个将压缩质量设为 4 的示例：

```typescript
import { constants } from 'zlib';
// 在你的初始化文件中的某处
await app.register(compression, { brotliOptions: { params: { [constants.BROTLI_PARAM_QUALITY]: 4 } } });
```

为了简化处理，你也可以告诉 `fastify-compress` 只使用 deflate 和 gzip 来压缩响应；这将导致响应体积可能更大，但传输速度会更快。

要指定编码方式，请向 `app.register` 方法传递第二个参数：

```typescript
await app.register(compression, { encodings: ['gzip', 'deflate'] });
```

上述代码将告诉 `fastify-compress` 仅使用 gzip 和 deflate 编码，并且在客户端同时支持两种编码时优先选择 gzip。