### 健康检查（Terminus）

Terminus 集成为你提供了 **就绪/存活** 健康检查功能。在复杂的后端架构中，健康检查至关重要。简而言之，健康检查通常由一个特殊的地址组成，例如：`https://my-website.com/health/readiness`。你的基础设施中的某个服务或组件（如 [Kubernetes](https://kubernetes.io/)）会持续检查这个地址。根据对该地址发起的 `GET` 请求返回的 HTTP 状态码，如果服务返回“不健康”的响应，它将采取相应的操作。

由于“健康”或“不健康”的定义取决于你提供的服务类型，**Terminus** 集成通过一组 **健康指标（health indicators）** 来帮助你定义这些状态。

例如，如果你的 Web 服务器使用 MongoDB 来存储数据，那么判断 MongoDB 是否仍在运行就变得非常重要。在这种情况下，你可以使用 `MongooseHealthIndicator`。如果配置正确——稍后会详细介绍——你的健康检查地址将根据 MongoDB 是否运行，返回健康或不健康的 HTTP 状态码。

#### 入门

要开始使用 `@nestjs/terminus`，我们需要安装所需的依赖。

```bash
$ npm install --save @nestjs/terminus
```

#### 设置健康检查

健康检查代表一组 **健康指标** 的汇总。健康指标用于检查某个服务是否处于健康或不健康的状态。如果所有分配的健康指标都正常，则健康检查结果为健康。由于很多应用程序需要类似的健康指标，[`@nestjs/terminus`](https://github.com/nestjs/terminus) 提供了一组预定义的指标，例如：

- `HttpHealthIndicator`
- `TypeOrmHealthIndicator`
- `MongooseHealthIndicator`
- `SequelizeHealthIndicator`
- `MikroOrmHealthIndicator`
- `PrismaHealthIndicator`
- `MicroserviceHealthIndicator`
- `GRPCHealthIndicator`
- `MemoryHealthIndicator`
- `DiskHealthIndicator`

要开始我们的第一个健康检查，我们先创建 `HealthModule` 并在它的导入数组中导入 `TerminusModule`。

> info **提示** 要使用 [Nest CLI](cli/overview) 创建模块，只需执行 `$ nest g module health` 命令。

```typescript
@@filename(health.module)
import { Module } from '@nestjs/common';
import { TerminusModule } from '@nestjs/terminus';

@Module({
  imports: [TerminusModule]
})
export class HealthModule {}
```

我们可以使用 [控制器（controller）](/controllers) 来执行健康检查，可以使用 [Nest CLI](cli/overview) 轻松创建。

```bash
$ nest g controller health
```

> info **提示** 强烈建议在你的应用程序中启用关闭钩子（shutdown hooks）。如果启用，Terminus 集成将使用该生命周期事件。[了解更多](fundamentals/lifecycle-events#application-shutdown)。

#### HTTP 健康检查

一旦我们安装了 `@nestjs/terminus`、导入了 `TerminusModule` 并创建了一个新控制器，就可以创建一个健康检查。

`HTTPHealthIndicator` 需要 `@nestjs/axios` 包，请确保已安装：

```bash
$ npm i --save @nestjs/axios axios
```

现在我们可以设置 `HealthController`：

```typescript
@@filename(health.controller)
import { Controller, Get } from '@nestjs/common';
import { HealthCheckService, HttpHealthIndicator, HealthCheck } from '@nestjs/terminus';

@Controller('health')
export class HealthController {
  constructor(
    private health: HealthCheckService,
    private http: HttpHealthIndicator,
  ) {}

  @Get()
  @HealthCheck()
  check() {
    return this.health.check([
      () => this.http.pingCheck('nestjs-docs', 'https://docs.nestjs.com'),
    ]);
  }
}
@@switch
import { Controller, Dependencies, Get } from '@nestjs/common';
import { HealthCheckService, HttpHealthIndicator, HealthCheck } from '@nestjs/terminus';

@Controller('health')
@Dependencies(HealthCheckService, HttpHealthIndicator)
export class HealthController {
  constructor(
    private health,
    private http,
  ) { }

  @Get()
  @HealthCheck()
  healthCheck() {
    return this.health.check([
      () => this.http.pingCheck('nestjs-docs', 'https://docs.nestjs.com'),
    ])
  }
}
```

```typescript
@@filename(health.module)
import { Module } from '@nestjs/common';
import { TerminusModule } from '@nestjs/terminus';
import { HttpModule } from '@nestjs/axios';
import { HealthController } from './health.controller';

@Module({
  imports: [TerminusModule, HttpModule],
  controllers: [HealthController],
})
export class HealthModule {}
@@switch
import { Module } from '@nestjs/common';
import { TerminusModule } from '@nestjs/terminus';
import { HttpModule } from '@nestjs/axios';
import { HealthController } from './health.controller';

@Module({
  imports: [TerminusModule, HttpModule],
  controllers: [HealthController],
})
export class HealthModule {}
```

现在我们的健康检查将向 `https://docs.nestjs.com` 发送一个 _GET_ 请求。如果从该地址获取到了健康的响应，那么访问 `http://localhost:3000/health` 将返回以下对象，并附带 200 状态码。

```json
{
  "status": "ok",
  "info": {
    "nestjs-docs": {
      "status": "up"
    }
  },
  "error": {},
  "details": {
    "nestjs-docs": {
      "status": "up"
    }
  }
}
```

该响应对象的接口可以通过 `@nestjs/terminus` 包中的 `HealthCheckResult` 接口访问。

|           |                                                                                                                                                                                             |                                      |
|-----------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------|
| `status`  | 如果任意健康指标失败，状态将为 `'error'`。如果 NestJS 应用正在关闭但仍接受 HTTP 请求，则健康检查将具有 `'shutting_down'` 状态。                                                            | `'error' \| 'ok' \| 'shutting_down'` |
| `info`    | 包含每个状态为 `'up'`（即“健康”）的健康指标的信息对象。                                                                             | `object`                             |
| `error`   | 包含每个状态为 `'down'`（即“不健康”）的健康指标的信息对象。                                                                          | `object`                             |
| `details` | 包含每个健康指标的详细信息的对象                                                                                                    | `object`                             |

##### 检查特定的 HTTP 响应码

在某些情况下，你可能希望检查特定的响应码并验证响应。例如，假设 `https://my-external-service.com` 返回了响应码 `204`。使用 `HttpHealthIndicator.responseCheck` 可以专门检查该响应码，并将其他响应码视为不健康。

如果返回的响应码不是 `204`，则以下示例将被视为不健康。第三个参数需要你提供一个函数（同步或异步），该函数返回一个布尔值来判断响应是否健康（`true`）或不健康（`false`）。

```typescript
@@filename(health.controller)
// 在 `HealthController` 类中

@Get()
@HealthCheck()
check() {
  return this.health.check([
    () =>
      this.http.responseCheck(
        'my-external-service',
        'https://my-external-service.com',
        (res) => res.status === 204,
      ),
  ]);
}
```

#### TypeORM 健康指标

Terminus 提供了将数据库检查添加到健康检查中的能力。要使用此健康指标，你需要先查看 [数据库章节](/techniques/sql)，并确保你的应用程序已建立数据库连接。

> info **提示** 在底层，`TypeOrmHealthIndicator` 只是执行了一个 `SELECT 1` 的 SQL 命令，通常用于验证数据库是否仍然存活。如果你使用的是 Oracle 数据库，则它会使用 `SELECT 1 FROM DUAL`。

```typescript
@@filename(health.controller)
@Controller('health')
export class HealthController {
  constructor(
    private health: HealthCheckService,
    private db: TypeOrmHealthIndicator,
  ) {}

  @Get()
  @HealthCheck()
  check() {
    return this.health.check([
      () => this.db.pingCheck('database'),
    ]);
  }
}
@@switch
@Controller('health')
@Dependencies(HealthCheckService, TypeOrmHealthIndicator)
export class HealthController {
  constructor(
    private health,
    private db,
  ) { }

  @Get()
  @HealthCheck()
  healthCheck() {
    return this.health.check([
      () => this.db.pingCheck('database'),
    ])
  }
}
```

如果你的数据库可以访问，现在当你向 `http://localhost:3000/health` 发送 `GET` 请求时，应该会看到以下 JSON 结果：

```json
{
  "status": "ok",
  "info": {
    "database": {
      "status": "up"
    }
  },
  "error": {},
  "details": {
    "database": {
      "status": "up"
    }
  }
}
```

如果你的应用使用 [多个数据库](techniques/database#multiple-databases)，你需要将每个连接注入到 `HealthController` 中。然后，你可以简单地将连接引用传递给 `TypeOrmHealthIndicator`。

```typescript
@@filename(health.controller)
@Controller('health')
export class HealthController {
  constructor(
    private health: HealthCheckService,
    private db: TypeOrmHealthIndicator,
    @InjectConnection('albumsConnection')
    private albumsConnection: Connection,
    @InjectConnection()
    private defaultConnection: Connection,
  ) {}

  @Get()
  @HealthCheck()
  check() {
    return this.health.check([
      () => this.db.pingCheck('albums-database', { connection: this.albumsConnection }),
      () => this.db.pingCheck('database', { connection: this.defaultConnection }),
    ]);
  }
}
```

#### 磁盘健康指标

使用 `DiskHealthIndicator` 我们可以检查存储使用情况。要开始使用，确保将 `DiskHealthIndicator` 注入到你的 `HealthController` 中。以下示例检查路径 `/` 的存储使用情况（在 Windows 上可以使用 `C:\\`）。如果该路径的使用空间超过总空间的 50%，健康检查将返回不健康的结果。

```typescript
@@filename(health.controller)
@Controller('health')
export class HealthController {
  constructor(
    private readonly health: HealthCheckService,
    private readonly disk: DiskHealthIndicator,
  ) {}

  @Get()
  @HealthCheck()
  check() {
    return this.health.check([
      () => this.disk.checkStorage('storage', { path: '/', thresholdPercent: 0.5 }),
    ]);
  }
}
@@switch
@Controller('health')
@Dependencies(HealthCheckService, DiskHealthIndicator)
export class HealthController {
  constructor(health, disk) {}

  @Get()
  @HealthCheck()
  healthCheck() {
    return this.health.check([
      () => this.disk.checkStorage('storage', { path: '/', thresholdPercent: 0.5 }),
    ])
  }
}
```

使用 `DiskHealthIndicator.checkStorage` 方法，你还可以检查固定大小的存储空间。以下示例中，如果路径 `/my-app/` 超过 250GB，则返回不健康的结果。

```typescript
@@filename(health.controller)
// 在 `HealthController` 类中

@Get()
@HealthCheck()
check() {
  return this.health.check([
    () => this.disk.checkStorage('storage', {  path: '/', threshold: 250 * 1024 * 1024 * 1024, })
  ]);
}
```

#### 内存健康指标

为了确保你的进程不超过一定的内存限制，可以使用 `MemoryHealthIndicator`。以下示例可用于检查进程的堆内存。

> info **提示** 堆是动态分配内存的区域（即通过 malloc 分配的内存）。从堆中分配的内存将保持分配状态，直到发生以下情况之一：
> - 内存被 _free_ 释放
> - 程序终止

```typescript
@@filename(health.controller)
@Controller('health')
export class HealthController {
  constructor(
    private health: HealthCheckService,
    private memory: MemoryHealthIndicator,
  ) {}

  @Get()
  @HealthCheck()
  check() {
    return this.health.check([
      () => this.memory.checkHeap('memory_heap', 150 * 1024 * 1024),
    ]);
  }
}
@@switch
@Controller('health')
@Dependencies(HealthCheckService, MemoryHealthIndicator)
export class HealthController {
  constructor(health, memory) {}

  @Get()
  @HealthCheck()
  healthCheck() {
    return this.health.check([
      () => this.memory.checkHeap('memory_heap', 150 * 1024 * 1024),
    ])
  }
}
```

你也可以使用 `MemoryHealthIndicator.checkRSS` 来验证进程的内存 RSS。此示例中，如果进程分配的内存超过 150MB，则返回不健康的结果。

> info **提示** RSS（Resident Set Size）用于显示分配给该进程并在 RAM 中的内存大小。
> 它不包括被交换出去的内存。它包括共享库中的内存，只要这些库的页面实际在内存中。它还包括所有堆栈和堆内存。

```typescript
@@filename(health.controller)
// 在 `HealthController` 类中

@Get()
@HealthCheck()
check() {
  return this.health.check([
    () => this.memory.checkRSS('memory_rss', 150 * 1024 * 1024),
  ]);
}
```

#### 自定义健康指标

在某些情况下，`@nestjs/terminus` 提供的预定义健康指标可能无法覆盖你所有的健康检查需求。在这种情况下，你可以根据自己的需求创建自定义健康指标。

首先，我们创建一个服务来表示我们的自定义指标。为了了解指标的结构，我们将创建一个示例 `DogHealthIndicator`。该服务在每个 `Dog` 对象类型为 `'goodboy'` 时应处于 `'up'` 状态。如果不满足该条件，则应抛出错误。

```typescript
@@filename(dog.health)
import { Injectable } from '@nestjs/common';
import { HealthIndicatorService } from '@nestjs/terminus';

export interface Dog {
  name: string;
  type: string;
}

@Injectable()
export class DogHealthIndicator {
  constructor(
    private readonly healthIndicatorService: HealthIndicatorService
  ) {}

  private dogs: Dog[] = [
    { name: 'Fido', type: 'goodboy' },
    { name: 'Rex', type: 'badboy' },
  ];

  async isHealthy(key: string){
    const indicator = this.healthIndicatorService.check(key);
    const badboys = this.dogs.filter(dog => dog.type === 'badboy');
    const isHealthy = badboys.length === 0;

    if (!isHealthy) {
      return indicator.down({ badboys: badboys.length });
    }

    return indicator.up();
  }
}
@@switch
import { Injectable } from '@nestjs/common';
import { HealthIndicatorService } from '@nestjs/terminus';

@Injectable()
@Dependencies(HealthIndicatorService)
export class DogHealthIndicator {
  constructor(healthIndicatorService) {
    this.healthIndicatorService = healthIndicatorService;
  }

  private dogs = [
    { name: 'Fido', type: 'goodboy' },
    { name: 'Rex', type: 'badboy' },
  ];

  async isHealthy(key){
    const indicator = this.healthIndicatorService.check(key);
    const badboys = this.dogs.filter(dog => dog.type === 'badboy');
    const isHealthy = badboys.length === 0;

    if (!isHealthy) {
      return indicator.down({ badboys: badboys.length });
    }

    return indicator.up();
  }
}
```

接下来，我们需要将健康指标注册为提供者。

```typescript
@@filename(health.module)
import { Module } from '@nestjs/common';
import { TerminusModule } from '@nestjs/terminus';
import { DogHealthIndicator } from './dog.health';

@Module({
  controllers: [HealthController],
  imports: [TerminusModule],
  providers: [DogHealthIndicator]
})
export class HealthModule { }
```

> info **提示** 在实际应用中，`DogHealthIndicator` 应该在一个单独的模块（如 `DogModule`）中提供，然后由 `HealthModule` 导入。

最后一步是将现在可用的健康指标添加到所需的健康检查端点中。为此，我们回到 `HealthController` 并将其添加到 `check` 方法中。

```typescript
@@filename(health.controller)
import { HealthCheckService, HealthCheck } from '@nestjs/terminus';
import { Injectable, Dependencies, Get } from '@nestjs/common';
import { DogHealthIndicator } from './dog.health';

@Injectable()
export class HealthController {
  constructor(
    private health: HealthCheckService,
    private dogHealthIndicator: DogHealthIndicator
  ) {}

  @Get()
  @HealthCheck()
  healthCheck() {
    return this.health.check([
      () => this.dogHealthIndicator.isHealthy('dog'),
    ])
  }
}
@@switch
import { HealthCheckService, HealthCheck } from '@nestjs/terminus';
import { Injectable, Get } from '@nestjs/common';
import { DogHealthIndicator } from './dog.health';

@Injectable()
@Dependencies(HealthCheckService, DogHealthIndicator)
export class HealthController {
  constructor(
    health,
    dogHealthIndicator
  ) {
    this.health = health;
    this.dogHealthIndicator = dogHealthIndicator;
  }

  @Get()
  @HealthCheck()
  healthCheck() {
    return this.health.check([
      () => this.dogHealthIndicator.isHealthy('dog'),
    ])
  }
}
```

#### 日志记录

Terminus 仅记录错误消息，例如健康检查失败时。通过 `TerminusModule.forRoot()` 方法，你可以更精细地控制错误日志的记录方式，甚至完全接管日志记录本身。

在本节中，我们将向你展示如何创建一个自定义日志记录器 `TerminusLogger`。该日志记录器继承自内置日志记录器。因此，你可以选择性地覆盖日志记录器的某部分。

> info **提示** 如果你想了解更多关于 NestJS 中自定义日志记录器的内容，[点击此处](/techniques/logger#injecting-a-custom-logger)。

```typescript
@@filename(terminus-logger.service)
import { Injectable, Scope, ConsoleLogger } from '@nestjs/common';

@Injectable({ scope: Scope.TRANSIENT })
export class TerminusLogger extends ConsoleLogger {
  error(message: any, stack?: string, context?: string): void;
  error(message: any, ...optionalParams: any[]): void;
  error(
    message: unknown,
    stack?: unknown,
    context?: unknown,
    ...rest: unknown[]
  ): void {
    // 在此处覆盖错误消息的记录方式
  }
}
```

一旦你创建了自己的自定义日志记录器，只需将其传递给 `TerminusModule.forRoot()` 即可：

```typescript
@@filename(health.module)
@Module({
imports: [
  TerminusModule.forRoot({
    logger: TerminusLogger,
  }),
],
})
export class HealthModule {}
```

要完全禁止来自 Terminus 的任何日志消息（包括错误消息），请按如下方式配置 Terminus：

```typescript
@@filename(health.module)
@Module({
imports: [
  TerminusModule.forRoot({
    logger: false,
  }),
],
})
export class HealthModule {}
```

Terminus 允许你配置健康检查错误在日志中的显示方式。

| 错误日志样式          | 描述                                                                                                                        | 示例                                                              |
|:------------------|:---------------------------------------------------------------------------------------------------------------------------|:-----------------------------------------------------------------|
| `json` （默认） | 以 JSON 对象形式打印健康检查结果摘要（仅在出错时）                                                     | <figure><img src="/assets/Terminus_Error_Log_Json.png" /></figure>   |
| `pretty`          | 以格式化框的形式打印健康检查结果摘要，并高亮显示成功/错误结果 | <figure><img src="/assets/Terminus_Error_Log_Pretty.png" /></figure> |

你可以通过 `errorLogStyle` 配置选项更改日志样式，如下所示：

```typescript
@@filename(health.module)
@Module({
  imports: [
    TerminusModule.forRoot({
      errorLogStyle: 'pretty',
    }),
  ]
})
export class HealthModule {}
```

#### 优雅关闭超时

如果你的应用程序需要延迟其关闭过程，Terminus 可以帮你处理。当与 Kubernetes 等编排器一起工作时，此设置尤其有用。通过将延迟设置为略长于就绪检查间隔的时间，你可以在关闭容器时实现零停机时间。

```typescript
@@filename(health.module)
@Module({
  imports: [
    TerminusModule.forRoot({
      gracefulShutdownTimeoutMs: 1000,
    }),
  ]
})
export class HealthModule {}
```

#### 更多示例

更多可运行的示例可以在 [这里](https://github.com/nestjs/terminus/tree/master/sample) 找到。