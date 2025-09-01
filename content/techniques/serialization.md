### 序列化

序列化是在对象通过网络响应返回之前进行的一个过程。这是提供数据转换和清理规则的合适位置。例如，像密码这样的敏感数据应始终从响应中排除。或者，某些属性可能需要额外的转换，例如仅发送实体的子集属性。手动执行这些转换可能会很繁琐且容易出错，并且可能无法确保覆盖所有情况。

#### 概述

Nest 提供了一个内置功能，帮助确保这些操作能够以简单的方式执行。`ClassSerializerInterceptor` 拦截器使用强大的 [class-transformer](https://github.com/typestack/class-transformer) 包，提供一种声明式且可扩展的方式来转换对象。它执行的基本操作是获取方法处理器返回的值，并对其应用 [class-transformer](https://github.com/typestack/class-transformer) 中的 `instanceToPlain()` 函数。在此过程中，它可以应用实体/DTO 类上通过 `class-transformer` 装饰器表达的规则，如下所述。

> info **提示** 序列化不适用于 [StreamableFile](https://docs.nestjs.com/techniques/streaming-files#streamable-file-class) 响应。

#### 排除属性

假设我们想要自动排除用户实体中的 `password` 属性。我们按如下方式注解实体：

```typescript
import { Exclude } from 'class-transformer';

export class UserEntity {
  id: number;
  firstName: string;
  lastName: string;

  @Exclude()
  password: string;

  constructor(partial: Partial<UserEntity>) {
    Object.assign(this, partial);
  }
}
```

现在考虑一个控制器，其方法处理器返回此类的实例：

```typescript
@UseInterceptors(ClassSerializerInterceptor)
@Get()
findOne(): UserEntity {
  return new UserEntity({
    id: 1,
    firstName: 'John',
    lastName: 'Doe',
    password: 'password',
  });
}
```

> **警告** 注意我们必须返回类的实例。如果你返回的是普通的 JavaScript 对象，例如 `{{ '{' }} user: new UserEntity() {{ '}' }}`，则该对象将无法正确序列化。

> info **提示** `ClassSerializerInterceptor` 从 `@nestjs/common` 导入。

当请求此端点时，客户端将收到以下响应：

```json
{
  "id": 1,
  "firstName": "John",
  "lastName": "Doe"
}
```

请注意，拦截器可以全局应用（如[此处](https://docs.nestjs.com/interceptors#binding-interceptors)所述）。拦截器与实体类声明的结合确保了任何返回 `UserEntity` 的方法都会删除 `password` 属性。这使您可以集中统一地执行该业务规则。

#### 暴露属性

你可以使用 `@Expose()` 装饰器为属性提供别名，或者执行一个函数来计算属性值（类似于 **getter** 方法），如下所示：

```typescript
@Expose()
get fullName(): string {
  return `${this.firstName} ${this.lastName}`;
}
```

#### 转换

你可以使用 `@Transform()` 装饰器执行额外的数据转换。例如，以下结构返回 `RoleEntity` 的 name 属性，而不是返回整个对象：

```typescript
@Transform(({ value }) => value.name)
role: RoleEntity;
```

#### 传递选项

你可以通过 `@SerializeOptions()` 装饰器传入选项对象来修改转换函数的默认行为。

```typescript
@SerializeOptions({
  excludePrefixes: ['_'],
})
@Get()
findOne(): UserEntity {
  return new UserEntity();
}
```

> info **提示** `@SerializeOptions()` 装饰器从 `@nestjs/common` 导入。

通过 `@SerializeOptions()` 传入的选项会作为第二个参数传递给底层的 `instanceToPlain()` 函数。在这个例子中，我们自动排除所有以 `_` 前缀开头的属性。

#### 转换普通对象

你可以通过使用 `@SerializeOptions` 装饰器在控制器级别强制执行转换。这确保了所有响应都会转换为指定类的实例，并应用 class-validator 或 class-transformer 的任何装饰器，即使返回的是普通对象也是如此。这种方法使得代码更简洁，无需重复实例化类或调用 `plainToInstance`。

在下面的示例中，尽管在两个条件分支中都返回了普通的 JavaScript 对象，它们将自动转换为 `UserEntity` 实例，并应用相关装饰器：

```typescript
@UseInterceptors(ClassSerializerInterceptor)
@SerializeOptions({ type: UserEntity })
@Get()
findOne(@Query() { id }: { id: number }): UserEntity {
  if (id === 1) {
    return {
      id: 1,
      firstName: 'John',
      lastName: 'Doe',
      password: 'password',
    };
  }

  return {
    id: 2,
    firstName: 'Kamil',
    lastName: 'Mysliwiec',
    password: 'password2',
  };
}
```

> info **提示** 通过指定控制器的预期返回类型，你可以利用 TypeScript 的类型检查功能，确保返回的普通对象符合 DTO 或实体的结构。`plainToInstance` 函数不提供这种级别的类型提示，如果普通对象与预期的 DTO 或实体结构不匹配，可能导致潜在的错误。

#### 示例

一个可运行的示例请参见 [这里](https://github.com/nestjs/nest/tree/master/sample/21-serializer)。

#### WebSockets 和微服务

虽然本章展示的是使用 HTTP 风格应用（例如 Express 或 Fastify）的示例，但 `ClassSerializerInterceptor` 对 WebSockets 和微服务同样有效，无论使用何种传输方式。

#### 了解更多

有关 `class-transformer` 包提供的可用装饰器和选项的更多信息，请参见 [这里](https://github.com/typestack/class-transformer)。