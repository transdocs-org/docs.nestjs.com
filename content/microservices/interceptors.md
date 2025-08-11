### 拦截器

[常规拦截器](/interceptors) 和微服务拦截器之间没有区别。以下示例使用了手动实例化的方法作用域拦截器。与基于 HTTP 的应用程序一样，你也可以使用控制器作用域的拦截器（即，在控制器类前面添加 `@UseInterceptors()` 装饰器）。

```typescript
@@filename()
@UseInterceptors(new TransformInterceptor())
@MessagePattern({ cmd: 'sum' })
accumulate(data: number[]): number {
  return (data || []).reduce((a, b) => a + b);
}
@@switch
@UseInterceptors(new TransformInterceptor())
@MessagePattern({ cmd: 'sum' })
accumulate(data) {
  return (data || []).reduce((a, b) => a + b);
}
```