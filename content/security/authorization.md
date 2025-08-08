### 授权（Authorization）

**授权** 是指确定用户能够做什么的过程。例如，管理员用户可以创建、编辑和删除帖子，而非管理员用户只能被授权阅读帖子。

授权与身份验证（authentication）是正交且独立的。然而，授权需要依赖某种身份验证机制。

处理授权的方法和策略有很多不同种类。具体项目采用哪种方法取决于其特定的应用需求。本章节将介绍几种可适用于不同需求的授权方法。

#### 基础的 RBAC 实现

基于角色的访问控制（**RBAC**）是一种围绕角色和权限定义的、策略中立的访问控制机制。在本节中，我们将演示如何使用 Nest 的 [守卫（guards）](/guards) 实现一个非常基础的 RBAC 机制。

首先，我们创建一个 `Role` 枚举来表示系统中的角色：

```typescript
@@filename(role.enum)
export enum Role {
  User = 'user',
  Admin = 'admin',
}
```

> info **提示** 在更复杂的系统中，你可能会将角色存储在数据库中，或从外部身份验证提供程序中获取。

有了这个定义后，我们可以创建一个 `@Roles()` 装饰器。该装饰器用于指定访问特定资源所需的权限角色。

```typescript
@@filename(roles.decorator)
import { SetMetadata } from '@nestjs/common';
import { Role } from '../enums/role.enum';

export const ROLES_KEY = 'roles';
export const Roles = (...roles: Role[]) => SetMetadata(ROLES_KEY, roles);
@@switch
import { SetMetadata } from '@nestjs/common';

export const ROLES_KEY = 'roles';
export const Roles = (...roles) => SetMetadata(ROLES_KEY, roles);
```

现在我们有了自定义的 `@Roles()` 装饰器，可以将其用于装饰任意路由处理器。

```typescript
@@filename(cats.controller)
@Post()
@Roles(Role.Admin)
create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
@@switch
@Post()
@Roles(Role.Admin)
@Bind(Body())
create(createCatDto) {
  this.catsService.create(createCatDto);
}
```

最后，我们创建一个 `RolesGuard` 类，它将当前用户分配的角色与当前路由所需的权限角色进行比较。为了访问路由的角色（自定义元数据），我们将使用 `Reflector` 辅助类，该类由框架提供，并从 `@nestjs/core` 包中导出。

```typescript
@@filename(roles.guard)
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Reflector } from '@nestjs/core';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.getAllAndOverride<Role[]>(ROLES_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (!requiredRoles) {
      return true;
    }
    const { user } = context.switchToHttp().getRequest();
    return requiredRoles.some((role) => user.roles?.includes(role));
  }
}
@@switch
import { Injectable, Dependencies } from '@nestjs/common';
import { Reflector } from '@nestjs/core';

@Injectable()
@Dependencies(Reflector)
export class RolesGuard {
  constructor(reflector) {
    this.reflector = reflector;
  }

  canActivate(context) {
    const requiredRoles = this.reflector.getAllAndOverride(ROLES_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (!requiredRoles) {
      return true;
    }
    const { user } = context.switchToHttp().getRequest();
    return requiredRoles.some((role) => user.roles.includes(role));
  }
}
```

