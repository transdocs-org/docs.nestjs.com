### 静态资源服务

为了提供像单页应用（SPA）这样的静态内容，我们可以使用 `@nestjs/serve-static` 包提供的 `ServeStaticModule`。

#### 安装

首先，我们需要安装所需的包：

```bash
$ npm install --save @nestjs/serve-static
```

#### 初始化

安装完成后，我们可以将 `ServeStaticModule` 导入到根模块 `AppModule` 中，并通过向 `forRoot()` 方法传递配置对象来对其进行配置。

```typescript
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { ServeStaticModule } from '@nestjs/serve-static';
import { join } from 'path';

@Module({
  imports: [
    ServeStaticModule.forRoot({
      rootPath: join(__dirname, '..', 'client'),
    }),
  ],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

完成上述配置后，构建你的静态网站，并将其内容放置在 `rootPath` 属性指定的目录中。

#### 配置

[ServeStaticModule](https://github.com/nestjs/serve-static) 提供了多种配置选项以自定义其行为。
你可以设置用于渲染静态应用的路径、指定排除的路径、启用或禁用 Cache-Control 响应头等。完整的配置选项列表请参见[这里](https://github.com/nestjs/serve-static/blob/master/lib/interfaces/serve-static-options.interface.ts)。

> warning **注意** 静态应用的默认 `renderPath` 是 `*`（所有路径），模块会响应 "index.html" 文件。
> 这样你就可以为你的 SPA 创建客户端路由。你在控制器中定义的路径会回退到服务器。
> 你可以通过设置 `serveRoot` 和 `renderPath` 并结合其他选项来改变这一行为。
> 另外，在 Fastify 适配器中实现了 `serveStaticOptions.fallthrough` 选项，用于模拟 Express 的 fallthrough 行为，将其设置为 `true` 后，当访问不存在的路由时会返回 `index.html` 而不是 404 错误。

#### 示例

一个可用的示例请参见[这里](https://github.com/nestjs/nest/tree/master/sample/24-serve-static)。