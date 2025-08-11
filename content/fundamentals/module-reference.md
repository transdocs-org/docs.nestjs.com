### 模块引用

Nest 提供了 `ModuleRef` 类，用于遍历内部的提供者列表，并通过使用其注入令牌作为查找键来获取任何提供者的引用。`ModuleRef` 类还提供了一种方法，可以动态实例化静态和作用域提供者。可以通过常规方式将 `ModuleRef` 注入到类中：

```typescript
@@filename(cats.service)
@Injectable()
export class CatsService {
  constructor(private moduleRef: ModuleRef) {}
}
@@switch
@Injectable()
@Dependencies(ModuleRef)
export class CatsService {
  constructor(moduleRef) {
    this.moduleRef = moduleRef;
  }
}
```

> info **提示** `ModuleRef` 类是从 `@nestjs/core` 包中导入的。

#### 获取实例

`ModuleRef` 实例（以下简称 **模块引用**）有一个 `get()` 方法。默认情况下，此方法使用其注入令牌/类名返回在 *当前模块* 中注册并已实例化的提供者、控制器或可注入对象（例如 guard、interceptor 等）。如果找不到该实例，则会抛出异常。

```typescript
@@filename(cats.service)
@Injectable()
export class CatsService implements OnModuleInit {
  private service: Service;
  constructor(private moduleRef: ModuleRef) {}

  onModuleInit() {
    this.service = this.moduleRef.get(Service);
  }
}
@@switch
@Injectable()
@Dependencies(ModuleRef)
export class CatsService {
  constructor(moduleRef) {
    this.moduleRef = moduleRef;
  }

  onModuleInit() {
    this.service = this.moduleRef.get(Service);
  }
}
```

> warning **警告** 不能使用 `get()` 方法检索作用域提供者（transient 或 request-scoped）。请使用下面描述的技术 <a href="https://docs.nestjs.com/fundamentals/module-ref#resolving-scoped-providers">below</a>。如何控制作用域请[点击此处](/fundamentals/injection-scopes)。

要从全局上下文中检索提供者（例如，如果该提供者是在不同模块中注入的），请将 `{{ '{' }} strict: false {{ '}' }}` 选项作为 `get()` 的第二个参数传入。

```typescript
this.moduleRef.get(Service, { strict: false });
```

#### 解析作用域提供者

要动态解析一个作用域提供者（transient 或 request-scoped），请使用 `resolve()` 方法，并将提供者的注入令牌作为参数传入。

```typescript
@@filename(cats.service)
@Injectable()
export class CatsService implements OnModuleInit {
  private transientService: TransientService;
  constructor(private moduleRef: ModuleRef) {}

  async onModuleInit() {
    this.transientService = await this.moduleRef.resolve(TransientService);
  }
}
@@switch
@Injectable()
@Dependencies(ModuleRef)
export class CatsService {
  constructor(moduleRef) {
    this.moduleRef = moduleRef;
  }

  async onModuleInit() {
    this.transientService = await this.moduleRef.resolve(TransientService);
  }
}
```

`resolve()` 方法从其自身的 **DI 容器子树** 中返回提供者的一个唯一实例。每个子树都有一个唯一的 **上下文标识符**。因此，如果您多次调用此方法并比较实例引用，会发现它们并不相等。

```typescript
@@filename(cats.service)
@Injectable()
export class CatsService implements OnModuleInit {
  constructor(private moduleRef: ModuleRef) {}

  async onModuleInit() {
    const transientServices = await Promise.all([
      this.moduleRef.resolve(TransientService),
      this.moduleRef.resolve(TransientService),
    ]);
    console.log(transientServices[0] === transientServices[1]); // false
  }
}
@@switch
@Injectable()
@Dependencies(ModuleRef)
export class CatsService {
  constructor(moduleRef) {
    this.moduleRef = moduleRef;
  }

  async onModuleInit() {
    const transientServices = await Promise.all([
      this.moduleRef.resolve(TransientService),
      this.moduleRef.resolve(TransientService),
    ]);
    console.log(transientServices[0] === transientServices[1]); // false
  }
}
```

