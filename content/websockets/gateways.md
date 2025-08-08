### 网关（Gateways）

在本节中讨论的大多数概念，如依赖注入、装饰器、异常过滤器、管道、守卫和拦截器，同样适用于网关。Nest 尽可能地抽象了实现细节，使得相同组件可以在基于 HTTP 的平台、WebSockets 和微服务中运行。本节介绍 Nest 中与 WebSockets 相关的特定功能。

在 Nest 中，一个网关就是一个使用 `@WebSocketGateway()` 装饰器标注的类。从技术上讲，网关是平台无关的，这意味着只要创建了适配器，它们就可以与任何 WebSockets 库兼容。Nest 开箱即用地支持两种 WS 平台：[socket.io](https://github.com/socketio/socket.io) 和 [ws](https://github.com/websockets/ws)。你可以根据自己的需求选择其中之一，也可以根据此[指南](/websockets/adapter)构建自己的适配器。

<figure><img class="illustrative-image" src="/assets/Gateways_1.png" /></figure>

> info **提示** 网关可以被视为[提供者](/providers)，这意味着它们可以通过类构造函数注入依赖项。此外，其他类（提供者和控制器）也可以注入网关。

#### 安装

要开始构建基于 WebSockets 的应用程序，首先安装必要的包：

```bash
@@filename()
$ npm i --save @nestjs/websockets @nestjs/platform-socket.io
@@switch
$ npm i --save @nestjs/websockets @nestjs/platform-socket.io
```

#### 概览

通常情况下，每个网关都会监听与 **HTTP 服务器** 相同的端口，除非你的应用程序不是 Web 应用程序，或者你手动更改了端口号。默认行为可以通过向 `@WebSocketGateway(80)` 装饰器传递参数进行修改，其中 `80` 是你选择的端口号。你还可以使用以下方式设置网关使用的[命名空间](https://socket.io/docs/v4/namespaces/)：

```typescript
@WebSocketGateway(80, { namespace: 'events' })
```

> warning **警告** 网关在被模块的 providers 数组引用之前不会被实例化。

你可以将任意支持的[选项](https://socket.io/docs/v4/server-options/)作为第二个参数传入 `@WebSocketGateway()` 装饰器，如下所示：

```typescript
@WebSocketGateway(81, { transports: ['websocket'] })
```

现在网关正在监听，但我们还没有订阅任何传入消息。让我们创建一个处理程序，用于订阅 `events` 消息，并将收到的数据原样返回给用户。

```typescript
@@filename(events.gateway)
@SubscribeMessage('events')
handleEvent(@MessageBody() data: string): string {
  return data;
}
@@switch
@Bind(MessageBody())
@SubscribeMessage('events')
handleEvent(data) {
  return data;
}
```

> info **提示** `@SubscribeMessage()` 和 `@MessageBody()` 装饰器来自 `@nestjs/websockets` 包。

一旦创建了网关，我们就可以在模块中注册它。

```typescript
import { Module } from '@nestjs/common';
import { EventsGateway } from './events.gateway';

@@filename(events.module)
@Module({
  providers: [EventsGateway]
})
export class EventsModule {}
```

你也可以将属性键传递给装饰器，以从传入的消息体中提取特定字段：

```typescript
@@filename(events.gateway)
@SubscribeMessage('events')
handleEvent(@MessageBody('id') id: number): number {
  // id === messageBody.id
  return id;
}
@@switch
@Bind(MessageBody('id'))
@SubscribeMessage('events')
handleEvent(id) {
  // id === messageBody.id
  return id;
}
```

如果你不想使用装饰器，下面的代码功能是等价的：

```typescript
@@filename(events.gateway)
@SubscribeMessage('events')
handleEvent(client: Socket, data: string): string {
  return data;
}
@@switch
@SubscribeMessage('events')
handleEvent(client, data) {
  return data;
}
```

在上面的例子中，`handleEvent()` 函数接受两个参数。第一个参数是平台特定的 [socket 实例](https://socket.io/docs/v4/server-api/#socket)，第二个参数是客户端发送的数据。不过，不推荐这种做法，因为它要求在每个单元测试中都要模拟 `socket` 实例。

当接收到 `events` 消息时，处理程序会发送一个包含相同数据的确认消息。此外，也可以使用特定库的方法发送消息，例如通过 `client.emit()` 方法。要访问连接的 socket 实例，请使用 `@ConnectedSocket()` 装饰器。

```typescript
@@filename(events.gateway)
@SubscribeMessage('events')
handleEvent(
  @MessageBody() data: string,
  @ConnectedSocket() client: Socket,
): string {
  return data;
}
@@switch
@Bind(MessageBody(), ConnectedSocket())
@SubscribeMessage('events')
handleEvent(data, client) {
  return data;
}
```

> info **提示** `@ConnectedSocket()` 装饰器来自 `@nestjs/websockets` 包。

不过，在这种情况下，你将无法使用拦截器。如果你不打算向用户返回响应，可以省略 `return` 语句（或显式返回一个“falsy”值，例如 `undefined`）。

当客户端发出如下消息时：

```typescript
socket.emit('events', { name: 'Nest' });
```

`handleEvent()` 方法将被调用。要在上述处理程序中监听消息的响应，客户端需要附加一个对应的确认监听器：

```typescript
socket.emit('events', { name: 'Nest' }, (data) => console.log(data));
```

#### 多次响应

确认响应只发送一次。此外，原生 WebSockets 实现并不支持此功能。为了解决这一限制，你可以返回一个包含两个属性的对象：`event` 是要发送的事件名称，`data` 是要转发给客户端的数据。

```typescript
@@filename(events.gateway)
@SubscribeMessage('events')
handleEvent(@MessageBody() data: unknown): WsResponse<unknown> {
  const event = 'events';
  return { event, data };
}
@@switch
@Bind(MessageBody())
@SubscribeMessage('events')
handleEvent(data) {
  const event = 'events';
  return { event, data };
}
```

> info **提示** `WsResponse` 接口来自 `@nestjs/websockets` 包。

> warning **警告** 如果你的 `data` 字段依赖 `ClassSerializerInterceptor`，则应返回一个实现了 `WsResponse` 接口的类实例，因为它会忽略纯 JavaScript 对象的响应。

为了监听传入的响应，客户端需要添加另一个事件监听器：

```typescript
socket.on('events', (data) => console.log(data));
```

#### 异步响应

消息处理程序既可以同步响应，也可以**异步响应**。因此，支持 `async` 方法。消息处理程序还可以返回一个 `Observable`，在这种情况下，结果值会在流完成之前持续发出。

```typescript
@@filename(events.gateway)
@SubscribeMessage('events')
onEvent(@MessageBody() data: unknown): Observable<WsResponse<number>> {
  const event = 'events';
  const response = [1, 2, 3];

  return from(response).pipe(
    map(data => ({ event, data })),
  );
}
@@switch
@Bind(MessageBody())
@SubscribeMessage('events')
onEvent(data) {
  const event = 'events';
  const response = [1, 2, 3];

  return from(response).pipe(
    map(data => ({ event, data })),
  );
}
```

在上面的例子中，消息处理程序会响应 **3 次**（每次对应数组中的一个元素）。

#### 生命周期钩子

有三个有用的生命周期钩子可用。它们都有对应的接口，如下表所示：

<table>
  <tr>
    <td>
      <code>OnGatewayInit</code>
    </td>
    <td>
      强制实现 <code>afterInit()</code> 方法。该方法接受一个库特定的服务器实例作为参数（如有需要，会展开其余参数）。
    </td>
  </tr>
  <tr>
    <td>
      <code>OnGatewayConnection</code>
    </td>
    <td>
      强制实现 <code>handleConnection()</code> 方法。该方法接受一个库特定的客户端 socket 实例作为参数。
    </td>
  </tr>
  <tr>
    <td>
      <code>OnGatewayDisconnect</code>
    </td>
    <td>
      强制实现 <code>handleDisconnect()</code> 方法。该方法接受一个库特定的客户端 socket 实例作为参数。
    </td>
  </tr>
</table>

> info **提示** 每个生命周期接口都来自 `@nestjs/websockets` 包。

#### 服务器与命名空间

有时，你可能希望直接访问原生的、**平台特定**的服务器实例。该实例的引用作为参数传递给 `afterInit()` 方法（`OnGatewayInit` 接口）。另一种方法是使用 `@WebSocketServer()` 装饰器。

```typescript
@WebSocketServer()
server: Server;
```

你也可以通过 `namespace` 属性获取对应的命名空间，如下所示：

```typescript
@WebSocketServer({ namespace: 'my-namespace' })
namespace: Namespace;
```

> warning **注意** `@WebSocketServer()` 装饰器来自 `@nestjs/websockets` 包。

当服务器实例准备就绪后，Nest 会自动将其赋值给该属性。

<app-banner-enterprise></app-banner-enterprise>

#### 示例

一个可用的示例请见 [此处](https://github.com/nestjs/nest/tree/master/sample/02-gateways)。