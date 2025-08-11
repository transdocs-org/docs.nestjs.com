### 生成 SDL

> warning **警告** 本章仅适用于代码优先（code first）方法。

要手动生成 GraphQL SDL 模式（即，不运行应用程序、不连接数据库、不连接解析器等），请使用 `GraphQLSchemaBuilderModule`。

```typescript
async function generateSchema() {
  const app = await NestFactory.create(GraphQLSchemaBuilderModule);
  await app.init();

  const gqlSchemaFactory = app.get(GraphQLSchemaFactory);
  const schema = await gqlSchemaFactory.create([RecipesResolver]);
  console.log(printSchema(schema));
}
```

> info **提示** `GraphQLSchemaBuilderModule` 和 `GraphQLSchemaFactory` 是从 `@nestjs/graphql` 包中导入的。`printSchema` 函数是从 `graphql` 包中导入的。

#### 使用方法

`gqlSchemaFactory.create()` 方法接受一个解析器类引用的数组。例如：

```typescript
const schema = await gqlSchemaFactory.create([
  RecipesResolver,
  AuthorsResolver,
  PostsResolvers,
]);
```

它还接受第二个可选参数，该参数是一个包含标量类的数组：

```typescript
const schema = await gqlSchemaFactory.create(
  [RecipesResolver, AuthorsResolver, PostsResolvers],
  [DurationScalar, DateScalar],
);
```

最后，您可以传递一个选项对象：

```typescript
const schema = await gqlSchemaFactory.create([RecipesResolver], {
  skipCheck: true,
  orphanedTypes: [],
});
```

- `skipCheck`：忽略模式验证；布尔值，默认为 `false`
- `orphanedTypes`：未被显式引用（不属于对象图）但需要生成的类列表。通常，如果一个类被声明但在图中没有其他引用，则会被省略。该属性的值是一个类引用数组。