### gRPC

[gRPC](https://github.com/grpc/grpc-node) 是一个现代、开源、高性能的 RPC 框架，可以在任何环境中运行。它能够高效地连接数据中心内外的服务，支持负载均衡、追踪、健康检查和身份验证的可插拔功能。

像许多 RPC 系统一样，gRPC 基于定义可远程调用的函数（方法）的概念。对于每个方法，你需要定义参数和返回类型。服务、参数和返回类型在 `.proto` 文件中使用 Google 开源的、与语言无关的 <a href="https://protobuf.dev">协议缓冲区</a> 机制进行定义。

通过 gRPC 传输器，Nest 使用 `.proto` 文件动态绑定客户端和服务器，从而简化远程过程调用的实现，并自动序列化和反序列化结构化数据。

#### 安装

要开始构建基于 gRPC 的微服务，请首先安装所需的包：

```bash
$ npm i --save @grpc/grpc-js @grpc/proto-loader
```

#### 概览

像其他 Nest 微服务传输层实现一样，你可以通过传递给 `createMicroservice()` 方法的选项对象中的 `transport` 属性选择 gRPC 传输机制。在下面的示例中，我们将设置一个 hero 服务。`options` 属性提供了该服务的元数据；其属性的描述 <a href="microservices/grpc#options">如下</a>。

```typescript
@@filename(main)
const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  transport: Transport.GRPC,
  options: {
    package: 'hero',
    protoPath: join(__dirname, 'hero/hero.proto'),
  },
});
@@switch
const app = await NestFactory.createMicroservice(AppModule, {
  transport: Transport.GRPC,
  options: {
    package: 'hero',
    protoPath: join(__dirname, 'hero/hero.proto'),
  },
});
```

> info **提示** `join()` 函数是从 `path` 包导入的；`Transport` 枚举是从 `@nestjs/microservices` 包导入的。

在 `nest-cli.json` 文件中，我们添加了 `assets` 属性，该属性允许我们分发非 TypeScript 文件，以及 `watchAssets` 属性，以启用对所有非 TypeScript 资源的监视。在我们的例子中，我们希望 `.proto` 文件自动复制到 `dist` 文件夹。

```json
{
  "compilerOptions": {
    "assets": ["**/*.proto"],
    "watchAssets": true
  }
}
```

#### 选项

<strong>gRPC</strong> 传输器选项对象暴露了以下描述的属性。

<table>
  <tr>
    <td><code>package</code></td>
    <td>Protobuf 包名称（与 <code>.proto</code> 文件中的 <code>package</code> 设置匹配）。必填</td>
  </tr>
  <tr>
    <td><code>protoPath</code></td>
    <td>
      指向 <code>.proto</code> 文件的绝对路径（或相对于根目录的路径）。必填
    </td>
  </tr>
  <tr>
    <td><code>url</code></td>
    <td>连接 URL。格式为 <code>ip 地址/dns 名称:端口</code> 的字符串（例如 <code>'0.0.0.0:50051'</code> 用于 Docker 服务器），定义传输器建立连接的地址/端口。可选，默认为 <code>'localhost:5000'</code></td>
  </tr>
  <tr>
    <td><code>protoLoader</code></td>
    <td>加载 <code>.proto</code> 文件的实用程序的 NPM 包名称。可选，默认为 <code>'@grpc/proto-loader'</code></td>
  </tr>
  <tr>
    <td><code>loader</code></td>
    <td>
      <code>@grpc/proto-loader</code> 选项。这些选项提供了对 <code>.proto</code> 文件行为的详细控制。可选。
      <a
        href="https://github.com/grpc/grpc-node/blob/master/packages/proto-loader/README.md"
        rel="nofollow"
        target="_blank"
        >此处</a
      > 了解更多详情
    </td>
  </tr>
  <tr>
    <td><code>credentials</code></td>
    <td>
      服务器凭证。可选。
      <a
        href="https://grpc.io/grpc/node/grpc.ServerCredentials.html"
        rel="nofollow"
        target="_blank"
        >在此处阅读更多</a
      >
    </td>
  </tr>
</table>

#### 示例 gRPC 服务

让我们定义一个名为 `HeroesService` 的示例 gRPC 服务。在上述 `options` 对象中，`protoPath` 属性设置了 `.proto` 定义文件 `hero.proto` 的路径。`hero.proto` 文件使用 <a href="https://developers.google.com/protocol-buffers">协议缓冲区</a> 进行结构化。它看起来如下：

```typescript
// hero/hero.proto
syntax = "proto3";

package hero;

service HeroesService {
  rpc FindOne (HeroById) returns (Hero) {}
}

message HeroById {
  int32 id = 1;
}

message Hero {
  int32 id = 1;
  string name = 2;
}
```

我们的 `HeroesService` 暴露了一个 `FindOne()` 方法。此方法期望一个类型为 `HeroById` 的输入参数，并返回一个 `Hero` 消息（协议缓冲区使用 `message` 元素来定义参数类型和返回类型）。

接下来，我们需要实现该服务。为了定义一个满足此定义的处理程序，我们在控制器中使用 `@GrpcMethod()` 装饰器，如下所示。此装饰器提供了将方法声明为 gRPC 服务方法所需的元数据。

> info **提示** 在之前的微服务章节中介绍的 `@MessagePattern()` 装饰器 (<a href="microservices/basics#request-response">了解更多</a>) 不用于基于 gRPC 的微服务。`@GrpcMethod()` 装饰器有效地取代了基于 gRPC 的微服务的这一功能。

```typescript
@@filename(heroes.controller)
@Controller()
export class HeroesController {
  @GrpcMethod('HeroesService', 'FindOne')
  findOne(data: HeroById, metadata: Metadata, call: ServerUnaryCall<any, any>): Hero {
    const items = [
      { id: 1, name: 'John' },
      { id: 2, name: 'Doe' },
    ];
    return items.find(({ id }) => id === data.id);
  }
}
@@switch
@Controller()
export class HeroesController {
  @GrpcMethod('HeroesService', 'FindOne')
  findOne(data, metadata, call) {
    const items = [
      { id: 1, name: 'John' },
      { id: 2, name: 'Doe' },
    ];
    return items.find(({ id }) => id === data.id);
  }
}
```

> info **提示** `@GrpcMethod()` 装饰器从 `@nestjs/microservices` 包导入，而 `Metadata` 和 `ServerUnaryCall` 则从 `grpc` 包导入。

上面显示的装饰器接受两个参数。第一个是服务名称（例如 `'HeroesService'`），对应于 `hero.proto` 中的 `HeroesService` 服务定义。第二个（字符串 `'FindOne'`）对应于 `hero.proto` 文件中 `HeroesService` 内定义的 `FindOne()` rpc 方法。

`findOne()` 处理方法接受三个参数，即调用者传递的 `data`、存储 gRPC 请求元数据的 `metadata` 和用于获取 `GrpcCall` 对象属性（如发送元数据到客户端的 `sendMetadata`）的 `call`。

`@GrpcMethod()` 装饰器的两个参数都是可选的。如果在没有第二个参数（例如 `'FindOne'`）的情况下调用，Nest 会自动将 `.proto` 文件中的 rpc 方法与处理程序关联起来，方法是将处理程序名称转换为大驼峰命名法（例如，`findOne` 处理程序与 `FindOne` rpc 调用定义相关联）。如下所示。

```typescript
@@filename(heroes.controller)
@Controller()
export class HeroesController {
  @GrpcMethod('HeroesService')
  findOne(data: HeroById, metadata: Metadata, call: ServerUnaryCall<any, any>): Hero {
    const items = [
      { id: 1, name: 'John' },
      { id: 2, name: 'Doe' },
    ];
    return items.find(({ id }) => id === data.id);
  }
}
@@switch
@Controller()
export class HeroesController {
  @GrpcMethod('HeroesService')
  findOne(data, metadata, call) {
    const items = [
      { id: 1, name: 'John' },
      { id: 2, name: 'Doe' },
    ];
    return items.find(({ id }) => id === data.id);
  }
}
```

你也可以省略第一个 `@GrpcMethod()` 参数。在这种情况下，Nest 会根据定义处理程序的 **类** 名称自动将处理程序与 proto 定义文件中的服务定义相关联。例如，在下面的代码中，类 `HeroesService` 根据名称 `'HeroesService'` 的匹配将其处理程序方法与 `hero.proto` 文件中的 `HeroesService` 服务定义相关联。

```typescript
@@filename(heroes.controller)
@Controller()
export class HeroesService {
  @GrpcMethod()
  findOne(data: HeroById, metadata: Metadata, call: ServerUnaryCall<any, any>): Hero {
    const items = [
      { id: 1, name: 'John' },
      { id: 2, name: 'Doe' },
    ];
    return items.find(({ id }) => id === data.id);
  }
}
@@switch
@Controller()
export class HeroesService {
  @GrpcMethod()
  findOne(data, metadata, call) {
    const items = [
      { id: 1, name: 'John' },
      { id: 2, name: 'Doe' },
    ];
    return items.find(({ id }) => id === data.id);
  }
}
```

#### 客户端

Nest 应用程序可以充当 gRPC 客户端，消费 `.proto` 文件中定义的服务。你可以通过 `ClientGrpc` 对象访问远程服务。你可以通过几种方式获取 `ClientGrpc` 对象。

首选的技术是导入 `ClientsModule`。使用 `register()` 方法将 `.proto` 文件中定义的服务包绑定到注入令牌，并配置服务。`name` 属性是注入令牌。对于 gRPC 服务，使用 `transport: Transport.GRPC`。`options` 属性是一个对象，其属性的描述 <a href="microservices/grpc#options">如下</a>。

```typescript
imports: [
  ClientsModule.register([
    {
      name: 'HERO_PACKAGE',
      transport: Transport.GRPC,
      options: {
        package: 'hero',
        protoPath: join(__dirname, 'hero/hero.proto'),
      },
    },
  ]),
];
```

> info **提示** `register()` 方法接受一个对象数组。通过提供以逗号分隔的注册对象列表，可以注册多个包。

注册后，我们可以使用 `@Inject()` 注入配置的 `ClientGrpc` 对象。然后我们使用 `ClientGrpc` 对象的 `getService()` 方法来检索服务实例，如下所示。

```typescript
@Injectable()
export class AppService implements OnModuleInit {
  private heroesService: HeroesService;

  constructor(@Inject('HERO_PACKAGE') private client: ClientGrpc) {}

  onModuleInit() {
    this.heroesService = this.client.getService<HeroesService>('HeroesService');
  }

  getHero(): Observable<string> {
    return this.heroesService.findOne({ id: 1 });
  }
}
```

> error **警告** 除非在 proto loader 配置中将 `keepCase` 选项设置为 `true`（微服务传输器配置中的 `options.loader.keepcase`），否则 gRPC 客户端不会发送名称中包含下划线 `_` 的字段。

请注意，这与在其他微服务传输方法中使用的技术略有不同。我们使用 `ClientGrpc` 类而不是 `ClientProxy` 类，该类提供了 `getService()` 方法。`getService()` 泛型方法接受一个服务名称作为参数并返回其实例（如果可用）。

或者，你可以使用 `@Client()` 装饰器实例化一个 `ClientGrpc` 对象，如下所示：

```typescript
@Injectable()
export class AppService implements OnModuleInit {
  @Client({
    transport: Transport.GRPC,
    options: {
      package: 'hero',
      protoPath: join(__dirname, 'hero/hero.proto'),
    },
  })
  client: ClientGrpc;

  private heroesService: HeroesService;

  onModuleInit() {
    this.heroesService = this.client.getService<HeroesService>('HeroesService');
  }

  getHero(): Observable<string> {
    return this.heroesService.findOne({ id: 1 });
  }
}
```

最后，对于更复杂的场景，可以使用 `ClientProxyFactory` 类注入一个动态配置的客户端，如 <a href="/microservices/basics#client">此处</a> 所述。

无论哪种方式，我们最终都会得到一个指向我们的 `HeroesService` 代理对象的引用，该代理对象公开了在 `.proto` 文件中定义的相同方法集。现在，当我们访问此代理对象（即 `heroesService`）时，gRPC 系统会自动序列化请求，将其转发到远程系统，返回响应并反序列化响应。由于 gRPC 为我们屏蔽了这些网络通信细节，`heroesService` 看起来并表现得像一个本地提供者。

注意，所有服务方法都是 **小驼峰命名法**（以便遵循语言的自然约定）。例如，虽然我们的 `.proto` 文件中的 `HeroesService` 定义包含 `FindOne()` 函数，但 `heroesService` 实例将提供 `findOne()` 方法。

```typescript
interface HeroesService {
  findOne(data: { id: number }): Observable<any>;
}
```

消息处理程序还可以返回一个 `Observable`，在这种情况下，结果值将在流完成之前发出。

```typescript
@@filename(heroes.controller)
@Get()
call(): Observable<any> {
  return this.heroesService.findOne({ id: 1 });
}
@@switch
@Get()
call() {
  return this.heroesService.findOne({ id: 1 });
}
```

要发送 gRPC 元数据（与请求一起），你可以传递第二个参数，如下所示：

```typescript
call(): Observable<any> {
  const metadata = new Metadata();
  metadata.add('Set-Cookie', 'yummy_cookie=choco');

  return this.heroesService.findOne({ id: 1 }, metadata);
}
```

> info **提示** `Metadata` 类是从 `grpc` 包导入的。

请注意，这将需要更新我们之前定义的 `HeroesService` 接口。

#### 示例

一个可用的工作示例请参见 [此处](https://github.com/nestjs/nest/tree/master/sample/04-grpc)。

#### gRPC 反射

[gRPC 服务器反射规范](https://grpc.io/docs/guides/reflection/#overview) 是一种允许 gRPC 客户端请求服务器暴露的 API 详细信息的标准，类似于为 REST API 暴露 OpenAPI 文档。这可以使使用开发人员调试工具（如 grpc-ui 或 postman）变得更容易。

要为你的服务器添加 gRPC 反射支持，首先安装所需的实现包：

```bash
$ npm i --save @grpc/reflection
```

然后，可以通过在 gRPC 服务器选项中使用 `onLoadPackageDefinition` 钩子将其集成到 gRPC 服务器中，如下所示：

```typescript
@@filename(main)
import { ReflectionService } from '@grpc/reflection';

const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  options: {
    onLoadPackageDefinition: (pkg, server) => {
      new ReflectionService(pkg).addToServer(server);
    },
  },
});
```

现在，你的服务器将使用反射规范响应请求 API 详细信息的消息。

#### gRPC 流

gRPC 本身支持长期存在的连接，通常称为 `流`。流对于聊天、观察或分块数据传输等场景非常有用。更多细节请参见官方文档 [此处](https://grpc.io/docs/guides/concepts/)。

Nest 支持 GRPC 流处理器的两种方式：

- RxJS `Subject` + `Observable` 处理器：可以在控制器方法中直接编写响应，或者传递给 `Subject`/`Observable` 消费者
- 纯 GRPC 调用流处理器：可以传递给某个执行器，该执行器将处理 Node 标准 `Duplex` 流处理器的其余分发。

<app-banner-enterprise></app-banner-enterprise>

#### 流示例

让我们定义一个新的 gRPC 服务 `HelloService`。`hello.proto` 文件使用 <a href="https://developers.google.com/protocol-buffers">协议缓冲区</a> 进行结构化。它看起来如下：

```typescript
// hello/hello.proto
syntax = "proto3";

package hello;

service HelloService {
  rpc BidiHello(stream HelloRequest) returns (stream HelloResponse);
  rpc LotsOfGreetings(stream HelloRequest) returns (HelloResponse);
}

message HelloRequest {
  string greeting = 1;
}

message HelloResponse {
  string reply = 1;
}
```

> info **提示** `LotsOfGreetings` 方法可以简单地使用 `@GrpcMethod` 装饰器实现（如上例所示），因为返回的流可以发出多个值。

基于此 `.proto` 文件，让我们定义 `HelloService` 接口：

```typescript
interface HelloService {
  bidiHello(upstream: Observable<HelloRequest>): Observable<HelloResponse>;
  lotsOfGreetings(
    upstream: Observable<HelloRequest>,
  ): Observable<HelloResponse>;
}

interface HelloRequest {
  greeting: string;
}

interface HelloResponse {
  reply: string;
}
```

> info **提示** proto 接口可以通过 [ts-proto](https://github.com/stephenh/ts-proto) 包自动生成，了解更多 [此处](https://github.com/stephenh/ts-proto/blob/main/NESTJS.markdown)。

#### Subject 策略

`@GrpcStreamMethod()` 装饰器将函数参数提供为 RxJS `Observable`。因此，我们可以接收和处理多个消息。

```typescript
@GrpcStreamMethod()
bidiHello(messages: Observable<any>, metadata: Metadata, call: ServerDuplexStream<any, any>): Observable<any> {
  const subject = new Subject();

  const onNext = message => {
    console.log(message);
    subject.next({
      reply: 'Hello, world!'
    });
  };
  const onComplete = () => subject.complete();
  messages.subscribe({
    next: onNext,
    complete: onComplete,
  });


  return subject.asObservable();
}
```

> warning **警告** 为了支持与 `@GrpcStreamMethod()` 装饰器的全双工交互，控制器方法必须返回一个 RxJS `Observable`。

> info **提示** `Metadata` 和 `ServerUnaryCall` 类/接口是从 `grpc` 包导入的。

根据服务定义（在 `.proto` 文件中），`BidiHello` 方法应该将请求流式传输到服务。为了从客户端向流发送多个异步消息，我们利用了 RxJS `ReplaySubject` 类。

```typescript
const helloService = this.client.getService<HelloService>('HelloService');
const helloRequest$ = new ReplaySubject<HelloRequest>();

helloRequest$.next({ greeting: 'Hello (1)!' });
helloRequest$.next({ greeting: 'Hello (2)!' });
helloRequest$.complete();

return helloService.bidiHello(helloRequest$);
```

在上面的示例中，我们向流中写入了两条消息（`next()` 调用），并通知服务我们已完成数据发送（`complete()` 调用）。

#### 调用流处理器

当方法返回值定义为 `stream` 时，`@GrpcStreamCall()` 装饰器将函数参数提供为 `grpc.ServerDuplexStream`，它支持标准方法如 `.on('data', callback)`、`.write(message)` 或 `.cancel()`。有关可用方法的完整文档，请参见 [此处](https://grpc.github.io/grpc/node/grpc-ClientDuplexStream.html)。

或者，当方法返回值不是 `stream` 时，`@GrpcStreamCall()` 装饰器提供两个函数参数，分别为 `grpc.ServerReadableStream`（了解更多 [此处](https://grpc.github.io/grpc/node/grpc-ServerReadableStream.html)）和 `callback`。

让我们从实现支持全双工交互的 `BidiHello` 开始。

```typescript
@GrpcStreamCall()
bidiHello(requestStream: any) {
  requestStream.on('data', message => {
    console.log(message);
    requestStream.write({
      reply: 'Hello, world!'
    });
  });
}
```

> info **提示** 此装饰器不需要提供任何特定的返回参数。期望流以与任何其他标准流类型类似的方式进行处理。

在上面的示例中，我们使用 `write()` 方法将对象写入响应流。传递给 `.on()` 方法的第二个参数的回调将在我们的服务接收到新数据块时被调用。

让我们实现 `LotsOfGreetings` 方法。

```typescript
@GrpcStreamCall()
lotsOfGreetings(requestStream: any, callback: (err: unknown, value: HelloResponse) => void) {
  requestStream.on('data', message => {
    console.log(message);
  });
  requestStream.on('end', () => callback(null, { reply: 'Hello, world!' }));
}
```

在这里，我们使用 `callback` 函数在 `requestStream` 处理完成后发送响应。

#### 健康检查

当在诸如 Kubernetes 的编排器中运行 gRPC 应用程序时，你可能需要知道它是否正在运行并且处于健康状态。[gRPC 健康检查规范](https://grpc.io/docs/guides/health-checking/) 是一个标准，允许 gRPC 客户端暴露其健康状态，以便编排器可以相应地采取行动。

要添加 gRPC 健康检查支持，首先安装 [grpc-node](https://github.com/grpc/grpc-node/tree/master/packages/grpc-health-check) 包：

```bash
$ npm i --save grpc-health-check
```

然后，可以通过在你的 gRPC 服务器选项中使用 `onLoadPackageDefinition` 钩子将其集成到 gRPC 服务中，如下所示。请注意，`protoPath` 需要同时包含健康检查和 hero 包。

```typescript
@@filename(main)
import { HealthImplementation, protoPath as healthCheckProtoPath } from 'grpc-health-check';

const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  options: {
    protoPath: [
      healthCheckProtoPath,
      protoPath: join(__dirname, 'hero/hero.proto'),
    ],
    onLoadPackageDefinition: (pkg, server) => {
      const healthImpl = new HealthImplementation({
        '': 'UNKNOWN',
      });

      healthImpl.addToServer(server);
      healthImpl.setStatus('', 'SERVING');
    },
  },
});
```

> info **提示** [gRPC 健康探测器](https://github.com/grpc-ecosystem/grpc-health-probe) 是一个有用的 CLI，用于在容器化环境中测试 gRPC 健康检查。

#### gRPC 元数据

元数据是关于特定 RPC 调用的信息，以键值对列表的形式表示，其中键是字符串，值通常是字符串，但也可以是二进制数据。元数据对 gRPC 本身是不透明的 - 它允许客户端向服务器提供与调用相关的信息，反之亦然。元数据可能包括身份验证令牌、请求标识符和用于监控的标签，以及诸如数据集中的记录数等数据信息。

要在 `@GrpcMethod()` 处理程序中读取元数据，请使用第二个参数（metadata），其类型为 `Metadata`（从 `grpc` 包导入）。

要从处理程序发送回元数据，请使用 `ServerUnaryCall#sendMetadata()` 方法（第三个处理程序参数）。

```typescript
@@filename(heroes.controller)
@Controller()
export class HeroesService {
  @GrpcMethod()
  findOne(data: HeroById, metadata: Metadata, call: ServerUnaryCall<any, any>): Hero {
    const serverMetadata = new Metadata();
    const items = [
      { id: 1, name: 'John' },
      { id: 2, name: 'Doe' },
    ];

    serverMetadata.add('Set-Cookie', 'yummy_cookie=choco');
    call.sendMetadata(serverMetadata);

    return items.find(({ id }) => id === data.id);
  }
}
@@switch
@Controller()
export class HeroesService {
  @GrpcMethod()
  findOne(data, metadata, call) {
    const serverMetadata = new Metadata();
    const items = [
      { id: 1, name: 'John' },
      { id: 2, name: 'Doe' },
    ];

    serverMetadata.add('Set-Cookie', 'yummy_cookie=choco');
    call.sendMetadata(serverMetadata);

    return items.find(({ id }) => id === data.id);
  }
}
```

同样，要在使用 `@GrpcStreamMethod()` 处理程序（[Subject 策略](microservices/grpc#subject-strategy)）注解的处理程序中读取元数据，请使用第二个参数（metadata），其类型为 `Metadata`（从 `grpc` 包导入）。

要从处理程序发送回元数据，请使用 `ServerDuplexStream#sendMetadata()` 方法（第三个处理程序参数）。

要从 [调用流处理程序](microservices/grpc#call-stream-handler)（使用 `@GrpcStreamCall()` 装饰器注解的处理程序）内部读取元数据，请在 `requestStream` 引用上监听 `metadata` 事件，如下所示：

```typescript
requestStream.on('metadata', (metadata: Metadata) => {
  const meta = metadata.get('X-Meta');
});
```