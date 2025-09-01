### 任务调度

任务调度允许您在指定的日期/时间、定期的时间间隔或经过指定间隔后一次执行任意代码（方法/函数）。在 Linux 系统中，通常由 [cron](https://zh.wikipedia.org/wiki/Cron) 这类操作系统级的工具来处理。对于 Node.js 应用程序，有多个包可以模拟类似 cron 的功能。Nest 提供了 `@nestjs/schedule` 包，它与流行的 Node.js [cron](https://github.com/kelektiv/node-cron) 包集成。在本章中，我们将介绍这个包的使用。

#### 安装

要开始使用它，我们首先安装所需的依赖项。

```bash
$ npm install --save @nestjs/schedule
```

要激活任务调度，请将 `ScheduleModule` 导入根模块 `AppModule`，并调用静态方法 `forRoot()`，如下所示：

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { ScheduleModule } from '@nestjs/schedule';

@Module({
  imports: [
    ScheduleModule.forRoot()
  ],
})
export class AppModule {}
```

调用 `.forRoot()` 方法会初始化调度器，并注册应用中存在的任何声明式 <a href="techniques/task-scheduling#declarative-cron-jobs">cron 任务</a>、<a href="techniques/task-scheduling#declarative-timeouts">超时任务</a> 和 <a href="techniques/task-scheduling#declarative-intervals">定时任务</a>。注册会在 `onApplicationBootstrap` 生命周期钩子中发生，确保所有模块都已加载并声明了任何计划任务。

#### 声明式 cron 任务

一个 cron 任务会安排一个任意函数（方法调用）自动运行。Cron 任务可以：

- 在指定的日期/时间执行一次。
- 按照周期性执行；周期性任务可以在指定的间隔内的特定时刻运行（例如，每小时一次，每周一次，每5分钟一次）

使用 `@Cron()` 装饰器声明一个 cron 任务，并在该装饰器后面定义包含要执行代码的方法，如下所示：

```typescript
import { Injectable, Logger } from '@nestjs/common';
import { Cron } from '@nestjs/schedule';

@Injectable()
export class TasksService {
  private readonly logger = new Logger(TasksService.name);

  @Cron('45 * * * * *')
  handleCron() {
    this.logger.debug('当当前秒数为 45 时调用');
  }
}
```

在此示例中，当当前秒数为 `45` 时，将调用 `handleCron()` 方法。换句话说，该方法将在每分钟的第 45 秒运行一次。

`@Cron()` 装饰器支持以下标准 [cron 表达式](http://crontab.org/)：

- 星号（例如 `*`）
- 范围（例如 `1-3,5`）
- 步长（例如 `*/2`）

在上面的示例中，我们向装饰器传递了 `45 * * * * *`。下面的说明展示了 cron 表达式字符串中每个位置的含义：

<pre class="language-javascript"><code class="language-javascript">
* * * * * *
| | | | | |
| | | | | 星期几
| | | | 月份
| | | 月份中的日期
| | 小时
| 分钟
秒（可选）
</code></pre>

一些 cron 表达式的示例：

<table>
  <tbody>
    <tr>
      <td><code>* * * * * *</code></td>
      <td>每秒</td>
    </tr>
    <tr>
      <td><code>45 * * * * *</code></td>
      <td>每分钟，在第 45 秒</td>
    </tr>
    <tr>
      <td><code>0 10 * * * *</code></td>
      <td>每小时，在第 10 分钟开始时</td>
    </tr>
    <tr>
      <td><code>0 */30 9-17 * * *</code></td>
      <td>上午9点到下午5点之间每30分钟一次</td>
    </tr>
    <tr>
      <td><code>0 30 11 * * 1-5</code></td>
      <td>周一至周五 11:30</td>
    </tr>
  </tbody>
</table>

`@nestjs/schedule` 包提供了一个包含常用 cron 表达式的便捷枚举。您可以按如下方式使用此枚举：

```typescript
import { Injectable, Logger } from '@nestjs/common';
import { Cron, CronExpression } from '@nestjs/schedule';

@Injectable()
export class TasksService {
  private readonly logger = new Logger(TasksService.name);

  @Cron(CronExpression.EVERY_30_SECONDS)
  handleCron() {
    this.logger.debug('每30秒调用一次');
  }
}
```

在此示例中，`handleCron()` 方法将每 `30` 秒调用一次。如果发生异常，它将被记录到控制台，因为每个用 `@Cron()` 注解的方法都会自动包裹在 `try-catch` 块中。

另外，您可以向 `@Cron()` 装饰器传递一个 JavaScript `Date` 对象。这样做会使任务在指定日期执行一次。

> info **提示** 使用 JavaScript 的日期运算来安排相对于当前日期的任务。例如，`@Cron(new Date(Date.now() + 10 * 1000))` 表示在应用启动后10秒执行一次任务。

此外，您还可以将附加选项作为第二个参数传递给 `@Cron()` 装饰器。

<table>
  <tbody>
    <tr>
      <td><code>name</code></td>
      <td>
        在声明后用于访问和控制 cron 任务。
      </td>
    </tr>
    <tr>
      <td><code>timeZone</code></td>
      <td>
        指定任务执行的时区。这会根据您的时区修改实际的执行时间。如果时区无效，将抛出错误。您可以在 <a href="http://momentjs.com/timezone/">Moment Timezone</a> 网站上查看所有可用的时区。
      </td>
    </tr>
    <tr>
      <td><code>utcOffset</code></td>
      <td>
        这允许您通过指定时区偏移量而不是使用 <code>timeZone</code> 参数。
      </td>
    </tr>
    <tr>
      <td><code>waitForCompletion</code></td>
      <td>
        如果为 <code>true</code>，则在当前的 onTick 回调完成之前，不会运行 cron 任务的其他实例。当当前 cron 任务正在运行时发生的任何新的计划执行都将被完全跳过。
      </td>
    </tr>
    <tr>
      <td><code>disabled</code></td>
      <td>
       表示任务是否将被执行。
      </td>
    </tr>
  </tbody>
</table>

```typescript
import { Injectable } from '@nestjs/common';
import { Cron, CronExpression } from '@nestjs/schedule';

@Injectable()
export class NotificationService {
  @Cron('* * 0 * * *', {
    name: 'notifications',
    timeZone: 'Europe/Paris',
  })
  triggerNotifications() {}
}
```

您可以在声明后通过 <a href="/techniques/task-scheduling#dynamic-schedule-module-api">动态 API</a> 访问和控制 cron 任务，或者动态创建一个 cron 任务（其 cron 表达式在运行时定义）。要通过 API 访问声明式 cron 任务，您必须通过将装饰器的第二个参数作为可选的 options 对象传递 `name` 属性来将任务与一个名称关联。

#### 声明式定时任务

要声明一个方法应在（周期性）指定间隔运行，请在方法定义前加上 `@Interval()` 装饰器。将间隔值（以毫秒为单位的数字）传递给装饰器，如下所示：

```typescript
@Interval(10000)
handleInterval() {
  this.logger.debug('每10秒调用一次');
}
```

> info **提示** 此机制在底层使用 JavaScript 的 `setInterval()` 函数。您也可以使用 cron 任务来安排周期性任务。

如果您想通过 <a href="/techniques/task-scheduling#dynamic-schedule-module-api">动态 API</a> 从声明类外部控制声明式定时任务，请使用以下构造将定时任务与名称关联：

```typescript
@Interval('notifications', 2500)
handleInterval() {}
```

如果发生异常，它将被记录到控制台，因为每个用 `@Interval()` 注解的方法都会自动包裹在 `try-catch` 块中。

<a href="techniques/task-scheduling#dynamic-intervals">动态 API</a> 还支持 **创建** 动态定时任务，其中定时任务的属性在运行时定义，并支持 **列出和删除** 它们。

<app-banner-enterprise></app-banner-enterprise>

#### 声明式超时任务

要声明一个方法应在（一次）指定的超时后运行，请在方法定义前加上 `@Timeout()` 装饰器。将相对于应用启动的时间偏移量（以毫秒为单位）传递给装饰器，如下所示：

```typescript
@Timeout(5000)
handleTimeout() {
  this.logger.debug('5秒后调用一次');
}
```

> info **提示** 此机制在底层使用 JavaScript 的 `setTimeout()` 函数。

如果发生异常，它将被记录到控制台，因为每个用 `@Timeout()` 注解的方法都会自动包裹在 `try-catch` 块中。

如果您想通过 <a href="/techniques/task-scheduling#dynamic-schedule-module-api">动态 API</a> 从声明类外部控制声明式超时任务，请使用以下构造将超时任务与名称关联：

```typescript
@Timeout('notifications', 2500)
handleTimeout() {}
```

<a href="techniques/task-scheduling#dynamic-timeouts">动态 API</a> 还支持 **创建** 动态超时任务，其中超时任务的属性在运行时定义，并支持 **列出和删除** 它们。

#### 动态调度模块 API

`@nestjs/schedule` 模块提供了一个动态 API，可以管理声明式的 <a href="techniques/task-scheduling#declarative-cron-jobs">cron 任务</a>、<a href="techniques/task-scheduling#declarative-timeouts">超时任务</a> 和 <a href="techniques/task-scheduling#declarative-intervals">定时任务</a>。该 API 还支持创建和管理 **动态** 的 cron 任务、超时任务和定时任务，其中属性在运行时定义。

#### 动态 cron 任务

可以通过 `SchedulerRegistry` API 从代码中的任何位置通过名称获取对 `CronJob` 实例的引用。首先，使用标准的构造函数注入注入 `SchedulerRegistry`：

```typescript
constructor(private schedulerRegistry: SchedulerRegistry) {}
```

> info **提示** 从 `@nestjs/schedule` 包中导入 `SchedulerRegistry`。

然后在类中使用它，如下所示。假设一个 cron 任务是通过以下声明创建的：

```typescript
@Cron('* * 8 * * *', {
  name: 'notifications',
})
triggerNotifications() {}
```

使用以下方式访问此任务：

```typescript
const job = this.schedulerRegistry.getCronJob('notifications');

job.stop();
console.log(job.lastDate());
```

`getCronJob()` 方法返回命名的 cron 任务。返回的 `CronJob` 对象具有以下方法：

- `stop()` - 停止一个计划运行的任务。
- `start()` - 重启一个已停止的任务。
- `setTime(time: CronTime)` - 停止任务，设置新时间，然后重新启动它。
- `lastDate()` - 返回一个 `DateTime` 表示，表示任务最后一次执行的日期。
- `nextDate()` - 返回一个 `DateTime` 表示，表示任务下一次执行的日期。
- `nextDates(count: number)` - 提供一个包含 `count` 个 `DateTime` 表示的数组，表示下一次触发任务执行的日期集合。`count` 默认为 0，返回空数组。

> info **提示** 在 `DateTime` 对象上使用 `toJSDate()` 方法将其渲染为等效于该 `DateTime` 的 JavaScript `Date`。

使用 `SchedulerRegistry#addCronJob` 方法动态 **创建** 新的 cron 任务，如下所示：

```typescript
addCronJob(name: string, seconds: string) {
  const job = new CronJob(`${seconds} * * * * *`, () => {
    this.logger.warn(`任务 ${name} 在 ${seconds} 秒时执行了！`);
  });

  this.schedulerRegistry.addCronJob(name, job);
  job.start();

  this.logger.warn(
    `任务 ${name} 已添加，每分钟在 ${seconds} 秒时运行！`,
  );
}
```

在此代码中，我们使用 `cron` 包中的 `CronJob` 对象创建 cron 任务。`CronJob` 构造函数接受一个 cron 表达式（与 `@Cron()` <a href="techniques/task-scheduling#declarative-cron-jobs">装饰器</a> 相同）作为第一个参数，以及一个在 cron 定时器触发时执行的回调作为第二个参数。`SchedulerRegistry#addCronJob` 方法接受两个参数：一个为 `CronJob` 命名，另一个是 `CronJob` 对象本身。

> warning **警告** 在访问之前请记得注入 `SchedulerRegistry`。从 `cron` 包中导入 `CronJob`。

使用 `SchedulerRegistry#deleteCronJob` 方法 **删除** 命名的 cron 任务，如下所示：

```typescript
deleteCron(name: string) {
  this.schedulerRegistry.deleteCronJob(name);
  this.logger.warn(`任务 ${name} 已删除！`);
}
```

使用 `SchedulerRegistry#getCronJobs` 方法 **列出** 所有 cron 任务，如下所示：

```typescript
getCrons() {
  const jobs = this.schedulerRegistry.getCronJobs();
  jobs.forEach((value, key, map) => {
    let next;
    try {
      next = value.nextDate().toJSDate();
    } catch (e) {
      next = '错误：下一次触发时间在过去！';
    }
    this.logger.log(`任务: ${key} -> 下一次: ${next}`);
  });
}
```

`getCronJobs()` 方法返回一个 `map`。在此代码中，我们遍历 `map` 并尝试访问每个 `CronJob` 的 `nextDate()` 方法。在 `CronJob` API 中，如果任务已经执行且没有未来的触发日期，则会抛出异常。

#### 动态定时任务

使用 `SchedulerRegistry#getInterval` 方法获取对定时任务的引用。如上所述，使用标准的构造函数注入注入 `SchedulerRegistry`：

```typescript
constructor(private schedulerRegistry: SchedulerRegistry) {}
```

并如下使用：

```typescript
const interval = this.schedulerRegistry.getInterval('notifications');
clearInterval(interval);
```

使用 `SchedulerRegistry#addInterval` 方法动态 **创建** 新的定时任务，如下所示：

```typescript
addInterval(name: string, milliseconds: number) {
  const callback = () => {
    this.logger.warn(`定时任务 ${name} 在 ${milliseconds} 毫秒时执行了！`);
  };

  const interval = setInterval(callback, milliseconds);
  this.schedulerRegistry.addInterval(name, interval);
}
```

在此代码中，我们创建了一个标准的 JavaScript 定时任务，然后将其传递给 `SchedulerRegistry#addInterval` 方法。该方法接受两个参数：一个为定时任务命名，另一个是定时任务本身。

使用 `SchedulerRegistry#deleteInterval` 方法 **删除** 命名的定时任务，如下所示：

```typescript
deleteInterval(name: string) {
  this.schedulerRegistry.deleteInterval(name);
  this.logger.warn(`定时任务 ${name} 已删除！`);
}
```

使用 `SchedulerRegistry#getIntervals` 方法 **列出** 所有定时任务，如下所示：

```typescript
getIntervals() {
  const intervals = this.schedulerRegistry.getIntervals();
  intervals.forEach(key => this.logger.log(`定时任务: ${key}`));
}
```

#### 动态超时任务

使用 `SchedulerRegistry#getTimeout` 方法获取对超时任务的引用。如上所述，使用标准的构造函数注入注入 `SchedulerRegistry`：

```typescript
constructor(private readonly schedulerRegistry: SchedulerRegistry) {}
```

并如下使用：

```typescript
const timeout = this.schedulerRegistry.getTimeout('notifications');
clearTimeout(timeout);
```

使用 `SchedulerRegistry#addTimeout` 方法动态 **创建** 新的超时任务，如下所示：

```typescript
addTimeout(name: string, milliseconds: number) {
  const callback = () => {
    this.logger.warn(`超时任务 ${name} 在 ${milliseconds} 毫秒后执行了！`);
  };

  const timeout = setTimeout(callback, milliseconds);
  this.schedulerRegistry.addTimeout(name, timeout);
}
```

在此代码中，我们创建了一个标准的 JavaScript 超时任务，然后将其传递给 `SchedulerRegistry#addTimeout` 方法。该方法接受两个参数：一个为超时任务命名，另一个是超时任务本身。

使用 `SchedulerRegistry#deleteTimeout` 方法 **删除** 命名的超时任务，如下所示：

```typescript
deleteTimeout(name: string) {
  this.schedulerRegistry.deleteTimeout(name);
  this.logger.warn(`超时任务 ${name} 已删除！`);
}
```

使用 `SchedulerRegistry#getTimeouts` 方法 **列出** 所有超时任务，如下所示：

```typescript
getTimeouts() {
  const timeouts = this.schedulerRegistry.getTimeouts();
  timeouts.forEach(key => this.logger.log(`超时任务: ${key}`));
}
```

#### 示例

一个可用的示例请参见 [这里](https://github.com/nestjs/nest/tree/master/sample/27-scheduling)。