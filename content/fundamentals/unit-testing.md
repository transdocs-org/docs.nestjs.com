### 测试

自动化测试被认为是任何严肃的软件开发工作中不可或缺的一部分。自动化使得在开发过程中快速简便地重复执行单个测试或测试套件变得容易。这有助于确保发布的软件符合质量和性能目标。自动化有助于增加测试覆盖率，并为开发人员提供更快的反馈循环。自动化不仅提高了单个开发人员的生产力，还确保了在开发生命周期的关键节点（如源代码控制签入、功能集成和版本发布）运行测试。

这类测试通常涵盖多种类型，包括单元测试、端到端（e2e）测试、集成测试等。虽然自动化测试的好处是毋庸置疑的，但设置这些测试可能会很繁琐。Nest 致力于推广最佳开发实践，包括有效的测试，因此它包含以下功能来帮助开发人员和团队构建和自动化测试。Nest：

- 自动为组件生成默认单元测试，为应用程序生成默认 e2e 测试
- 提供默认的测试工具（例如一个构建隔离模块/应用程序加载器的测试运行器）
- 默认集成 [Jest](https://github.com/facebook/jest) 和 [Supertest](https://github.com/visionmedia/supertest)，同时保持对测试工具的无关性
- 在测试环境中提供 Nest 的依赖注入系统，以便轻松模拟组件

如前所述，你可以使用任何你喜爱的 **测试框架**，因为 Nest 不强制使用任何特定的工具。只需替换所需的元素（例如测试运行器），你仍然可以享受 Nest 提供的现成测试设施带来的好处。

#### 安装

要开始使用，请首先安装所需的包：

```bash
$ npm i --save-dev @nestjs/testing
```

#### 单元测试

在下面的示例中，我们测试了两个类：`CatsController` 和 `CatsService`。如前所述，[Jest](https://github.com/facebook/jest) 被作为默认的测试框架提供。它既作为测试运行器，也提供断言函数和测试双工具，帮助进行模拟、间谍等操作。在以下基本测试中，我们手动实例化这些类，并确保控制器和服务满足其 API 合约。

```typescript
@@filename(cats.controller.spec)
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

describe('CatsController', () => {
  let catsController: CatsController;
  let catsService: CatsService;

  beforeEach(() => {
    catsService = new CatsService();
    catsController = new CatsController(catsService);
  });

  describe('findAll', () => {
    it('should return an array of cats', async () => {
      const result = ['test'];
      jest.spyOn(catsService, 'findAll').mockImplementation(() => result);

      expect(await catsController.findAll()).toBe(result);
    });
  });
});
@@switch
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

describe('CatsController', () => {
  let catsController;
  let catsService;

  beforeEach(() => {
    catsService = new CatsService();
    catsController = new CatsController(catsService);
  });

  describe('findAll', () => {
    it('should return an array of cats', async () => {
      const result = ['test'];
      jest.spyOn(catsService, 'findAll').mockImplementation(() => result);

      expect(await catsController.findAll()).toBe(result);
    });
  });
});
```

> info **提示** 将测试文件放在被测试类附近。测试文件应以 `.spec` 或 `.test` 作为后缀。

由于上面的示例很简单，我们实际上没有测试任何 Nest 特有的内容。确实，我们甚至没有使用依赖注入（请注意，我们向 `catsController` 传递了一个 `CatsService` 的实例）。这种测试形式——我们手动实例化被测试类——通常称为 **隔离测试**，因为它与框架无关。让我们介绍一些更高级的功能，帮助你测试更广泛使用 Nest 特性的应用程序。

#### 测试工具

`@nestjs/testing` 包提供了一组工具，可以实现更稳健的测试过程。让我们使用内置的 `Test` 类重写前面的示例：

```typescript
@@filename(cats.controller.spec)
import { Test } from '@nestjs/testing';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

describe('CatsController', () => {
  let catsController: CatsController;
  let catsService: CatsService;

  beforeEach(async () => {
    const moduleRef = await Test.createTestingModule({
        controllers: [CatsController],
        providers: [CatsService],
      }).compile();

    catsService = moduleRef.get(CatsService);
    catsController = moduleRef.get(CatsController);
  });

  describe('findAll', () => {
    it('should return an array of cats', async () => {
      const result = ['test'];
      jest.spyOn(catsService, 'findAll').mockImplementation(() => result);

      expect(await catsController.findAll()).toBe(result);
    });
  });
});
@@switch
import { Test } from '@nestjs/testing';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

describe('CatsController', () => {
  let catsController;
  let catsService;

  beforeEach(async () => {
    const moduleRef = await Test.createTestingModule({
        controllers: [CatsController],
        providers: [CatsService],
      }).compile();

    catsService = moduleRef.get(CatsService);
    catsController = moduleRef.get(CatsController);
  });

  describe('findAll', () => {
    it('should return an array of cats', async () => {
      const result = ['test'];
      jest.spyOn(catsService, 'findAll').mockImplementation(() => result);

      expect(await catsController.findAll()).toBe(result);
    });
  });
});
```

`Test` 类对于提供一个应用程序执行上下文非常有用，它基本上模拟了完整的 Nest 运行时，但提供了让你轻松管理类实例的钩子，包括模拟和覆盖。`Test` 类有一个 `createTestingModule()` 方法，该方法以一个模块元数据对象作为参数（即你传递给 `@Module()` 装饰器的同一个对象）。此方法返回一个 `TestingModule` 实例，该实例又提供了几个方法。对于单元测试，重要的是 `compile()` 方法。此方法引导一个模块及其依赖项（类似于在传统的 `main.ts` 文件中使用 `NestFactory.create()` 引导应用程序的方式），并返回一个准备好进行测试的模块。

> info **提示** `compile()` 方法是 **异步的**，因此必须使用 `await`。一旦模块被编译完成，你可以使用 `get()` 方法检索它声明的任何 **静态** 实例（控制器和提供者）。

`TestingModule` 继承自 [模块引用](/fundamentals/module-ref) 类，因此它能够动态解析作用域提供者（瞬态或请求作用域）。使用 `resolve()` 方法来实现这一点（`get()` 方法只能检索静态实例）。

```typescript
const moduleRef = await Test.createTestingModule({
  controllers: [CatsController],
  providers: [CatsService],
}).compile();

catsService = await moduleRef.resolve(CatsService);
```

> warning **警告** `resolve()` 方法返回一个提供者的唯一实例，该实例来自其自身的 **DI 容器子树**。每个子树都有一个唯一的上下文标识符。因此，如果你多次调用此方法并比较实例引用，你会发现它们不相等。

> info **提示** 想要了解更多关于模块引用的功能，请点击 [这里](/fundamentals/module-ref)。

你可以使用 [自定义提供者](/fundamentals/custom-providers) 覆盖生产环境中使用的任何提供者，以用于测试目的。例如，你可以模拟一个数据库服务，而不是连接到真实的数据库。我们将在下一节中介绍覆盖，但它们也适用于单元测试。

<app-banner-courses></app-banner-courses>

#### 自动模拟

Nest 还允许你为所有缺失的依赖项定义一个模拟工厂。这在你有一个类具有大量依赖项，并且手动模拟所有依赖项需要花费大量时间和设置时非常有用。要使用此功能，你需要将 `createTestingModule()` 与 `useMocker()` 方法链式调用，并传入一个依赖项模拟工厂。此工厂可以接收一个可选的 token，它是一个实例 token，任何对 Nest 提供者有效的 token 都可以，然后返回一个模拟实现。以下是一个使用 [`jest-mock`](https://www.npmjs.com/package/jest-mock) 创建一个通用模拟器，以及使用 `jest.fn()` 创建 `CatsService` 的特定模拟的示例。

```typescript
// ...
import { ModuleMocker, MockMetadata } from 'jest-mock';

const moduleMocker = new ModuleMocker(global);

describe('CatsController', () => {
  let controller: CatsController;

  beforeEach(async () => {
    const moduleRef = await Test.createTestingModule({
      controllers: [CatsController],
    })
      .useMocker((token) => {
        const results = ['test1', 'test2'];
        if (token === CatsService) {
          return { findAll: jest.fn().mockResolvedValue(results) };
        }
        if (typeof token === 'function') {
          const mockMetadata = moduleMocker.getMetadata(
            token,
          ) as MockMetadata<any, any>;
          const Mock = moduleMocker.generateFromMetadata(
            mockMetadata,
          ) as ObjectConstructor;
          return new Mock();
        }
      })
      .compile();

    controller = moduleRef.get(CatsController);
  });
});
```

你也可以像获取自定义提供者一样从测试容器中检索这些模拟对象，例如 `moduleRef.get(CatsService)`。

> info **提示** 通用的模拟工厂，如 [`@golevelup/ts-jest`](https://github.com/golevelup/nestjs/tree/master/packages/testing) 中的 `createMock`，也可以直接传入。

> info **提示** `REQUEST` 和 `INQUIRER` 提供者不能被自动模拟，因为它们已经在上下文中预定义了。不过，你可以使用自定义提供者语法或 `.overrideProvider` 方法来 **覆盖** 它们。

#### 端到端测试

与专注于单个模块和类的单元测试不同，端到端（e2e）测试关注类和模块在更高级别的交互——更接近最终用户与生产系统交互的方式。随着应用程序的增长，手动测试每个 API 端点的端到端行为变得困难。自动化端到端测试有助于我们确保系统的整体行为正确并满足项目需求。要执行 e2e 测试，我们使用与前面 **单元测试** 中类似的配置。此外，Nest 还使我们能够轻松使用 [Supertest](https://github.com/visionmedia/supertest) 库来模拟 HTTP 请求。

```typescript
@@filename(cats.e2e-spec)
import * as request from 'supertest';
import { Test } from '@nestjs/testing';
import { CatsModule } from '../../src/cats/cats.module';
import { CatsService } from '../../src/cats/cats.service';
import { INestApplication } from '@nestjs/common';

describe('Cats', () => {
  let app: INestApplication;
  let catsService = { findAll: () => ['test'] };

  beforeAll(async () => {
    const moduleRef = await Test.createTestingModule({
      imports: [CatsModule],
    })
      .overrideProvider(CatsService)
      .useValue(catsService)
      .compile();

    app = moduleRef.createNestApplication();
    await app.init();
  });

  it(`/GET cats`, () => {
    return request(app.getHttpServer())
      .get('/cats')
      .expect(200)
      .expect({
        data: catsService.findAll(),
      });
  });

  afterAll(async () => {
    await app.close();
  });
});
@@switch
import * as request from 'supertest';
import { Test } from '@nestjs/testing';
import { CatsModule } from '../../src/cats/cats.module';
import { CatsService } from '../../src/cats/cats.service';
import { INestApplication } from '@nestjs/common';

describe('Cats', () => {
  let app: INestApplication;
  let catsService = { findAll: () => ['test'] };

  beforeAll(async () => {
    const moduleRef = await Test.createTestingModule({
      imports: [CatsModule],
    })
      .overrideProvider(CatsService)
      .useValue(catsService)
      .compile();

    app = moduleRef.createNestApplication();
    await app.init();
  });

  it(`/GET cats`, () => {
    return request(app.getHttpServer())
      .get('/cats')
      .expect(200)
      .expect({
        data: catsService.findAll(),
      });
  });

  afterAll(async () => {
    await app.close();
  });
});
```

> info **提示** 如果你使用 [Fastify](/techniques/performance) 作为你的 HTTP 适配器，它需要稍微不同的配置，并具有内置的测试功能：
>
> ```ts
> let app: NestFastifyApplication;
>
> beforeAll(async () => {
>   app = moduleRef.createNestApplication<NestFastifyApplication>(
>     new FastifyAdapter(),
>   );
>
>   await app.init();
>   await app.getHttpAdapter().getInstance().ready();
> });
>
> it(`/GET cats`, () => {
>   return app
>     .inject({
>       method: 'GET',
>       url: '/cats',
>     })
>     .then((result) => {
>       expect(result.statusCode).toEqual(200);
>       expect(result.payload).toEqual(/* expectedPayload */);
>     });
> });
>
> afterAll(async () => {
>   await app.close();
> });
> ```

在这个示例中，我们建立在前面描述的一些概念之上。除了我们之前使用的 `compile()` 方法之外，现在我们还使用 `createNestApplication()` 方法来实例化一个完整的 Nest 运行环境。

需要注意的一个问题是，当你使用 `compile()` 方法编译应用程序时，此时 `HttpAdapterHost#httpAdapter` 将为 `undefined`。这是因为在此编译阶段还没有创建 HTTP 适配器或服务器。如果你的测试需要 `httpAdapter`，你应该使用 `createNestApplication()` 方法创建应用程序实例，或者重构你的项目，以避免在初始化依赖图时依赖此属性。

好的，让我们逐步分析这个示例：

我们将运行中的应用程序引用保存在 `app` 变量中，以便我们可以使用它来模拟 HTTP 请求。

我们使用 Supertest 中的 `request()` 函数来模拟 HTTP 测试。我们希望这些 HTTP 请求路由到我们运行的 Nest 应用程序，因此我们将 `request()` 函数传入 Nest 底层 HTTP 监听器的引用（反过来，这可能由 Express 平台提供）。因此构造 `request(app.getHttpServer())`。对 `request()` 的调用为我们提供了一个被包装的 HTTP 服务器，现在它连接到了 Nest 应用程序，该服务器暴露了模拟实际 HTTP 请求的方法。例如，使用 `request(...).get('/cats')` 将发起一个与通过网络接收的 `get '/cats'` 请求相同的请求。

在这个示例中，我们还提供了一个替代（测试双）实现的 `CatsService`，它简单地返回一个我们可以在测试中验证的硬编码值。使用 `overrideProvider()` 来提供这样的替代实现。同样，Nest 提供了使用 `overrideModule()`、`overrideGuard()`、`overrideInterceptor()`、`overrideFilter()` 和 `overridePipe()` 方法分别覆盖模块、守卫、拦截器、过滤器和管道的方法。

除了 `overrideModule()` 之外，每个覆盖方法都返回一个对象，该对象具有与 [自定义提供者](https://docs.nestjs.com/fundamentals/custom-providers) 描述的相同的方法：

- `useClass`：你提供一个类，该类将被实例化以提供用于覆盖对象（提供者、守卫等）的实例。
- `useValue`：你提供一个实例，该实例将用于覆盖对象。
- `useFactory`：你提供一个返回实例的函数，该实例将用于覆盖对象。

另一方面，`overrideModule()` 返回一个具有 `useModule()` 方法的对象，你可以使用它来提供一个将覆盖原始模块的模块，如下所示：

```typescript
const moduleRef = await Test.createTestingModule({
  imports: [AppModule],
})
  .overrideModule(CatsModule)
  .useModule(AlternateCatsModule)
  .compile();
```

每种覆盖方法类型都会返回 `TestingModule` 实例，因此可以用 [流式接口](https://en.wikipedia.org/wiki/Fluent_interface) 的方式链式调用其他方法。你应该在链的最后使用 `compile()` 方法，让 Nest 实例化并初始化模块。

此外，有时你可能希望提供一个自定义的日志记录器，例如在 CI 服务器上运行测试时。使用 `setLogger()` 方法并传入一个符合 `LoggerService` 接口的对象，以指示 `TestModuleBuilder` 在测试期间如何记录日志（默认情况下，只有 "error" 日志会被输出到控制台）。

编译后的模块具有几个有用的方法，如下表所示：

<table>
  <tr>
    <td>
      <code>createNestApplication()</code>
    </td>
    <td>
      基于给定模块创建并返回一个 Nest 应用程序（<code>INestApplication</code> 实例）。
      请注意，你必须使用 <code>init()</code> 方法手动初始化应用程序。
    </td>
  </tr>
  <tr>
    <td>
      <code>createNestMicroservice()</code>
    </td>
    <td>
      基于给定模块创建并返回一个 Nest 微服务（<code>INestMicroservice</code> 实例）。
    </td>
  </tr>
  <tr>
    <td>
      <code>get()</code>
    </td>
    <td>
      检索应用程序上下文中可用的控制器或提供者（包括守卫、过滤器等）的静态实例。继承自 <a href="/fundamentals/module-ref">模块引用</a> 类。
    </td>
  </tr>
  <tr>
     <td>
      <code>resolve()</code>
    </td>
    <td>
      检索应用程序上下文中可用的控制器或提供者（包括守卫、过滤器等）的动态创建的作用域实例（请求或瞬态）。继承自 <a href="/fundamentals/module-ref">模块引用</a> 类。
    </td>
  </tr>
  <tr>
    <td>
      <code>select()</code>
    </td>
    <td>
      遍历模块的依赖图；可用于从选定的模块中检索特定实例（与 <code>get()</code> 方法中的严格模式（<code>strict: true</code>）一起使用）。
    </td>
  </tr>
</table>

> info **提示** 将你的 e2e 测试文件放在 `test` 目录中。测试文件应以 `.e2e-spec` 作为后缀。

#### 覆盖全局注册的增强器

如果你有一个全局注册的守卫（或管道、拦截器或过滤器），你需要采取更多步骤来覆盖该增强器。回顾原始注册如下所示：

```typescript
providers: [
  {
    provide: APP_GUARD,
    useClass: JwtAuthGuard,
  },
],
```

这是通过 `APP_*` token 将守卫注册为一个 "multi"-provider。为了能够在此处替换 `JwtAuthGuard`，注册需要使用现有的提供者：

```typescript
providers: [
  {
    provide: APP_GUARD,
    useExisting: JwtAuthGuard,
    // ^^^^^^^^ 注意这里使用了 'useExisting' 而不是 'useClass'
  },
  JwtAuthGuard,
],
```

> info **提示** 将 `useClass` 改为 `useExisting`，以引用一个已注册的提供者，而不是让 Nest 在 token 后面实例化它。

现在，`JwtAuthGuard` 对 Nest 来说是一个可见的常规提供者，可以在创建 `TestingModule` 时被覆盖：

```typescript
const moduleRef = await Test.createTestingModule({
  imports: [AppModule],
})
  .overrideProvider(JwtAuthGuard)
  .useClass(MockAuthGuard)
  .compile();
```

现在你的所有测试在每次请求中都将使用 `MockAuthGuard`。

#### 测试请求作用域实例

[请求作用域](/fundamentals/injection-sopes) 提供者为每个传入的 **请求** 唯一创建。请求处理完成后，该实例将被垃圾回收。这带来了一个问题，因为我们无法访问为被测试请求生成的特定依赖注入子树。

我们知道（根据前面的部分），`resolve()` 方法可以用来检索一个动态实例化的类。另外，如 <a href="https://docs.nestjs.com/fundamentals/module-ref#resolving-scoped-providers">此处</a> 所述，我们知道可以传递一个唯一的上下文标识符来控制 DI 容器子树的生命周期。如何在测试环境中利用这一点？

策略是预先生成一个上下文标识符，并强制 Nest 在所有传入请求中使用该特定标识符来创建子树。这样我们就可以访问为被测试请求创建的实例。

为此，使用 `jest.spyOn()` 在 `ContextIdFactory` 上：

```typescript
const contextId = ContextIdFactory.create();
jest
  .spyOn(ContextIdFactory, 'getByRequest')
  .mockImplementation(() => contextId);
```

现在我们可以使用 `contextId` 来访问任何后续请求所生成的 DI 容器子树中的实例。

```typescript
catsService = await moduleRef.resolve(CatsService, contextId);
```