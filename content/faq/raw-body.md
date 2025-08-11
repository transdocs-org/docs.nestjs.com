### 原始请求体

拥有对原始请求体访问权限的一个最常见用例是执行 Webhook 签名验证。通常，为了执行 Webhook 签名验证，需要未反序列化的请求体来计算 HMAC 哈希值。

> warning **警告**：仅当启用内置的全局请求体解析中间件时才能使用此功能，也就是说，在创建应用时不能传递 `bodyParser: false`。

#### 与 Express 一起使用

首先在创建 Nest Express 应用时启用该选项：

```typescript
import { NestFactory } from '@nestjs/core';
import type { NestExpressApplication } from '@nestjs/platform-express';
import { AppModule } from './app.module';

// 在 "bootstrap" 函数中
const app = await NestFactory.create<NestExpressApplication>(AppModule, {
  rawBody: true,
});
await app.listen(process.env.PORT ?? 3000);
```

要在控制器中访问原始请求体，可以使用一个便捷接口 `RawBodyRequest` 来在请求对象上暴露 `rawBody` 字段：使用接口 `RawBodyRequest` 类型：

```typescript
import { Controller, Post, RawBodyRequest, Req } from '@nestjs/common';
import { Request } from 'express';

@Controller('cats')
class CatsController {
  @Post()
  create(@Req() req: RawBodyRequest<Request>) {
    const raw = req.rawBody; // 返回一个 `Buffer`。
  }
}
```

#### 注册不同的解析器

默认情况下，只注册了 `json` 和 `urlencoded` 解析器。如果你想要动态注册其他解析器，则需要显式地进行注册。

例如，要注册一个 `text` 解析器，可以使用以下代码：

```typescript
app.useBodyParser('text');
```

> warning **警告**：请确保在调用 `NestFactory.create` 时提供了正确的应用类型。对于 Express 应用，正确的类型是 `NestExpressApplication`，否则将找不到 `.useBodyParser` 方法。

#### 请求体解析大小限制

如果你的应用需要解析比 Express 默认的 `100kb` 更大的请求体，请使用以下代码：

```typescript
app.useBodyParser('json', { limit: '10mb' });
```

`.useBodyParser` 方法会尊重在应用选项中传递的 `rawBody` 选项。

#### 与 Fastify 一起使用

首先在创建 Nest Fastify 应用时启用该选项：

```typescript
import { NestFactory } from '@nestjs/core';
import {
  FastifyAdapter,
  NestFastifyApplication,
} from '@nestjs/platform-fastify';
import { AppModule } from './app.module';

// 在 "bootstrap" 函数中
const app = await NestFactory.create<NestFastifyApplication>(
  AppModule,
  new FastifyAdapter(),
  {
    rawBody: true,
  },
);
await app.listen(process.env.PORT ?? 3000);
```

要在控制器中访问原始请求体，可以使用一个便捷接口 `RawBodyRequest` 来在请求对象上暴露 `rawBody` 字段：使用接口 `RawBodyRequest` 类型：

```typescript
import { Controller, Post, RawBodyRequest, Req } from '@nestjs/common';
import { FastifyRequest } from 'fastify';

@Controller('cats')
class CatsController {
  @Post()
  create(@Req() req: RawBodyRequest<FastifyRequest>) {
    const raw = req.rawBody; // 返回一个 `Buffer`。
  }
}
```

#### 注册不同的解析器

默认情况下，只注册了 `application/json` 和 `application/x-www-form-urlencoded` 解析器。如果你想要动态注册其他解析器，则需要显式地进行注册。

例如，要注册一个 `text/plain` 解析器，可以使用以下代码：

```typescript
app.useBodyParser('text/plain');
```

> warning **警告**：请确保在调用 `NestFactory.create` 时提供了正确的应用类型。对于 Fastify 应用，正确的类型是 `NestFastifyApplication`，否则将找不到 `.useBodyParser` 方法。

#### 请求体解析大小限制

如果你的应用需要解析比 Fastify 默认的 `1MiB` 更大的请求体，请使用以下代码：

```typescript
const bodyLimit = 10_485_760; // 10MiB
app.useBodyParser('application/json', { bodyLimit });
```

`.useBodyParser` 方法会尊重在应用选项中传递的 `rawBody` 选项。