### 循环依赖

当两个类相互依赖时，就会出现循环依赖。例如，类 A 需要类 B，而类 B 也反过来需要类 A。在 Nest 中，循环依赖可能出现在模块之间，以及提供者（provider）之间。

虽然应尽可能避免循环依赖，但有时却无法做到。在这种情况下，Nest 提供了两种方式来解决提供者之间的循环依赖。在本章中，我们将介绍其中一种技术——使用 **前向引用（forward referencing）**，以及另一种方式——使用 **ModuleRef** 类从 DI 容器中获取提供者的实例。

我们还将介绍如何解决模块之间的循环依赖。

> warning **警告** 当使用“桶文件”（barrel files）或 `index.ts` 文件来组织导入时，也可能引发循环依赖。在导入模块或提供者类时应避免使用桶文件。例如，当导入与桶文件处于同一目录的文件时，不应该使用桶文件。也就是说，`cats/cats.controller` 不应通过导入 `cats` 来引入 `cats/cats.service` 文件。更多详情请参阅 [这个 GitHub issue](https://github.com/nestjs/nest/issues/1181#issuecomment-430197191)。

#### 前向引用

一个 **前向引用（forward reference）** 允许 Nest 使用 `forwardRef()` 工具函数引用尚未定义的类。例如，如果 `CatsService` 和 `CommonService` 相互依赖，那么两者都可以使用 `@Inject()` 和 `forwardRef()` 工具来解决循环依赖问题。否则，Nest 将无法实例化它们，因为所有必要的元数据都不可用。以下是一个示例：

```typescript
@@filename(cats.service)
@Injectable()
export class CatsService {
  constructor(
    @Inject(forwardRef(() => CommonService))
    private commonService: CommonService,
  ) {}
}
@@switch
@Injectable()
@Dependencies(forwardRef(() => CommonService))
export class CatsService {
  constructor(commonService) {
    this.commonService = commonService;
  }
}
```

> info **提示** `forwardRef()` 函数是从 `@nestjs/common` 包中导入的。

以上是关系的一边。现在我们对 `CommonService` 做同样的处理：

```typescript
@@filename(common.service)
@Injectable()
export class CommonService {
  constructor(
    @Inject(forwardRef(() => CatsService))
    private catsService: CatsService,
  ) {}
}
@@switch
@Injectable()
@Dependencies(forwardRef(() => CatsService))
export class CommonService {
  constructor(catsService) {
    this.catsService = catsService;
  }
}
```

> warning **警告** 实例化的顺序是不确定的。请确保你的代码不依赖于哪个构造函数先被调用。如果循环依赖中包含作用域为 `Scope.REQUEST` 的提供者，可能会导致依赖为 `undefined`。更多信息请参阅 [这里](https://github.com/nestjs/nest/issues/5778)。

#### 使用 ModuleRef 类作为替代方案

除了使用 `forwardRef()`，你还可以通过重构代码并使用 `ModuleRef` 类在循环依赖的一边获取提供者实例。有关 `ModuleRef` 工具类的更多信息，请参阅 [这里](/fundamentals/module-ref)。

#### 模块间的前向引用

为了解决模块之间的循环依赖，可以在两个模块的关联中都使用相同的 `forwardRef()` 工具函数。例如：

```typescript
@@filename(common.module)
@Module({
  imports: [forwardRef(() => CatsModule)],
})
export class CommonModule {}
```

以上是关系的一边。现在我们对 `CatsModule` 做同样的处理：

```typescript
@@filename(cats.module)
@Module({
  imports: [forwardRef(() => CommonModule)],
})
export class CatsModule {}
```