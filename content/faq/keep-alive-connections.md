### 保持连接（Keep Alive）

默认情况下，NestJS 的 HTTP 适配器会在关闭应用程序前等待响应完成。但有时这种行为并不符合预期，或者不是我们所期望的。有些请求可能会使用 `Connection: Keep-Alive` 头信息，并且会持续较长时间。

对于这些你总是希望应用程序退出而不等待请求结束的场景，可以在创建 NestJS 应用程序时启用 `forceCloseConnections` 选项。

> warning **提示** 大多数用户不需要启用此选项。但如果你发现应用程序在预期时间内没有退出，则可能需要启用该选项。通常当 `app.enableShutdownHooks()` 被启用时，并且你发现应用程序没有重新启动或退出，尤其是在使用 `--watch` 运行 NestJS 应用程序进行开发时。

#### 使用方法

在你的 `main.ts` 文件中，创建 NestJS 应用程序时启用该选项：

```typescript
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule, {
    forceCloseConnections: true,
  });
  await app.listen(process.env.PORT ?? 3000);
}

bootstrap();
```