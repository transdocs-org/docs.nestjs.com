### 异步提供者

有时，应用程序的启动需要延迟到一个或多个**异步任务**完成之后。例如，在与数据库的连接建立之前，你可能不希望开始接收请求。你可以通过使用异步提供者来实现这种功能。

其语法是在 `useFactory` 语法中使用 `async/await`。工厂函数返回一个 `Promise`，并且工厂函数内部可以 `await` 异步任务。在实例化任何依赖（注入）该提供者的类之前，Nest 会等待该 Promise 解析完成。

```typescript
{
  provide: 'ASYNC_CONNECTION',
  useFactory: async () => {
    const connection = await createConnection(options);
    return connection;
  },
}
```

> info **提示** 在[此处](/fundamentals/custom-providers)了解更多关于自定义提供者的语法。

#### 注入

异步提供者与其他提供者一样，通过它们的令牌注入到其他组件中。在上面的例子中，你可以使用 `@Inject('ASYNC_CONNECTION')` 来进行注入。

#### 示例

[TypeORM 教程](/recipes/sql-typeorm)中包含了一个更完整的异步提供者的示例。