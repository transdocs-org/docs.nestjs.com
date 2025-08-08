### MQTT

[MQTT](https://mqtt.org/)（消息队列遥测传输）是一种开源的轻量级消息协议，优化了低延迟通信。该协议通过**发布/订阅**模型，提供了一种可扩展且经济高效的设备连接方式。一个基于MQTT构建的通信系统包括发布服务器、代理（broker）和一个或多个客户端。它专为资源受限的设备以及低带宽、高延迟或不可靠网络设计。

#### 安装

要开始构建基于MQTT的微服务，首先安装所需的包：

```bash
$ npm i --save mqtt
```

#### 概览

要使用MQTT传输器，请将以下选项对象传递给 `createMicroservice()` 方法：

```typescript
@@filename(main)
const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  transport: Transport.MQTT,
  options: {
    url: 'mqtt://localhost:1883',
  },
});
@@switch
const app = await NestFactory.createMicroservice(AppModule, {
  transport: Transport.MQTT,
  options: {
    url: 'mqtt://localhost:1883',
  },
});
```

> info **提示** `Transport` 枚举从 `@nestjs/microservices` 包中导入。

#### 选项

`options` 对象是特定于所选传输器的。<strong>MQTT</strong> 传输器暴露了 [此处](https://github.com/mqttjs/MQTT.js/#mqttclientstreambuilder-options) 描述的属性。

#### 客户端

与其他微服务传输器一样，您有 <a href="https://docs.nestjs.com/microservices/basics#client">多种选项</a> 来创建 `ClientProxy` 实例。

一种创建实例的方法是使用 `ClientsModule`。要使用 `ClientsModule` 创建客户端实例，请导入它并使用 `register()` 方法传递一个选项对象，其属性与上面 `createMicroservice()` 方法中显示的相同，同时添加一个 `name` 属性作为注入令牌。[点击此处](https://docs.nestjs.com/microservices/basics#client) 阅读更多关于 `ClientsModule` 的内容。

```typescript
@Module({
  imports: [
    ClientsModule.register([
      {
        name: 'MATH_SERVICE',
        transport: Transport.MQTT,
        options: {
          url: 'mqtt://localhost:1883',
        }
      },
    ]),
  ]
  ...
})
```

也可以使用其他创建客户端的方式（如 `ClientProxyFactory` 或 `@Client()`）。您可 <a href="https://docs.nestjs.com/microservices/basics#client">点击此处</a> 阅读相关内容。

#### 上下文

在更复杂的场景中，您可能需要访问有关传入请求的更多信息。当使用MQTT传输器时，您可以访问 `MqttContext` 对象。

```typescript
@@filename()
@MessagePattern('notifications')
getNotifications(@Payload() data: number[], @Ctx() context: MqttContext) {
  console.log(`主题: ${context.getTopic()}`);
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('notifications')
getNotifications(data, context) {
  console.log(`主题: ${context.getTopic()}`);
}
```

> info **提示** `@Payload()`、`@Ctx()` 和 `MqttContext` 从 `@nestjs/microservices` 包中导入。

要访问原始的MQTT [数据包](https://github.com/mqttjs/mqtt-packet)，请使用 `MqttContext` 对象的 `getPacket()` 方法，如下所示：

```typescript
@@filename()
@MessagePattern('notifications')
getNotifications(@Payload() data: number[], @Ctx() context: MqttContext) {
  console.log(context.getPacket());
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('notifications')
getNotifications(data, context) {
  console.log(context.getPacket());
}
```

#### 通配符

订阅可以是明确的主题，也可以包含通配符。有两种通配符：`+` 和 `#`。`+` 是单级通配符，而 `#` 是多级通配符，可覆盖多个主题层级。

```typescript
@@filename()
@MessagePattern('sensors/+/temperature/+')
getTemperature(@Ctx() context: MqttContext) {
  console.log(`主题: ${context.getTopic()}`);
}
@@switch
@Bind(Ctx())
@MessagePattern('sensors/+/temperature/+')
getTemperature(context) {
  console.log(`主题: ${context.getTopic()}`);
}
```

#### 服务质量 (QoS)

使用 `@MessagePattern` 或 `@EventPattern` 装饰器创建的任何订阅都将使用 QoS 0。如果需要更高的 QoS，可以在建立连接时使用 `subscribeOptions` 块进行全局设置，如下所示：

```typescript
@@filename(main)
const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  transport: Transport.MQTT,
  options: {
    url: 'mqtt://localhost:1883',
    subscribeOptions: {
      qos: 2
    },
  },
});
@@switch
const app = await NestFactory.createMicroservice(AppModule, {
  transport: Transport.MQTT,
  options: {
    url: 'mqtt://localhost:1883',
    subscribeOptions: {
      qos: 2
    },
  },
});
```

如果需要特定主题的 QoS，请考虑创建一个 [自定义传输器](https://docs.nestjs.com/microservices/custom-transport)。

#### 记录构建器

要配置消息选项（调整 QoS 级别、设置 Retain 或 DUP 标志，或向有效载荷添加其他属性），您可以使用 `MqttRecordBuilder` 类。例如，要将 `QoS` 设置为 `2`，请使用 `setQoS` 方法，如下所示：

```typescript
const userProperties = { 'x-version': '1.0.0' };
const record = new MqttRecordBuilder(':cat:')
  .setProperties({ userProperties })
  .setQoS(1)
  .build();
client.send('replace-emoji', record).subscribe(...);
```

> info **提示** `MqttRecordBuilder` 类从 `@nestjs/microservices` 包中导出。

同时，您也可以在服务器端读取这些选项，方法是访问 `MqttContext`。

```typescript
@@filename()
@MessagePattern('replace-emoji')
replaceEmoji(@Payload() data: string, @Ctx() context: MqttContext): string {
  const { properties: { userProperties } } = context.getPacket();
  return userProperties['x-version'] === '1.0.0' ? '🐱' : '🐈';
}
@@switch
@Bind(Payload(), Ctx())
@MessagePattern('replace-emoji')
replaceEmoji(data, context) {
  const { properties: { userProperties } } = context.getPacket();
  return userProperties['x-version'] === '1.0.0' ? '🐱' : '🐈';
}
```

在某些情况下，您可能希望为多个请求配置用户属性，可以将这些选项传递给 `ClientProxyFactory`。

```typescript
import { Module } from '@nestjs/common';
import { ClientProxyFactory, Transport } from '@nestjs/microservices';

@Module({
  providers: [
    {
      provide: 'API_v1',
      useFactory: () =>
        ClientProxyFactory.create({
          transport: Transport.MQTT,
          options: {
            url: 'mqtt://localhost:1833',
            userProperties: { 'x-version': '1.0.0' },
          },
        }),
    },
  ],
})
export class ApiModule {}
```

#### 实例状态更新

要实时获取连接和底层驱动程序实例的状态更新，可以订阅 `status` 流。此流提供特定于所选驱动程序的状态更新。对于MQTT驱动程序，`status` 流会发出 `connected`、`disconnected`、`reconnecting` 和 `closed` 事件。

```typescript
this.client.status.subscribe((status: MqttStatus) => {
  console.log(status);
});
```

> info **提示** `MqttStatus` 类型从 `@nestjs/microservices` 包中导入。

同样，您可以订阅服务器的 `status` 流以接收服务器状态的通知。

```typescript
const server = app.connectMicroservice<MicroserviceOptions>(...);
server.status.subscribe((status: MqttStatus) => {
  console.log(status);
});
```

#### 监听MQTT事件

在某些情况下，您可能希望监听微服务发出的内部事件。例如，您可以监听 `error` 事件，以便在发生错误时触发其他操作。为此，请使用 `on()` 方法，如下所示：

```typescript
this.client.on('error', (err) => {
  console.error(err);
});
```

同样，您可以监听服务器的内部事件：

```typescript
server.on<MqttEvents>('error', (err) => {
  console.error(err);
});
```

> info **提示** `MqttEvents` 类型从 `@nestjs/microservices` 包中导入。

#### 访ing底层驱动程序实例

对于更高级的用例，您可能需要访问底层驱动程序实例。这在手动关闭连接或使用驱动程序特定方法的场景中非常有用。但请注意，对于大多数情况，您**不需要**直接访问驱动程序。

为此，您可以使用 `unwrap()` 方法，该方法返回底层驱动程序实例。泛型类型参数应指定您期望的驱动程序实例类型。

```typescript
const mqttClient = this.client.unwrap<import('mqtt').MqttClient>();
```

同样，您可以访问服务器的底层驱动程序实例：

```typescript
const mqttClient = server.unwrap<import('mqtt').MqttClient>();
```