### 管道（Pipes）

管道是一个使用 `@Injectable()` 装饰器标注的类，并实现了 `PipeTransform` 接口。

<figure>
  <img class="illustrative-image" src="/assets/Pipe_1.png" />
</figure>

管道通常有两种典型使用场景：

- **转换（Transformation）**：将输入数据转换为所需的格式（例如，将字符串转换为整数）
- **验证（Validation）**：评估输入数据，如果数据有效，则直接通过；否则抛出异常

在这两种情况下，管道都作用于由 <a href="controllers#route-parameters">控制器路由处理函数</a> 处理的参数。Nest 在调用方法之前插入一个管道，该管道接收传给方法的参数并对它们进行处理。任何转换或验证操作都在此时发生，之后使用（可能被转换过的）参数调用路由处理函数。

Nest 自带许多开箱即用的内置管道。你也可以构建自己的自定义管道。在本章中，我们将介绍内置管道，并展示如何将它们绑定到路由处理函数。然后我们将介绍几个自定义管道的示例，以展示如何从头开始构建一个管道。

> info **提示** 管道运行在异常区域（exceptions zone）内。这意味着当管道抛出异常时，该异常将由异常层处理（全局异常过滤器以及当前上下文中应用的任何[异常过滤器](/exception-filters)）。因此，当管道中抛出异常时，后续不会执行控制器方法。这为你提供了一种最佳实践方式，用于在系统边界处对外部来源的数据进行验证。

#### 内置管道

Nest 提供了许多开箱即用的内置管道：

- `ValidationPipe`
- `ParseIntPipe`
- `ParseFloatPipe`
- `ParseBoolPipe`
- `ParseArrayPipe`
- `ParseUUIDPipe`
- `ParseEnumPipe`
- `DefaultValuePipe`
- `ParseFilePipe`
- `ParseDatePipe`

这些管道都从 `@nestjs/common` 包中导出。

我们来看一个使用 `ParseIntPipe` 的简单示例。这是 **转换** 使用场景的一个示例，该管道确保方法处理函数的参数被转换为 JavaScript 整数（如果转换失败则抛出异常）。在本章稍后，我们将展示一个简单的自定义 `ParseIntPipe` 实现。下面的示例技术也适用于其他内置的转换管道（`ParseBoolPipe`、`ParseFloatPipe`、`ParseEnumPipe`、`ParseArrayPipe`、`ParseDatePipe` 和 `ParseUUIDPipe`，在本章中我们将它们统称为 `Parse*` 管道）。

#### 绑定管道

要使用管道，我们需要将管道类的实例绑定到适当的上下文中。在 `ParseIntPipe` 示例中，我们希望将管道与特定的路由处理方法关联，并确保它在方法调用之前执行。我们通过以下方式实现这一点，我们称之为在方法参数级别绑定管道：

```typescript
@Get(':id')
async findOne(@Param('id', ParseIntPipe) id: number) {
  return this.catsService.findOne(id);
}
```

这确保了以下两个条件之一成立：要么我们在 `findOne()` 方法中接收到的参数是一个数字（如 `this.catsService.findOne()` 的调用所期望的），要么在调用路由处理函数之前抛出异常。

例如，假设调用如下路由：

```bash
GET localhost:3000/abc
```

Nest 将抛出如下异常：

```json
{
  "statusCode": 400,
  "message": "Validation failed (numeric string is expected)",
  "error": "Bad Request"
}
```

该异常将阻止 `findOne()` 方法体的执行。

在上面的示例中，我们传递的是类（`ParseIntPipe`），而不是实例，让框架负责实例化并启用依赖注入。与管道和守卫一样，我们也可以传递一个就地实例。如果我们想通过传递选项来自定义内置管道的行为，则传递就地实例非常有用：

```typescript
@Get(':id')
async findOne(
  @Param('id', new ParseIntPipe({ errorHttpStatusCode: HttpStatus.NOT_ACCEPTABLE }))
  id: number,
) {
  return this.catsService.findOne(id);
}
```

其他转换管道（所有 **Parse\*** 管道）的绑定方式类似。这些管道都在验证路由参数、查询字符串参数和请求体值的上下文中工作。

例如，对于查询字符串参数：

```typescript
@Get()
async findOne(@Query('id', ParseIntPipe) id: number) {
  return this.catsService.findOne(id);
}
```

以下是一个使用 `ParseUUIDPipe` 来解析字符串参数并验证其是否为 UUID 的示例：

