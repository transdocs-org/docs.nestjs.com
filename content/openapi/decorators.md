### 装饰器

所有可用的 OpenAPI 装饰器都带有 `Api` 前缀，以便与核心装饰器区分开来。以下列出了所有导出的装饰器，并标明了每个装饰器可以应用的级别。

|                           |                     |
| ------------------------- | ------------------- |
| `@ApiBasicAuth()`         | 方法 / 控制器       |
| `@ApiBearerAuth()`        | 方法 / 控制器       |
| `@ApiBody()`              | 方法                |
| `@ApiConsumes()`          | 方法 / 控制器       |
| `@ApiCookieAuth()`        | 方法 / 控制器       |
| `@ApiExcludeController()` | 控制器              |
| `@ApiExcludeEndpoint()`   | 方法                |
| `@ApiExtension()`         | 方法                |
| `@ApiExtraModels()`       | 方法 / 控制器       |
| `@ApiHeader()`            | 方法 / 控制器       |
| `@ApiHideProperty()`      | 模型                |
| `@ApiOAuth2()`            | 方法 / 控制器       |
| `@ApiOperation()`         | 方法                |
| `@ApiParam()`             | 方法 / 控制器       |
| `@ApiProduces()`          | 方法 / 控制器       |
| `@ApiSchema()`            | 模型                |
| `@ApiProperty()`          | 模型                |
| `@ApiPropertyOptional()`  | 模型                |
| `@ApiQuery()`             | 方法 / 控制器       |
| `@ApiResponse()`          | 方法 / 控制器       |
| `@ApiSecurity()`          | 方法 / 控制器       |
| `@ApiTags()`              | 方法 / 控制器       |
| `@ApiCallbacks()`         | 方法 / 控制器       |