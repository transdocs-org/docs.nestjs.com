### 概览

> info **提示** 本章节介绍 Nest Devtools 与 Nest 框架的集成。如果你在寻找 Devtools 应用，请访问 [Devtools](https://devtools.nestjs.com) 网站。

要开始调试你的本地应用，请打开 `main.ts` 文件，并确保在应用选项对象中将 `snapshot` 属性设置为 `true`，如下所示：

```typescript
async function bootstrap() {
  const app = await NestFactory.create(AppModule, {
    snapshot: true,
  });
  await app.listen(process.env.PORT ?? 3000);
}
```

这将指示框架收集必要的元数据，使 Nest Devtools 能够可视化你的应用图谱。

接下来，安装所需的依赖：

```bash
$ npm i @nestjs/devtools-integration
```

> warning **警告** 如果你在应用中使用了 `@nestjs/graphql` 包，请确保安装最新版本（`npm i @nestjs/graphql@11`）。

安装完该依赖后，打开 `app.module.ts` 文件，并导入我们刚刚安装的 `DevtoolsModule`：

```typescript
@Module({
  imports: [
    DevtoolsModule.register({
      http: process.env.NODE_ENV !== 'production',
    }),
  ],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

> warning **警告** 这里我们检查 `NODE_ENV` 环境变量的原因是：**你永远不应该在生产环境中使用该模块！**

一旦导入了 `DevtoolsModule` 并且你的应用正在运行（使用 `npm run start:dev`），你应该能够访问 [Devtools](https://devtools.nestjs.com) 网址并看到应用的图谱。

<figure><img src="/assets/devtools/modules-graph.png" /></figure>

> info **提示** 如上图所示，每个模块都连接到了 `InternalCoreModule`。`InternalCoreModule` 是一个全局模块，始终被导入到根模块中。由于它被注册为全局节点，Nest 会自动在所有模块与 `InternalCoreModule` 节点之间创建连接。如果你想在图谱中隐藏全局模块，可以使用侧边栏中的 "**Hide global modules（隐藏全局模块）**" 复选框。

因此我们可以看到，`DevtoolsModule` 会使你的应用暴露一个额外的 HTTP 服务器（端口 8000），供 Devtools 应用对你的应用进行内省。

为了确认一切按预期运行，将图谱视图切换为 "Classes（类）"。你应该会看到如下界面：

<figure><img src="/assets/devtools/classes-graph.png" /></figure>

要聚焦于某个特定节点，点击矩形区域，图谱会弹出一个带有 **"Focus（聚焦）** 按钮的窗口。你也可以使用侧边栏中的搜索栏来查找特定节点。

> info **提示** 如果你点击 **Inspect（检查）** 按钮，应用会跳转到 `/debug` 页面，并选中该特定节点。

<figure><img src="/assets/devtools/node-popup.png" /></figure>

> info **提示** 要将图谱导出为图片，请点击图谱右上角的 **Export as PNG（导出为 PNG）** 按钮。

使用位于侧边栏（左侧）的表单控件，你可以控制连接的接近程度，例如可视化特定的应用子树：

<figure><img src="/assets/devtools/subtree-view.png" /></figure>

当你团队中有 **新开发者** 并想向他们展示你的应用结构时，这个功能尤其有用。你也可以使用此功能可视化某个特定模块（例如 `TasksModule`）及其所有依赖项，这在将大型应用拆分为更小模块（例如独立的微服务）时非常方便。

你可以观看以下视频，了解 **Graph Explorer（图谱探索器）** 功能的实际使用效果：

<figure>
  <iframe
    width="1000"
    height="565"
    src="https://www.youtube.com/embed/bW8V-ssfnvM"
    title="YouTube video player"
    frameBorder="0"
    allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
    allowFullScreen
  ></iframe>
</figure>

#### 调查 "Cannot resolve dependency（无法解析依赖）" 错误

> info **注意** 此功能适用于 `@nestjs/core` >= `v9.3.10`。

你可能遇到的最常见错误信息之一就是 Nest 无法解析某个提供者的依赖。使用 Nest Devtools，你可以轻松识别问题并学习如何解决它。

首先，打开 `main.ts` 文件并更新 `bootstrap()` 调用，如下所示：

```typescript
bootstrap().catch((err) => {
  fs.writeFileSync('graph.json', PartialGraphHost.toString() ?? '');
  process.exit(1);
});
```

同时确保将 `abortOnError` 设置为 `false`：

```typescript
const app = await NestFactory.create(AppModule, {
  snapshot: true,
  abortOnError: false, // <--- 此处
});
```

现在，每当你的应用因 **"Cannot resolve dependency"（无法解析依赖）** 错误而启动失败时，你会在根目录中发现一个名为 `graph.json` 的文件（代表部分图谱）。你可以将此文件拖放至 Devtools 中（请确保将当前模式从 "Interactive（交互式）" 切换为 "Preview（预览）"）：

<figure><img src="/assets/devtools/drag-and-drop.png" /></figure>

上传成功后，你应该会看到如下图谱和对话框：

<figure><img src="/assets/devtools/partial-graph-modules-view.png" /></figure>

如你所见，高亮显示的 `TasksModule` 是我们需要关注的模块。此外，在对话框中你还可以看到一些修复此问题的指导建议。

如果我们切换到 "Classes（类）" 视图，将看到如下内容：

<figure><img src="/assets/devtools/partial-graph-classes-view.png" /></figure>

该图谱表明，我们想注入到 `TasksService` 中的 `DiagnosticsService` 在 `TasksModule` 模块上下文中未找到，因此我们很可能只需要将 `DiagnosticsModule` 导入到 `TasksModule` 模块中即可修复此问题！

#### 路由探索器

当你进入 **Routes explorer（路由探索器）** 页面时，你应该能看到所有已注册的入口点：

<figure><img src="/assets/devtools/routes.png" /></figure>

> info **提示** 此页面不仅显示 HTTP 路由，还显示所有其他入口点（例如 WebSockets、gRPC、GraphQL 解析器等）。

入口点会根据其所属控制器进行分组。你也可以使用搜索栏查找特定入口点。

如果你点击某个特定入口点，**流程图** 将会显示。该图展示了该入口点的执行流程（例如绑定到该路由的守卫、拦截器、管道等）。当你想了解特定路由的请求/响应周期，或调试某个特定守卫/拦截器/管道未执行的原因时，这非常有用。

#### 沙箱

要即时执行 JavaScript 代码并与你的应用实时交互，请进入 **Sandbox（沙箱）** 页面：

<figure><img src="/assets/devtools/sandbox.png" /></figure>

该交互式环境可用于**实时**测试和调试 API 端点，让开发者能够快速识别和修复问题，而无需使用例如 HTTP 客户端。我们还可以绕过身份验证层，因此不再需要额外的登录步骤，甚至不需要为测试创建特殊用户账号。对于事件驱动的应用，我们也可以直接从沙箱触发事件，并查看应用的响应。

所有日志都会输出到沙箱的控制台中，因此我们可以轻松查看发生了什么。

只需**即时**执行代码，即可立即看到结果，而无需重新构建应用或重启服务器。

<figure><img src="/assets/devtools/sandbox-table.png" /></figure>

> info **提示** 要漂亮地显示对象数组，请使用 `console.table()`（或简写为 `table()`）函数。

你可以观看以下视频，了解 **Interactive Playground（交互式游乐场）** 功能的实际使用效果：

<figure>
  <iframe
    width="1000"
    height="565"
    src="https://www.youtube.com/embed/liSxEN_VXKM"
    title="YouTube video player"
    frameBorder="0"
    allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
    allowFullScreen
  ></iframe>
</figure>

#### 启动性能分析器

要查看所有类节点（控制器、提供者、增强器等）及其对应的实例化时间，请进入 **Bootstrap performance（启动性能）** 页面：

<figure><img src="/assets/devtools/bootstrap-performance.png" /></figure>

此页面在你想识别应用启动过程中最慢的部分时非常有用（例如当你想优化应用的启动时间时，这在例如无服务器环境中至关重要）。

#### 审计

要查看应用在分析序列化图谱时自动生成的审计信息（错误/警告/提示），请进入 **Audit（审计）** 页面：

<figure><img src="/assets/devtools/audit.png" /></figure>

> info **提示** 上图并未展示所有可用的审计规则。

当你想识别应用中潜在问题时，此页面非常实用。

#### 预览静态文件

要将序列化图谱保存到文件中，请使用以下代码：

```typescript
await app.listen(process.env.PORT ?? 3000); // 或者 await app.init()
fs.writeFileSync('./graph.json', app.get(SerializedGraph).toString());
```

> info **提示** `SerializedGraph` 是从 `@nestjs/core` 包中导出的。

然后你可以拖放或上传该文件：

<figure><img src="/assets/devtools/drag-and-drop.png" /></figure>

这在你想与他人（例如同事）分享你的图谱，或想离线分析时非常有用。