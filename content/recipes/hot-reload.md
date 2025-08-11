### 热重载

对应用程序启动过程影响最大的是 **TypeScript 编译**。幸运的是，使用 [webpack](https://github.com/webpack/webpack) 的 HMR（模块热替换）功能，我们不需要每次更改代码时都重新编译整个项目。这大大减少了实例化应用程序所需的时间，也使迭代开发变得更加容易。

> warning **警告** 请注意，`webpack` 不会自动将你的资源文件（例如 `graphql` 文件）复制到 `dist` 文件夹。同样，`webpack` 也不兼容 glob 静态路径（例如 `TypeOrmModule` 中的 `entities` 属性）。

### 使用 CLI

如果你使用的是 [Nest CLI](https://docs.nestjs.com/cli/overview)，配置过程非常简单。CLI 封装了 `webpack`，这使得我们可以使用 `HotModuleReplacementPlugin` 插件。

#### 安装

首先安装所需的包：

```bash
$ npm i --save-dev webpack-node-externals run-script-webpack-plugin webpack
```

> info **提示** 如果你使用的是 **Yarn Berry**（而非经典的 Yarn），请安装 `webpack-pnp-externals` 包，而不是 `webpack-node-externals`。

#### 配置

安装完成后，在应用程序的根目录下创建一个 `webpack-hmr.config.js` 文件。

```typescript
const nodeExternals = require('webpack-node-externals');
const { RunScriptWebpackPlugin } = require('run-script-webpack-plugin');

module.exports = function (options, webpack) {
  return {
    ...options,
    entry: ['webpack/hot/poll?100', options.entry],
    externals: [
      nodeExternals({
        allowlist: ['webpack/hot/poll?100'],
      }),
    ],
    plugins: [
      ...options.plugins,
      new webpack.HotModuleReplacementPlugin(),
      new webpack.WatchIgnorePlugin({
        paths: [/\.js$/, /\.d\.ts$/],
      }),
      new RunScriptWebpackPlugin({ name: options.output.filename, autoRestart: false }),
    ],
  };
};
```

> info **提示** 对于 **Yarn Berry**（非经典 Yarn），请不要使用 `externals` 配置属性中的 `nodeExternals`，而是使用 `webpack-pnp-externals` 包中的 `WebpackPnpExternals`：`WebpackPnpExternals({{ '{' }} exclude: ['webpack/hot/poll?100'] {{ '}' }})`。

该函数接收一个包含默认 webpack 配置的对象作为第一个参数，以及由 Nest CLI 使用的底层 `webpack` 包的引用作为第二个参数。此外，它返回一个修改后的 webpack 配置，其中包含了 `HotModuleReplacementPlugin`、`WatchIgnorePlugin` 和 `RunScriptWebpackPlugin` 插件。

#### 模块热替换（HMR）

要启用 **HMR**，请打开应用程序的入口文件（`main.ts`），并添加以下与 webpack 相关的指令：

```typescript
declare const module: any;

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(process.env.PORT ?? 3000);

  if (module.hot) {
    module.hot.accept();
    module.hot.dispose(() => app.close());
  }
}
bootstrap();
```

为了简化执行过程，请在 `package.json` 文件中添加一条脚本：

```json
"start:dev": "nest build --webpack --webpackPath webpack-hmr.config.js --watch"
```

现在只需打开命令行并运行以下命令即可：

```bash
$ npm run start:dev
```

### 不使用 CLI

如果你没有使用 [Nest CLI](https://docs.nestjs.com/cli/overview)，配置会稍微复杂一些（需要手动完成更多步骤）。

#### 安装

首先安装所需的包：

```bash
$ npm i --save-dev webpack webpack-cli webpack-node-externals ts-loader run-script-webpack-plugin
```

> info **提示** 如果你使用的是 **Yarn Berry**（而非经典的 Yarn），请安装 `webpack-pnp-externals` 包，而不是 `webpack-node-externals`。

#### 配置

安装完成后，在应用程序的根目录下创建一个 `webpack.config.js` 文件。

```typescript
const webpack = require('webpack');
const path = require('path');
const nodeExternals = require('webpack-node-externals');
const { RunScriptWebpackPlugin } = require('run-script-webpack-plugin');

module.exports = {
  entry: ['webpack/hot/poll?100', './src/main.ts'],
  target: 'node',
  externals: [
    nodeExternals({
      allowlist: ['webpack/hot/poll?100'],
    }),
  ],
  module: {
    rules: [
      {
        test: /.tsx?$/,
        use: 'ts-loader',
        exclude: /node_modules/,
      },
    ],
  },
  mode: 'development',
  resolve: {
    extensions: ['.tsx', '.ts', '.js'],
  },
  plugins: [new webpack.HotModuleReplacementPlugin(), new RunScriptWebpackPlugin({ name: 'server.js', autoRestart: false })],
  output: {
    path: path.join(__dirname, 'dist'),
    filename: 'server.js',
  },
};
```

> info **提示** 对于 **Yarn Berry**（非经典 Yarn），请不要使用 `externals` 配置属性中的 `nodeExternals`，而是使用 `webpack-pnp-externals` 包中的 `WebpackPnpExternals`：`WebpackPnpExternals({{ '{' }} exclude: ['webpack/hot/poll?100'] {{ '}' }})`。

该配置向 webpack 说明了关于你的应用程序的一些关键信息：入口文件的位置、用于存放 **编译后** 文件的目录、以及你想用来编译源文件的加载器。通常，即使你并不完全理解所有选项，也可以直接使用这个配置文件。

#### 模块热替换（HMR）

要启用 **HMR**，请打开应用程序的入口文件（`main.ts`），并添加以下与 webpack 相关的指令：

```typescript
declare const module: any;

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(process.env.PORT ?? 3000);

  if (module.hot) {
    module.hot.accept();
    module.hot.dispose(() => app.close());
  }
}
bootstrap();
```

为了简化执行过程，请在 `package.json` 文件中添加一条脚本：

```json
"start:dev": "webpack --config webpack.config.js --watch"
```

现在只需打开命令行并运行以下命令即可：

```bash
$ npm run start:dev
```

#### 示例

一个可用的示例请参见 [此处](https://github.com/nestjs/nest/tree/master/sample/08-webpack)。