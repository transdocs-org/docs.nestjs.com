### 适配器

WebSocket 模块是平台无关的，因此你可以通过使用 `WebSocketAdapter` 接口引入自己的库（甚至是原生实现）。该接口要求实现几个方法，如下表所述：

<table>
  <tr>
    <td><code>create</code></td>
    <td>根据传入的参数创建一个 socket 实例</td>
  </tr>
  <tr>
    <td><code>bindClientConnect</code></td>
    <td>绑定客户端连接事件</td>
  </tr>
  <tr>
    <td><code>bindClientDisconnect</code></td>
    <td>绑定客户端断开连接事件（可选*）</td>
  </tr>
  <tr>
    <td><code>bindMessageHandlers</code></td>
    <td>将传入的消息绑定到对应的消息处理程序</td>
  </tr>
  <tr>
    <td><code>close</code></td>
    <td>终止服务器实例</td>
  </tr>
</table>

#### 扩展 socket.io

[socket.io](https://github.com/socketio/socket.io) 包被封装在 `IoAdapter` 类中。如果你想增强适配器的基本功能该怎么办？例如，你的技术需求要求能够跨多个负载均衡的 Web 服务实例广播事件。为此，你可以扩展 `IoAdapter` 并重写一个负责实例化新的 socket.io 服务器的方法。但首先，让我们安装所需的包。

> warning **警告** 要在多个负载均衡的实例中使用 socket.io，你必须通过在客户端 socket.io 配置中设置 `transports: ['websocket']` 来禁用轮询，或者在负载均衡器中启用基于 cookie 的路由。仅使用 Redis 是不够的。详见[此处](https://socket.io/docs/v4/using-multiple-nodes/#enabling-sticky-session)。

```bash
$ npm i --save redis socket.io @socket.io/redis-adapter
```

安装完包之后，我们可以创建一个 `RedisIoAdapter` 类。

```typescript
import { IoAdapter } from '@nestjs/platform-socket.io';
import { ServerOptions } from 'socket.io';
import { createAdapter } from '@socket.io/redis-adapter';
import { createClient } from 'redis';

export class RedisIoAdapter extends IoAdapter {
  private adapterConstructor: ReturnType<typeof createAdapter>;

  async connectToRedis(): Promise<void> {
    const pubClient = createClient({ url: `redis://localhost:6379` });
    const subClient = pubClient.duplicate();

    await Promise.all([pubClient.connect(), subClient.connect()]);

    this.adapterConstructor = createAdapter(pubClient, subClient);
  }

  createIOServer(port: number, options?: ServerOptions): any {
    const server = super.createIOServer(port, options);
    server.adapter(this.adapterConstructor);
    return server;
  }
}
```

然后，只需切换到你新创建的 Redis 适配器。

```typescript
const app = await NestFactory.create(AppModule);
const redisIoAdapter = new RedisIoAdapter(app);
await redisIoAdapter.connectToRedis();

app.useWebSocketAdapter(redisIoAdapter);
```

#### Ws 库

另一个可用的适配器是 `WsAdapter`，它在框架和快速且经过充分测试的 [ws](https://github.com/websockets/ws) 库之间充当代理。此适配器完全兼容原生浏览器 WebSockets，并且比 socket.io 包要快得多。不幸的是，它开箱即用的功能要少得多。不过在某些情况下，你可能并不需要这些功能。

> info **提示** `ws` 库不支持命名空间（由 `socket.io` 推广的通信通道）。不过，为了在某种程度上模拟此功能，你可以在不同路径上挂载多个 `ws` 服务器（例如：`@WebSocketGateway({{ '{' }} path: '/users' {{ '}' }})`）。

要使用 `ws`，我们首先需要安装所需的包：

```bash
$ npm i --save @nestjs/platform-ws
```

安装包后，我们可以切换适配器：

```typescript
const app = await NestFactory.create(AppModule);
app.useWebSocketAdapter(new WsAdapter(app));
```

> info **提示** `WsAdapter` 从 `@nestjs/platform-ws` 导入。

`WsAdapter` 被设计为处理 `{{ '{' }} event: string, data: any {{ '}' }}` 格式的消息。如果你需要接收和处理其他格式的消息，则需要配置一个消息解析器来将其转换为此要求的格式。

```typescript
const wsAdapter = new WsAdapter(app, {
  // 处理 [event, data] 格式的消息
  messageParser: (data) => {
    const [event, payload] = JSON.parse(data.toString());
    return { event, data: payload };
  },
});
```

或者，你也可以通过 `setMessageParser` 方法在适配器创建后配置消息解析器。

#### 高级（自定义适配器）

为了演示目的，我们将手动集成 [ws](https://github.com/websockets/ws) 库。如前所述，该库的适配器已经创建，并作为 `WsAdapter` 类从 `@nestjs/platform-ws` 包中导出。以下是一个简化版的实现可能的样子：

```typescript
@@filename(ws-adapter)
import * as WebSocket from 'ws';
import { WebSocketAdapter, INestApplicationContext } from '@nestjs/common';
import { MessageMappingProperties } from '@nestjs/websockets';
import { Observable, fromEvent, EMPTY } from 'rxjs';
import { mergeMap, filter } from 'rxjs/operators';

export class WsAdapter implements WebSocketAdapter {
  constructor(private app: INestApplicationContext) {}

  create(port: number, options: any = {}): any {
    return new WebSocket.Server({ port, ...options });
  }

  bindClientConnect(server, callback: Function) {
    server.on('connection', callback);
  }

  bindMessageHandlers(
    client: WebSocket,
    handlers: MessageMappingProperties[],
    process: (data: any) => Observable<any>,
  ) {
    fromEvent(client, 'message')
      .pipe(
        mergeMap(data => this.bindMessageHandler(data, handlers, process)),
        filter(result => result),
      )
      .subscribe(response => client.send(JSON.stringify(response)));
  }

  bindMessageHandler(
    buffer,
    handlers: MessageMappingProperties[],
    process: (data: any) => Observable<any>,
  ): Observable<any> {
    const message = JSON.parse(buffer.data);
    const messageHandler = handlers.find(
      handler => handler.message === message.event,
    );
    if (!messageHandler) {
      return EMPTY;
    }
    return process(messageHandler.callback(message.data));
  }

  close(server) {
    server.close();
  }
}
```

> info **提示** 当你想使用 [ws](https://github.com/websockets/ws) 库时，请使用内置的 `WsAdapter`，而不要自己创建适配器。

然后，我们可以使用 `useWebSocketAdapter()` 方法设置自定义适配器：

```typescript
@@filename(main)
const app = await NestFactory.create(AppModule);
app.useWebSocketAdapter(new WsAdapter(app));
```

#### 示例

一个使用 `WsAdapter` 的完整示例可在[此处](https://github.com/nestjs/nest/tree/master/sample/16-gateways-ws)找到。