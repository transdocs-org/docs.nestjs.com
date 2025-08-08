### 嵌套命令（Nest Commander）

在 [独立应用程序](/standalone-applications) 文档的基础上，还有一个 [nest-commander](https://jmcdo29.github.io/nest-commander) 包，用于以类似于典型的 Nest 应用程序的结构来编写命令行应用程序。

> info **提示** `nest-commander` 是一个第三方包，不由 NestJS 核心团队整体维护。请在 [对应的仓库](https://github.com/jmcdo29/nest-commander/issues/new/choose) 中报告您在使用该库时遇到的任何问题。

#### 安装

与任何其他包一样，使用前需要先安装它。

```bash
$ npm i nest-commander
```

#### 命令文件

`nest-commander` 通过类上的 `@Command()` 装饰器和类方法上的 `@Option()` 装饰器，使用 [装饰器](https://www.typescriptlang.org/docs/handbook/decorators.html) 轻松编写新的命令行应用程序。每个命令文件都应实现 `CommandRunner` 抽象类，并使用 `@Command()` 装饰器进行装饰。

每个命令在 Nest 中都视为一个 `@Injectable()`，因此正常的依赖注入仍然可以按预期工作。需要注意的唯一一点是抽象类 `CommandRunner`，每个命令都应实现它。`CommandRunner` 抽象类确保所有命令都有一个 `run` 方法，该方法返回一个 `Promise<void>`，并接收参数 `string[]` 和 `Record<string, any>`。`run` 方法是你启动所有逻辑的地方，它将接收所有未匹配选项标志的参数，并将它们作为数组传递，以防你确实需要处理多个参数。至于选项，`Record<string, any>` 中的属性名与 `@Option()` 装饰器提供的 `name` 属性匹配，而它们的值则与选项处理函数的返回值匹配。如果你希望获得更好的类型安全性，也可以为你的选项创建一个接口。

#### 运行命令

类似于在 NestJS 应用程序中使用 `NestFactory` 创建服务器并通过 `listen` 运行它的方式，`nest-commander` 包提供了一个简单易用的 API 来运行你的命令。导入 `CommandFactory` 并使用 `static` 方法 `run`，传入应用程序的根模块。这看起来可能如下所示：

```ts
import { CommandFactory } from 'nest-commander';
import { AppModule } from './app.module';

async function bootstrap() {
  await CommandFactory.run(AppModule);
}

bootstrap();
```

默认情况下，使用 `CommandFactory` 时 Nest 的日志记录器是禁用的。不过你可以通过 `run` 函数的第二个参数传入日志记录器。你可以提供一个自定义的 NestJS 日志记录器，或者提供一个你想保留的日志级别的数组 —— 如果你只想打印 Nest 的错误日志，至少提供 `['error']` 可能会很有用。

```ts
import { CommandFactory } from 'nest-commander';
import { AppModule } from './app.module';
import { LogService } './log.service';

async function bootstrap() {
  await CommandFactory.run(AppModule, new LogService());

  // 或者，如果你只想打印 Nest 的警告和错误日志
  await CommandFactory.run(AppModule, ['warn', 'error']);
}

bootstrap();
```

就这样。在底层，`CommandFactory` 会自动调用 `NestFactory` 并在必要时调用 `app.close()`，因此你无需担心内存泄漏的问题。如果你需要添加一些错误处理，可以使用 `try/catch` 包裹 `run` 调用，或者可以在 `bootstrap()` 调用后链接 `.catch()` 方法。

#### 测试

如果你无法轻松测试一个超级棒的命令行脚本，那写它又有什么用呢？幸运的是，`nest-commander` 提供了一些工具，可以很好地与 NestJS 生态系统集成，对于熟悉 Nest 的开发者来说会感到非常亲切。在测试模式下构建命令时，你可以使用 `CommandTestFactory` 而不是 `CommandFactory`，并传入你的元数据，其工作方式非常类似于 `@nestjs/testing` 中的 `Test.createTestingModule`。事实上，它在底层使用了这个包。你仍然可以在调用 `compile()` 之前链式调用 `overrideProvider` 方法，以便在测试中替换 DI 组件。

#### 综合示例

以下类相当于一个 CLI 命令，它可以接收子命令 `basic`，也可以直接调用，支持 `-n`、`-s` 和 `-b`（以及它们的长格式），并且每个选项都有自定义解析器。如常规 commander 所期望的那样，也支持 `--help` 选项。

```ts
import { Command, CommandRunner, Option } from 'nest-commander';
import { LogService } from './log.service';

interface BasicCommandOptions {
  string?: string;
  boolean?: boolean;
  number?: number;
}

@Command({ name: 'basic', description: '一个参数解析器' })
export class BasicCommand extends CommandRunner {
  constructor(private readonly logService: LogService) {
    super()
  }

  async run(
    passedParam: string[],
    options?: BasicCommandOptions,
  ): Promise<void> {
    if (options?.boolean !== undefined && options?.boolean !== null) {
      this.runWithBoolean(passedParam, options.boolean);
    } else if (options?.number) {
      this.runWithNumber(passedParam, options.number);
    } else if (options?.string) {
      this.runWithString(passedParam, options.string);
    } else {
      this.runWithNone(passedParam);
    }
  }

  @Option({
    flags: '-n, --number [number]',
    description: '一个基本的数字解析器',
  })
  parseNumber(val: string): number {
    return Number(val);
  }

  @Option({
    flags: '-s, --string [string]',
    description: '一个字符串返回值',
  })
  parseString(val: string): string {
    return val;
  }

  @Option({
    flags: '-b, --boolean [boolean]',
    description: '一个布尔值解析器',
  })
  parseBoolean(val: string): boolean {
    return JSON.parse(val);
  }

  runWithString(param: string[], option: string): void {
    this.logService.log({ param, string: option });
  }

  runWithNumber(param: string[], option: number): void {
    this.logService.log({ param, number: option });
  }

  runWithBoolean(param: string[], option: boolean): void {
    this.logService.log({ param, boolean: option });
  }

  runWithNone(param: string[]): void {
    this.logService.log({ param });
  }
}
```

确保将命令类添加到模块中：

```ts
@Module({
  providers: [LogService, BasicCommand],
})
export class AppModule {}
```

现在，为了能够在你的 main.ts 中运行 CLI，你可以这样做：

```ts
async function bootstrap() {
  await CommandFactory.run(AppModule);
}

bootstrap();
```

就这样，你已经拥有了一个命令行应用程序。

#### 更多信息

访问 [nest-commander 文档网站](https://jmcdo29.github.io/nest-commander)，获取更多信息、示例和 API 文档。