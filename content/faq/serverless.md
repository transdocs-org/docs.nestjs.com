### 无服务器架构（Serverless）

无服务器计算是一种云计算执行模型，云服务提供商按需分配机器资源，代表客户管理服务器。当应用程序未被使用时，不会为其分配任何计算资源。其定价基于应用程序实际消耗的资源量（[来源](https://en.wikipedia.org/wiki/Serverless_computing)）。

使用 **无服务器架构**，你只需专注于应用程序代码中的各个函数。诸如 AWS Lambda、Google Cloud Functions 和 Microsoft Azure Functions 这类服务会自动管理所有物理硬件、虚拟机操作系统以及 Web 服务器软件。

> info **提示** 本章不讨论无服务器函数的优缺点，也不深入探讨任何云服务提供商的具体细节。

#### 冷启动

冷启动是指你的代码在一段时间内首次被执行。根据你所使用的云服务提供商，它可能涉及多个不同的操作，从下载代码、引导运行时环境，直到最终执行你的代码。  
此过程会根据多种因素增加**显著的延迟**，包括编程语言、应用程序所需的依赖包数量等。

冷启动非常重要，尽管有一些因素我们无法控制，但我们仍可以在自身层面做很多事情来尽可能缩短冷启动时间。

虽然你可以将 Nest 视为一个专为复杂企业级应用设计的完整框架，但它也**适用于更“简单”的应用**（或脚本）。例如，借助 [独立应用](/standalone-applications) 功能，你可以在简单的 Worker、CRON 任务、CLI 或无服务器函数中使用 Nest 的依赖注入（DI）系统。

#### 基准测试

为了更好地理解在无服务器函数场景下使用 Nest 或其他知名库（如 `express`）所带来的开销，我们可以比较 Node.js 运行以下脚本所需的时间：

```typescript
// #1 Express
import * as express from 'express';

async function bootstrap() {
  const app = express();
  app.get('/', (req, res) => res.send('Hello world!'));
  await new Promise<void>((resolve) => app.listen(3000, resolve));
}
bootstrap();

// #2 Nest（使用 @nestjs/platform-express）
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule, { logger: ['error'] });
  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();

// #3 作为独立应用的 Nest（无 HTTP 服务器）
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { AppService } from './app.service';

async function bootstrap() {
  const app = await NestFactory.createApplicationContext(AppModule, {
    logger: ['error'],
  });
  console.log(app.get(AppService).getHello());
}
bootstrap();

// #4 原生 Node.js 脚本
async function bootstrap() {
  console.log('Hello world!');
}
bootstrap();
```

对于以上所有脚本，我们使用了 `tsc`（TypeScript）编译器，因此代码未被打包（未使用 `webpack`）。

|                                      |                   |
| ------------------------------------ | ----------------- |
| Express                              | 0.0079s (7.9ms)   |
| Nest with `@nestjs/platform-express` | 0.1974s (197.4ms) |
| Nest（独立应用）                     | 0.1117s (111.7ms) |
| 原生 Node.js 脚本                    | 0.0071s (7.1ms)   |

> info **注意** 测试机器：MacBook Pro Mid 2014，2.5 GHz 四核 Intel Core i7，16 GB 1600 MHz DDR3，SSD。

现在，我们再次运行所有基准测试，但这次使用 `webpack`（如果你安装了 [Nest CLI](/cli/overview)，可以运行 `nest build --webpack`）将应用程序打包成一个可执行的 JavaScript 文件。  
不过，我们不会使用 Nest CLI 自带的默认 `webpack` 配置，而是确保将所有依赖项（`node_modules`）一起打包，如下所示：

```javascript
module.exports = (options, webpack) => {
  const lazyImports = [
    '@nestjs/microservices/microservices-module',
    '@nestjs/websockets/socket-module',
  ];

  return {
    ...options,
    externals: [],
    plugins: [
      ...options.plugins,
      new webpack.IgnorePlugin({
        checkResource(resource) {
          if (lazyImports.includes(resource)) {
            try {
              require.resolve(resource);
            } catch (err) {
              return true;
            }
          }
          return false;
        },
      }),
    ],
  };
};
```

> info **提示** 要让 Nest CLI 使用此配置，请在项目根目录下创建一个名为 `webpack.config.js` 的新文件。

使用此配置，我们得到了以下结果：

|                                      |                  |
| ------------------------------------ | ---------------- |
| Express                              | 0.0068s (6.8ms)  |
| Nest with `@nestjs/platform-express` | 0.0815s (81.5ms) |
| Nest（独立应用）                     | 0.0319s (31.9ms) |
| 原生 Node.js 脚本                    | 0.0066s (6.6ms)  |

> info **注意** 测试机器：MacBook Pro Mid 2014，2.5 GHz 四核 Intel Core i7，16 GB 1600 MHz DDR3，SSD。

> info **提示** 你还可以通过进一步的代码压缩和优化技术（使用 `webpack` 插件等）来进一步提升性能。

如你所见，编译方式（是否打包代码）对整体启动时间有显著影响。使用 `webpack`，你可以将一个 Nest 独立应用（包含一个模块、一个控制器和一个服务的初始项目）的启动时间平均降低至约 32ms，而基于 Express 的常规 HTTP NestJS 应用的启动时间则可降低至约 81.5ms。

对于更复杂的 Nest 应用，例如通过 `$ nest g resource` 生成的包含 10 个资源（即 10 个模块、10 个控制器、10 个服务、20 个 DTO 类、50 个 HTTP 端点 + `AppModule`）的应用，在同样配置的机器上整体启动时间约为 0.1298s（129.8ms）。不过，将单体应用作为无服务器函数运行通常没有太多意义，所以请将此基准测试视为应用程序增长时启动时间可能增加的一个示例。

#### 运行时优化

到目前为止，我们讨论了编译时优化。然而，与编译无关的是你如何在应用程序中定义提供者（Providers）以及加载 Nest 模块的方式，这在应用程序规模变大时尤为重要。

例如，假设你将数据库连接定义为 [异步提供者](/fundamentals/async-providers)。异步提供者的设计是为了延迟应用程序的启动，直到一个或多个异步任务完成。  
这意味着，如果某个无服务器函数平均需要 2 秒来连接数据库（在启动时），那么该函数在冷启动时至少需要额外的两秒（因为它必须等待数据库连接建立）才能返回响应。

由此可见，在**无服务器环境**中，启动时间非常重要，因此你在构建提供者时应采用不同的方式。  
另一个例子是使用 Redis 作为缓存，但仅在某些特定场景下使用。在这种情况下，你不应将 Redis 连接定义为异步提供者，因为即使在当前函数调用中不需要 Redis，它也会拖慢启动时间。

此外，有时你可以使用 `LazyModuleLoader` 类按需懒加载整个模块，如 [本章](/fundamentals/lazy-loading-modules) 所述。缓存是一个很好的例子。  
假设你的应用程序有一个 `CacheModule`，它内部连接了 Redis，并导出了 `CacheService` 用于与 Redis 存储交互。如果你不是在所有函数调用中都需要它，那么你可以在需要时按需懒加载。这样，当冷启动发生时，不需要缓存的调用可以更快地启动。

```typescript
if (request.method === RequestMethod[RequestMethod.GET]) {
  const { CacheModule } = await import('./cache.module');
  const moduleRef = await this.lazyModuleLoader.load(() => CacheModule);

  const { CacheService } = await import('./cache.service');
  const cacheService = moduleRef.get(CacheService);

  return cacheService.get(ENDPOINT_KEY);
}
```

另一个很好的例子是一个 Webhook 或 Worker，根据某些特定条件（例如输入参数）执行不同的操作。  
在这种情况下，你可以在路由处理函数中指定一个条件，按需懒加载适合当前函数调用的模块，而其他模块则按需懒加载。

```typescript
if (workerType === WorkerType.A) {
  const { WorkerAModule } = await import('./worker-a.module');
  const moduleRef = await this.lazyModuleLoader.load(() => WorkerAModule);
  // ...
} else if (workerType === WorkerType.B) {
  const { WorkerBModule } = await import('./worker-b.module');
  const moduleRef = await this.lazyModuleLoader.load(() => WorkerBModule);
  // ...
}
```

#### 示例集成

你的应用程序入口文件（通常是 `main.ts`）的写法**取决于多个因素**，因此**没有一种通用模板**适用于所有场景。  
例如，启动无服务器函数所需的初始化文件会因云服务商（AWS、Azure、GCP 等）而异。  
此外，如果你希望运行一个具有多个路由/端点的典型 HTTP 应用，还是仅提供一个路由（或执行特定代码段），你的代码也会不同（例如，对于每个函数一个端点的方案，你可以使用 `NestFactory.createApplicationContext`，而无需启动 HTTP 服务器、设置中间件等）。

为了演示，我们将使用 `@nestjs/platform-express` 创建一个完整的 HTTP 路由器，并将其与 [Serverless](https://www.serverless.com/) 框架集成（本例中目标为 AWS Lambda）。如前所述，你的代码会因云服务商和其他因素而有所不同。

首先，安装所需的包：

```bash
$ npm i @codegenie/serverless-express aws-lambda
$ npm i -D @types/aws-lambda serverless-offline
```

> info **提示** 为了加快开发周期，我们安装了 `serverless-offline` 插件，用于模拟 AWS Lambda 和 API Gateway。

安装完成后，创建 `serverless.yml` 文件以配置 Serverless 框架：

```yaml
service: serverless-example

plugins:
  - serverless-offline

provider:
  name: aws
  runtime: nodejs14.x

functions:
  main:
    handler: dist/main.handler
    events:
      - http:
          method: ANY
          path: /
      - http:
          method: ANY
          path: '{proxy+}'
```

> info **提示** 欲了解 Serverless 框架的更多信息，请访问其 [官方文档](https://www.serverless.com/framework/docs/)。

接下来，编辑 `main.ts` 文件，更新引导代码以添加必要的样板代码：

```typescript
import { NestFactory } from '@nestjs/core';
import serverlessExpress from '@codegenie/serverless-express';
import { Callback, Context, Handler } from 'aws-lambda';
import { AppModule } from './app.module';

let server: Handler;

async function bootstrap(): Promise<Handler> {
  const app = await NestFactory.create(AppModule);
  await app.init();

  const expressApp = app.getHttpAdapter().getInstance();
  return serverlessExpress({ app: expressApp });
}

export const handler: Handler = async (
  event: any,
  context: Context,
  callback: Callback,
) => {
  server = server ?? (await bootstrap());
  return server(event, context, callback);
};
```

> info **提示** 如果你希望创建多个无服务器函数并共享模块，我们建议使用 [CLI 的 Monorepo 模式](/cli/monorepo#monorepo-mode)。

> warning **警告** 如果你使用了 `@nestjs/swagger` 包，还需要一些额外步骤才能在无服务器函数中正常工作。更多信息请查看这个 [讨论帖](https://github.com/nestjs/swagger/issues/199)。

接下来，打开 `tsconfig.json` 文件，确保启用了 `esModuleInterop` 选项，以便 `@codegenie/serverless-express` 正确加载。

```json
{
  "compilerOptions": {
    ...
    "esModuleInterop": true
  }
}
```

现在你可以构建你的应用（使用 `nest build` 或 `tsc`），并通过 `serverless` CLI 在本地运行 Lambda 函数：

```bash
$ npm run build
$ npx serverless offline
```

应用程序启动后，打开浏览器并访问 `http://localhost:3000/dev/[ANY_ROUTE]`（其中 `[ANY_ROUTE]` 是你在应用中注册的任意端点）。

在前面的章节中，我们已经展示了使用 `webpack` 并打包你的应用可以显著减少整体启动时间。  
然而，要使我们的示例正常运行，你还需要在 `webpack.config.js` 文件中添加一些额外配置。一般来说，为确保 `handler` 函数被正确识别，你需要将 `output.libraryTarget` 属性设置为 `commonjs2`。

```javascript
return {
  ...options,
  externals: [],
  output: {
    ...options.output,
    libraryTarget: 'commonjs2',
  },
  // ... 其他配置
};
```

完成以上配置后，你现在可以使用 `$ nest build --webpack` 编译函数代码（然后使用 `$ npx serverless offline` 进行测试）。

推荐（但**不是必须**，因为这会减慢构建过程）安装 `terser-webpack-plugin` 并覆盖其配置以在压缩生产构建时保留类名。如果不这样做，当在应用中使用 `class-validator` 时可能会导致行为异常。

```javascript
const TerserPlugin = require('terser-webpack-plugin');

return {
  ...options,
  externals: [],
  optimization: {
    minimizer: [
      new TerserPlugin({
        terserOptions: {
          keep_classnames: true,
        },
      }),
    ],
  },
  output: {
    ...options.output,
    libraryTarget: 'commonjs2',
  },
  // ... 其他配置
};
```

#### 使用独立应用功能

或者，如果你希望你的函数非常轻量，并且不需要任何与 HTTP 相关的功能（如路由、守卫、拦截器、管道等），你可以直接使用 `NestFactory.createApplicationContext`（如前所述），而不是运行整个 HTTP 服务器（及其底层的 `express`），如下所示：

```typescript
@@filename(main)
import { HttpStatus } from '@nestjs/common';
import { NestFactory } from '@nestjs/core';
import { Callback, Context, Handler } from 'aws-lambda';
import { AppModule } from './app.module';
import { AppService } from './app.service';

export const handler: Handler = async (
  event: any,
  context: Context,
  callback: Callback,
) => {
  const appContext = await NestFactory.createApplicationContext(AppModule);
  const appService = appContext.get(AppService);

  return {
    body: appService.getHello(),
    statusCode: HttpStatus.OK,
  };
};
```

> info **提示** 请注意，`NestFactory.createApplicationContext` 不会将控制器方法用增强器（如守卫、拦截器等）包装。如需此类功能，必须使用 `NestFactory.create` 方法。

你还可以将 `event` 对象传递给某个提供者，例如 `EventsService`，由它根据输入值和业务逻辑处理并返回相应的结果。

```typescript
export const handler: Handler = async (
  event: any,
  context: Context,
  callback: Callback,
) => {
  const appContext = await NestFactory.createApplicationContext(AppModule);
  const eventsService = appContext.get(EventsService);
  return eventsService.process(event);
};
```