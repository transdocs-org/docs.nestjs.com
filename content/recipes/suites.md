### 套件（原名 Automock）

套件（Suites）是一个有特定理念且灵活的测试元框架，旨在提升后端系统的软件测试体验。通过将多种测试工具整合到一个统一的框架中，Suites 简化了可靠测试的创建过程，有助于开发高质量的软件。

> info **提示** `Suites` 是一个第三方包，不由 NestJS 核心团队维护。如发现该库存在问题，请向 [对应的仓库](https://github.com/suites-dev/suites) 提交 issue。

#### 简介

控制反转（IoC）是 NestJS 框架的核心原则之一，它支持模块化和可测试的架构。虽然 NestJS 提供了用于创建测试模块的内置工具，但 Suites 提供了一种替代方法，强调对独立单元或小单元组合的测试。Suites 使用一个虚拟的依赖容器，其中的模拟对象（mock）会自动创建，无需手动将每个提供者（provider）替换为模拟对象。这种方法既可以替代 NestJS 的 `Test.createTestingModule` 方法，也可以与之并用，提供更灵活的单元测试方式。

#### 安装

要在 NestJS 中使用 Suites，请安装以下依赖包：

```bash
$ npm i -D @suites/unit @suites/di.nestjs @suites/doubles.jest
```

> info **提示** `Suites` 也支持 Vitest 和 Sinon 作为测试模拟库，分别为 `@suites/doubles.vitest` 和 `@suites/doubles.sinon`。

#### 示例与模块设置

考虑一个包含 `CatsApiService`、`CatsDAL`、`HttpClient` 和 `Logger` 的 `CatsService` 模块设置。这将是本示例的基础：

```typescript
@@filename(cats.module)
import { HttpModule } from '@nestjs/axios';
import { PrismaModule } from '../prisma.module';

@Module({
  imports: [HttpModule.register({ baseUrl: 'https://api.cats.com/' }), PrismaModule],
  providers: [CatsService, CatsApiService, CatsDAL, Logger],
  exports: [CatsService],
})
export class CatsModule {}
```

`HttpModule` 和 `PrismaModule` 都向宿主模块导出了提供者。

现在我们从单独测试 `CatsHttpService` 开始。该服务负责从 API 获取猫咪数据并记录操作。

```typescript
@@filename(cats-http.service)
@Injectable()
export class CatsHttpService {
  constructor(private httpClient: HttpClient, private logger: Logger) {}

  async fetchCats(): Promise<Cat[]> {
    this.logger.log('从 API 获取猫咪数据');
    const response = await this.httpClient.get('/cats');
    return response.data;
  }
}
```

我们希望隔离 `CatsHttpService` 并模拟其依赖项 `HttpClient` 和 `Logger`。Suites 提供了 `.solitary()` 方法，可轻松实现这一目标。

```typescript
@@filename(cats-http.service.spec)
import { TestBed, Mocked } from '@suites/unit';

describe('Cats Http Service 单元测试', () => {
  let catsHttpService: CatsHttpService;
  let httpClient: Mocked<HttpClient>;
  let logger: Mocked<Logger>;

  beforeAll(async () => {
    // 隔离 CatsHttpService 并模拟 HttpClient 和 Logger
    const { unit, unitRef } = await TestBed.solitary(CatsHttpService).compile();

    catsHttpService = unit;
    httpClient = unitRef.get(HttpClient);
    logger = unitRef.get(Logger);
  });

  it('应从 API 获取猫咪数据并记录操作', async () => {
    const catsFixtures: Cat[] = [{ id: 1, name: 'Catty' }, { id: 2, name: 'Mitzy' }];
    httpClient.get.mockResolvedValue({ data: catsFixtures });

    const cats = await catsHttpService.fetchCats();

    expect(logger.log).toHaveBeenCalledWith('从 API 获取猫咪数据');
    expect(httpClient.get).toHaveBeenCalledWith('/cats');
    expect(cats).toEqual<Cat[]>(catsFixtures);
  });
});
```

在上面的示例中，Suites 使用 `TestBed.solitary()` 自动模拟了 `CatsHttpService` 的依赖项。这使得设置更加简单，因为无需手动模拟每个依赖项。

- **自动模拟依赖项**：Suites 会为被测单元的所有依赖项生成模拟对象。
- **模拟对象的空行为**：初始时这些模拟对象没有预定义行为，你需要根据测试需要指定其行为。
- `unit` 和 `unitRef` 属性：
  - `unit` 指向被测类的实际实例，包含其模拟的依赖项。
  - `unitRef` 是一个引用，允许你访问模拟的依赖项。

#### 使用 `TestingModule` 测试 `CatsApiService`

对于 `CatsApiService`，我们想确保 `HttpModule` 在 `CatsModule` 宿主模块中被正确导入和配置。这包括验证 `Axios` 的基础 URL（及其他配置）是否设置正确。

在这种情况下，我们不会使用 Suites，而是使用 Nest 的 `TestingModule` 来测试 `HttpModule` 的实际配置。我们将使用 `nock` 来模拟 HTTP 请求，而不是在此场景中模拟 `HttpClient`。

```typescript
@@filename(cats-api.service)
import { HttpClient } from '@nestjs/axios';

@Injectable()
export class CatsApiService {
  constructor(private httpClient: HttpClient) {}

  async getCatById(id: number): Promise<Cat> {
    const response = await this.httpClient.get(`/cats/${id}`);
    return response.data;
  }
}
```

我们需要使用真实、未模拟的 `HttpClient` 来测试 `CatsApiService`，以确保 DI 和 `Axios`（HTTP）的配置正确。这包括导入 `CatsModule` 并使用 `nock` 模拟 HTTP 请求。

```typescript
@@filename(cats-api.service.integration.test)
import { Test } from '@nestjs/testing';
import * as nock from 'nock';

describe('Cats Api Service 集成测试', () => {
  let catsApiService: CatsApiService;

  beforeAll(async () => {
    const moduleRef = await Test.createTestingModule({
      imports: [CatsModule],
    }).compile();

    catsApiService = moduleRef.get(CatsApiService);
  });

  afterEach(() => {
    nock.cleanAll();
  });

  it('应使用真实 HttpClient 按 ID 获取猫咪', async () => {
    const catFixture: Cat = { id: 1, name: 'Catty' };

    nock('https://api.cats.com') // 使该 URL 与 HttpModule 注册中的相同
      .get('/cats/1')
      .reply(200, catFixture);

    const cat = await catsApiService.getCatById(1);
    expect(cat).toEqual<Cat>(catFixture);
  });
});
```

#### 可交互测试示例

接下来，我们测试 `CatsService`，它依赖于 `CatsApiService` 和 `CatsDAL`。我们将模拟 `CatsApiService` 并暴露 `CatsDAL`。

```typescript
@@filename(cats.dal)
import { PrismaClient } from '@prisma/client';

@Injectable()
export class CatsDAL {
  constructor(private prisma: PrismaClient) {}

  async saveCat(cat: Cat): Promise<Cat> {
    return this.prisma.cat.create({data: cat});
  }
}
```

接下来是 `CatsService`，它依赖于 `CatsApiService` 和 `CatsDAL`：

```typescript
@@filename(cats.service)
@Injectable()
export class CatsService {
  constructor(
    private catsApiService: CatsApiService,
    private catsDAL: CatsDAL
  ) {}

  async getAndSaveCat(id: number): Promise<Cat> {
    const cat = await this.catsApiService.getCatById(id);
    return this.catsDAL.saveCat(cat);
  }
}
```

现在，我们使用 Suites 的可交互测试来测试 `CatsService`：

```typescript
@@filename(cats.service.spec)
import { TestBed, Mocked } from '@suites/unit';
import { PrismaClient } from '@prisma/client';

describe('Cats Service 可交互单元测试', () => {
  let catsService: CatsService;
  let prisma: Mocked<PrismaClient>;
  let catsApiService: Mocked<CatsApiService>;

  beforeAll(async () => {
    // 可交互测试设置，暴露 CatsDAL 并模拟 CatsApiService
    const { unit, unitRef } = await TestBed.sociable(CatsService)
      .expose(CatsDAL)
      .mock(CatsApiService)
      .final({ getCatById: async () => ({ id: 1, name: 'Catty' })})
      .compile();

    catsService = unit;
    prisma = unitRef.get(PrismaClient);
  });

  it('应获取猫咪并保存', async () => {
    const catFixture: Cat = { id: 1, name: 'Catty' };
    prisma.cat.create.mockResolvedValue(catFixture);

    const savedCat = await catsService.getAndSaveCat(1);

    expect(prisma.cat.create).toHaveBeenCalledWith({ data: catFixture });
    expect(savedCat).toEqual(catFixture);
  });
});
```

在这个示例中，我们使用 `.sociable()` 方法设置测试环境。我们使用 `.expose()` 方法允许与 `CatsDAL` 进行真实交互，同时使用 `.mock()` 方法模拟 `CatsApiService`。`.final()` 方法为 `CatsApiService` 设置固定行为，确保测试结果的一致性。

这种方法强调在真实交互中测试 `CatsService`，涉及到 `Prisma` 的处理。Suites 会直接使用 `CatsDAL`，而仅对其依赖项（如 `Prisma`）进行模拟。

请注意，这种方法仅用于**验证行为**，不同于加载整个测试模块。可交互测试适用于在隔离直接依赖项的情况下验证单元行为，尤其是当你关注单元行为和交互时非常有用。

#### 集成测试与数据库

对于 `CatsDAL`，可以测试真实数据库（如 SQLite 或 PostgreSQL，例如使用 Docker Compose）。但在本示例中，我们将模拟 `Prisma` 并专注于可交互测试。模拟 `Prisma` 的原因是避免 I/O 操作，专注于 `CatsService` 的行为。当然，你也可以进行包含真实 I/O 操作和实时数据库的测试。

#### 可交互单元测试、集成测试与模拟

- **可交互单元测试**：关注在模拟深层依赖项的同时测试单元之间的交互和行为。在本示例中，我们模拟了 `Prisma` 并暴露了 `CatsDAL`。
- **集成测试**：涉及真实 I/O 操作和完整的依赖注入（DI）配置。使用 `HttpModule` 和 `nock` 测试 `CatsApiService` 被视为集成测试，因为它验证了 `HttpClient` 的真实配置和交互。在此场景中，我们将使用 Nest 的 `TestingModule` 来加载实际模块配置。

**使用模拟时要小心谨慎。** 确保测试 I/O 操作和 DI 配置（尤其是涉及 HTTP 或数据库交互时）。在通过集成测试验证这些组件后，你可以放心地在可交互单元测试中模拟它们，以专注于行为和交互。Suites 的可交互测试旨在验证单元在与其直接依赖项隔离的情况下行为是否正确，而集成测试确保整个系统配置和 I/O 操作正常运行。

#### 测试 IoC 容器注册

验证 DI 容器是否正确配置非常重要，以防止运行时错误。这包括确保所有提供者、服务和模块都已正确注册和注入。测试 DI 容器配置有助于尽早发现配置错误，防止仅在运行时才出现的问题。

为了确认 IoC 容器已正确设置，我们可以创建一个集成测试，加载实际模块配置并验证所有提供者是否已正确注册和注入。

```typescript
import { Test, TestingModule } from '@nestjs/testing';
import { CatsModule } from './cats.module';
import { CatsService } from './cats.service';

describe('Cats Module 集成测试', () => {
  let moduleRef: TestingModule;

  beforeAll(async () => {
    moduleRef = await Test.createTestingModule({
      imports: [CatsModule],
    }).compile();
  });

  it('应从 IoC 容器中解析导出的提供者', () => {
    const catsService = moduleRef.get(CatsService);
    expect(catsService).toBeDefined();
  });
});
```

#### 孤立测试、可交互测试、集成测试与端到端测试的对比

#### 孤立单元测试

- **重点**：完全隔离测试单个单元（类）。
- **使用场景**：测试 `CatsHttpService`。
- **工具**：Suites 的 `TestBed.solitary()` 方法。
- **示例**：模拟 `HttpClient` 并测试 `CatsHttpService`。

#### 可交互单元测试

- **重点**：验证单元之间的交互，同时模拟深层依赖。
- **使用场景**：测试 `CatsService`，模拟 `CatsApiService` 并暴露 `CatsDAL`。
- **工具**：Suites 的 `TestBed.sociable()` 方法。
- **示例**：模拟 `Prisma` 并测试 `CatsService`。

#### 集成测试

- **重点**：涉及真实 I/O 操作和完整配置的模块（IoC 容器）。
- **使用场景**：使用 `HttpModule` 和 `nock` 测试 `CatsApiService`。
- **工具**：Nest 的 `TestingModule`。
- **示例**：测试 `HttpClient` 的真实配置和交互。

#### 端到端测试（E2E）

- **重点**：从更高层面测试类和模块的交互。
- **使用场景**：从最终用户的角度测试系统的完整行为。
- **工具**：Nest 的 `TestingModule`、`supertest`。
- **示例**：使用 `supertest` 模拟 HTTP 请求来测试 `CatsModule`。

更多关于设置和运行 E2E 测试的详细信息，请参阅 [NestJS 官方测试指南](https://docs.nestjs.com/fundamentals/testing#end-to-end-testing)。