### 异步本地存储

`AsyncLocalStorage` 是一个 [Node.js API](https://nodejs.org/api/async_context.html#async_context_class_asynclocalstorage)（基于 `async_hooks` API），它提供了一种替代方式来在应用程序中传递本地状态，而无需显式地将状态作为函数参数传递。它类似于其他语言中的线程本地存储。

Async Local Storage 的主要思想是：我们可以通过 `AsyncLocalStorage#run` 方法来“包裹”某个函数调用。所有在被包裹的调用中执行的代码都可以访问同一个 `store`，而该 `store` 对每个调用链来说都是唯一的。

在 NestJS 的上下文中，这意味着如果我们能在请求生命周期中的某个位置包裹住其余的请求代码，我们就可以访问和修改仅对该请求可见的状态，这可以作为 REQUEST 作用域提供者的替代方案，并克服其某些限制。

或者，我们可以使用 ALS 仅在系统的某一部分（例如 _transaction_ 对象）中传播上下文，而无需在服务间显式传递，这可以提高隔离性和封装性。

#### 自定义实现

NestJS 本身并未提供 `AsyncLocalStorage` 的内置抽象，因此让我们通过一个简单的 HTTP 场景来实现它，以更好地理解整个概念：

> info **提示** 如果需要使用现成的[专用包](recipes/async-local-storage#nestjs-cls)，请继续阅读下方内容。

1. 首先，在某个共享的源文件中创建一个新的 `AsyncLocalStorage` 实例。由于我们使用的是 NestJS，我们也将其封装成一个模块，并添加一个自定义提供者。

```ts
@@filename(als.module)
@Module({
  providers: [
    {
      provide: AsyncLocalStorage,
      useValue: new AsyncLocalStorage(),
    },
  ],
  exports: [AsyncLocalStorage],
})
export class AlsModule {}
```
> info **提示** `AsyncLocalStorage` 从 `async_hooks` 模块导入。

2. 我们只关注 HTTP 请求，因此使用中间件将 `next` 函数用 `AsyncLocalStorage#run` 包裹起来。由于中间件是请求最先触发的部分，这将使得 `store` 在所有增强器和系统其余部分中都可用。

```ts
@@filename(app.module)
@Module({
  imports: [AlsModule],
  providers: [CatsService],
  controllers: [CatsController],
})
export class AppModule implements NestModule {
  constructor(
    // 在模块构造函数中注入 AsyncLocalStorage，
    private readonly als: AsyncLocalStorage
  ) {}

  configure(consumer: MiddlewareConsumer) {
    // 绑定中间件，
    consumer
      .apply((req, res, next) => {
        // 根据请求填充 store 的默认值，
        const store = {
          userId: req.headers['x-user-id'],
        };
        // 将 "next" 函数作为回调传递给 "als.run" 方法，并带上 store。
        this.als.run(store, () => next());
      })
      .forRoutes('*path');
  }
}
@@switch
@Module({
  imports: [AlsModule],
  providers: [CatsService],
  controllers: [CatsController],
})
@Dependencies(AsyncLocalStorage)
export class AppModule {
  constructor(als) {
    // 在模块构造函数中注入 AsyncLocalStorage，
    this.als = als
  }

  configure(consumer) {
    // 绑定中间件，
    consumer
      .apply((req, res, next) => {
        // 根据请求填充 store 的默认值，
        const store = {
          userId: req.headers['x-user-id'],
        };
        // 将 "next" 函数作为回调传递给 "als.run" 方法，并带上 store。
        this.als.run(store, () => next());
      })
      .forRoutes('*path');
  }
}
```

3. 现在，在请求生命周期中的任何位置，我们都可以访问本地的 store 实例。

```ts
@@filename(cats.service)
@Injectable()
export class CatsService {
  constructor(
    // 可以注入提供的 ALS 实例。
    private readonly als: AsyncLocalStorage,
    private readonly catsRepository: CatsRepository,
  ) {}

  getCatForUser() {
    // "getStore" 方法将始终返回与当前请求关联的 store 实例。
    const userId = this.als.getStore()["userId"] as number;
    return this.catsRepository.getForUser(userId);
  }
}
@@switch
@Injectable()
@Dependencies(AsyncLocalStorage, CatsRepository)
export class CatsService {
  constructor(als, catsRepository) {
    // 可以注入提供的 ALS 实例。
    this.als = als
    this.catsRepository = catsRepository
  }

  getCatForUser() {
    // "getStore" 方法将始终返回与当前请求关联的 store 实例。
    const userId = this.als.getStore()["userId"] as number;
    return this.catsRepository.getForUser(userId);
  }
}
```

4. 这样就完成了。现在我们可以通过 ALS 在请求上下文中共享状态，而无需注入整个 `REQUEST` 对象。

> warning **警告** 请注意，虽然该技术在许多用例中非常有用，但它会隐式地隐藏代码流程（创建了隐式上下文），因此请谨慎使用，尤其是避免创建上下文中的“上帝对象”([God objects](https://en.wikipedia.org/wiki/God_object))。

### NestJS CLS

[nestjs-cls](https://github.com/Papooch/nestjs-cls) 包提供了比直接使用 `AsyncLocalStorage` 更好的开发者体验（CLS 是 _continuation-local storage_ 的缩写）。它将实现抽象为一个 `ClsModule`，提供了多种初始化 `store` 的方式，适用于不同的传输协议（不仅仅是 HTTP），还支持强类型。

之后，可以通过可注入的 `ClsService` 访问 store，或者通过使用 [Proxy Providers](https://www.npmjs.com/package/nestjs-cls#proxy-providers) 完全将其从业务逻辑中抽象出来。

> info **提示** `nestjs-cls` 是一个第三方包，不由 NestJS 核心团队维护。如有问题，请在 [对应的仓库](https://github.com/Papooch/nestjs-cls/issues) 中报告。

#### 安装

除了对 `@nestjs` 相关库的 peer dependency 依赖外，它仅使用 Node.js 的内置 API。像其他包一样安装即可：

```bash
npm i nestjs-cls
```

#### 使用

类似[上方](recipes/async-local-storage#custom-implementation) 所描述的功能，使用 `nestjs-cls` 可以如下实现：

1. 在根模块中导入 `ClsModule`。

```ts
@@filename(app.module)
@Module({
  imports: [
    // 注册 ClsModule，
    ClsModule.forRoot({
      middleware: {
        // 自动为所有路由挂载 ClsMiddleware，
        mount: true,
        // 并使用 setup 方法提供默认的 store 值。
        setup: (cls, req) => {
          cls.set('userId', req.headers['x-user-id']);
        },
      },
    }),
  ],
  providers: [CatsService],
  controllers: [CatsController],
})
export class AppModule {}
```

2. 然后使用 `ClsService` 来访问 store 中的值。

```ts
@@filename(cats.service)
@Injectable()
export class CatsService {
  constructor(
    // 可以注入提供的 ClsService 实例，
    private readonly cls: ClsService,
    private readonly catsRepository: CatsRepository,
  ) {}

  getCatForUser() {
    // 使用 "get" 方法获取存储的值。
    const userId = this.cls.get('userId');
    return this.catsRepository.getForUser(userId);
  }
}
@@switch
@Injectable()
@Dependencies(AsyncLocalStorage, CatsRepository)
export class CatsService {
  constructor(cls, catsRepository) {
    // 可以注入提供的 ClsService 实例，
    this.cls = cls
    this.catsRepository = catsRepository
  }

  getCatForUser() {
    // 使用 "get" 方法获取存储的值。
    const userId = this.cls.get('userId');
    return this.catsRepository.getForUser(userId);
  }
}
```

3. 为了对 `ClsService` 管理的 store 值进行强类型支持（并获得字符串键的自动补全），我们可以在注入时使用可选的类型参数 `ClsService<MyClsStore>`。

```ts
export interface MyClsStore extends ClsStore {
  userId: number;
}
```

> info **提示** 该包还支持自动生成请求 ID，并通过 `cls.getId()` 获取，或者通过 `cls.get(CLS_REQ)` 获取整个 Request 对象。

#### 测试

由于 `ClsService` 只是一个可注入的提供者，因此可以在单元测试中完全模拟它。

但在某些集成测试中，我们可能仍希望使用真实的 `ClsService` 实现。在这种情况下，我们需要使用 `ClsService#run` 或 `ClsService#runWith` 方法包裹上下文相关的代码。

```ts
describe('CatsService', () => {
  let service: CatsService
  let cls: ClsService
  const mockCatsRepository = createMock<CatsRepository>()

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      // 设置测试模块，与平时相同。
      providers: [
        CatsService,
        {
          provide: CatsRepository,
          useValue: mockCatsRepository
        }
      ],
      imports: [
        // 导入静态版本的 ClsModule，它只提供 ClsService，
        // 但不会以任何方式初始化 store。
        ClsModule
      ],
    }).compile()

    service = module.get(CatsService)

    // 同时获取 ClsService 供后续使用。
    cls = module.get(ClsService)
  })

  describe('getCatForUser', () => {
    it('retrieves cat based on user id', async () => {
      const expectedUserId = 42
      mocksCatsRepository.getForUser.mockImplementationOnce(
        (id) => ({ userId: id })
      )

      // 使用 `runWith` 方法包裹测试调用，
      // 在其中可以手动传递 store 值。
      const cat = await cls.runWith(
        { userId: expectedUserId },
        () => service.getCatForUser()
      )

      expect(cat.userId).toEqual(expectedUserId)
    })
  })
})
```

#### 更多信息

请访问 [NestJS CLS GitHub 页面](https://github.com/Papooch/nestjs-cls) 获取完整的 API 文档和更多代码示例。