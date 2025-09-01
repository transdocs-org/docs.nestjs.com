### 守卫

微服务守卫与[常规 HTTP 应用程序守卫](/guards)之间没有根本的区别。
唯一的不同之处在于，你应该使用 `RpcException` 而不是抛出 `HttpException`。

> info **Hint** `RpcException` 类是从 `@nestjs/microservices` 包中导出的。

#### 绑定守卫

以下示例使用了方法作用域的守卫。与基于 HTTP 的应用程序一样，你也可以使用控制器作用域的守卫（即，在控制器类前添加 `@UseGuards()` 装饰器）。

```typescript
@@filename()
@UseGuards(AuthGuard)
@MessagePattern({ cmd: 'sum' })
accumulate(data: number[]): number {
  return (data || []).reduce((a, b) => a + b);
}
@@switch
@UseGuards(AuthGuard)
@MessagePattern({ cmd: 'sum' })
accumulate(data) {
  return (data || []).reduce((a, b) => a + b);
}
```
