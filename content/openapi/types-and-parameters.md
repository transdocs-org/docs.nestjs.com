### 类型和参数

`SwaggerModule` 会搜索路由处理程序中的所有 `@Body()`、`@Query()` 和 `@Param()` 装饰器来生成 API 文档。它还会利用反射创建相应的模型定义。请看以下代码：

```typescript
@Post()
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

> info **提示**：要显式设置请求体定义，请使用 `@ApiBody()` 装饰器（从 `@nestjs/swagger` 包导入）。

根据 `CreateCatDto`，将生成如下模型定义的 Swagger UI：

<figure><img src="/assets/swagger-dto.png" /></figure>

如您所见，尽管类中声明了一些属性，但定义仍然是空的。为了让 `SwaggerModule` 看到类的属性，我们需要使用 `@ApiProperty()` 装饰器标注这些属性，或者使用 CLI 插件（在 **Plugin** 部分有更多说明），该插件会自动完成此操作：

```typescript
import { ApiProperty } from '@nestjs/swagger';

export class CreateCatDto {
  @ApiProperty()
  name: string;

  @ApiProperty()
  age: number;

  @ApiProperty()
  breed: string;
}
```

> info **提示**：可以考虑使用 Swagger 插件（参见 [Plugin](/openapi/cli-plugin) 部分），它将自动为您完成这些操作，而不必手动为每个属性添加注解。

现在，我们打开浏览器，查看生成的 `CreateCatDto` 模型：

<figure><img src="/assets/swagger-dto2.png" /></figure>

此外，`@ApiProperty()` 装饰器还允许设置各种 [Schema Object](https://swagger.io/specification/#schemaObject) 属性：

```typescript
@ApiProperty({
  description: 'The age of a cat',
  minimum: 1,
  default: 1,
})
age: number;
```

> info **提示**：可以使用 `@ApiPropertyOptional()` 快捷装饰器代替显式指定 `{{"@ApiProperty({ required: false })"}}`。

要显式设置属性的类型，请使用 `type` 键：

```typescript
@ApiProperty({
  type: Number,
})
age: number;
```

#### 数组

当属性是数组时，我们必须手动指示数组类型，如下所示：

```typescript
@ApiProperty({ type: [String] })
names: string[];
```

> info **提示**：可以考虑使用 Swagger 插件（参见 [Plugin](/openapi/cli-plugin) 部分），它将自动检测数组。

可以将类型作为数组的第一个元素（如上所示），或者将 `isArray` 属性设置为 `true`。

<app-banner-enterprise></app-banner-enterprise>

#### 循环依赖

当类之间存在循环依赖时，请使用惰性函数为 `SwaggerModule` 提供类型信息：

```typescript
@ApiProperty({ type: () => Node })
node: Node;
```

> info **提示**：可以考虑使用 Swagger 插件（参见 [Plugin](/openapi/cli-plugin) 部分），它将自动检测循环依赖。

#### 泛型和接口

由于 TypeScript 不会存储有关泛型或接口的元数据，当您在 DTO 中使用它们时，`SwaggerModule` 可能在运行时无法正确生成模型定义。例如，以下代码将无法被 Swagger 模块正确检查：

```typescript
createBulk(@Body() usersDto: CreateUserDto[])
```

为克服此限制，您可以显式设置类型：

```typescript
@ApiBody({ type: [CreateUserDto] })
createBulk(@Body() usersDto: CreateUserDto[])
```

#### 枚举

要标识一个 `enum`，我们必须手动在 `@ApiProperty` 上使用包含值数组的 `enum` 属性。

```typescript
@ApiProperty({ enum: ['Admin', 'Moderator', 'User']})
role: UserRole;
```

或者，定义一个实际的 TypeScript 枚举，如下所示：

```typescript
export enum UserRole {
  Admin = 'Admin',
  Moderator = 'Moderator',
  User = 'User',
}
```

然后，您可以将该枚举直接与 `@Query()` 参数装饰器一起使用，并结合 `@ApiQuery()` 装饰器。

```typescript
@ApiQuery({ name: 'role', enum: UserRole })
async filterByRole(@Query('role') role: UserRole = UserRole.User) {}
```

<figure><img src="/assets/enum_query.gif" /></figure>

当 `isArray` 设置为 **true** 时，`enum` 可以作为 **多选框** 使用：

<figure><img src="/assets/enum_query_array.gif" /></figure>

#### 枚举模式

默认情况下，`enum` 属性会在 `parameter` 上添加 [Enum](https://swagger.io/docs/specification/data-models/enums/) 的原始定义。

```yaml
- breed:
    type: 'string'
    enum:
      - Persian
      - Tabby
      - Siamese
```

以上规范在大多数情况下都适用。但是，如果您使用的是将规范作为 **输入** 并生成 **客户端** 代码的工具，那么生成的代码中可能会出现重复的 `enums`。请考虑以下代码片段：

```typescript
// 生成的客户端代码
export class CatDetail {
  breed: CatDetailEnum;
}

export class CatInformation {
  breed: CatInformationEnum;
}

export enum CatDetailEnum {
  Persian = 'Persian',
  Tabby = 'Tabby',
  Siamese = 'Siamese',
}

