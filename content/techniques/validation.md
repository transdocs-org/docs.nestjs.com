### 验证

验证发送到 Web 应用程序中的任何数据的正确性是一种最佳实践。为了自动验证传入的请求，Nest 提供了几个开箱即用的管道：

- `ValidationPipe`
- `ParseIntPipe`
- `ParseBoolPipe`
- `ParseArrayPipe`
- `ParseUUIDPipe`

`ValidationPipe` 利用了强大的 [class-validator](https://github.com/typestack/class-validator) 包及其声明式验证装饰器。`ValidationPipe` 提供了一种便捷的方法，为所有传入的客户端负载强制执行验证规则，这些规则在每个模块的本地类/DTO声明中通过简单的注解进行声明。

#### 概述

在 [Pipes](/pipes) 章节中，我们介绍了如何构建简单的管道，并将它们绑定到控制器、方法或全局应用来演示其工作流程。请务必回顾该章节以更好地理解本章节的内容。在这里，我们将专注于 `ValidationPipe` 的各种**实际用例**，并展示如何使用它的一些高级自定义功能。

#### 使用内置的 ValidationPipe

要开始使用它，我们首先安装所需的依赖项。

```bash
$ npm i --save class-validator class-transformer
```

> info **提示** `ValidationPipe` 从 `@nestjs/common` 包中导出。

由于此管道使用了 [`class-validator`](https://github.com/typestack/class-validator) 和 [`class-transformer`](https://github.com/typestack/class-transformer) 库，因此提供了许多可选配置。你可以通过传递给管道的配置对象来设置这些选项。以下是内置的选项：

```typescript
export interface ValidationPipeOptions extends ValidatorOptions {
  transform?: boolean;
  disableErrorMessages?: boolean;
  exceptionFactory?: (errors: ValidationError[]) => any;
}
```

除了这些选项外，所有 `class-validator` 的选项（继承自 `ValidatorOptions` 接口）都可用：

<table>
  <tr>
    <th>选项</th>
    <th>类型</th>
    <th>描述</th>
  </tr>
  <tr>
    <td><code>enableDebugMessages</code></td>
    <td><code>boolean</code></td>
    <td>如果设置为 true，验证器将在控制台打印额外的警告信息，当某些地方出错时。</td>
  </tr>
  <tr>
    <td><code>skipUndefinedProperties</code></td>
    <td><code>boolean</code></td>
    <td>如果设置为 true，验证器将跳过验证对象中值为 undefined 的属性。</td>
  </tr>
  <tr>
    <td><code>skipNullProperties</code></td>
    <td><code>boolean</code></td>
    <td>如果设置为 true，验证器将跳过验证对象中值为 null 的属性。</td>
  </tr>
  <tr>
    <td><code>skipMissingProperties</code></td>
    <td><code>boolean</code></td>
    <td>如果设置为 true，验证器将跳过验证对象中值为 null 或 undefined 的属性。</td>
  </tr>
  <tr>
    <td><code>whitelist</code></td>
    <td><code>boolean</code></td>
    <td>如果设置为 true，验证器将从验证（返回）对象中移除所有未使用任何验证装饰器的属性。</td>
  </tr>
  <tr>
    <td><code>forbidNonWhitelisted</code></td>
    <td><code>boolean</code></td>
    <td>如果设置为 true，验证器将不会移除非白名单属性，而是抛出异常。</td>
  </tr>
  <tr>
    <td><code>forbidUnknownValues</code></td>
    <td><code>boolean</code></td>
    <td>如果设置为 true，尝试验证未知对象将立即失败。</td>
  </tr>
  <tr>
    <td><code>disableErrorMessages</code></td>
    <td><code>boolean</code></td>
    <td>如果设置为 true，验证错误将不会返回给客户端。</td>
  </tr>
  <tr>
    <td><code>errorHttpStatusCode</code></td>
    <td><code>number</code></td>
    <td>该设置允许你指定在发生错误时抛出的异常类型。默认情况下它抛出 <code>BadRequestException</code>。</td>
  </tr>
  <tr>
    <td><code>exceptionFactory</code></td>
    <td><code>Function</code></td>
    <td>接收一个验证错误数组并返回一个要抛出的异常对象。</td>
  </tr>
  <tr>
    <td><code>groups</code></td>
    <td><code>string[]</code></td>
    <td>在对象验证期间要使用的组。</td>
  </tr>
  <tr>
    <td><code>always</code></td>
    <td><code>boolean</code></td>
    <td>设置装饰器中 <code>always</code> 选项的默认值。可以在装饰器选项中覆盖默认值。</td>
  </tr>
  <tr>
    <td><code>strictGroups</code></td>
    <td><code>boolean</code></td>
    <td>如果 <code>groups</code> 未提供或为空，忽略至少包含一个组的装饰器。</td>
  </tr>
  <tr>
    <td><code>dismissDefaultMessages</code></td>
    <td><code>boolean</code></td>
    <td>如果设置为 true，验证将不使用默认消息。错误消息在未显式设置时始终为 <code>undefined</code>。</td>
  </tr>
  <tr>
    <td><code>validationError.target</code></td>
    <td><code>boolean</code></td>
    <td>指示是否在 <code>ValidationError</code> 中暴露目标。</td>
  </tr>
  <tr>
    <td><code>validationError.value</code></td>
    <td><code>boolean</code></td>
    <td>指示是否在 <code>ValidationError</code> 中暴露验证值。</td>
  </tr>
  <tr>
    <td><code>stopAtFirstError</code></td>
    <td><code>boolean</code></td>
    <td>当设置为 true 时，验证给定属性时在遇到第一个错误后停止。默认为 false。</td>
  </tr>
</table>

> info **注意** 有关 `class-validator` 包的更多信息，请查看其 [仓库](https://github.com/typestack/class-validator)。

#### 自动验证

我们首先将 `ValidationPipe` 绑定到应用程序级别，从而确保所有端点都免受不正确数据的影响。

```typescript
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalPipes(new ValidationPipe());
  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```

为了测试我们的管道，让我们创建一个基本端点。

```typescript
@Post()
create(@Body() createUserDto: CreateUserDto) {
  return 'This action adds a new user';
}
```

> info **提示** 由于 TypeScript 不会存储有关**泛型或接口**的元数据，因此当在 DTO 中使用它们时，`ValidationPipe` 可能无法正确验证传入的数据。因此，请考虑在 DTO 中使用具体类。

> info **Hint** 在导入 DTO 时，不能使用仅类型导入，因为它们在运行时会被擦除，即请记住使用 `import {{ '{' }} CreateUserDto {{ '}' }}` 而不是 `import type {{ '{' }} CreateUserDto {{ '}' }}`。

现在我们可以在 `CreateUserDto` 中添加一些验证规则。我们使用 `class-validator` 包提供的装饰器来完成此操作，详细信息请参见 [这里](https://github.com/typestack/class-validator#validation-decorators)。这样，任何使用 `CreateUserDto` 的路由都会自动强制执行这些验证规则。

```typescript
import { IsEmail, IsNotEmpty } from 'class-validator';

export class CreateUserDto {
  @IsEmail()
  email: string;

  @IsNotEmpty()
  password: string;
}
```

有了这些规则后，如果一个请求命中我们的端点，并且请求体中的 `email` 属性无效，应用程序将自动以 `400 Bad Request` 状态码和以下响应体进行响应：

```json
{
  "statusCode": 400,
  "error": "Bad Request",
  "message": ["email must be an email"]
}
```

除了验证请求体之外，`ValidationPipe` 还可用于其他请求对象属性。假设我们希望在端点路径中接受 `:id`。为了确保只接受此请求参数的数字，我们可以使用以下构造：

```typescript
@Get(':id')
findOne(@Param() params: FindOneParams) {
  return 'This action returns a user';
}
```

`FindOneParams` 与 DTO 一样，只是一个使用 `class-validator` 定义验证规则的类。它看起来像这样：

```typescript
import { IsNumberString } from 'class-validator';

export class FindOneParams {
  @IsNumberString()
  id: string;
}
```

#### 禁用详细错误

错误消息有助于解释请求中哪里出错了。但是，某些生产环境可能希望禁用详细错误。可以通过向 `ValidationPipe` 传递一个选项对象来实现：

```typescript
app.useGlobalPipes(
  new ValidationPipe({
    disableErrorMessages: true,
  }),
);
```

结果是详细错误消息将不会显示在响应体中。

#### 移除属性

我们的 `ValidationPipe` 还可以过滤掉方法处理程序不应接收的属性。在这种情况下，我们可以**白名单**允许的属性，并且任何未包含在白名单中的属性将自动从结果对象中移除。例如，如果我们的处理程序期望 `email` 和 `password` 属性，但请求还包括一个 `age` 属性，这个属性可以自动从生成的 DTO 中移除。要启用此行为，请将 `whitelist` 设置为 `true`。

```typescript
app.useGlobalPipes(
  new ValidationPipe({
    whitelist: true,
  }),
);
```

当设置为 true 时，这将自动移除非白名单属性（那些在验证类中没有任何装饰器的属性）。

或者，你可以在存在非白名单属性时停止请求的处理，并向用户返回错误响应。要启用此功能，请将 `forbidNonWhitelisted` 选项设置为 `true`，同时将 `whitelist` 设置为 `true`。

<app-banner-courses></app-banner-courses>

#### 转换负载对象

通过网络传入的负载是普通的 JavaScript 对象。`ValidationPipe` 可以自动将负载转换为根据其 DTO 类型化的对象。要启用自动转换，请将 `transform` 设置为 `true`。这可以在方法级别进行：

```typescript
@@filename(cats.controller)
@Post()
@UsePipes(new ValidationPipe({ transform: true }))
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

要全局启用此行为，请在全局管道上设置选项：

```typescript
app.useGlobalPipes(
  new ValidationPipe({
    transform: true,
  }),
);
```

启用自动转换选项后，`ValidationPipe` 还将执行原始类型转换。在以下示例中，`findOne()` 方法接受一个参数，该参数表示提取的 `id` 路径参数：

```typescript
@Get(':id')
findOne(@Param('id') id: number) {
  console.log(typeof id === 'number'); // true
  return 'This action returns a user';
}
```

默认情况下，每个路径参数和查询参数都是以 `string` 类型通过网络传输的。在上面的示例中，我们将 `id` 类型指定为 `number`（在方法签名中）。因此，`ValidationPipe` 将尝试自动将字符串标识符转换为数字。

#### 显式转换

在上一节中，我们展示了 `ValidationPipe` 如何根据预期类型隐式转换查询和路径参数。但是，此功能需要启用自动转换。

或者（在禁用自动转换的情况下），你可以使用 `ParseIntPipe` 或 `ParseBoolPipe` 显式转换值（注意 `ParseStringPipe` 是不需要的，因为如前所述，默认情况下每个路径参数和查询参数都是以 `string` 类型传输的）。

```typescript
@Get(':id')
findOne(
  @Param('id', ParseIntPipe) id: number,
  @Query('sort', ParseBoolPipe) sort: boolean,
) {
  console.log(typeof id === 'number'); // true
  console.log(typeof sort === 'boolean'); // true
  return 'This action returns a user';
}
```

> info **提示** `ParseIntPipe` 和 `ParseBoolPipe` 从 `@nestjs/common` 包中导出。

#### 映射类型

当你构建 **CRUD**（创建/读取/更新/删除）等功能时，通常需要构建基础实体类型的变体。Nest 提供了多个实用函数来执行类型转换，使此任务更加方便。

> **警告** 如果你的应用程序使用了 `@nestjs/swagger` 包，请参见 [本章节](/openapi/mapped-types) 获取有关映射类型的更多信息。同样，如果你使用了 `@nestjs/graphql` 包，请参见 [本章节](/graphql/mapped-types)。这两个包都严重依赖类型，因此它们需要使用不同的导入。因此，如果你使用了 `@nestjs/mapped-types`（而不是适当的 `@nestjs/swagger` 或 `@nestjs/graphql`，具体取决于你的应用程序类型），你可能会遇到各种未记录的副作用。

在构建输入验证类型（也称为 DTO）时，构建同一类型的 **create** 和 **update** 变体通常很有用。例如，**create** 变体可能需要所有字段，而 **update** 变体可能使所有字段变为可选。

Nest 提供了 `PartialType()` 实用函数，使此任务更简单并减少样板代码。

`PartialType()` 函数返回一个类型（类），其所有属性都设置为可选。例如，假设我们有如下 **create** 类型：

```typescript
export class CreateCatDto {
  name: string;
  age: number;
  breed: string;
}
```

默认情况下，所有这些字段都是必需的。要创建一个具有相同字段但每个字段都是可选的类型，请使用 `PartialType()`，并传入类引用（`CreateCatDto`）作为参数：

```typescript
export class UpdateCatDto extends PartialType(CreateCatDto) {}
```

> info **提示** `PartialType()` 函数从 `@nestjs/mapped-types` 包中导入。

`PickType()` 函数通过从输入类型中选择一组属性来构造一个新类型（类）。例如，假设我们从如下类型开始：

```typescript
export class CreateCatDto {
  name: string;
  age: number;
  breed: string;
}
```

我们可以使用 `PickType()` 实用函数从该类中选择一组属性：

```typescript
export class UpdateCatAgeDto extends PickType(CreateCatDto, ['age'] as const) {}
```

> info **提示** `PickType()` 函数从 `@nestjs/mapped-types` 包中导入。

`OmitType()` 函数通过从输入类型中选择所有属性，然后移除一组特定的键来构造一个类型。例如，假设我们从如下类型开始：

```typescript
export class CreateCatDto {
  name: string;
  age: number;
  breed: string;
}
```

我们可以生成一个派生类型，它具有除 `name` 外的所有属性，如下所示。在此构造中，`OmitType` 的第二个参数是一个属性名称数组。

```typescript
export class UpdateCatDto extends OmitType(CreateCatDto, ['name'] as const) {}
```

> info **提示** `OmitType()` 函数从 `@nestjs/mapped-types` 包中导入。

`IntersectionType()` 函数将两个类型合并为一个新类型（类）。例如，假设我们从如下两个类型开始：

```typescript
export class CreateCatDto {
  name: string;
  breed: string;
}

export class AdditionalCatInfo {
  color: string;
}
```

我们可以生成一个新类型，包含两个类型中的所有属性。

```typescript
export class UpdateCatDto extends IntersectionType(
  CreateCatDto,
  AdditionalCatInfo,
) {}
```

> info **提示** `IntersectionType()` 函数从 `@nestjs/mapped-types` 包中导入。

类型映射实用函数是可组合的。例如，以下代码将生成一个类型（类），它具有 `CreateCatDto` 类型的所有属性（除了 `name`），并且这些属性将被设置为可选：

```typescript
export class UpdateCatDto extends PartialType(
  OmitType(CreateCatDto, ['name'] as const),
) {}
```

#### 解析和验证数组

TypeScript 不会存储有关泛型或接口的元数据，因此当在 DTO 中使用它们时，`ValidationPipe` 可能无法正确验证传入的数据。例如，在以下代码中，`createUserDtos` 将不会被正确验证：

```typescript
@Post()
createBulk(@Body() createUserDtos: CreateUserDto[]) {
  return 'This action adds new users';
}
```

要验证数组，请创建一个专用类，该类包含包装数组的属性，或者使用 `ParseArrayPipe`。

```typescript
@Post()
createBulk(
  @Body(new ParseArrayPipe({ items: CreateUserDto }))
  createUserDtos: CreateUserDto[],
) {
  return 'This action adds new users';
}
```

此外，`ParseArrayPipe` 在解析查询参数时也可能派上用场。我们考虑一个 `findByIds()` 方法，它根据作为查询参数传递的标识符返回用户。

```typescript
@Get()
findByIds(
  @Query('ids', new ParseArrayPipe({ items: Number, separator: ',' }))
  ids: number[],
) {
  return 'This action returns users by ids';
}
```

这种构造验证来自如下 HTTP `GET` 请求的查询参数：

```bash
GET /?ids=1,2,3
```

#### WebSockets 和微服务

虽然本章展示了使用 HTTP 风格应用程序（例如 Express 或 Fastify）的示例，但 `ValidationPipe` 对于 WebSockets 和微服务同样有效，无论使用何种传输方式。

#### 了解更多

阅读更多关于 `class-validator` 包提供的自定义验证器、错误消息和可用装饰器的信息，请参见 [这里](https://github.com/typestack/class-validator)。