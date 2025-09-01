### 独立应用程序

挂载 Nest 应用程序有几种方式。你可以创建一个 Web 应用、一个微服务，或者一个不带任何网络监听器的 Nest **独立应用程序（standalone application）**。Nest 独立应用程序是对 Nest **IoC 容器**的封装，这个容器保存了所有已实例化的类。我们可以直接通过独立应用程序对象，从任何导入的模块中获取对任意现有实例的引用。因此，你可以在任何地方使用 Nest 框架的功能，例如在脚本化的 **CRON 任务** 中。你甚至可以在其基础上构建一个 **CLI 工具**。

#### 入门指南

要创建一个 Nest 独立应用程序，请使用以下代码结构：

```typescript
@@filename()
async function bootstrap() {
  const app = await NestFactory.createApplicationContext(AppModule);
  // 你的应用程序逻辑在这里 ...
}
bootstrap();
```

#### 从静态模块中获取提供者

独立应用程序对象允许你获取 Nest 应用程序中注册的任意实例。假设我们在 `TasksModule` 模块中有一个 `TasksService` 提供者，并且该模块被我们的 `AppModule` 导入。这个类提供了一组我们想在 CRON 任务中调用的方法。

```typescript
@@filename()
const tasksService = app.get(TasksService);
```

要访问 `TasksService` 实例，可以使用 `get()` 方法。`get()` 方法像一个**查询**，会在每个已注册的模块中查找实例。你可以传递任意提供者的 token。或者，为了进行严格的上下文检查，可以传入一个带有 `strict: true` 属性的选项对象。启用该选项后，你必须通过特定模块导航以从所选上下文中获取特定实例。

```typescript
@@filename()
const tasksService = app.select(TasksModule).get(TasksService, { strict: true });
```

以下是对从独立应用程序对象中获取实例引用的方法的总结：

<table>
  <tr>
    <td>
      <code>get()</code>
    </td>
    <td>
      获取应用程序上下文中可用的控制器或提供者（包括守卫、过滤器等）的实例。
    </td>
  </tr>
  <tr>
    <td>
      <code>select()</code>
    </td>
    <td>
      遍历模块图以从选定的模块中提取特定实例（与上述的严格模式一起使用）。
    </td>
  </tr>
</table>

> info **提示** 在非严格模式下，默认选择根模块。要选择其他模块，你需要手动逐步遍历模块图。

请注意，独立应用程序没有网络监听器，因此在此上下文中不可用任何与 HTTP 相关的 Nest 功能（例如中间件、拦截器、管道、守卫等）。

例如，即使你在应用程序中注册了一个全局拦截器，然后使用 `app.get()` 方法获取控制器的实例，该拦截器也不会执行。

#### 从动态模块中获取提供者

处理 [动态模块](/fundamentals/dynamic-modules) 时，我们应该向 `app.select` 提供相同的对象，该对象代表应用中已注册的动态模块。例如：

```typescript
@@filename()
export const dynamicConfigModule = ConfigModule.register({ folder: './config' });

@Module({
  imports: [dynamicConfigModule],
})
export class AppModule {}
```

然后你可以稍后选择该模块：

```typescript
@@filename()
const configService = app.select(dynamicConfigModule).get(ConfigService, { strict: true });
```

#### 终止阶段

如果你希望脚本执行完毕后 Node 应用程序关闭（例如用于运行 CRON 任务的脚本），你必须在 `bootstrap` 函数的最后调用 `app.close()` 方法，如下所示：

```typescript
@@filename()
async function bootstrap() {
  const app = await NestFactory.createApplicationContext(AppModule);
  // 应用程序逻辑...
  await app.close();
}
bootstrap();
```

正如 [生命周期事件](/fundamentals/lifecycle-events) 章节中提到的，这将触发生命周期钩子。

#### 示例

一个完整的示例请参见 [这里](https://github.com/nestjs/nest/tree/master/sample/18-context)。