```typescript
@@filename()
@Get(':uuid')
async findOne(@Param('uuid', new ParseUUIDPipe()) uuid: string) {
  return this.catsService.findOne(uuid);
}
@@switch
@Get(':uuid')
@Bind(Param('uuid', new ParseUUIDPipe()))
async findOne(uuid) {
  return this.catsService.findOne(uuid);
}
```

> info **提示** 使用 `ParseUUIDPipe()` 时，它会解析版本 3、4 或 5 的 UUID。如果你只需要特定版本的 UUID，可以在管道选项中传入版本号。

我们已经看到了绑定各种 `Parse*` 内置管道的示例。绑定验证管道略有不同；我们将在下一节中讨论。

> info **提示** 也请参见 [验证技术](/techniques/validation) 以获取验证管道的详细示例。

#### 自定义管道

如前所述，你可以构建自己的自定义管道。虽然 Nest 提供了强大的内置 `ParseIntPipe` 和 `ValidationPipe`，但我们将从头开始构建它们的简单自定义版本，以了解自定义管道是如何构建的。

我们从一个简单的 `ValidationPipe` 开始。最初，我们让它接收一个输入值并立即返回相同的值，表现得像一个恒等函数。

```typescript
@@filename(validation.pipe)
import { PipeTransform, Injectable, ArgumentMetadata } from '@nestjs/common';

@Injectable()
export class ValidationPipe implements PipeTransform {
  transform(value: any, metadata: ArgumentMetadata) {
    return value;
  }
}
@@switch
import { Injectable } from '@nestjs/common';

@Injectable()
export class ValidationPipe {
  transform(value, metadata) {
    return value;
  }
}
```

> info **提示** `PipeTransform<T, R>` 是一个泛型接口，任何管道都必须实现该接口。泛型接口使用 `T` 表示输入值的类型，使用 `R` 表示 `transform()` 方法的返回类型。

每个管道都必须实现 `transform()` 方法以满足 `PipeTransform` 接口契约。此方法有两个参数：

- `value`
- `metadata`

`value` 参数是当前正在处理的方法参数（在它被路由处理函数接收之前），`metadata` 是当前正在处理的方法参数的元数据。元数据对象具有以下属性：

```typescript
export interface ArgumentMetadata {
  type: 'body' | 'query' | 'param' | 'custom';
  metatype?: Type<unknown>;
  data?: string;
}
```

这些属性描述了当前正在处理的参数。

<table>
  <tr>
    <td>
      <code>type</code>
    </td>
    <td>表示参数是 body
      <code>@Body()</code>、query
      <code>@Query()</code>、param
      <code>@Param()</code> 还是自定义参数（详见
      <a routerLink="/custom-decorators">此处</a>）。</td>
  </tr>
  <tr>
    <td>
      <code>metatype</code>
    </td>
    <td>
      提供参数的元类型，例如
      <code>String</code>。注意：如果你在路由处理函数方法签名中省略了类型声明，或者使用的是普通 JavaScript，则该值为
      <code>undefined</code>。
    </td>
  </tr>
  <tr>
    <td>
      <code>data</code>
    </td>
    <td>传递给装饰器的字符串，例如
      <code>@Body('string')</code>。如果装饰器括号为空，则为
      <code>undefined</code>。</td>
  </tr>
</table>

> warning **警告** TypeScript 接口在转译期间会消失。因此，如果一个方法参数的类型声明为接口而不是类，`metatype` 的值将是 `Object`。

#### 基于模式的验证

让我们让验证管道更有用一些。仔细看一下 `CatsController` 的 `create()` 方法，我们可能希望在调用服务方法之前确保 post 请求体对象是有效的。

```typescript
@@filename()
@Post()
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
@@switch
@Post()
async create(@Body() createCatDto) {
  this.catsService.create(createCatDto);
}
```

我们关注 `createCatDto` 请求体参数。它的类型是 `CreateCatDto`：

```typescript
@@filename(create-cat.dto)
export class CreateCatDto {
  name: string;
  age: number;
  breed: string;
}
```

我们想要确保任何对 `create` 方法的请求都包含一个有效的请求体。因此我们必须验证 `createCatDto` 对象的三个成员。我们可以在路由处理函数内部进行验证，但这不是理想的做法，因为它会违反 **单一职责原则（SRP）**。

另一种方法是创建一个 **验证器类** 并将任务委托给它。但这有一个缺点，我们必须记住在每个方法的开头都调用这个验证器。

