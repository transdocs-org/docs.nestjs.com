### 复杂度控制

> warning **警告** 本章节仅适用于代码优先（code first）的方法。

查询复杂度允许你定义某些字段的复杂程度，并通过设置**最大复杂度**来限制查询。其核心思想是通过一个简单的数字来表示每个字段的复杂度。通常的默认值是将每个字段的复杂度设为 `1`。此外，GraphQL 查询的复杂度计算可以通过所谓的复杂度估算器（complexity estimators）进行自定义。复杂度估算器是一个简单的函数，用于计算字段的复杂度。你可以在规则中添加任意数量的复杂度估算器，它们将依次执行。第一个返回数值复杂度的估算器将决定该字段的复杂度。

`@nestjs/graphql` 包可以很好地与 [graphql-query-complexity](https://github.com/slicknode/graphql-query-complexity) 这类工具集成，该工具提供了一种基于成本分析的解决方案。使用该库，你可以拒绝执行那些被认为代价过高的查询。

#### 安装

要开始使用，请先安装所需的依赖项。

```bash
$ npm install --save graphql-query-complexity
```

#### 入门

安装完成后，我们可以定义 `ComplexityPlugin` 类：

```typescript
import { GraphQLSchemaHost } from '@nestjs/graphql';
import { Plugin } from '@nestjs/apollo';
import {
  ApolloServerPlugin,
  BaseContext,
  GraphQLRequestListener,
} from '@apollo/server';
import { GraphQLError } from 'graphql';
import {
  fieldExtensionsEstimator,
  getComplexity,
  simpleEstimator,
} from 'graphql-query-complexity';

@Plugin()
export class ComplexityPlugin implements ApolloServerPlugin {
  constructor(private gqlSchemaHost: GraphQLSchemaHost) {}

  async requestDidStart(): Promise<GraphQLRequestListener<BaseContext>> {
    const maxComplexity = 20;
    const { schema } = this.gqlSchemaHost;

    return {
      async didResolveOperation({ request, document }) {
        const complexity = getComplexity({
          schema,
          operationName: request.operationName,
          query: document,
          variables: request.variables,
          estimators: [
            fieldExtensionsEstimator(),
            simpleEstimator({ defaultComplexity: 1 }),
          ],
        });
        if (complexity > maxComplexity) {
          throw new GraphQLError(
            `查询过于复杂：${complexity}。允许的最大复杂度：${maxComplexity}`,
          );
        }
        console.log('查询复杂度：', complexity);
      },
    };
  }
}
```

为了演示方便，我们将允许的最大复杂度设为 `20`。在上面的例子中，我们使用了两个估算器：`simpleEstimator` 和 `fieldExtensionsEstimator`。

- `simpleEstimator`: 简单估算器为每个字段返回固定的复杂度值
- `fieldExtensionsEstimator`: 字段扩展估算器从你的 schema 中每个字段提取复杂度值

> info **提示** 记得将此类添加到任意模块的 providers 数组中。

#### 字段级别的复杂度

有了这个插件后，我们现在可以通过将 `complexity` 属性传递给 `@Field()` 装饰器的选项对象，来定义任意字段的复杂度，如下所示：

```typescript
@Field({ complexity: 3 })
title: string;
```

或者，你也可以定义一个估算函数：

```typescript
@Field({ complexity: (options: ComplexityEstimatorArgs) => ... })
title: string;
```

#### 查询/变更级别的复杂度

此外，`@Query()` 和 `@Mutation()` 装饰器也可以像如下方式一样指定 `complexity` 属性：

```typescript
@Query({ complexity: (options: ComplexityEstimatorArgs) => options.args.count * options.childComplexity })
items(@Args('count') count: number) {
  return this.itemsService.getItems({ count });
}
```