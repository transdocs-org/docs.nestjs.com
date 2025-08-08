### 简介

[OpenAPI](https://swagger.io/specification/) 规范是一种与语言无关的定义格式，用于描述 RESTful API。Nest 提供了一个专用的 [模块](https://github.com/nestjs/swagger)，允许通过装饰器生成此类规范。

#### 安装

要开始使用它，我们首先安装所需的依赖项。

```bash
$ npm install --save @nestjs/swagger
```

#### 引导

安装过程完成后，打开 `main.ts` 文件，并使用 `SwaggerModule` 类初始化 Swagger：

```typescript
@@filename(main)
import { NestFactory } from '@nestjs/core';
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  const config = new DocumentBuilder()
    .setTitle('Cats example')
    .setDescription('The cats API description')
    .setVersion('1.0')
    .addTag('cats')
    .build();
  const documentFactory = () => SwaggerModule.createDocument(app, config);
  SwaggerModule.setup('api', app, documentFactory);

  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```

> info **提示** 工厂方法 `SwaggerModule.createDocument()` 专门用于在请求时生成 Swagger 文档。这种方法有助于节省一些初始化时间，生成的文档是一个符合 [OpenAPI Document](https://swagger.io/specification/#openapi-document) 规范的可序列化对象。除了通过 HTTP 提供文档外，还可以将其保存为 JSON 或 YAML 文件，并以多种方式使用。

`DocumentBuilder` 有助于构建符合 OpenAPI 规范的基础文档。它提供了多种方法，可以设置标题、描述、版本等属性。为了创建一个包含所有 HTTP 路由的完整文档，我们使用 `SwaggerModule` 类的 `createDocument()` 方法。此方法接受两个参数：应用程序实例和 Swagger 配置对象。或者，我们还可以提供第三个参数，其类型应为 `SwaggerDocumentOptions`。有关此内容的更多信息，请参见 [文档选项部分](/openapi/introduction#document-options)。

一旦我们创建了文档，就可以调用 `setup()` 方法。它接受以下参数：

1. 挂载 Swagger UI 的路径
2. 应用程序实例
3. 上面实例化的文档对象
4. 可选的配置参数（更多内容请参见[此处](/openapi/introduction#setup-options)）

现在，您可以运行以下命令以启动 HTTP 服务器：

```bash
$ npm run start
```

当应用程序运行时，打开浏览器并访问 `http://localhost:3000/api`。您应该会看到 Swagger UI。

<figure><img src="/assets/swagger1.png" /></figure>

如您所见，`SwaggerModule` 会自动反映您的所有端点。

> info **提示** 要生成并下载 Swagger JSON 文件，请访问 `http://localhost:3000/api-json`（假设您的 Swagger 文档可通过 `http://localhost:3000/api` 访问）。
> 还可以使用 `@nestjs/swagger` 中的 setup 方法将其暴露在自定义路由上，如下所示：
>
> ```typescript
> SwaggerModule.setup('swagger', app, documentFactory, {
>   jsonDocumentUrl: 'swagger/json',
> });
> ```
>
> 这会将其暴露在 `http://localhost:3000/swagger/json` 路径下。

> warning **警告** 在使用 `fastify` 和 `helmet` 时，可能会遇到 [CSP](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP) 冲突问题。要解决此问题，请按以下方式配置 CSP：
>
> ```typescript
> app.register(helmet, {
>   contentSecurityPolicy: {
>     directives: {
>       defaultSrc: [`'self'`],
>       styleSrc: [`'self'`, `'unsafe-inline'`],
>       imgSrc: [`'self'`, 'data:', 'validator.swagger.io'],
>       scriptSrc: [`'self'`, `https: 'unsafe-inline'`],
>     },
>   },
> });
>
> // 如果您不打算使用 CSP，可以使用以下配置：
> app.register(helmet, {
>   contentSecurityPolicy: false,
> });
> ```

#### 文档选项

在创建文档时，可以提供一些额外选项来微调库的行为。这些选项应为 `SwaggerDocumentOptions` 类型，具体如下：

```TypeScript
export interface SwaggerDocumentOptions {
  /**
   * 要包含在规范中的模块列表
   */
  include?: Function[];

  /**
   * 应被检查并包含在规范中的额外模型
   */
  extraModels?: Function[];

  /**
   * 如果为 `true`，swagger 将忽略通过 `setGlobalPrefix()` 方法设置的全局前缀
   */
  ignoreGlobalPrefix?: boolean;

  /**
   * 如果为 `true`，swagger 还将加载 `include` 模块导入的模块中的路由
   */
  deepScanRoutes?: boolean;

  /**
   * 自定义 operationIdFactory，用于根据 `controllerKey`、`methodKey` 和版本生成 `operationId`
   * @default () => controllerKey_methodKey_version
   */
  operationIdFactory?: OperationIdFactory;

  /**
   * 自定义 linkNameFactory，用于生成响应中 `links` 字段的链接名称
   *
   * @see [Link objects](https://swagger.io/docs/specification/links/)
   *
   * @default () => `${controllerKey}_${methodKey}_from_${fieldKey}`
   */
  linkNameFactory?: (
    controllerKey: string,
    methodKey: string,
    fieldKey: string
  ) => string;

  /*
   * 根据控制器名称自动生成标签。
   * 如果为 `false`，则必须使用 `@ApiTags()` 装饰器来定义标签。
   * 否则，将使用控制器名称（去除后缀 `Controller`）。
   * @default true
   */
  autoTagControllers?: boolean;
}
```

例如，如果您希望库生成的操作名称像 `createUser` 而不是 `UsersController_createUser`，可以这样设置：

```TypeScript
const options: SwaggerDocumentOptions =  {
  operationIdFactory: (
    controllerKey: string,
    methodKey: string
  ) => methodKey
};
const documentFactory = () => SwaggerModule.createDocument(app, config, options);
```

#### 设置选项

您可以通过传递一个符合 `SwaggerCustomOptions` 接口的对象作为 `SwaggerModule#setup` 方法的第四个参数来配置 Swagger UI。

```TypeScript
export interface SwaggerCustomOptions {
  /**
   * 如果为 `true`，Swagger 资源路径将添加全局前缀（通过 `setGlobalPrefix()` 设置）。
   * 默认值：`false`。
   * @see https://docs.nestjs.com/faq/global-prefix
   */
  useGlobalPrefix?: boolean;

  /**
   * 如果为 `false`，将不提供 Swagger UI。仅 API 定义（JSON 和 YAML）可访问（路径为 `/{path}-json` 和 `/{path}-yaml`）。
   * 要完全禁用 Swagger UI 和 API 定义，请使用 `raw: false`。
   * 默认值：`true`。
   * @deprecated 使用 `ui` 替代。
   */
  swaggerUiEnabled?: boolean;

  /**
   * 如果为 `false`，将不提供 Swagger UI。仅 API 定义（JSON 和 YAML）可访问（路径为 `/{path}-json` 和 `/{path}-yaml`）。
   * 要完全禁用 Swagger UI 和 API 定义，请使用 `raw: false`。
   * 默认值：`true`。
   */
  ui?: boolean;

  /**
   * 如果为 `true`，将提供所有格式的原始定义。
   * 您也可以传入一个数组来指定要提供的格式，例如 `raw: ['json']` 表示仅提供 JSON 定义。
   * 如果省略或设置为空数组，则不提供任何定义（JSON 或 YAML）。
   * 使用此选项可以控制 Swagger 相关端点的可用性。
   * 默认值：`true`。
   */
  raw?: boolean | Array<'json' | 'yaml'>;

  /**
   * 指定在 Swagger UI 中加载的 API 定义的 URL。
   */
  swaggerUrl?: string;

  /**
   * 提供的 JSON API 定义的路径。
   * 默认值：`<path>-json`。
   */
  jsonDocumentUrl?: string;

  /**
   * 提供的 YAML API 定义的路径。
   * 默认值：`<path>-yaml`。
   */
  yamlDocumentUrl?: string;

  /**
   * 钩子函数，可在文档被提供之前对其进行修改。
   * 在文档生成后且作为 JSON 和 YAML 提供之前调用。
   */
  patchDocumentOnRequest?: <TRequest = any, TResponse = any>(
    req: TRequest,
    res: TResponse,
    document: OpenAPIObject
  ) => OpenAPIObject;

  /**
   * 如果为 `true`，将在 Swagger UI 界面中显示 OpenAPI 定义的选择器。
   * 默认值：`false`。
   */
  explorer?: boolean;

  /**
   * 额外的 Swagger UI 选项
   */
  swaggerOptions?: SwaggerUiOptions;

  /**
   * 要注入到 Swagger UI 页面中的自定义 CSS 样式。
   */
  customCss?: string;

  /**
   * 要加载到 Swagger UI 页面中的自定义 CSS 样式表的 URL（或多个 URL）。
   */
  customCssUrl?: string | string[];

  /**
   * 要加载到 Swagger UI 页面中的自定义 JavaScript 文件的 URL（或多个 URL）。
   */
  customJs?: string | string[];

  /**
   * 要加载到 Swagger UI 页面中的自定义 JavaScript 脚本。
   */
  customJsStr?: string | string[];

  /**
   * Swagger UI 页面的自定义图标。
   */
  customfavIcon?: string;

  /**
   * Swagger UI 页面的自定义标题。
   */
  customSiteTitle?: string;

  /**
   * 包含自定义 Swagger UI 静态资源的文件系统路径（例如：./node_modules/swagger-ui-dist）。
   */
  customSwaggerUiPath?: string;

  /**
   * @deprecated 此属性无效。
   */
  validatorUrl?: string;

  /**
   * @deprecated 此属性无效。
   */
  url?: string;

  /**
   * @deprecated 此属性无效。
   */
  urls?: Record<'url' | 'name', string>[];
}
```

> info **提示** `ui` 和 `raw` 是独立的选项。禁用 Swagger UI（`ui: false`）不会禁用 API 定义（JSON/YAML）。反之，禁用 API 定义（`raw: []`）也不会禁用 Swagger UI。
>
> 例如，以下配置将禁用 Swagger UI，但仍允许访问 API 定义：
>
> ```typescript
> const options: SwaggerCustomOptions = {
>   ui: false, // 禁用 Swagger UI
>   raw: ['json'], // JSON API 定义仍然可访问（YAML 被禁用）
> };
> SwaggerModule.setup('api', app, options);
> ```
>
> 在这种情况下，`http://localhost:3000/api-json` 仍然可访问，但 `http://localhost:3000/api`（Swagger UI）不可访问。

#### 示例

一个可用的示例位于 [此处](https://github.com/nestjs/nest/tree/master/sample/11-swagger)。