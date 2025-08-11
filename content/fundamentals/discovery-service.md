### 发现服务（Discovery Service）

`@nestjs/core` 提供的 `DiscoveryService` 是一个强大的工具，它允许开发者动态地检查和获取 NestJS 应用中的提供者（providers）、控制器（controllers）及其他元数据。这对于构建依赖运行时反射（introspection）的插件、装饰器（decorators）或高级功能特别有用。通过使用 `DiscoveryService`，开发者可以创建更加灵活和模块化的架构，从而在应用中实现自动化和动态行为。

#### 入门指南

在使用 `DiscoveryService` 之前，你需要在计划使用它的模块中导入 `DiscoveryModule`。这确保了该服务可以被依赖注入。以下是一个在 NestJS 模块中进行配置的示例：

```typescript
import { Module } from '@nestjs/common';
import { DiscoveryModule } from '@nestjs/core';
import { ExampleService } from './example.service';

@Module({
  imports: [DiscoveryModule],
  providers: [ExampleService],
})
export class ExampleModule {}
```

一旦模块配置完成，就可以将 `DiscoveryService` 注入到任何需要动态发现功能的提供者或服务中。

```typescript
@@filename(example.service)
@Injectable()
export class ExampleService {
  constructor(private readonly discoveryService: DiscoveryService) {}
}
@@switch
@Injectable()
@Dependencies(DiscoveryService)
export class ExampleService {
  constructor(discoveryService) {
    this.discoveryService = discoveryService;
  }
}
```

#### 发现提供者和控制器

`DiscoveryService` 的一个核心能力是检索应用中所有已注册的提供者。这对于根据特定条件动态处理提供者非常有用。以下代码片段演示了如何访问所有提供者：

```typescript
const providers = this.discoveryService.getProviders();
console.log(providers);
```

每个提供者对象包含诸如其实例、令牌（token）和元数据等信息。同样地，如果你需要检索应用中所有已注册的控制器，可以使用以下代码：

```typescript
const controllers = this.discoveryService.getControllers();
console.log(controllers);
```

该功能在需要动态处理控制器的场景下特别有用，例如分析追踪或自动注册机制。

#### 提取元数据

除了发现提供者和控制器之外，`DiscoveryService` 还支持检索附加到这些组件上的元数据。这在使用自定义装饰器在运行时存储元数据时特别有价值。

例如，考虑一个使用自定义装饰器为提供者附加特定元数据的场景：

```typescript
import { DiscoveryService } from '@nestjs/core';

export const FeatureFlag = DiscoveryService.createDecorator();
```

将此装饰器应用于一个服务，可以使其存储稍后可查询的元数据：

```typescript
import { Injectable } from '@nestjs/common';
import { FeatureFlag } from './custom-metadata.decorator';

@Injectable()
@FeatureFlag('experimental')
export class CustomService {}
```

一旦通过这种方式为提供者附加了元数据，`DiscoveryService` 就可以轻松地根据分配的元数据筛选提供者。以下代码片段展示了如何检索具有特定元数据值的提供者：

```typescript
const providers = this.discoveryService.getProviders();

const [provider] = providers.filter(
  (item) =>
    this.discoveryService.getMetadataByDecorator(FeatureFlag, item) ===
    'experimental',
);

console.log(
  'Providers with the "experimental" feature flag metadata:',
  provider,
);
```

#### 总结

`DiscoveryService` 是一个功能强大且灵活的工具，它支持在 NestJS 应用中进行运行时反射。通过允许动态发现提供者、控制器和元数据，它在构建可扩展的框架、插件和自动化驱动的功能方面发挥着关键作用。无论你是需要扫描和处理提供者、提取元数据用于高级处理，还是创建模块化和可扩展的架构，`DiscoveryService` 都提供了一种高效且结构化的方式来实现这些目标。