要跨多个 `resolve()` 调用生成一个实例，并确保它们共享同一个生成的 DI 容器子树，可以将上下文标识符传递给 `resolve()` 方法。使用 `ContextIdFactory` 类生成上下文标识符。此类提供了一个 `create()` 方法，用于返回一个合适的唯一标识符。

```typescript
@@filename(cats.service)
@Injectable()
export class CatsService implements OnModuleInit {
  constructor(private moduleRef: ModuleRef) {}

  async onModuleInit() {
    const contextId = ContextIdFactory.create();
    const transientServices = await Promise.all([
      this.moduleRef.resolve(TransientService, contextId),
      this.moduleRef.resolve(TransientService, contextId),
    ]);
    console.log(transientServices[0] === transientServices[1]); // true
  }
}
@@switch
@Injectable()
@Dependencies(ModuleRef)
export class CatsService {
  constructor(moduleRef) {
    this.moduleRef = moduleRef;
  }

  async onModuleInit() {
    const contextId = ContextIdFactory.create();
    const transientServices = await Promise.all([
      this.moduleRef.resolve(TransientService, contextId),
      this.moduleRef.resolve(TransientService, contextId),
    ]);
    console.log(transientServices[0] === transientServices[1]); // true
  }
}
```

> info **提示** `ContextIdFactory` 类是从 `@nestjs/core` 包中导入的。

#### 注册 `REQUEST` 提供者

手动生成的上下文标识符（通过 `ContextIdFactory.create()` 创建）表示的 DI 子树中，`REQUEST` 提供者为 `undefined`，因为这些子树不是由 Nest 的依赖注入系统实例化和管理的。

要为手动创建的 DI 子树注册一个自定义 `REQUEST` 对象，请使用 `ModuleRef#registerRequestByContextId()` 方法，如下所示：

```typescript
const contextId = ContextIdFactory.create();
this.moduleRef.registerRequestByContextId(/* YOUR_REQUEST_OBJECT */, contextId);
```

#### 获取当前子树

有时，您可能希望在 **请求上下文** 中解析一个 request-scoped 提供者的实例。假设 `CatsService` 是 request-scoped 的，并且您想解析同样标记为 request-scoped 提供者的 `CatsRepository` 实例。为了共享相同的 DI 容器子树，您必须获取当前的上下文标识符，而不是生成一个新的（例如使用上面显示的 `ContextIdFactory.create()` 函数）。要获取当前的上下文标识符，请首先使用 `@Inject()` 装饰器注入请求对象：

```typescript
@@filename(cats.service)
@Injectable()
export class CatsService {
  constructor(
    @Inject(REQUEST) private request: Record<string, unknown>,
  ) {}
}
@@switch
@Injectable()
@Dependencies(REQUEST)
export class CatsService {
  constructor(request) {
    this.request = request;
  }
}
```

> info **提示** 有关请求提供者的更多信息，请参见[此处](https://docs.nestjs.com/fundamentals/injection-scopes#request-provider)。

现在，使用 `ContextIdFactory` 类的 `getByRequest()` 方法根据请求对象创建一个上下文标识符，并将其传递给 `resolve()` 调用：

```typescript
const contextId = ContextIdFactory.getByRequest(this.request);
const catsRepository = await this.moduleRef.resolve(CatsRepository, contextId);
```

#### 动态实例化自定义类

要动态实例化一个 **之前未注册为提供者** 的类，请使用模块引用的 `create()` 方法。

```typescript
@@filename(cats.service)
@Injectable()
export class CatsService implements OnModuleInit {
  private catsFactory: CatsFactory;
  constructor(private moduleRef: ModuleRef) {}

  async onModuleInit() {
    this.catsFactory = await this.moduleRef.create(CatsFactory);
  }
}
@@switch
@Injectable()
@Dependencies(ModuleRef)
export class CatsService {
  constructor(moduleRef) {
    this.moduleRef = moduleRef;
  }

  async onModuleInit() {
    this.catsFactory = await this.moduleRef.create(CatsFactory);
  }
}
```

这种技术使您可以在框架容器之外有条件地实例化不同的类。