> info **提示** 有关在上下文敏感的方式中使用 `Reflector` 的更多细节，请参阅执行上下文章节中的 [反射与元数据](/fundamentals/execution-context#reflection-and-metadata) 部分。

> warning **注意** 此示例被称为“**基础**”是因为我们仅检查了路由处理器级别的角色。在实际应用中，你可能有一些端点/处理器涉及多个操作，每个操作都需要一组特定的权限。在这种情况下，你需要在业务逻辑中的某处提供一种机制来检查角色，这会稍微增加维护难度，因为将没有一个集中化的机制来将权限与特定操作关联起来。

在本示例中，我们假设 `request.user` 包含用户实例及其允许的角色（在 `roles` 属性下）。在你的应用中，你可能应在自定义的 **身份验证守卫** 中完成该关联 —— 详见 [身份验证](/security/authentication) 章节。

为确保此示例正常运行，你的 `User` 类应如下所示：

```typescript
class User {
  // ...其他属性
  roles: Role[];
}
```

最后，确保注册 `RolesGuard`，例如在控制器级别或全局注册：

```typescript
providers: [
  {
    provide: APP_GUARD,
    useClass: RolesGuard,
  },
],
```

当一个权限不足的用户请求某个端点时，Nest 会自动返回以下响应：

```json
{
  "statusCode": 403,
  "message": "Forbidden resource",
  "error": "Forbidden"
}
```

> info **提示** 如果你想返回不同的错误响应，可以抛出自定义异常，而不是返回布尔值。

<app-banner-courses-auth></app-banner-courses-auth>

#### 基于声明的授权（Claims-based authorization）

当创建一个身份时，可能会被分配一个或多个由可信方签发的声明（claims）。声明是一个名值对，表示主体能做什么，而不是主体是谁。

要在 Nest 中实现基于声明的授权，你可以按照我们在 [RBAC](/security/authorization#basic-rbac-implementation) 小节中展示的相同步骤操作，唯一的区别是：不是检查特定角色，而是比较 **权限（permissions）**。每个用户都有一组分配的权限。同样，每个资源/端点也应定义访问它们所需的权限（例如，通过专门的 `@RequirePermissions()` 装饰器）。

```typescript
@@filename(cats.controller)
@Post()
@RequirePermissions(Permission.CREATE_CAT)
create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
@@switch
@Post()
@RequirePermissions(Permission.CREATE_CAT)
@Bind(Body())
create(createCatDto) {
  this.catsService.create(createCatDto);
}
```

> info **提示** 在上面的示例中，`Permission`（类似于我们在 RBAC 小节中展示的 `Role`）是一个包含系统中所有可用权限的 TypeScript 枚举。

#### 集成 CASL

[CASL](https://casl.js.org/) 是一个同构的授权库，用于限制客户端可以访问的资源。它设计为可逐步采用，可以在简单的声明式授权和完整的基于主体和属性的授权之间轻松扩展。

首先，安装 `@casl/ability` 包：

```bash
$ npm i @casl/ability
```

> info **提示** 在本示例中，我们选择了 CASL，但你也可以根据自己的偏好和项目需求使用其他库，例如 `accesscontrol` 或 `acl`。

安装完成后，为了演示 CASL 的工作机制，我们将定义两个实体类：`User` 和 `Article`。

```typescript
class User {
  id: number;
  isAdmin: boolean;
}
```

`User` 类包含两个属性：`id`（唯一用户标识符）和 `isAdmin`（表示用户是否具有管理员权限）。

```typescript
class Article {
  id: number;
  isPublished: boolean;
  authorId: number;
}
```

`Article` 类包含三个属性：分别是 `id`、`isPublished` 和 `authorId`。其中 `id` 是文章的唯一标识符，`isPublished` 表示文章是否已发布，而 `authorId` 是撰写该文章的用户 ID。

现在我们回顾并细化本示例的需求：

- 管理员可以管理（创建/读取/更新/删除）所有实体
- 普通用户只能读取所有内容
- 用户可以更新自己撰写过的文章（`article.authorId === userId`）
- 已发布的文章不能被删除（`article.isPublished === true`）

基于此，我们可以先创建一个 `Action` 枚举，表示用户可以对实体执行的所有操作：

```typescript
export enum Action {
  Manage = 'manage',
  Create = 'create',
  Read = 'read',
  Update = 'update',
  Delete = 'delete',
}
```

> warning **注意** `manage` 是 CASL 中的一个特殊关键字，表示“任何”操作。

为了封装 CASL 库，我们现在生成 `CaslModule` 和 `CaslAbilityFactory`。

```bash
$ nest g module casl
$ nest g class casl/casl-ability.factory
```

完成后，我们可以在 `CaslAbilityFactory` 上定义 `createForUser()` 方法。该方法将为给定用户创建 `Ability` 对象：

```typescript
type Subjects = InferSubjects<typeof Article | typeof User> | 'all';

export type AppAbility = MongoAbility<[Action, Subjects]>;

@Injectable()
export class CaslAbilityFactory {
  createForUser(user: User) {
    const { can, cannot, build } = new AbilityBuilder(createMongoAbility);

    if (user.isAdmin) {
      can(Action.Manage, 'all'); // 拥有所有内容的读写权限
    } else {
      can(Action.Read, 'all'); // 拥有所有内容的只读权限
    }

    can(Action.Update, Article, { authorId: user.id });
    cannot(Action.Delete, Article, { isPublished: true });

    return build({
      // 详情请参考：https://casl.js.org/v6/en/guide/subject-type-detection#use-classes-as-subject-types
      detectSubjectType: (item) =>
        item.constructor as ExtractSubjectType<Subjects>,
    });
  }
}
```

> warning **注意** `all` 是 CASL 中的一个特殊关键字，表示“任何主题”。

> info **提示** 从 CASL v6 开始，`MongoAbility` 成为默认的权限类，取代了旧版的 `Ability`，以更好地支持使用类似 MongoDB 语法的条件权限。尽管名称如此，它并不绑定于 MongoDB —— 它可以通过简单地比较对象与使用类似 MongoDB 的语法编写的条件来处理任何类型的数据。

> info **提示** `MongoAbility`、`AbilityBuilder`、`AbilityClass` 和 `ExtractSubjectType` 类均从 `@casl/ability` 包中导出。

> info **提示** `detectSubjectType` 选项让 CASL 理解如何从对象中提取主题类型。更多详情请参阅 [CASL 文档](https://casl.js.org/v6/en/guide/subject-type-detection#use-classes-as-subject-types)。

在上述示例中，我们使用 `AbilityBuilder` 类创建了一个 `MongoAbility` 实例。正如你可能猜到的那样，`can` 和 `cannot` 接受相同的参数但具有不同的含义：`can` 表示允许对指定主题执行某个操作，而 `cannot` 表示禁止。两者最多可接受 4 个参数。有关这些函数的更多信息，请参阅官方 [CASL 文档](https://casl.js.org/v6/en/guide/intro)。

最后，确保将 `CaslAbilityFactory` 添加到 `CaslModule` 模块定义中的 `providers` 和 `exports` 数组中：

```typescript
import { Module } from '@nestjs/common';
import { CaslAbilityFactory } from './casl-ability.factory';

@Module({
  providers: [CaslAbilityFactory],
  exports: [CaslAbilityFactory],
})
export class CaslModule {}
```

完成后，只要导入了 `CaslModule`，就可以通过标准的构造函数注入方式将 `CaslAbilityFactory` 注入到任意类中：

```typescript
constructor(private caslAbilityFactory: CaslAbilityFactory) {}
```

然后可以如下方式使用它：

```typescript
const ability = this.caslAbilityFactory.createForUser(user);
if (ability.can(Action.Read, 'all')) {
  // "user" 可以读取所有内容
}
```

> info **提示** 有关 `MongoAbility` 类的更多信息，请参阅官方 [CASL 文档](https://casl.js.org/v6/en/guide/intro)。

例如，假设我们有一个非管理员用户。在这种情况下，用户应该能够读取文章，但不能创建新文章或删除现有文章：

```typescript
const user = new User();
user.isAdmin = false;

const ability = this.caslAbilityFactory.createForUser(user);
ability.can(Action.Read, Article); // true
ability.can(Action.Delete, Article); // false
ability.can(Action.Create, Article); // false
```

> info **提示** 虽然 `MongoAbility` 和 `AbilityBuilder` 类都提供了 `can` 和 `cannot` 方法，但它们的目的不同，且接受的参数略有不同。

此外，正如我们在需求中指定的那样，用户应该能够更新自己的文章：

```typescript
const user = new User();
user.id = 1;

const article = new Article();
article.authorId = user.id;

const ability = this.caslAbilityFactory.createForUser(user);
ability.can(Action.Update, article); // true

article.authorId = 2;
ability.can(Action.Update, article); // false
```

如你所见，`MongoAbility` 实例允许我们以相当可读的方式检查权限。同样地，`AbilityBuilder` 也允许我们以类似的方式定义权限（并指定各种条件）。如需更多示例，请参阅官方文档。

#### 高级用法：实现一个 `PoliciesGuard`

在本节中，我们将演示如何构建一个更高级的守卫，它会检查用户是否满足特定的 **授权策略**，这些策略可以在方法级别进行配置（你也可以扩展它以支持类级别配置）。在本示例中，我们将使用 CASL 包作为演示工具，但并非必须使用此库。此外，我们将使用在上一节中创建的 `CaslAbilityFactory` 提供者。

首先，我们明确需求。目标是提供一种机制，允许为每个路由处理器指定策略检查。我们将支持对象和函数两种方式（用于更简单的检查或偏好函数式风格代码的开发者）。

首先定义策略处理器的接口：

```typescript
import { AppAbility } from '../casl/casl-ability.factory';

interface IPolicyHandler {
  handle(ability: AppAbility): boolean;
}

type PolicyHandlerCallback = (ability: AppAbility) => boolean;

export type PolicyHandler = IPolicyHandler | PolicyHandlerCallback;
```

如上所述，我们提供了两种定义策略处理器的方式：一种是实现了 `IPolicyHandler` 接口的类实例，另一种是符合 `PolicyHandlerCallback` 类型的函数。

有了这些定义后，我们可以创建一个 `@CheckPolicies()` 装饰器。该装饰器用于指定访问特定资源所需满足的策略。

```typescript
export const CHECK_POLICIES_KEY = 'check_policy';
export const CheckPolicies = (...handlers: PolicyHandler[]) =>
  SetMetadata(CHECK_POLICIES_KEY, handlers);
```

现在我们创建一个 `PoliciesGuard`，它将提取并执行绑定到路由处理器的所有策略处理器。

```typescript
@Injectable()
export class PoliciesGuard implements CanActivate {
  constructor(
    private reflector: Reflector,
    private caslAbilityFactory: CaslAbilityFactory,
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const policyHandlers =
      this.reflector.get<PolicyHandler[]>(
        CHECK_POLICIES_KEY,
        context.getHandler(),
      ) || [];

    const { user } = context.switchToHttp().getRequest();
    const ability = this.caslAbilityFactory.createForUser(user);

    return policyHandlers.every((handler) =>
      this.execPolicyHandler(handler, ability),
    );
  }

  private execPolicyHandler(handler: PolicyHandler, ability: AppAbility) {
    if (typeof handler === 'function') {
      return handler(ability);
    }
    return handler.handle(ability);
  }
}
```

> info **提示** 在此示例中，我们假设 `request.user` 包含用户实例。在你的应用中，你可能应在自定义的 **身份验证守卫** 中完成该关联 —— 详见 [身份验证](/security/authentication) 章节。

让我们逐步解释这个示例。`policyHandlers` 是通过 `@CheckPolicies()` 装饰器分配给方法的处理器数组。接着，我们使用 `CaslAbilityFactory#create` 方法构造 `Ability` 对象，用于验证用户是否有足够的权限执行特定操作。我们将此对象传递给策略处理器，该处理器可以是函数，也可以是实现了 `IPolicyHandler` 接口并暴露 `handle()` 方法返回布尔值的类实例。最后，我们使用 `Array#every` 方法确保每个处理器都返回 `true`。

最后，要测试这个守卫，请将其绑定到任意路由处理器，并注册一个内联策略处理器（函数式方法），如下所示：

```typescript
@Get()
@UseGuards(PoliciesGuard)
@CheckPolicies((ability: AppAbility) => ability.can(Action.Read, Article))
findAll() {
  return this.articlesService.findAll();
}
```

或者，我们也可以定义一个实现了 `IPolicyHandler` 接口的类：

```typescript
export class ReadArticlePolicyHandler implements IPolicyHandler {
  handle(ability: AppAbility) {
    return ability.can(Action.Read, Article);
  }
}
```

并如下使用它：

```typescript
@Get()
@UseGuards(PoliciesGuard)
@CheckPolicies(new ReadArticlePolicyHandler())
findAll() {
  return this.articlesService.findAll();
}
```

> warning **注意** 由于我们必须使用 `new` 关键字在原地实例化策略处理器，因此 `ReadArticlePolicyHandler` 类无法使用依赖注入。可以通过 `ModuleRef#get` 方法解决这个问题（详见 [此处](/fundamentals/module-ref)）。基本上，你应允许通过 `@CheckPolicies()` 装饰器传入 `Type<IPolicyHandler>` 类型，而不是注册函数和实例。然后，在守卫内部，你可以使用类型引用通过 `moduleRef.get(YOUR_HANDLER_TYPE)` 获取实例，甚至可以使用 `ModuleRef#create` 方法动态实例化它。