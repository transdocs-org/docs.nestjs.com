### 自定义传输器

Nest 提供了多种**传输器**，以及一个允许开发者构建新的自定义传输策略的 API。
传输器使你可以使用可插拔的通信层和简单的应用层消息协议将组件通过网络连接起来（阅读完整[文章](https://dev.to/nestjs/integrate-nestjs-with-external-services-using-microservice-transporters-part-1-p3)）。

> info **提示** 使用 Nest 构建微服务并不一定意味着你必须使用 `@nestjs/microservices` 包。例如，如果你想与外部服务通信（比如其他用不同语言编写的微服务），你可能不需要 `@nestjs/microservices` 库提供的所有功能。
> 事实上，如果你不需要使用 `@EventPattern` 或 `@MessagePattern` 等装饰器来声明性地定义订阅者，运行一个[独立应用](/application-context)并手动维护连接/订阅频道就足以满足大多数使用场景，并且会为你提供更大的灵活性。

使用自定义传输器，你可以集成任何消息系统/协议（包括 Google Cloud Pub/Sub、Amazon Kinesis 等），或者在现有传输器的基础上扩展功能（例如为 MQTT 添加 [QoS](https://github.com/mqttjs/MQTT.js/blob/master/README.md#qos)）。

> info **提示** 为了更好地理解 Nest 微服务的工作原理以及如何扩展现有传输器的功能，我们建议阅读 [NestJS Microservices in Action](https://dev.to/johnbiundo/series/4724) 和 [Advanced NestJS Microservices](https://dev.to/nestjs/part-1-introduction-and-setup-1a2l) 文章系列。

#### 创建策略

首先，让我们定义一个表示我们自定义传输器的类。

```typescript
import { CustomTransportStrategy, Server } from '@nestjs/microservices';

class GoogleCloudPubSubServer
  extends Server
  implements CustomTransportStrategy
{
  /**
   * 在运行 "app.listen()" 时触发。
   */
  listen(callback: () => void) {
    callback();
  }

  /**
   * 在应用程序关闭时触发。
   */
  close() {}

  /**
   * 如果你不希望传输器用户
   * 能够注册事件监听器，可以忽略此方法。大多数自定义实现
   * 都不需要这个方法。
   */
  on(event: string, callback: Function) {
    throw new Error('方法未实现。');
  }

  /**
   * 如果你不希望传输器用户
   * 能够获取底层的原生服务器，可以忽略此方法。大多数自定义实现
   * 都不需要这个方法。
   */
  unwrap<T = never>(): T {
    throw new Error('方法未实现。');
  }
}
```

> warning **警告** 请注意，在本章中我们不会实现一个功能齐全的 Google Cloud Pub/Sub 服务器，因为这需要深入研究传输器特定的技术细节。

在上面的例子中，我们声明了 `GoogleCloudPubSubServer` 类并提供了由 `CustomTransportStrategy` 接口强制要求的 `listen()` 和 `close()` 方法。
此外，我们的类继承了从 `@nestjs/microservices` 包导入的 `Server` 类，该类提供了一些有用的方法，例如 Nest 运行时用于注册消息处理程序的方法。或者，如果你想扩展现有传输策略的功能，可以继承相应的服务器类，例如 `ServerRedis`。
按照惯例，我们在类名后添加了 `"Server"` 后缀，因为它将负责订阅消息/事件（并在必要时做出响应）。

有了这些，我们现在可以使用我们的自定义策略，而不是使用内置的传输器，如下所示：

```typescript
const app = await NestFactory.createMicroservice<MicroserviceOptions>(
  AppModule,
  {
    strategy: new GoogleCloudPubSubServer(),
  },
);
```

基本上，我们不再传递带有 `transport` 和 `options` 属性的普通传输器选项对象，而是传递一个 `strategy` 属性，其值是我们自定义传输器类的一个实例。

回到我们的 `GoogleCloudPubSubServer` 类，在实际应用中，我们将在 `listen()` 方法中建立与消息代理/外部服务的连接并注册订阅/监听特定频道（然后在 `close()` 清理方法中移除订阅并关闭连接），
但由于这需要很好地理解 Nest 微服务之间的通信方式，我们建议阅读这篇[文章系列](https://dev.to/nestjs/part-1-introduction-and-setup-1a2l)。
在本章中，我们将重点介绍 `Server` 类提供的功能以及如何利用它们构建自定义策略。

例如，假设在我们的应用程序中某处定义了以下消息处理程序：

```typescript
@MessagePattern('echo')
echo(@Payload() data: object) {
  return data;
}
```

这个消息处理程序将由 Nest 运行时自动注册。使用 `Server` 类，你可以看到已注册的消息模式，还可以访问并执行分配给它们的实际方法。
为了测试这一点，让我们在 `listen()` 方法中 `callback` 函数被调用之前添加一个简单的 `console.log`：

```typescript
listen(callback: () => void) {
  console.log(this.messageHandlers);
  callback();
}
```

在应用程序重启后，你会在终端中看到如下日志：

```typescript
Map { 'echo' => [AsyncFunction] { isEventHandler: false } }
```

> info **提示** 如果我们使用的是 `@EventPattern` 装饰器，则你会看到相同的输出，但 `isEventHandler` 属性将被设置为 `true`。

如你所见，`messageHandlers` 属性是一个 `Map` 集合，包含了所有的消息（和事件）处理程序，其中模式作为键使用。
现在，你可以使用一个键（例如 `"echo"`）来获取消息处理程序的引用：

```typescript
async listen(callback: () => void) {
  const echoHandler = this.messageHandlers.get('echo');
  console.log(await echoHandler('Hello world!'));
  callback();
}
```

一旦我们执行 `echoHandler` 并传入一个任意字符串作为参数（这里是 `"Hello world!"`），我们应该会在控制台中看到：

```json
Hello world!
```

这意味着我们的方法处理程序已正确执行。

当使用 `CustomTransportStrategy` 和 [拦截器](/interceptors) 时，处理程序会被包装成 RxJS 流。这意味着你需要订阅它们以执行流的底层逻辑（例如，在拦截器执行后继续执行控制器逻辑）。

下面是一个示例：

```typescript
async listen(callback: () => void) {
  const echoHandler = this.messageHandlers.get('echo');
  const streamOrResult = await echoHandler('Hello World');
  if (isObservable(streamOrResult)) {
    streamOrResult.subscribe();
  }
  callback();
}
```

#### 客户端代理

正如我们在第一部分中提到的，你不一定需要使用 `@nestjs/microservices` 包来创建微服务，但如果你决定使用它并需要集成自定义策略，则还需要提供一个“客户端”类。

> info **提示** 再次说明，实现一个与所有 `@nestjs/microservices` 功能（例如流式处理）兼容的完整客户端类，需要很好地理解框架使用的通信技术。要了解更多信息，请查看这篇[文章](https://dev.to/nestjs/part-4-basic-client-component-16f9)。

为了与外部服务通信/发送和发布消息（或事件），你可以使用特定库的 SDK 包，或者实现一个继承 `ClientProxy` 的自定义客户端类，如下所示：

```typescript
import { ClientProxy, ReadPacket, WritePacket } from '@nestjs/microservices';

class GoogleCloudPubSubClient extends ClientProxy {
  async connect(): Promise<any> {}
  async close() {}
  async dispatchEvent(packet: ReadPacket<any>): Promise<any> {}
  publish(
    packet: ReadPacket<any>,
    callback: (packet: WritePacket<any>) => void,
  ): Function {}
  unwrap<T = never>(): T {
    throw new Error('方法未实现。');
  }
}
```

> warning **警告** 请注意，在本章中我们不会实现一个功能齐全的 Google Cloud Pub/Sub 客户端，因为这需要深入研究传输器特定的技术细节。

如你所见，`ClientProxy` 类要求我们提供几个用于建立和关闭连接以及发布消息（`publish`）和事件（`dispatchEvent`）的方法。
请注意，如果你不需要支持请求-响应通信模式，可以将 `publish()` 方法留空。同样，如果你不需要支持基于事件的通信，可以跳过 `dispatchEvent()` 方法。

为了观察这些方法何时被执行，让我们添加多个 `console.log` 调用，如下所示：

```typescript
class GoogleCloudPubSubClient extends ClientProxy {
  async connect(): Promise<any> {
    console.log('connect');
  }

  async close() {
    console.log('close');
  }

  async dispatchEvent(packet: ReadPacket<any>): Promise<any> {
    return console.log('event to dispatch: ', packet);
  }

  publish(
    packet: ReadPacket<any>,
    callback: (packet: WritePacket<any>) => void,
  ): Function {
    console.log('message:', packet);

    // 在实际应用中，"callback" 函数应使用
    // 响应者发送回来的负载来执行。在这里，我们简单模拟（5 秒延迟）
    // 响应已返回，通过传递与最初传入相同的 "data"。
    //
    // WritePacket 上的 "isDisposed" 布尔值告诉响应不再有进一步的数据
    // 预期。如果没有发送或为 false，则会向 Observable 发送数据。
    setTimeout(() => callback({ 
      response: packet.data,
      isDisposed: true,
    }), 5000);

    return () => console.log('teardown');
  }

  unwrap<T = never>(): T {
    throw new Error('方法未实现。');
  }
}
```

有了这些，让我们创建一个 `GoogleCloudPubSubClient` 类的实例并运行 `send()` 方法（你可能在前面章节中见过），并订阅返回的 observable 流。

```typescript
const googlePubSubClient = new GoogleCloudPubSubClient();
googlePubSubClient
  .send('pattern', 'Hello world!')
  .subscribe((response) => console.log(response));
```

现在，你应该在终端中看到以下输出：

```typescript
connect
message: { pattern: 'pattern', data: 'Hello world!' }
Hello world! // <-- 5 秒后
```

为了测试我们的 "teardown" 方法（即我们的 `publish()` 方法返回的方法）是否正确执行，让我们在流上应用一个 timeout 操作符，将其设置为 2 秒，以确保它比我们的 `setTimeout` 调用 `callback` 函数更早抛出错误。

```typescript
const googlePubSubClient = new GoogleCloudPubSubClient();
googlePubSubClient
  .send('pattern', 'Hello world!')
  .pipe(timeout(2000))
  .subscribe(
    (response) => console.log(response),
    (error) => console.error(error.message),
  );
```

> info **提示** `timeout` 操作符是从 `rxjs/operators` 包导入的。

应用 `timeout` 操作符后，你的终端输出应该如下所示：

```typescript
connect
message: { pattern: 'pattern', data: 'Hello world!' }
teardown // <-- teardown
Timeout has occurred
```

要分发一个事件（而不是发送消息），请使用 `emit()` 方法：

```typescript
googlePubSubClient.emit('event', 'Hello world!');
```

你应该在控制台中看到如下内容：

```typescript
connect
event to dispatch:  { pattern: 'event', data: 'Hello world!' }
```

#### 消息序列化

如果你需要在客户端对响应的序列化添加一些自定义逻辑，你可以使用一个继承 `ClientProxy` 类或其子类之一的自定义类。对于修改成功请求，你可以重写 `serializeResponse` 方法；对于修改通过此客户端的任何错误，你可以重写 `serializeError` 方法。要使用这个自定义类，你可以通过 `ClientsModule.register()` 方法的 `customClass` 属性传递该类。以下是一个自定义 `ClientProxy` 的示例，它将每个错误序列化为 `RpcException`。

```typescript
@@filename(error-handling.proxy)
import { ClientTcp, RpcException } from '@nestjs/microservices';

class ErrorHandlingProxy extends ClientTCP {
  serializeError(err: Error) {
    return new RpcException(err);
  }
}
```

然后在 `ClientsModule` 中使用它：

```typescript
@@filename(app.module)
@Module({
  imports: [
    ClientsModule.register([{
      name: 'CustomProxy',
      customClass: ErrorHandlingProxy,
    }]),
  ]
})
export class AppModule
```

> info **提示** 这里传递给 `customClass` 的是类本身，而不是类的实例。Nest 将在幕后为你创建实例，并将传递给 `options` 属性的任何选项传递给新的 `ClientProxy`。