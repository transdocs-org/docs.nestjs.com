### 服务器发送事件（Server-Sent Events）

服务器发送事件（Server-Sent Events，SSE）是一种服务器推送技术，允许客户端通过 HTTP 连接自动从服务器接收更新。每条通知以一个由两个换行符结尾的文本块形式发送（了解更多请参见[此处](https://developer.mozilla.org/zh-CN/docs/Web/API/Server-sent_events)）。

#### 使用方式

要在一个路由上启用服务器发送事件（该路由注册在**控制器类**中），请使用 `@Sse()` 装饰器来修饰方法处理器。

```typescript
@Sse('sse')
sse(): Observable<MessageEvent> {
  return interval(1000).pipe(map((_) => ({ data: { hello: 'world' } })));
}
```

> info **提示** `@Sse()` 装饰器和 `MessageEvent` 接口从 `@nestjs/common` 包导入，而 `Observable`、`interval` 和 `map` 则从 `rxjs` 包导入。

> warning **警告** 服务器发送事件的路由必须返回一个 `Observable` 流。

在上面的示例中，我们定义了一个名为 `sse` 的路由，它将允许我们传播实时更新。这些事件可以通过 [EventSource API](https://developer.mozilla.org/zh-CN/docs/Web/API/EventSource) 来监听。

`sse` 方法返回一个 `Observable`，它会发出多个 `MessageEvent`（在本例中，每秒钟发出一个新的 `MessageEvent`）。`MessageEvent` 对象必须符合以下接口以匹配规范：

```typescript
export interface MessageEvent {
  data: string | object;
  id?: string;
  type?: string;
  retry?: number;
}
```

完成此设置后，我们现在可以在客户端应用程序中创建一个 `EventSource` 类的实例，并将 `/sse` 路由作为构造函数参数传入（该路径与我们之前在 `@Sse()` 装饰器中传递的端点相匹配）。

`EventSource` 实例会向 HTTP 服务器建立一个持久连接，并以 `text/event-stream` 格式接收事件。该连接会一直保持开放，直到调用 `EventSource.close()` 方法将其关闭。

一旦连接建立，来自服务器的入站消息将以事件的形式传递给你的代码。如果入站消息中包含事件字段，则触发的事件名称与事件字段的值相同。如果没有事件字段，则会触发一个通用的 `message` 事件（[来源](https://developer.mozilla.org/zh-CN/docs/Web/API/EventSource)）。

```javascript
const eventSource = new EventSource('/sse');
eventSource.onmessage = ({ data }) => {
  console.log('收到新消息', JSON.parse(data));
};
```

#### 示例

一个完整的示例可以在这里查看：[示例代码](https://github.com/nestjs/nest/tree/master/sample/28-sse)。