如何创建验证中间件？这可能可行，但不幸的是，不可能创建一个可以在整个应用程序中所有上下文中使用的 **通用中间件**。这是因为中间件不了解 **执行上下文**，包括将要调用的处理函数及其任何参数。

当然，这正是管道设计的用途。让我们继续完善我们的验证管道。

<app-banner-courses></app-banner-courses>

#### 对象模式验证

有几种方法可以在一个干净、[DRY](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself) 的方式中进行对象验证。一种常见的方法是使用 **基于模式的** 验证。让我们尝试这种方法。

[Zod](https://zod.dev/) 库允许你以一种简单的方式创建模式，并具有可读性高的 API。让我们构建一个使用 Zod 模式的验证管道。

首先安装所需的包：

```bash
$ npm install --save zod
```

在下面的代码示例中，我们创建了一个简单的类，它接收一个模式作为构造函数参数。然后我们应用 `schema.parse()` 方法，该方法根据提供的模式验证我们的传入参数。

如前所述，**验证管道** 要么返回未更改的值，要么抛出异常。

在下一节中，你将看到我们如何使用 `@UsePipes()` 装饰器为给定的控制器方法提供适当的模式。这样做使我们的验证管道可以在不同上下文中重用，正如我们希望的那样。

```typescript
@@filename()
import { PipeTransform, ArgumentMetadata, BadRequestException } from '@nestjs/common';
import { ZodSchema  } from 'zod';

export class ZodValidationPipe implements PipeTransform {
  constructor(private schema: ZodSchema) {}

  transform(value: unknown, metadata: ArgumentMetadata) {
    try {
      const parsedValue = this.schema.parse(value);
      return parsedValue;
    } catch (error) {
      throw new BadRequestException('Validation failed');
    }
  }
}
@@switch
import { BadRequestException } from '@nestjs/common';

export class ZodValidationPipe {
  constructor(private schema) {}

  transform(value, metadata) {
    try {
      const parsedValue = this.schema.parse(value);
      return parsedValue;
    } catch (error) {
      throw new BadRequestException('Validation failed');
    }
  }
}
```

#### 绑定验证管道

前面我们已经看到了如何绑定转换管道（如 `ParseIntPipe` 和其他 `Parse*` 管道）。

绑定验证管道也非常简单。

在这种情况下，我们需要在方法调用级别绑定管道。在当前示例中，我们需要执行以下操作来使用 `ZodValidationPipe`：

1. 创建 `ZodValidationPipe` 的实例
2. 在管道的类构造函数中传入特定上下文的 Zod 模式
3. 将管道绑定到方法

Zod 模式示例：

```typescript
import { z } from 'zod';

export const createCatSchema = z
  .object({
    name: z.string(),
    age: z.number(),
    breed: z.string(),
  })
  .required();

export type CreateCatDto = z.infer<typeof createCatSchema>;
```

我们使用 `@UsePipes()` 装饰器来实现，如下所示：

```typescript
@@filename(cats.controller)
@Post()
@UsePipes(new ZodValidationPipe(createCatSchema))
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
@@switch
@Post()
@Bind(Body())
@UsePipes(new ZodValidationPipe(createCatSchema))
async create(createCatDto) {
  this.catsService.create(createCatDto);
}
```

> info **提示** `@UsePipes()` 装饰器从 `@nestjs/common` 包中导入。

> warning **警告** `zod` 库要求在你的 `tsconfig.json` 文件中启用 `strictNullChecks` 配置。

#### 类验证器

> warning **警告** 本节中的技术需要使用 TypeScript，如果你的应用使用的是普通 JavaScript，则不可用。

让我们看一下验证技术的另一种实现方式。

Nest 与 [class-validator](https://github.com/typestack/class-validator) 库配合良好。这个强大的库允许你使用基于装饰器的验证。基于装饰器的验证非常强大，特别是当与 Nest 的 **管道** 功能结合使用时，因为我们能够访问被处理属性的 `metatype`。在开始之前，我们需要安装所需的包：

```bash
$ npm i --save class-validator class-transformer
```

安装完成后，我们可以在 `CreateCatDto` 类中添加几个装饰器。这里我们看到了这种方法的一个显著优势：`CreateCatDto` 类仍然是我们 Post 请求体对象的唯一真实来源（而不必创建一个单独的验证类）。

```typescript
@@filename(create-cat.dto)
import { IsString, IsInt } from 'class-validator';

export class CreateCatDto {
  @IsString()
  name: string;

  @IsInt()
  age: number;

  @IsString()
  breed: string;
}
```

> info **提示** 有关 class-validator 装饰器的更多信息，请参见 [这里](https://github.com/typestack/class-validator#usage)。

现在我们可以创建一个使用这些注解的 `ValidationPipe` 类。

```typescript
@@filename(validation.pipe)
import { PipeTransform, Injectable, ArgumentMetadata, BadRequestException } from '@nestjs/common';
import { validate } from 'class-validator';
import { plainToInstance } from 'class-transformer';

@Injectable()
export class ValidationPipe implements PipeTransform<any> {
  async transform(value: any, { metatype }: ArgumentMetadata) {
    if (!metatype || !this.toValidate(metatype)) {
      return value;
    }
    const object = plainToInstance(metatype, value);
    const errors = await validate(object);
    if (errors.length > 0) {
      throw new BadRequestException('Validation failed');
    }
    return value;
  }

  private toValidate(metatype: Function): boolean {
    const types: Function[] = [String, Boolean, Number, Array, Object];
    return !types.includes(metatype);
  }
}
```

> info **提示** 再次提醒，你不必自己构建通用的验证管道，因为 Nest 已经提供了开箱即用的 `ValidationPipe`。内置的 `ValidationPipe` 提供了比本章中构建的示例更多的选项，本章的示例为了展示自定义管道的机制而保持基础。你可以在[这里](/techniques/validation)找到完整细节和大量示例。

> warning **注意** 我们上面使用了 [class-transformer](https://github.com/typestack/class-transformer) 库，它与 **class-validator** 是同一个作者开发的，因此它们配合得非常好。

让我们逐步理解这段代码。首先，注意 `transform()` 方法被标记为 `async`。这是可能的，因为 Nest 支持同步和 **异步** 管道。我们让这个方法异步是因为一些 class-validator 验证可以是异步的（使用 Promise）。

其次，我们使用解构来提取 `metatype` 字段（从 `ArgumentMetadata` 中提取该字段）到我们的 `metatype` 参数中。这只是获取完整的 `ArgumentMetadata` 后再分配 `metatype` 变量的简写形式。

第三，注意辅助函数 `toValidate()`。它的作用是在当前处理的参数是原生 JavaScript 类型时跳过验证步骤（这些类型不能附加验证装饰器，因此没有理由让它们通过验证步骤）。

第四，我们使用 class-transformer 的 `plainToInstance()` 函数将原始 JavaScript 参数对象转换为有类型的对象，以便我们可以应用验证。我们必须这样做的原因是，传入的 post 请求体对象在从网络请求反序列化后 **没有类型信息**（这是底层平台如 Express 的工作方式）。class-validator 需要使用我们之前为 DTO 定义的验证装饰器，因此我们需要执行此转换，以便将传入的请求体视为具有适当装饰的对象，而不是一个普通的对象。

最后，如前所述，由于这是一个 **验证管道**，它要么返回未更改的值，要么抛出异常。

最后一步是绑定 `ValidationPipe`。管道可以是参数作用域、方法作用域、控制器作用域或全局作用域。在我们基于 Zod 的验证管道示例中，我们看到了在方法级别绑定管道的示例。
在下面的示例中，我们将把管道实例绑定到路由处理函数的 `@Body()` 装饰器上，以便我们的管道在验证 post 请求体时被调用。

```typescript
@@filename(cats.controller)
@Post()
async create(
  @Body(new ValidationPipe()) createCatDto: CreateCatDto,
) {
  this.catsService.create(createCatDto);
}
```

参数作用域的管道在验证逻辑仅涉及一个指定参数时非常有用。

#### 全局作用域管道

由于 `ValidationPipe` 被设计为尽可能通用，我们可以通过将其设置为 **全局作用域** 的管道来充分发挥其效用，这样它将在整个应用程序中的每个控制器和每个路由处理函数上应用。

```typescript
@@filename(main)
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalPipes(new ValidationPipe());
  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```

> warning **注意** 对于 <a href="faq/hybrid-application">混合应用</a>，`useGlobalPipes()` 方法不会为网关和微服务设置管道。对于“标准”（非混合）微服务应用，`useGlobalPipes()` 确实会全局挂载管道。

全局管道在整个应用程序中使用，适用于每个控制器和每个路由处理函数。

请注意，在依赖注入方面，从任何模块外部注册的全局管道（如上面示例中的 `useGlobalPipes()`）无法注入依赖项，因为绑定是在任何模块上下文之外进行的。为了解决这个问题，你可以直接从任何模块设置一个全局管道，使用以下结构：

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { APP_PIPE } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_PIPE,
      useClass: ValidationPipe,
    },
  ],
})
export class AppModule {}
```

> info **提示** 使用此方法对管道进行依赖注入时，请注意无论在哪个模块中使用此结构，管道实际上是全局的。应该在哪里使用它？选择定义管道的模块（例如上面示例中的 `ValidationPipe`）。此外，`useClass` 不是处理自定义提供者注册的唯一方式。了解更多[这里](/fundamentals/custom-providers)。

#### 内置的 ValidationPipe

再次提醒，你不必自己构建通用的验证管道，因为 Nest 已经提供了开箱即用的 `ValidationPipe`。内置的 `ValidationPipe` 提供了比我们在本章中构建的示例更多的选项，本章的示例为了展示自定义管道的机制而保持基础。你可以在[这里](/techniques/validation)找到完整细节和大量示例。

#### 转换使用场景

验证并不是自定义管道的唯一用途。在本章开头，我们提到管道还可以将输入数据 **转换** 为所需格式。这是可能的，因为 `transform` 函数返回的值会完全覆盖参数的先前值。

这在什么时候有用？考虑客户端传递的数据需要在处理之前进行某些更改——例如将字符串转换为整数——以便路由处理函数能够正确处理。此外，某些必需的数据字段可能缺失，我们希望应用默认值。**转换管道** 可以在客户端请求和请求处理函数之间插入一个处理函数来完成这些功能。

下面是一个简单的 `ParseIntPipe` 示例，它负责将字符串解析为整数值。（如上所述，Nest 有一个更复杂的内置 `ParseIntPipe`；我们包含这个示例只是为了展示自定义转换管道的机制）。

```typescript
@@filename(parse-int.pipe)
import { PipeTransform, Injectable, ArgumentMetadata, BadRequestException } from '@nestjs/common';

