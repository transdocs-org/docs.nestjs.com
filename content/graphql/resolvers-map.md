### 解析器（Resolvers）

解析器提供将 [GraphQL](https://graphql.org/) 操作（查询、变更或订阅）转换为数据的指令。它们返回的结构与我们在 schema 中指定的相同——可以是同步返回，也可以返回一个最终解析为该结构的 Promise。通常，你会手动创建一个**解析器映射（resolver map）**。而 `@nestjs/graphql` 包则使用你用来注解类的装饰器提供的元数据自动生成解析器映射。为了演示使用该包功能创建 GraphQL API 的过程，我们将创建一个简单的作者（author）API。

#### 代码优先（Code first）

在代码优先方法中，我们不遵循手动编写 GraphQL SDL（Schema Definition Language）来创建 GraphQL schema 的常规流程。相反，我们使用 TypeScript 装饰器从 TypeScript 类定义中生成 SDL。`@nestjs/graphql` 包读取通过装饰器定义的元数据，并自动为你生成 schema。

#### 对象类型（Object types）

GraphQL schema 中大多数定义都是**对象类型**。你定义的每个对象类型都应该代表应用程序客户端可能需要交互的领域对象。例如，我们的示例 API 需要能够获取作者列表及其文章，因此我们需要定义 `Author` 类型和 `Post` 类型以支持此功能。

如果我们使用 schema 优先方法，我们会用类似下面的 SDL 定义这样的 schema：

```graphql
type Author {
  id: Int!
  firstName: String
  lastName: String
  posts: [Post!]!
}
```

在这种情况下，采用代码优先方法时，我们使用 TypeScript 类和装饰器来定义 schema。上面 SDL 的等价代码优先写法如下：

```typescript
@@filename(authors/models/author.model)
import { Field, Int, ObjectType } from '@nestjs/graphql';
import { Post } from './post';

@ObjectType()
export class Author {
  @Field(type => Int)
  id: number;

  @Field({ nullable: true })
  firstName?: string;

  @Field({ nullable: true })
  lastName?: string;

  @Field(type => [Post])
  posts: Post[];
}
```

> info **提示**：TypeScript 的元数据反射系统有几个限制，例如无法确定类包含哪些属性或识别某个属性是可选还是必填。由于这些限制，我们必须显式使用 `@Field()` 装饰器在 schema 定义类中提供每个字段的 GraphQL 类型和可选性元数据，或者使用 [CLI 插件](/graphql/cli-plugin) 自动生成这些内容。

像任何类一样，`Author` 对象类型由一组字段组成，每个字段声明一个类型。字段的类型对应于 [GraphQL 类型](https://graphql.org/learn/schema/)。字段的 GraphQL 类型可以是另一个对象类型，也可以是标量类型。GraphQL 标量类型是一个基本类型（如 `ID`、`String`、`Boolean` 或 `Int`），它解析为单个值。

> info **提示**：除了 GraphQL 内置的标量类型外，你还可以定义自定义标量类型（阅读 [更多](/graphql/scalars)）。

上面的 `Author` 对象类型定义将导致 Nest 生成我们之前展示的 SDL：

```graphql
type Author {
  id: Int!
  firstName: String
  lastName: String
  posts: [Post!]!
}
```

`@Field()` 装饰器接受一个可选的类型函数（如 `type => Int`）和一个可选的选项对象。

当 TypeScript 类型系统和 GraphQL 类型系统之间存在歧义时，需要类型函数。具体来说：对于 `string` 和 `boolean` 类型，**不需要** 类型函数；对于 `number` 类型（必须映射到 GraphQL 的 `Int` 或 `Float`），**需要** 类型函数。类型函数只需返回所需的 GraphQL 类型（如本章各种示例所示）。

选项对象可以包含以下任意键值对：

- `nullable`：指定字段是否可为空（在 `@nestjs/graphql` 中，默认字段不可为空）；`boolean`
- `description`：设置字段描述；`string`
- `deprecationReason`：标记字段为已弃用；`string`

例如：

```typescript
@Field({ description: `Book title`, deprecationReason: 'Not useful in v2 schema' })
title: string;
```

> info **提示**：你也可以为整个对象类型添加描述或将其标记为弃用：`@ObjectType({{ '{' }} description: 'Author model' {{ '}' }})`。

当字段是数组时，我们必须在 `Field()` 装饰器的类型函数中手动指示数组类型，如下所示：

```typescript
@Field(type => [Post])
posts: Post[];
```

> info **提示**：使用数组括号表示法（`[ ]`），我们可以指示数组的深度。例如，使用 `[[Int]]` 将表示一个整数矩阵。

要声明数组项（而不是数组本身）可为空，将 `nullable` 属性设置为 `'items'`，如下所示：

```typescript
@Field(type => [Post], { nullable: 'items' })
posts: Post[];
```

> info **提示**：如果数组及其项都可为空，则将 `nullable` 设置为 `'itemsAndList'`。

现在 `Author` 对象类型已经创建，让我们定义 `Post` 对象类型。

```typescript
@@filename(posts/models/post.model)
import { Field, Int, ObjectType } from '@nestjs/graphql';

@ObjectType()
export class Post {
  @Field(type => Int)
  id: number;

  @Field()
  title: string;

  @Field(type => Int, { nullable: true })
  votes?: number;
}
```

`Post` 对象类型将在 SDL 中生成以下部分的 GraphQL schema：

```graphql
type Post {
  id: Int!
  title: String!
  votes: Int
}
```

#### 代码优先解析器（Code first resolver）

至此，我们已经定义了可以在数据图中存在的对象（类型定义），但客户端尚无法与这些对象交互。为了解决这个问题，我们需要创建一个解析器类。在代码优先方法中，解析器类既定义了解析器函数，也生成了 **Query 类型**。这在我们逐步讲解下面的示例时会变得清晰：

```typescript
@@filename(authors/authors.resolver)
@Resolver(() => Author)
export class AuthorsResolver {
  constructor(
    private authorsService: AuthorsService,
    private postsService: PostsService,
  ) {}

  @Query(() => Author)
  async author(@Args('id', { type: () => Int }) id: number) {
    return this.authorsService.findOneById(id);
  }

  @ResolveField()
  async posts(@Parent() author: Author) {
    const { id } = author;
    return this.postsService.findAll({ authorId: id });
  }
}
```

> info **提示**：所有装饰器（如 `@Resolver`、`@ResolveField`、`@Args` 等）都从 `@nestjs/graphql` 包导出。

你可以定义多个解析器类。Nest 将在运行时合并这些类。有关代码组织的更多信息，请参见下面的 [模块](/graphql/resolvers#module) 部分。

> warning **注意**：`AuthorsService` 和 `PostsService` 类中的逻辑可以尽可能简单或复杂。本示例的主要目的是展示如何构建解析器以及它们如何与其他提供者交互。

在上面的示例中，我们创建了 `AuthorsResolver`，它定义了一个查询解析器函数和一个字段解析器函数。要创建解析器，我们创建一个类，将解析器函数作为方法，并用 `@Resolver()` 装饰器注解该类。

在这个示例中，我们定义了一个查询处理程序，根据请求中发送的 `id` 获取作者对象。要指定该方法是查询处理程序，请使用 `@Query()` 装饰器。

传递给 `@Resolver()` 装饰器的参数是可选的，但在我们的图变得非平凡时会派上用场。它用于提供一个父对象，字段解析器函数在遍历对象图时会使用它。

在这个示例中，由于该类包含一个字段解析器函数（用于 `Author` 对象类型的 `posts` 属性），我们必须为 `@Resolver()` 装饰器提供一个值，以指示该类中定义的所有字段解析器的父类型（即对应的 `ObjectType` 类名）。正如示例中所清楚的那样，当编写字段解析器函数时，有必要访问父对象（即正在解析的字段所属的对象）。在这个示例中，我们使用一个字段解析器填充作者的 `posts` 数组，该字段解析器调用一个服务，该服务以作者的 `id` 作为参数。因此，需要在 `@Resolver()` 装饰器中标识父对象。注意，对应的 `@Parent()` 方法参数装饰器用于在字段解析器中提取对该父对象的引用。

我们可以在该类中定义多个 `@Query()` 解析器函数（也可以在其他解析器类中定义），它们将在生成的 SDL 中聚合到一个 **Query 类型** 定义中，并在解析器映射中生成相应的条目。这允许你将查询定义在使用它们的模型和服务附近，并在模块中保持良好的组织。

> info **提示**：Nest CLI 提供了一个生成器（schematic），它可以自动生成 **所有样板代码**，帮助我们避免做所有这些工作，使开发体验更加简单。阅读有关此功能的更多信息 [这里](/recipes/crud-generator)。

#### 查询类型名称（Query type names）

在上面的示例中，`@Query()` 装饰器根据方法名生成 GraphQL schema 查询类型名称。例如，考虑上面示例中的以下构造：

```typescript
@Query(() => Author)
async author(@Args('id', { type: () => Int }) id: number) {
  return this.authorsService.findOneById(id);
}
```

这将在我们的 schema 中为 `author` 查询生成以下条目（查询类型使用与方法名相同的名称）：

```graphql
type Query {
  author(id: Int!): Author
}
```

> info **提示**：了解更多关于 GraphQL 查询的内容 [这里](https://graphql.org/learn/queries/)。

按照惯例，我们倾向于解耦这些名称；例如，我们倾向于在查询处理程序方法中使用类似 `getAuthor()` 的名称，但仍然在查询类型中使用 `author`。字段解析器也是如此。我们可以通过将映射名称作为 `@Query()` 和 `@ResolveField()` 装饰器的参数传递来轻松实现这一点，如下所示：

```typescript
@@filename(authors/authors.resolver)
@Resolver(() => Author)
export class AuthorsResolver {
  constructor(
    private authorsService: AuthorsService,
    private postsService: PostsService,
  ) {}

  @Query(() => Author, { name: 'author' })
  async getAuthor(@Args('id', { type: () => Int }) id: number) {
    return this.authorsService.findOneById(id);
  }

  @ResolveField('posts', () => [Post])
  async getPosts(@Parent() author: Author) {
    const { id } = author;
    return this.postsService.findAll({ authorId: id });
  }
}
```

上面的 `getAuthor` 处理方法将导致生成以下部分的 GraphQL schema（在 SDL 中）：

```graphql
type Query {
  author(id: Int!): Author
}
```

#### 查询装饰器选项（Query decorator options）

`@Query()` 装饰器的选项对象（我们在上面传递了 `{{ '{' }}name: 'author'{{ '}' }}`）接受多个键值对：

- `name`：查询的名称；`string`
- `description`：用于生成 GraphQL schema 文档的描述（例如，在 GraphQL Playground 中）；`string`
- `deprecationReason`：设置查询元数据以显示查询为已弃用（例如，在 GraphQL Playground 中）；`string`
- `nullable`：查询是否可以返回空数据响应；`boolean` 或 `'items'` 或 `'itemsAndList'`（有关 `'items'` 和 `'itemsAndList'` 的详细信息，请参见上文）

#### Args 装饰器选项（Args decorator options）

使用 `@Args()` 装饰器从请求中提取参数以供方法处理程序使用。其工作方式与 [REST 路由参数提取](/controllers#route-parameters) 非常相似。

通常你的 `@Args()` 装饰器会很简单，不需要像上面的 `getAuthor()` 方法那样传递对象参数。例如，如果标识符的类型是字符串，则以下构造就足够了，并且只需从传入的 GraphQL 请求中提取命名字段作为方法参数：

```typescript
@Args('id') id: string
```

在 `getAuthor()` 的情况下，使用的是 `number` 类型，这带来了一个挑战。`number` TypeScript 类型没有提供足够的信息来判断预期的 GraphQL 表示形式（例如，`Int` 还是 `Float`）。因此，我们必须**显式**传递类型引用。我们通过向 `Args()` 装饰器传递第二个参数（包含参数选项）来实现这一点，如下所示：

```typescript
@Query(() => Author, { name: 'author' })
async getAuthor(@Args('id', { type: () => Int }) id: number) {
  return this.authorsService.findOneById(id);
}
```

选项对象允许我们指定以下可选键值对：

- `type`：返回 GraphQL 类型的函数
- `defaultValue`：默认值；`any`
- `description`：描述元数据；`string`
- `deprecationReason`：弃用字段并提供描述原因的元数据；`string`
- `nullable`：字段是否可为空

查询处理方法可以接受多个参数。假设我们想根据 `firstName` 和 `lastName` 获取作者。在这种情况下，我们可以调用 `@Args` 两次：

```typescript
getAuthor(
  @Args('firstName', { nullable: true }) firstName?: string,
  @Args('lastName', { defaultValue: '' }) lastName?: string,
) {}
```

> info **提示**：对于 `firstName`（GraphQL 可为空字段），无需将 `null` 或 `undefined` 等非值类型添加到该字段的类型中。只需注意，在你的解析器中，你需要对这些可能的非值类型进行类型守卫检查，因为 GraphQL 可为空字段将允许这些类型传递到你的解析器中。

#### 专用参数类（Dedicated arguments class）

使用内联 `@Args()` 调用时，像上面的示例代码会变得臃肿。相反，你可以创建一个专用的 `GetAuthorArgs` 参数类，并在处理方法中如下访问它：

```typescript
@Args() args: GetAuthorArgs
```

使用 `@ArgsType()` 创建 `GetAuthorArgs` 类，如下所示：

```typescript
@@filename(authors/dto/get-author.args)
import { MinLength } from 'class-validator';
import { Field, ArgsType } from '@nestjs/graphql';

@ArgsType()
class GetAuthorArgs {
  @Field({ nullable: true })
  firstName?: string;

  @Field({ defaultValue: '' })
  @MinLength(3)
  lastName: string;
}
```

> info **提示**：再次强调，由于 TypeScript 的元数据反射系统的限制，必须使用 `@Field` 装饰器手动指示类型和可选性，或者使用 [CLI 插件](/graphql/cli-plugin)。此外，对于 `firstName`（GraphQL 可为空字段），无需将 `null` 或 `undefined` 等非值类型添加到该字段的类型中。只需注意，在你的解析器中，你需要对这些可能的非值类型进行类型守卫检查，因为 GraphQL 可为空字段将允许这些类型传递到你的解析器中。

这将导致生成以下部分的 GraphQL schema（在 SDL 中）：

```graphql
type Query {
  author(firstName: String, lastName: String = ''): Author
}
```

> info **提示**：请注意，像 `GetAuthorArgs` 这样的参数类与 `ValidationPipe` 配合得很好（阅读 [更多](/techniques/validation)）。

#### 类继承（Class inheritance）

你可以使用标准的 TypeScript 类继承来创建带有通用实用类型功能（字段和字段属性、验证等）的基类，这些功能可以被扩展。例如，你可能有一组始终包含标准 `offset` 和 `limit` 字段的分页相关参数，但还有其他特定类型的索引字段。你可以设置如下所示的类层次结构。

基类 `@ArgsType()`：

```typescript
@ArgsType()
class PaginationArgs {
  @Field(() => Int)
  offset: number = 0;

  @Field(() => Int)
  limit: number = 10;
}
```

特定类型的 `@ArgsType()` 子类：

```typescript
@ArgsType()
class GetAuthorArgs extends PaginationArgs {
  @Field({ nullable: true })
  firstName?: string;

  @Field({ defaultValue: '' })
  @MinLength(3)
  lastName: string;
}
```

同样的方法也可以用于 `@ObjectType()` 对象。在基类中定义通用属性：

```typescript
@ObjectType()
class Character {
  @Field(() => Int)
  id: number;

  @Field()
  name: string;
}
```

在子类中添加特定类型的属性：

```typescript
@ObjectType()
class Warrior extends Character {
  @Field()
  level: number;
}
```

你也可以对解析器使用继承。通过结合继承和 TypeScript 泛型，你可以确保类型安全。例如，要创建一个具有通用 `findAll` 查询的基类，可以使用如下构造：

```typescript
function BaseResolver<T extends Type<unknown>>(classRef: T): any {
  @Resolver({ isAbstract: true })
  abstract class BaseResolverHost {
    @Query(() => [classRef], { name: `findAll${classRef.name}` })
    async findAll(): Promise<T[]> {
      return [];
    }
  }
  return BaseResolverHost;
}
```

请注意以下几点：

- 必须显式返回类型（如上面的 `any`），否则 TypeScript 会抱怨私有类定义的使用。建议：定义接口而不是使用 `any`。
- `Type` 从 `@nestjs/common` 包导入
- `isAbstract: true` 属性表示不应为此类生成 SDL（Schema Definition Language 语句）。注意，你也可以为其他类型设置此属性以抑制 SDL 生成。

以下是生成 `BaseResolver` 具体子类的方法：

```typescript
@Resolver(() => Recipe)
export class RecipesResolver extends BaseResolver(Recipe) {
  constructor(private recipesService: RecipesService) {
    super();
  }
}
```

此构造将生成以下 SDL：

```graphql
type Query {
  findAllRecipe: [Recipe!]!
}
```

#### 泛型（Generics）

我们在上面看到了泛型的一种用法。这个强大的 TypeScript 特性可以用于创建有用的抽象。例如，以下是一个基于 [此文档](https://graphql.org/learn/pagination/#pagination-and-edges) 的游标分页实现示例：

```typescript
import { Field, ObjectType, Int } from '@nestjs/graphql';
import { Type } from '@nestjs/common';

interface IEdgeType<T> {
  cursor: string;
  node: T;
}

export interface IPaginatedType<T> {
  edges: IEdgeType<T>[];
  nodes: T[];
  totalCount: number;
  hasNextPage: boolean;
}

export function Paginated<T>(classRef: Type<T>): Type<IPaginatedType<T>> {
  @ObjectType(`${classRef.name}Edge`)
  abstract class EdgeType {
    @Field(() => String)
    cursor: string;

    @Field(() => classRef)
    node: T;
  }

  @ObjectType({ isAbstract: true })
  abstract class PaginatedType implements IPaginatedType<T> {
    @Field(() => [EdgeType], { nullable: true })
    edges: EdgeType[];

    @Field(() => [classRef], { nullable: true })
    nodes: T[];

    @Field(() => Int)
    totalCount: number;

    @Field()
    hasNextPage: boolean;
  }
  return PaginatedType as Type<IPaginatedType<T>>;
}
```

定义了上面的基类后，我们现在可以轻松创建继承此行为的专用类型。例如：

```typescript
@ObjectType()
class PaginatedAuthor extends Paginated(Author) {}
```

#### Schema 优先（Schema first）

如 [前一章](/graphql/quick-start) 所述，在 schema 优先方法中，我们从手动定义 SDL 中的 schema 类型开始（阅读 [更多](https://graphql.org/learn/schema/#type-language)）。考虑以下 SDL 类型定义。

> info **提示**：为了方便本章内容，我们将所有 SDL 聚合在一个位置（例如，一个 `.graphql` 文件，如下所示）。在实践中，你可能会发现以模块化方式组织代码更为合适。例如，可以为每个领域实体创建单独的 SDL 文件，其中包含代表每个实体的类型定义，以及相关的服务、解析器代码和 Nest 模块定义类，并将它们放在该实体的专用目录中。Nest 将在运行时聚合所有单独的 schema 类型定义。

```graphql
type Author {
  id: Int!
  firstName: String
  lastName: String
  posts: [Post]
}

type Post {
  id: Int!
  title: String!
  votes: Int
}

type Query {
  author(id: Int!): Author
}
```

#### Schema 优先解析器（Schema first resolver）

上面的 schema 暴露了一个单一的查询 - `author(id: Int!): Author`。

> info **提示**：了解更多关于 GraphQL 查询的内容 [这里](https://graphql.org/learn/queries/)。

现在让我们创建一个 `AuthorsResolver` 类来解析作者查询：

```typescript
@@filename(authors/authors.resolver)
@Resolver('Author')
export class AuthorsResolver {
  constructor(
    private authorsService: AuthorsService,
    private postsService: PostsService,
  ) {}

  @Query()
  async author(@Args('id') id: number) {
    return this.authorsService.findOneById(id);
  }

  @ResolveField()
  async posts(@Parent() author) {
    const { id } = author;
    return this.postsService.findAll({ authorId: id });
  }
}
```

> info **提示**：所有装饰器（如 `@Resolver`、`@ResolveField`、`@Args` 等）都从 `@nestjs/graphql` 包导出。

> warning **注意**：`AuthorsService` 和 `PostsService` 类中的逻辑可以尽可能简单或复杂。本示例的主要目的是展示如何构建解析器以及它们如何与其他提供者交互。

`@Resolver()` 装饰器是必需的。它接受一个可选的字符串参数，表示类名。每当类包含 `@ResolveField()` 装饰器时，都需要此类名，以告知 Nest 装饰的方法与父类型相关（在我们的示例中是 `Author` 类型）。或者，而不是在类顶部设置 `@Resolver()`，可以在每个方法上这样做：

```typescript
@Resolver('Author')
@ResolveField()
async posts(@Parent() author) {
  const { id } = author;
  return this.postsService.findAll({ authorId: id });
}
```

在这种情况下（在方法级别使用 `@Resolver()` 装饰器），如果你在类中有多个 `@ResolveField()` 装饰器，则必须为所有这些装饰器添加 `@Resolver()`。这不被认为是最佳实践（因为它会产生额外的开销）。

> info **提示**：传递给 `@Resolver()` 的任何类名参数 **不会** 影响查询（`@Query()` 装饰器）或变更（`@Mutation()` 装饰器）。

> warning **警告**：在 **代码优先** 方法中，**不支持** 在方法级别使用 `@Resolver` 装饰器。

在上面的示例中，`@Query()` 和 `@ResolveField()` 装饰器根据方法名与 GraphQL schema 类型相关联。例如，考虑上面示例中的以下构造：

```typescript
@Query()
async author(@Args('id') id: number) {
  return this.authorsService.findOneById(id);
}
```

这将在我们的 schema 中为 `author` 查询生成以下条目（查询类型使用与方法名相同的名称）：

```graphql
type Query {
  author(id: Int!): Author
}
```

按照惯例，我们倾向于解耦这些名称，使用类似 `getAuthor()` 或 `getPosts()` 的名称作为解析器方法。我们可以通过将映射名称作为装饰器的参数传递来轻松实现这一点，如下所示：

```typescript
@@filename(authors/authors.resolver)
@Resolver('Author')
export class AuthorsResolver {
  constructor(
    private authorsService: AuthorsService,
    private postsService: PostsService,
  ) {}

  @Query('author')
  async getAuthor(@Args('id') id: number) {
    return this.authorsService.findOneById(id);
  }

  @ResolveField('posts')
  async getPosts(@Parent() author) {
    const { id } = author;
    return this.postsService.findAll({ authorId: id });
  }
}
```

> info **提示**：Nest CLI 提供了一个生成器（schematic），它可以自动生成 **所有样板代码**，帮助我们避免做所有这些工作，使开发体验更加简单。阅读有关此功能的更多信息 [这里](/recipes/crud-generator)。

#### 生成类型（Generating types）

假设我们使用 schema 优先方法并启用了类型生成功能（使用 `outputAs: 'class'`，如 [前一章](/graphql/quick-start) 所示），一旦你运行应用程序，它将在你指定的位置（在 `GraphQLModule.forRoot()` 方法中）生成以下文件。例如，在 `src/graphql.ts` 中：

```typescript
@@filename(graphql)
export (class Author {
  id: number;
  firstName?: string;
  lastName?: string;
  posts?: Post[];
})
export class Post {
  id: number;
  title: string;
  votes?: number;
}

export abstract class IQuery {
  abstract author(id: number): Author | Promise<Author>;
}
```

通过生成类（而不是默认的生成接口技术），你可以将声明式验证 **装饰器** 与 schema 优先方法结合使用，这是一种非常有用的技术（阅读 [更多](/techniques/validation)）。例如，你可以在生成的 `CreatePostInput` 类上添加 `class-validator` 装饰器，如下所示，以强制对 `title` 字段进行最小和最大字符串长度限制：

```typescript
import { MinLength, MaxLength } from 'class-validator';

export class CreatePostInput {
  @MinLength(3)
  @MaxLength(50)
  title: string;
}
```

> warning **注意**：要启用输入（和参数）的自动验证，请使用 `ValidationPipe`。阅读更多关于验证的内容 [这里](/techniques/validation)，关于管道的内容 [这里](/pipes)。

但是，如果你直接将装饰器添加到自动生成的文件中，它们将在每次文件生成时被 **覆盖**。相反，创建一个单独的文件并扩展生成的类。

```typescript
import { MinLength, MaxLength } from 'class-validator';
import { Post } from '../../graphql.ts';

export class CreatePostInput extends Post {
  @MinLength(3)
  @MaxLength(50)
  title: string;
}
```

#### GraphQL 参数装饰器（GraphQL argument decorators）

我们可以使用专用装饰器访问标准的 GraphQL 解析器参数。以下是 Nest 装饰器与代表的普通 Apollo 参数的对比。

<table>
  <tbody>
    <tr>
      <td><code>@Root()</code> 和 <code>@Parent()</code></td>
      <td><code>root</code>/<code>parent</code></td>
    </tr>
    <tr>
      <td><code>@Context(param?: string)</code></td>
      <td><code>context</code> / <code>context[param]</code></td>
    </tr>
    <tr>
      <td><code>@Info(param?: string)</code></td>
      <td><code>info</code> / <code>info[param]</code></td>
    </tr>
    <tr>
      <td><code>@Args(param?: string)</code></td>
      <td><code>args</code> / <code>args[param]</code></td>
    </tr>
  </tbody>
</table>

这些参数的含义如下：

- `root`：包含从父字段解析器返回结果的对象，或者在顶级 `Query` 字段的情况下，包含从服务器配置传递的 `rootValue`。
- `context`：由特定查询中的所有解析器共享的对象；通常用于包含每个请求的状态。
- `info`：包含查询执行状态信息的对象。
- `args`：包含查询中传入字段的参数的对象。

<app-banner-devtools></app-banner-devtools>

#### 模块（Module）

完成上述步骤后，我们就声明性地指定了 `GraphQLModule` 生成解析器映射所需的所有信息。`GraphQLModule` 使用反射来内省通过装饰器提供的元数据，并自动将类转换为正确的解析器映射。

你还需要做的唯一一件事是 **提供**（即在某个模块中列为 `provider`）解析器类（`AuthorsResolver`），并在某处导入模块（`AuthorsModule`），以便 Nest 能够使用它。

例如，我们可以在一个 `AuthorsModule` 中完成此操作，该模块还可以提供此上下文中所需的其他服务。确保在某处（例如根模块或由根模块导入的其他模块）导入 `AuthorsModule`。

```typescript
@@filename(authors/authors.module)
@Module({
  imports: [PostsModule],
  providers: [AuthorsService, AuthorsResolver],
})
export class AuthorsModule {}
```

> info **提示**：按所谓的 **领域模型** 组织代码是有帮助的（类似于你在 REST API 中组织入口点的方式）。在这种方法中，将模型（`ObjectType` 类）、解析器和服务保留在代表领域模型的 Nest 模块中。每个模块的这些组件保留在一个文件夹中。当你这样做，并使用 [Nest CLI](/cli/overview) 生成每个元素时，Nest 会自动为你连接所有这些部分（定位文件在适当的文件夹中，生成 `provider` 和 `imports` 数组中的条目等）。