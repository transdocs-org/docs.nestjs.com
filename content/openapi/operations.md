### 操作

在 OpenAPI 术语中，路径是你的 API 公开的端点（资源），例如 `/users` 或 `/reports/summary`，而操作则是用于操作这些路径的 HTTP 方法，例如 `GET`、`POST` 或 `DELETE`。

#### 标签

要将控制器绑定到特定标签，请使用 `@ApiTags(...tags)` 装饰器。

```typescript
@ApiTags('cats')
@Controller('cats')
export class CatsController {}
```

#### 请求头

要定义作为请求一部分的自定义请求头，请使用 `@ApiHeader()` 装饰器。

```typescript
@ApiHeader({
  name: 'X-MyHeader',
  description: 'Custom header',
})
@Controller('cats')
export class CatsController {}
```

#### 响应

要定义自定义的 HTTP 响应，请使用 `@ApiResponse()` 装饰器。

```typescript
@Post()
@ApiResponse({ status: 201, description: 'The record has been successfully created.'})
@ApiResponse({ status: 403, description: 'Forbidden.'})
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

Nest 提供了一组继承自 `@ApiResponse` 装饰器的快捷 **API 响应** 装饰器：

- `@ApiOkResponse()`
- `@ApiCreatedResponse()`
- `@ApiAcceptedResponse()`
- `@ApiNoContentResponse()`
- `@ApiMovedPermanentlyResponse()`
- `@ApiFoundResponse()`
- `@ApiBadRequestResponse()`
- `@ApiUnauthorizedResponse()`
- `@ApiNotFoundResponse()`
- `@ApiForbiddenResponse()`
- `@ApiMethodNotAllowedResponse()`
- `@ApiNotAcceptableResponse()`
- `@ApiRequestTimeoutResponse()`
- `@ApiConflictResponse()`
- `@ApiPreconditionFailedResponse()`
- `@ApiTooManyRequestsResponse()`
- `@ApiGoneResponse()`
- `@ApiPayloadTooLargeResponse()`
- `@ApiUnsupportedMediaTypeResponse()`
- `@ApiUnprocessableEntityResponse()`
- `@ApiInternalServerErrorResponse()`
- `@ApiNotImplementedResponse()`
- `@ApiBadGatewayResponse()`
- `@ApiServiceUnavailableResponse()`
- `@ApiGatewayTimeoutResponse()`
- `@ApiDefaultResponse()`

```typescript
@Post()
@ApiCreatedResponse({ description: 'The record has been successfully created.'})
@ApiForbiddenResponse({ description: 'Forbidden.'})
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

要为请求指定返回模型，我们必须创建一个类并使用 `@ApiProperty()` 装饰器注解所有属性。

```typescript
export class Cat {
  @ApiProperty()
  id: number;

  @ApiProperty()
  name: string;

  @ApiProperty()
  age: number;

  @ApiProperty()
  breed: string;
}
```

然后可以将 `Cat` 模型与响应装饰器的 `type` 属性结合使用。

```typescript
@ApiTags('cats')
@Controller('cats')
export class CatsController {
  @Post()
  @ApiCreatedResponse({
    description: 'The record has been successfully created.',
    type: Cat,
  })
  async create(@Body() createCatDto: CreateCatDto): Promise<Cat> {
    return this.catsService.create(createCatDto);
  }
}
```

让我们打开浏览器并验证生成的 `Cat` 模型：

<figure><img src="/assets/swagger-response-type.png" /></figure>

除了为每个端点或控制器单独定义响应之外，你还可以使用 `DocumentBuilder` 类为所有端点定义全局响应。当你希望为应用程序中的所有端点定义全局响应（例如，对于 `401 Unauthorized` 或 `500 Internal Server Error` 等错误）时，此方法非常有用。

```typescript
const config = new DocumentBuilder()
  .addGlobalResponse({
    status: 500,
    description: 'Internal server error',
  })
  // 其他配置
  .build();
```

#### 文件上传

你可以使用 `@ApiBody` 装饰器和 `@ApiConsumes()` 为特定方法启用文件上传。以下是使用 [文件上传](/techniques/file-upload) 技术的完整示例：

```typescript
@UseInterceptors(FileInterceptor('file'))
@ApiConsumes('multipart/form-data')
@ApiBody({
  description: 'List of cats',
  type: FileUploadDto,
})
uploadFile(@UploadedFile() file: Express.Multer.File) {}
```

其中 `FileUploadDto` 定义如下：

```typescript
class FileUploadDto {
  @ApiProperty({ type: 'string', format: 'binary' })
  file: any;
}
```

要处理多个文件上传，你可以如下定义 `FilesUploadDto`：

```typescript
class FilesUploadDto {
  @ApiProperty({ type: 'array', items: { type: 'string', format: 'binary' } })
  files: any[];
}
```