@Injectable()
export class ParseIntPipe implements PipeTransform<string, number> {
  transform(value: string, metadata: ArgumentMetadata): number {
    const val = parseInt(value, 10);
    if (isNaN(val)) {
      throw new BadRequestException('Validation failed');
    }
    return val;
  }
}
@@switch
import { Injectable, BadRequestException } from '@nestjs/common';

@Injectable()
export class ParseIntPipe {
  transform(value, metadata) {
    const val = parseInt(value, 10);
    if (isNaN(val)) {
      throw new BadRequestException('Validation failed');
    }
    return val;
  }
}
```

然后我们可以将这个管道绑定到选定的参数上，如下所示：

```typescript
@@filename()
@Get(':id')
async findOne(@Param('id', new ParseIntPipe()) id) {
  return this.catsService.findOne(id);
}
@@switch
@Get(':id')
@Bind(Param('id', new ParseIntPipe()))
async findOne(id) {
  return this.catsService.findOne(id);
}
```

另一个有用的转换场景是使用请求中提供的 id 从数据库中选择一个 **现有用户** 实体：

```typescript
@@filename()
@Get(':id')
findOne(@Param('id', UserByIdPipe) userEntity: UserEntity) {
  return userEntity;
}
@@switch
@Get(':id')
@Bind(Param('id', UserByIdPipe))
findOne(userEntity) {
  return userEntity;
}
```

我们将这个管道的实现留给读者，但请注意，像所有其他转换管道一样，它接收一个输入值（一个 `id`）并返回一个输出值（一个 `UserEntity` 对象）。这可以通过将样板代码抽象到公共管道中，使你的代码更具声明性且更符合 [DRY](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself) 原则。

#### 提供默认值

`Parse*` 管道期望参数值是已定义的。它们在接收到 `null` 或 `undefined` 值时会抛出异常。为了允许端点处理缺失的查询字符串参数值，我们必须在 `Parse*` 管道操作这些值之前提供一个默认值注入。`DefaultValuePipe` 正是为此目的而设计的。只需在 `@Query()` 装饰器中实例化一个 `DefaultValuePipe`，然后在相关 `Parse*` 管道之前使用它，如下所示：

```typescript
@@filename()
@Get()
async findAll(
  @Query('activeOnly', new DefaultValuePipe(false), ParseBoolPipe) activeOnly: boolean,
  @Query('page', new DefaultValuePipe(0), ParseIntPipe) page: number,
) {
  return this.catsService.findAll({ activeOnly, page });
}
```