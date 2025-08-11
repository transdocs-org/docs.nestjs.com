### 其他功能

本页面列出了你可能会觉得有用的其他所有可用功能。

#### 全局前缀

要忽略通过 `setGlobalPrefix()` 设置的全局路由前缀，请使用 `ignoreGlobalPrefix`：

```typescript
const document = SwaggerModule.createDocument(app, options, {
  ignoreGlobalPrefix: true,
});
```

#### 全局参数

你可以使用 `DocumentBuilder` 为所有路由定义参数，如下所示：

```typescript
const config = new DocumentBuilder()
  .addGlobalParameters({
    name: 'tenantId',
    in: 'header',
  })
  // 其他配置
  .build();
```

#### 全局响应

你可以使用 `DocumentBuilder` 为所有路由定义全局响应。这对于在应用程序的所有端点中设置统一的响应非常有用，例如像 `401 Unauthorized` 或 `500 Internal Server Error` 这样的错误代码。

```typescript
const config = new DocumentBuilder()
  .addGlobalResponse({
    status: 500,
    description: 'Internal server error',
  })
  // 其他配置
  .build();
```

#### 多个规范

`SwaggerModule` 提供了一种支持多个规范的方式。换句话说，你可以在不同的端点上提供不同的文档和不同的 UI 界面。

要支持多个规范，你的应用程序必须采用模块化的方式编写。`createDocument()` 方法接受第三个参数 `extraOptions`，它是一个包含名为 `include` 属性的对象。`include` 属性的值是你想要包含在该 Swagger 规范中的模块数组。

你可以按照如下方式设置对多个规范的支持：

```typescript
import { NestFactory } from '@nestjs/core';
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';
import { AppModule } from './app.module';
import { CatsModule } from './cats/cats.module';
import { DogsModule } from './dogs/dogs.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  /**
   * createDocument(application, configurationOptions, extraOptions);
   *
   * createDocument 方法接受一个可选的第三个参数 "extraOptions"
   * 它是一个包含 "include" 属性的对象，你可以在其中传递一个想要包含在该 Swagger 规范中的模块数组
   * 例如：CatsModule 和 DogsModule 将会有两个独立的 Swagger 规范，
   * 它们将通过两个不同的端点在两个不同的 SwaggerUI 上展示。
   */

  const options = new DocumentBuilder()
    .setTitle('Cats 示例')
    .setDescription('Cats API 描述')
    .setVersion('1.0')
    .addTag('cats')
    .build();

  const catDocumentFactory = () =>
    SwaggerModule.createDocument(app, options, {
      include: [CatsModule],
    });
  SwaggerModule.setup('api/cats', app, catDocumentFactory);

  const secondOptions = new DocumentBuilder()
    .setTitle('Dogs 示例')
    .setDescription('Dogs API 描述')
    .setVersion('1.0')
    .addTag('dogs')
    .build();

  const dogDocumentFactory = () =>
    SwaggerModule.createDocument(app, secondOptions, {
      include: [DogsModule],
    });
  SwaggerModule.setup('api/dogs', app, dogDocumentFactory);

  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```

现在你可以使用以下命令启动服务器：

```bash
$ npm run start
```

访问 `http://localhost:3000/api/cats` 可以看到猫的 Swagger UI：

<figure><img src="/assets/swagger-cats.png" /></figure>

而访问 `http://localhost:3000/api/dogs` 则会展示狗的 Swagger UI：

<figure><img src="/assets/swagger-dogs.png" /></figure>

#### 探索栏中的下拉菜单

要启用探索栏下拉菜单中对多个规范的支持，你需要设置 `explorer: true` 并在 `SwaggerCustomOptions` 中配置 `swaggerOptions.urls`。

> info **提示** 确保 `swaggerOptions.urls` 指向你的 Swagger 文档的 JSON 格式！要指定 JSON 文档，请在 `SwaggerCustomOptions` 中使用 `jsonDocumentUrl`。更多设置选项，请查看[这里](/openapi/introduction#setup-options)。

以下是如何在探索栏的下拉菜单中设置多个规范的示例：

```typescript
import { NestFactory } from '@nestjs/core';
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';
import { AppModule } from './app.module';
import { CatsModule } from './cats/cats.module';
import { DogsModule } from './dogs/dogs.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // 主 API 配置
  const options = new DocumentBuilder()
    .setTitle('多规范示例')
    .setDescription('多规范的描述')
    .setVersion('1.0')
    .build();

  // 创建主 API 文档
  const document = SwaggerModule.createDocument(app, options);

  // 设置支持下拉菜单的主 API Swagger UI
  SwaggerModule.setup('api', app, document, {
    explorer: true,
    swaggerOptions: {
      urls: [
        {
          name: '1. API',
          url: 'api/swagger.json',
        },
        {
          name: '2. Cats API',
          url: 'api/cats/swagger.json',
        },
        {
          name: '3. Dogs API',
          url: 'api/dogs/swagger.json',
        },
      ],
    },
    jsonDocumentUrl: '/api/swagger.json',
  });

  // Cats API 配置
  const catOptions = new DocumentBuilder()
    .setTitle('Cats 示例')
    .setDescription('Cats API 的描述')
    .setVersion('1.0')
    .addTag('cats')
    .build();

  // 创建 Cats API 文档
  const catDocument = SwaggerModule.createDocument(app, catOptions, {
    include: [CatsModule],
  });

  // 设置 Cats API Swagger UI
  SwaggerModule.setup('api/cats', app, catDocument, {
    jsonDocumentUrl: '/api/cats/swagger.json',
  });

  // Dogs API 配置
  const dogOptions = new DocumentBuilder()
    .setTitle('Dogs 示例')
    .setDescription('Dogs API 的描述')
    .setVersion('1.0')
    .addTag('dogs')
    .build();

  // 创建 Dogs API 文档
  const dogDocument = SwaggerModule.createDocument(app, dogOptions, {
    include: [DogsModule],
  });

  // 设置 Dogs API Swagger UI
  SwaggerModule.setup('api/dogs', app, dogDocument, {
    jsonDocumentUrl: '/api/dogs/swagger.json',
  });

  await app.listen(3000);
}

bootstrap();
```

在此示例中，我们设置了一个主 API，并分别为 Cats 和 Dogs 设置了独立的规范，每个规范都可以通过探索栏中的下拉菜单访问。