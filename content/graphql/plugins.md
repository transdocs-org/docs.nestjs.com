### 使用 Apollo 的插件

插件使您能够通过在响应特定事件时执行自定义操作来扩展 Apollo Server 的核心功能。目前，这些事件对应于 GraphQL 请求生命周期的各个阶段，以及 Apollo Server 自身的启动（详情请[点击此处](https://www.apollographql.com/docs/apollo-server/integrations/plugins/)）。例如，一个基本的日志插件可能会记录发送到 Apollo Server 的每个请求所关联的 GraphQL 查询字符串。

#### 自定义插件

要创建插件，请声明一个使用 `@nestjs/apollo` 包导出的 `@Plugin` 装饰器的类。此外，为了获得更好的代码自动补全功能，建议实现 `@apollo/server` 包中的 `ApolloServerPlugin` 接口。

```typescript
import { ApolloServerPlugin, GraphQLRequestListener } from '@apollo/server';
import { Plugin } from '@nestjs/apollo';

@Plugin()
export class LoggingPlugin implements ApolloServerPlugin {
  async requestDidStart(): Promise<GraphQLRequestListener<any>> {
    console.log('请求开始');
    return {
      async willSendResponse() {
        console.log('即将发送响应');
      },
    };
  }
}
```

完成上述设置后，我们可以将 `LoggingPlugin` 注册为提供者。

```typescript
@Module({
  providers: [LoggingPlugin],
})
export class CommonModule {}
```

Nest 会自动实例化插件并将其应用到 Apollo Server。

#### 使用外部插件

Apollo 提供了一些开箱即用的插件。要使用现有插件，只需导入它并添加到 `plugins` 数组中：

```typescript
GraphQLModule.forRoot({
  // ...
  plugins: [ApolloServerOperationRegistry({ /* 选项 */ })]
}),
```

> info **提示** `ApolloServerOperationRegistry` 插件是从 `@apollo/server-plugin-operation-registry` 包中导出的。

#### 与 Mercurius 配合使用的插件

某些现有的 Mercurius 特定 Fastify 插件必须在 Mercurius 插件之后加载到插件树中（详情请[点击此处](https://mercurius.dev/#/docs/plugins)）。

> warning **警告** [mercurius-upload](https://github.com/mercurius-js/mercurius-upload) 是一个例外，应该在主文件中注册。

为此，`MercuriusDriver` 暴露了一个可选的 `plugins` 配置选项。它是一个对象数组，每个对象包含两个属性：`plugin` 和 `options`。因此，注册 [cache 插件](https://github.com/mercurius-js/cache) 将如下所示：

```typescript
GraphQLModule.forRoot({
  driver: MercuriusDriver,
  // ...
  plugins: [
    {
      plugin: cache,
      options: {
        ttl: 10,
        policy: {
          Query: {
            add: true
          }
        }
      },
    }
  ]
}),
```