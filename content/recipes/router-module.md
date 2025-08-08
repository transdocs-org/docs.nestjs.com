### 路由模块

> info **提示** 本章节仅适用于基于 HTTP 的应用程序。

在 HTTP 应用程序中（例如 REST API），处理程序的路由路径是通过连接控制器中声明的（可选的）前缀（在 `@Controller` 装饰器中定义）以及方法装饰器中指定的路径（例如 `@Get('users')`）拼接而成的。你可以在[这一节](/controllers#路由)中了解更多相关内容。此外，你还可以为应用程序中注册的所有路由定义一个[全局前缀](/faq/global-prefix)，或者启用[版本控制](/techniques/versioning)功能。

另外，有些特殊情况中，你可能希望在模块级别定义一个前缀（即该模块内注册的所有控制器都会继承该前缀），这时候就派上用场了。例如，假设有一个 REST 应用程序，其中有一组不同的端点被应用程序的某个叫做 "Dashboard" 的部分使用。在这种情况下，你不需要在每个控制器中重复 `/dashboard` 前缀，而是可以使用一个实用的 `RouterModule` 模块，如下所示：

```typescript
@Module({
  imports: [
    DashboardModule,
    RouterModule.register([
      {
        path: 'dashboard',
        module: DashboardModule,
      },
    ]),
  ],
})
export class AppModule {}
```

> info **提示** `RouterModule` 类是从 `@nestjs/core` 包中导出的。

此外，你还可以定义层级结构。这意味着每个模块都可以拥有 `children` 子模块。子模块将继承其父模块的前缀。在下面的示例中，我们将 `AdminModule` 注册为 `DashboardModule` 和 `MetricsModule` 的父模块。

```typescript
@Module({
  imports: [
    AdminModule,
    DashboardModule,
    MetricsModule,
    RouterModule.register([
      {
        path: 'admin',
        module: AdminModule,
        children: [
          {
            path: 'dashboard',
            module: DashboardModule,
          },
          {
            path: 'metrics',
            module: MetricsModule,
          },
        ],
      },
    ]),
  ],
})
```

> info **提示** 此功能应谨慎使用，因为过度使用可能会导致代码难以维护。

在上面的示例中，任何在 `DashboardModule` 中注册的控制器都会自动加上 `/admin/dashboard` 前缀（模块会从上到下递归地拼接路径 —— 父级到子级）。同理，`MetricsModule` 中定义的每个控制器也会自动加上模块级别的 `/admin/metrics` 前缀。