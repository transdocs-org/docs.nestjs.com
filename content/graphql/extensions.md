### 扩展

> warning **警告** 本章节仅适用于代码优先的方法。

扩展是一个**高级、低级别的功能**，它允许你在类型配置中定义任意的数据。在字段上附加自定义元数据可以让你创建更复杂、通用的解决方案。例如，通过扩展，你可以定义访问特定字段所需的字段级角色。这些角色可以在运行时被解析，以确定调用者是否具有检索特定字段的足够权限。

#### 添加自定义元数据

要为字段附加自定义元数据，请使用从 `@nestjs/graphql` 包导出的 `@Extensions()` 装饰器。

```typescript
@Field()
@Extensions({ role: Role.ADMIN })
password: string;
```

在上面的示例中，我们为 `role` 元数据属性赋值了 `Role.ADMIN`。`Role` 是一个简单的 TypeScript 枚举，用于对系统中所有可用的用户角色进行分组。

注意，除了可以在字段上设置元数据之外，你还可以在类级别和方法级别（例如，查询处理程序上）使用 `@Extensions()` 装饰器。

#### 使用自定义元数据

利用自定义元数据的逻辑可以根据需要设计得足够复杂。例如，你可以创建一个简单的拦截器，用于记录每次方法调用的事件，或者创建一个 [字段中间件](/graphql/field-middleware)，将访问某个字段所需的权限角色与调用者的权限进行匹配（字段级权限系统）。

为了便于说明，我们定义一个 `checkRoleMiddleware`，它将用户的角色（这里为硬编码）与访问目标字段所需的角色进行比较：

```typescript
export const checkRoleMiddleware: FieldMiddleware = async (
  ctx: MiddlewareContext,
  next: NextFn,
) => {
  const { info } = ctx;
  const { extensions } = info.parentType.getFields()[info.fieldName];

  /**
   * 在实际应用中，"userRole" 变量
   * 应该代表调用者（用户）的角色（例如，"ctx.user.role"）。
   */
  const userRole = Role.USER;
  if (userRole === extensions.role) {
    // 或者直接 "return null" 以忽略该字段
    throw new ForbiddenException(
      `用户没有足够的权限访问 "${info.fieldName}" 字段。`,
    );
  }
  return next();
};
```

完成以上设置后，我们可以为 `password` 字段注册中间件，如下所示：

```typescript
@Field({ middleware: [checkRoleMiddleware] })
@Extensions({ role: Role.ADMIN })
password: string;
```