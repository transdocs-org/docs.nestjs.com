### 事件

[事件发射器](https://www.npmjs.com/package/@nestjs/event-emitter) 包（`@nestjs/event-emitter`）提供了一个简单的观察者实现，允许你订阅并监听应用程序中发生的各种事件。由于单个事件可以有多个互不依赖的监听器，因此事件是一种将应用程序的不同部分解耦的绝佳方式。

`EventEmitterModule` 在内部使用了 [eventemitter2](https://github.com/EventEmitter2/EventEmitter2) 包。

#### 入门

首先安装所需的包：

```shell
$ npm i --save @nestjs/event-emitter
```

安装完成后，将 `EventEmitterModule` 导入到根模块 `AppModule`，并调用静态方法 `forRoot()`，如下所示：

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { EventEmitterModule } from '@nestjs/event-emitter';

@Module({
  imports: [
    EventEmitterModule.forRoot()
  ],
})
export class AppModule {}
```

`.forRoot()` 方法会初始化事件发射器，并注册应用程序中存在的所有声明式事件监听器。注册会在 `onApplicationBootstrap` 生命周期钩子中发生，确保所有模块都已加载并声明了任何计划任务。

要配置底层的 `EventEmitter` 实例，请将配置对象传递给 `.forRoot()` 方法，如下所示：

```typescript
EventEmitterModule.forRoot({
  // 设置为 `true` 以使用通配符
  wildcard: false,
  // 用于分隔命名空间的分隔符
  delimiter: '.',
  // 如果希望在添加新监听器时触发 newListener 事件，请设置为 `true`
  newListener: false,
  // 如果希望在移除监听器时触发 removeListener 事件，请设置为 `true`
  removeListener: false,
  // 可分配给一个事件的最大监听器数量
  maxListeners: 10,
  // 当分配的监听器数量超过最大限制时，在内存泄漏消息中显示事件名称
  verboseMemoryLeak: false,
  // 如果没有监听器时发出 error 事件，不抛出 uncaughtException 错误
  ignoreErrors: false,
});
```

#### 触发事件

要触发（即发射）一个事件，请首先通过标准的构造函数注入方式注入 `EventEmitter2`：

```typescript
constructor(private eventEmitter: EventEmitter2) {}
```

> info **提示** 从 `@nestjs/event-emitter` 包导入 `EventEmitter2`。

然后在类中使用它，如下所示：

```typescript
this.eventEmitter.emit(
  'order.created',
  new OrderCreatedEvent({
    orderId: 1,
    payload: {},
  }),
);
```

#### 监听事件

要声明一个事件监听器，请在方法定义前使用 `@OnEvent()` 装饰器，并在方法中编写要执行的代码，如下所示：

```typescript
@OnEvent('order.created')
handleOrderCreatedEvent(payload: OrderCreatedEvent) {
  // 处理 "OrderCreatedEvent" 事件
}
```

> warning **警告** 事件订阅者不能是请求作用域的。

第一个参数可以是用于简单事件发射器的 `string` 或 `symbol`，在通配符发射器的情况下，可以是 `string | symbol | Array<string | symbol>`。

第二个参数（可选）是一个监听器选项对象，如下所示：

```typescript
export type OnEventOptions = OnOptions & {
  /**
   * 如果为 "true"，将给定的监听器插入到监听器数组的开头而不是末尾。
   *
   * @see https://github.com/EventEmitter2/EventEmitter2#emitterprependlistenerevent-listener-options
   *
   * @default false
   */
  prependListener?: boolean;

  /**
   * 如果为 "true"，处理事件时不会抛出错误。否则，会抛出错误。
   *
   * @default true
   */
  suppressErrors?: boolean;
};
```

> info **提示** 有关 `OnOptions` 选项对象的更多信息，请参阅 [`eventemitter2`](https://github.com/EventEmitter2/EventEmitter2#emitteronevent-listener-options-objectboolean) 文档。

```typescript
@OnEvent('order.created', { async: true })
handleOrderCreatedEvent(payload: OrderCreatedEvent) {
  // 处理 "OrderCreatedEvent" 事件
}
```

要使用命名空间/通配符，请将 `wildcard` 选项传递给 `EventEmitterModule#forRoot()` 方法。当启用命名空间/通配符时，事件可以是用分隔符分隔的字符串（如 `foo.bar`）或数组（如 `['foo', 'bar']`）。分隔符也是可配置的配置项（`delimiter`）。启用命名空间功能后，你可以使用通配符订阅事件：

```typescript
@OnEvent('order.*')
handleOrderEvents(payload: OrderCreatedEvent | OrderRemovedEvent | OrderUpdatedEvent) {
  // 处理事件
}
```

请注意，这种通配符仅适用于一个层级。例如，`order.*` 参数将匹配 `order.created` 和 `order.shipped`，但不匹配 `order.delayed.out_of_stock`。要监听此类事件，请使用 `多级通配符` 模式（即 `**`），详见 `EventEmitter2` [文档](https://github.com/EventEmitter2/EventEmitter2#multi-level-wildcards)。

使用此模式，你可以创建一个监听所有事件的事件监听器，例如：

```typescript
@OnEvent('**')
handleEverything(payload: any) {
  // 处理事件
}
```

> info **提示** `EventEmitter2` 类提供了许多用于与事件交互的有用方法，如 `waitFor` 和 `onAny`。你可以 [这里](https://github.com/EventEmitter2/EventEmitter2) 阅读更多相关内容。

#### 防止事件丢失

在 `onApplicationBootstrap` 生命周期钩子之前或期间触发的事件（例如来自模块构造函数或 `onModuleInit` 方法的事件）可能会被遗漏，因为 `EventSubscribersLoader` 可能尚未完成监听器的注册。

为了避免此问题，你可以使用 `EventEmitterReadinessWatcher` 的 `waitUntilReady` 方法，该方法返回一个 promise，在所有监听器注册完成后解析。你可以在模块的 `onApplicationBootstrap` 生命周期钩子中调用此方法，以确保所有事件都被正确捕获。

```typescript
await this.eventEmitterReadinessWatcher.waitUntilReady();
this.eventEmitter.emit(
  'order.created',
  new OrderCreatedEvent({ orderId: 1, payload: {} }),
);
```

> info **注意** 此方法仅在 `onApplicationBootstrap` 生命周期钩子完成之前触发的事件需要使用。

#### 示例

一个完整示例可以在 [这里](https://github.com/nestjs/nest/tree/master/sample/30-event-emitter) 找到。