#### 扩展

要为请求添加扩展，请使用 `@ApiExtension()` 装饰器。扩展名必须以 `x-` 开头。

```typescript
@ApiExtension('x-foo', { hello: 'world' })
```

#### 高级：通用 `ApiResponse`

通过提供 [原始定义](/openapi/types-and-parameters#raw-definitions) 的能力，我们可以在 Swagger UI 中定义通用模式。假设我们有以下 DTO：

```ts
export class PaginatedDto<TData> {
  @ApiProperty()
  total: number;

  @ApiProperty()
  limit: number;

  @ApiProperty()
  offset: number;

  results: TData[];
}
```

我们跳过了对 `results` 的装饰，因为我们将在稍后为其提供原始定义。现在，我们再定义另一个 DTO 并将其命名为 `CatDto`，如下所示：

```ts
export class CatDto {
  @ApiProperty()
  name: string;

  @ApiProperty()
  age: number;

  @ApiProperty()
  breed: string;
}
```

有了这些，我们可以如下定义一个 `PaginatedDto<CatDto>` 响应：

```ts
@ApiOkResponse({
  schema: {
    allOf: [
      { $ref: getSchemaPath(PaginatedDto) },
      {
        properties: {
          results: {
            type: 'array',
            items: { $ref: getSchemaPath(CatDto) },
          },
        },
      },
    ],
  },
})
async findAll(): Promise<PaginatedDto<CatDto>> {}
```

在此示例中，我们指定了响应将具有 `PaginatedDto` 的所有属性，并且 `results` 属性的类型为 `Array<CatDto>`。

- `getSchemaPath()` 是一个函数，它从 OpenAPI 规范文件中返回给定模型的 OpenAPI Schema 路径。
- `allOf` 是 OAS 3 提供的一个概念，用于覆盖各种与继承相关的用例。

最后，由于 `PaginatedDto` 未被任何控制器直接引用，`SwaggerModule` 还无法生成相应的模型定义。在这种情况下，我们必须将其作为 [额外模型](/openapi/types-and-parameters#extra-models) 添加。例如，我们可以在控制器级别使用 `@ApiExtraModels()` 装饰器，如下所示：

```ts
@Controller('cats')
@ApiExtraModels(PaginatedDto)
export class CatsController {}
```

如果你现在运行 Swagger，此特定端点生成的 `swagger.json` 应该具有以下响应定义：

```json
"responses": {
  "200": {
    "description": "",
    "content": {
      "application/json": {
        "schema": {
          "allOf": [
            {
              "$ref": "#/components/schemas/PaginatedDto"
            },
            {
              "properties": {
                "results": {
                  "$ref": "#/components/schemas/CatDto"
                }
              }
            }
          ]
        }
      }
    }
  }
}
```

为了使其可重用，我们可以为 `PaginatedDto` 创建一个自定义装饰器，如下所示：

```ts
export const ApiPaginatedResponse = <TModel extends Type<any>>(
  model: TModel,
) => {
  return applyDecorators(
    ApiExtraModels(PaginatedDto, model),
    ApiOkResponse({
      schema: {
        allOf: [
          { $ref: getSchemaPath(PaginatedDto) },
          {
            properties: {
              results: {
                type: 'array',
                items: { $ref: getSchemaPath(model) },
              },
            },
          },
        ],
      },
    }),
  );
};
```

> info **提示** `Type<any>` 接口和 `applyDecorators` 函数是从 `@nestjs/common` 包中导入的。

为了确保 `SwaggerModule` 将为我们模型生成定义，我们必须像之前在控制器中添加 `PaginatedDto` 一样，将其添加为额外模型。

完成此操作后，我们可以在端点上使用自定义的 `@ApiPaginatedResponse()` 装饰器：

```ts
@ApiPaginatedResponse(CatDto)
async findAll(): Promise<PaginatedDto<CatDto>> {}
```

对于客户端生成工具，这种方法在如何为客户端生成 `PaginatedResponse<TModel>` 时存在歧义。以下是一个客户端生成器对上述 `GET /` 端点的结果示例。

```typescript
// Angular
findAll(): Observable<{ total: number, limit: number, offset: number, results: CatDto[] }>
```

如你所见，这里的 **返回类型** 是模糊的。为了解决此问题，你可以在 `ApiPaginatedResponse` 的 `schema` 中添加一个 `title` 属性：

```typescript
export const ApiPaginatedResponse = <TModel extends Type<any>>(
  model: TModel,
) => {
  return applyDecorators(
    ApiOkResponse({
      schema: {
        title: `PaginatedResponseOf${model.name}`,
        allOf: [
          // ...
        ],
      },
    }),
  );
};
```

现在客户端生成工具的结果将变为：

```ts
// Angular
findAll(): Observable<PaginatedResponseOfCatDto>
```