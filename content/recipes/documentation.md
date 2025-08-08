### 文档

**Compodoc** 是一个用于 Angular 应用程序的文档生成工具。由于 Nest 与 Angular 拥有相似的项目和代码结构，**Compodoc** 同样适用于 Nest 应用程序。

#### 安装配置

在已有的 Nest 项目中安装配置 Compodoc 非常简单。首先，在你的操作系统终端中运行以下命令以添加开发依赖：

```bash
$ npm i -D @compodoc/compodoc
```

#### 生成文档

使用以下命令生成项目文档（需要 npm 6 版本以支持 `npx` 功能）。更多选项请参阅[官方文档](https://compodoc.app/guides/usage.html)。

```bash
$ npx @compodoc/compodoc -p tsconfig.json -s
```

打开浏览器并访问 [http://localhost:8080](http://localhost:8080)，你将看到一个初始的 Nest CLI 项目界面：

<figure><img src="/assets/documentation-compodoc-1.jpg" /></figure>
<figure><img src="/assets/documentation-compodoc-2.jpg" /></figure>

#### 参与贡献

你可以在这里[参与并贡献](https://github.com/compodoc/compodoc) Compodoc 项目。