export enum CatInformationEnum {
  Persian = 'Persian',
  Tabby = 'Tabby',
  Siamese = 'Siamese',
}
```

> info **提示**：上面的代码片段是使用名为 [NSwag](https://github.com/RicoSuter/NSwag) 的工具生成的。

您可以看到现在有两个完全相同的 `enums`。
为了解决这个问题，您可以在装饰器中传递 `enumName` 和 `enum` 属性。

```typescript
export class CatDetail {
  @ApiProperty({ enum: CatBreed, enumName: 'CatBreed' })
  breed: CatBreed;
}
```

`enumName` 属性使 `@nestjs/swagger` 能够将 `CatBreed` 转换为其自己的 `schema`，从而使 `CatBreed` 枚举可复用。规范将如下所示：

```yaml
CatDetail:
  type: 'object'
  properties:
    ...
    - breed:
        schema:
          $ref: '#/components/schemas/CatBreed'
CatBreed:
  type: string
  enum:
    - Persian
    - Tabby
    - Siamese
```

> info **提示**：任何将 `enum` 作为属性的 **装饰器** 也将接受 `enumName`。

#### 属性值示例

您可以通过使用 `example` 键为属性设置单个示例，如下所示：

```typescript
@ApiProperty({
  example: 'persian',
})
breed: string;
```

如果您想提供多个示例，可以使用 `examples` 键并传入如下结构的对象：

```typescript
@ApiProperty({
  examples: {
    Persian: { value: 'persian' },
    Tabby: { value: 'tabby' },
    Siamese: { value: 'siamese' },
    'Scottish Fold': { value: 'scottish_fold' },
  },
})
breed: string;
```

#### 原始定义

在某些情况下（例如深度嵌套的数组或矩阵），您可能需要手动定义类型：

```typescript
@ApiProperty({
  type: 'array',
  items: {
    type: 'array',
    items: {
      type: 'number',
    },
  },
})
coords: number[][];
```

还可以像这样指定原始对象模式：

```typescript
@ApiProperty({
  type: 'object',
  properties: {
    name: {
      type: 'string',
      example: 'Error'
    },
    status: {
      type: 'number',
      example: 400
    }
  },
  required: ['name', 'status']
})
rawDefinition: Record<string, any>;
```

要在控制器类中手动定义输入/输出内容，请使用 `schema` 属性：

```typescript
@ApiBody({
  schema: {
    type: 'array',
    items: {
      type: 'array',
      items: {
        type: 'number',
      },
    },
  },
})
async create(@Body() coords: number[][]) {}
```

#### 额外模型

要定义未在控制器中直接引用但应由 Swagger 模块检查的额外模型，请使用 `@ApiExtraModels()` 装饰器：

```typescript
@ApiExtraModels(ExtraModel)
export class CreateCatDto {}
```

> info **提示**：您只需对特定模型类使用一次 `@ApiExtraModels()`。

或者，您可以将一个带有 `extraModels` 属性的选项对象传递给 `SwaggerModule.createDocument()` 方法，如下所示：

```typescript
const documentFactory = () =>
  SwaggerModule.createDocument(app, options, {
    extraModels: [ExtraModel],
  });
```

要获取对模型的引用（`$ref`），请使用 `getSchemaPath(ExtraModel)` 函数：

```typescript
'application/vnd.api+json': {
   schema: { $ref: getSchemaPath(ExtraModel) },
},
```

#### oneOf、anyOf、allOf

要组合模式，您可以使用 `oneOf`、`anyOf` 或 `allOf` 关键字（[阅读更多](https://swagger.io/docs/specification/data-models/oneof-anyof-allof-not/)）。

```typescript
@ApiProperty({
  oneOf: [
    { $ref: getSchemaPath(Cat) },
    { $ref: getSchemaPath(Dog) },
  ],
})
pet: Cat | Dog;
```

如果您想定义一个具有多态性的数组（即成员跨越多个模式的数组），则应使用原始定义（见上文）手动定义您的类型。

```typescript
type Pet = Cat | Dog;

@ApiProperty({
  type: 'array',
  items: {
    oneOf: [
      { $ref: getSchemaPath(Cat) },
      { $ref: getSchemaPath(Dog) },
    ],
  },
})
pets: Pet[];
```

> info **提示**：`getSchemaPath()` 函数从 `@nestjs/swagger` 导入。

`Cat` 和 `Dog` 都必须使用 `@ApiExtraModels()` 装饰器（在类级别）定义为额外模型。

#### 模式名称和描述

您可能已经注意到，生成的模式名称基于原始模型类的名称（例如，`CreateCatDto` 模型生成 `CreateCatDto` 模式）。如果您想更改模式名称，可以使用 `@ApiSchema()` 装饰器。

示例如下：

```typescript
@ApiSchema({ name: 'CreateCatRequest' })
class CreateCatDto {}
```

上述模型将被转换为 `CreateCatRequest` 模式。

默认情况下，生成的模式中不会添加任何描述。您可以使用 `description` 属性添加描述：

```typescript
@ApiSchema({ description: 'Description of the CreateCatDto schema' })
class CreateCatDto {}
```

这样，描述将包含在模式中，如下所示：

```yaml
schemas:
  CreateCatDto:
    type: object
    description: Description of the CreateCatDto schema
```