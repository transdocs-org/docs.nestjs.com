### 共享模型

> warning **警告** 本章仅适用于代码优先（code first）的方法。

在项目后端中使用 TypeScript 的一大优势是，可以通过使用一个通用的 TypeScript 包，在基于 TypeScript 的前端应用中复用相同的模型。

但这里存在一个问题：使用代码优先方法创建的模型被大量 GraphQL 相关的装饰器所修饰。这些装饰器在前端中是无关的，而且会对性能产生负面影响。

#### 使用模型 shim

为了解决这个问题，NestJS 提供了一个“shim”，允许你通过 `webpack`（或类似）配置，将原始的装饰器替换为无操作的代码。要使用此 shim，需要在 `@nestjs/graphql` 包和该 shim 之间配置一个别名。

例如，对于 webpack，可以按如下方式配置：

```typescript
resolve: { // 参见: https://webpack.js.org/configuration/resolve/
  alias: {
      "@nestjs/graphql": path.resolve(__dirname, "../node_modules/@nestjs/graphql/dist/extra/graphql-model-shim")
  }
}
```

> info **提示** [TypeORM](/techniques/database) 包也有一个类似的 shim，你可以在[这里](https://github.com/typeorm/typeorm/blob/master/extra/typeorm-model-shim.js)找到它。
