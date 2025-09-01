### 授权

**授权**是指决定用户可以执行哪些操作的过程。例如，管理员用户可以创建、编辑和删除文章，而非管理员用户仅被授权阅读文章。

授权与认证是正交且独立的。然而，授权需要依赖认证机制。

处理授权有许多不同的方法和策略。具体项目采用哪种方法取决于其特定的应用需求。本章介绍了几种可适应不同需求的授权方法。

#### 基础 RBAC 实现

基于角色的访问控制（**RBAC**）是一种围绕角色和权限定义的策略中立的访问控制机制。本节将演示如何使用 Nest [守卫](/guards)实现一个非常基础的 RBAC 机制。

首先，我们创建一个表示系统中角色的 `Role` 枚举：
```typescript
@@filename(role.enum)
export enum Role {
  User = 'user',
  Admin = 'admin',
}
```
> info **提示** 在更复杂的系统中，你可以将角色存储在数据库中，或从外部身份验证提供程序中拉取。

有了这些准备，我们就可以创建一个 `@Roles()` 装饰器。该装饰器允许指定访问特定资源所需的权限角色。
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
现在我们已经有了一个自定义的 `@Roles()` 装饰器，可以用它来装饰任何路由处理函数。
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
最后，我们创建一个 `RolesGuard` 类，该类将当前用户分配的角色与正在处理的当前路由实际所需的角色进行比较。为了访问路由的角色（自定义元数据），我们将使用 `Reflector` 辅助类，该类由框架开箱即用地提供，并从 `@nestjs/core` 包中导出。
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
> info **提示** 有关在上下文敏感的方式中使用 `Reflector` 的更多细节，请参阅执行上下文章节的[反射与元数据](/fundamentals/execution-context#reflection-and-metadata)部分。

> warning **注意** 此示例之所以命名为“**basic**”，是因为我们仅在路由处理程序级别检查角色的存在。在实际应用中，您可能会拥有涉及多个操作的端点/处理程序，而每个操作都需要一组特定的权限。在这种情况下，您必须在业务逻辑中的某处提供一种角色检查机制，这使得维护变得稍微困难，因为没有一个集中位置将权限与特定操作关联起来。

在此示例中，我们假设 `request.user` 包含用户实例以及允许的角色（在 `roles` 属性下）。在您的应用中，您可能会在自定义的**身份验证守卫**中进行此关联——有关更多详细信息，请参阅[身份验证](/security/authentication)章节。

为了确保此示例正常工作，您的 `User` 类必须如下所示：
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
当权限不足的用户请求某个端点时，Nest 会自动返回以下响应：
```typescript
{
  "statusCode": 403,
  "message": "禁止访问的资源",
  "error": "禁止访问"
}
```
> info **提示** 如果你想返回不同的错误响应，应该抛出自定义的特定异常，而不是返回布尔值。

<app-banner-courses-auth></app-banner-courses-auth>

#### 基于声明的授权

当身份被创建时，它可能会被分配一个或多个由可信方颁发的声明。声明是一个“名称-值”对，表示主体**可以做什么**，而不是主体**是什么**。

要在 Nest 中实现基于声明的授权，你可以按照我们在 [RBAC](/security/authorization#basic-rbac-implementation) 一节中展示的相同步骤进行，但有一个显著区别：你需要比较的是**权限**而不是特定角色。每个用户都会被分配一组权限；同样，每个资源/端点都会定义访问它们所需的权限（例如，通过一个专用的 `@RequirePermissions()` 装饰器）。
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
> info **提示** 在上面的示例中，`Permission`（与我们在 RBAC 一节中展示的 `Role` 类似）是一个 TypeScript 枚举，它包含了系统中所有可用的权限。

#### 集成 CASL

[CASL](https://casl.js.org/) 是一个同构的授权库，用于限制给定客户端可以访问哪些资源。它被设计为可逐步采用，并且可以简单地在基于声明的简单授权与功能完整的基于主体和属性的授权之间轻松扩展。

首先，安装 `@casl/ability` 包：
```bash
$ npm i @casl/ability
```
> info **提示** 在本示例中，我们选择了 CASL，但你也可以根据偏好和项目需求使用其他库，例如 `accesscontrol` 或 `acl`。

安装完成后，为了说明 CASL 的机制，我们将定义两个实体类：`User` 和 `Article`。
```typescript
class User {
  id: number;
  isAdmin: boolean;
}
```
`User` 类包含两个属性：`id`，即唯一的用户标识符；`isAdmin`，用于指示用户是否具有管理员权限。
```typescript
class Article {
  id: number;
  isPublished: boolean;
  authorId: number;
}
```
`Article` 类有三个属性，分别是 `id`、`isPublished` 和 `authorId`。`id` 是文章的唯一标识符，`isPublished` 表示文章是否已发布，`authorId` 是撰写该文章的用户的 ID。

现在让我们回顾并完善此示例的需求：

- 管理员可以管理（创建/读取/更新/删除）所有实体
- 用户对所有内容具有只读访问权限
- 用户可以更新自己的文章（`article.authorId === userId`）
- 已发布的文章不能被删除（`article.isPublished === true`）

考虑到这些需求，我们可以首先创建一个 `Action` 枚举，表示用户可以对实体执行的所有可能操作：
```typescript
export enum Action {
  Manage = '管理',
  Create = '创建',
  Read = '读取',
  Update = '更新',
  Delete = '删除',
}
```
> warning **注意** `manage` 是 CASL 中的一个特殊关键字，表示“任何操作”。

为了封装 CASL 库，我们现在来生成 `CaslModule` 和 `CaslAbilityFactory`。
```bash
$ nest g module casl
$ nest g class casl/casl-ability.factory
```
有了这些准备，我们就可以在 `CaslAbilityFactory` 上定义 `createForUser()` 方法。该方法会为指定用户创建 `Ability` 对象：
```typescript
type Subjects = InferSubjects<typeof Article | typeof User> | 'all';

export type AppAbility = MongoAbility<[Action, Subjects]>;

@Injectable()
export class CaslAbilityFactory {
  createForUser(user: User) {
    const { can, cannot, build } = new AbilityBuilder(createMongoAbility);

    if (user.isAdmin) {
      can(Action.Manage, 'all'); // 对所有资源拥有读写权限
    } else {
      can(Action.Read, 'all'); // 对所有资源拥有只读权限
    }

    can(Action.Update, Article, { authorId: user.id });
    cannot(Action.Delete, Article, { isPublished: true });

    return build({
      // 详情请阅读 https://casl.js.org/v6/en/guide/subject-type-detection#use-classes-as-subject-types
      detectSubjectType: (item) =>
        item.constructor as ExtractSubjectType<Subjects>,
    });
  }
}
```
> warning **注意** `all` 是 CASL 中的特殊关键字，表示“任意主体”。

> info **提示** 自 CASL v6 起，`MongoAbility` 成为默认的 Ability 类，取代了旧的 `Ability`，以便更好地支持基于条件的权限，并使用类似 MongoDB 的语法。尽管名字如此，它并不依赖 MongoDB——只需将对象与用类 Mongo 语法编写的条件进行比较，即可适用于任何类型的数据。

> info **提示** `MongoAbility`、`AbilityBuilder`、`AbilityClass` 和 `ExtractSubjectType` 这些类都从 `@casl/ability` 包导出。

> info **提示** `detectSubjectType` 选项让 CASL 知道如何从对象中获取主体类型。更多信息请阅读 [CASL 文档](https://casl.js.org/v6/en/guide/subject-type-detection#use-classes-as-subject-types)。

在上面的示例中，我们使用 `AbilityBuilder` 类创建了 `MongoAbility` 实例。正如你可能已经猜到的，`can` 和 `cannot` 接受相同的参数，但含义相反：`can` 允许对指定主体执行某个动作，而 `cannot` 禁止执行。两者最多都可接受 4 个参数。要了解更多关于这些函数的信息，请访问官方 [CASL 文档](https://casl.js.org/v6/en/guide/intro)。

最后，请确保在 `CaslModule` 模块定义的 `providers` 和 `exports` 数组中添加 `CaslAbilityFactory`：
```typescript
import { Module } from '@nestjs/common';
import { CaslAbilityFactory } from './casl-ability.factory';

@Module({
  providers: [CaslAbilityFactory],
  exports: [CaslAbilityFactory],
})
export class CaslModule {}
```
有了这一步，只要宿主上下文导入了 `CaslModule`，我们就可以通过标准的构造函数注入方式，将 `CaslAbilityFactory` 注入到任何类中：
```typescript
constructor(private caslAbilityFactory: CaslAbilityFactory) {}
```
然后按如下方式在类中使用它。
```typescript
const ability = this.caslAbilityFactory.createForUser(user);
if (ability.can(Action.Read, 'all')) {
  // “user” 拥有对所有内容的读取权限
}
```
> info **提示** 在官方 [CASL 文档](https://casl.js.org/v6/en/guide/intro) 中了解更多关于 `MongoAbility` 类的信息。

例如，假设我们有一个不是管理员的用户。在这种情况下，该用户应该能够读取文章，但禁止创建新文章或删除现有文章。
```typescript
const user = new User();
user.isAdmin = false;

const ability = this.caslAbilityFactory.createForUser(user);
ability.can(Action.Read, Article); // true
ability.can(Action.Delete, Article); // false
ability.can(Action.Create, Article); // false
```
> info **提示** 尽管 `MongoAbility` 和 `AbilityBuilder` 两个类都提供了 `can` 和 `cannot` 方法，但它们的用途不同，接受的参数也略有差异。

同时，正如我们在需求中所指定的，用户应该能够更新自己的文章：
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
如你所见，`MongoAbility` 实例让我们能够以非常易读的方式检查权限。同样地，`AbilityBuilder` 也允许我们以类似的方式定义权限（并指定各种条件）。要查看更多示例，请访问官方文档。

#### 进阶：实现 `PoliciesGuard`

本节将演示如何构建一个更复杂的守卫，它会检查用户是否满足在方法级别配置的特定**授权策略**（你也可以扩展它以支持类级别的策略配置）。在此示例中，我们将仅出于说明目的使用 CASL 包，但并不要求你一定要使用该库。我们还会用到上一节创建的 `CaslAbilityFactory` 提供者。

首先，让我们明确需求。目标是提供一种机制，允许在每个路由处理器上指定策略检查。我们将同时支持对象和函数（用于更简单的检查，以及偏好函数式代码的人）。

我们先从定义策略处理器的接口开始：
```typescript
import { AppAbility } from '../casl/casl-ability.factory';

interface IPolicyHandler {
  handle(ability: AppAbility): boolean;
}

type PolicyHandlerCallback = (ability: AppAbility) => boolean;

export type PolicyHandler = IPolicyHandler | PolicyHandlerCallback;
```
如上所述，我们提供了两种定义策略处理器的方式：对象（实现 `IPolicyHandler` 接口的类的实例）和函数（符合 `PolicyHandlerCallback` 类型）。

在此基础上，我们可以创建一个 `@CheckPolicies()` 装饰器。该装饰器允许指定访问特定资源时必须满足的策略。
```typescript
export const CHECK_POLICIES_KEY = 'check_policy';
export const CheckPolicies = (...handlers: PolicyHandler[]) =>
  SetMetadata(CHECK_POLICIES_KEY, handlers);
```
现在让我们创建一个 `PoliciesGuard`，它将提取并执行绑定到路由处理器的所有策略处理器。
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
> info **提示** 在此示例中，我们假设 `request.user` 包含用户实例。在你的应用中，你可能会在自定义的 **身份验证守卫** 中建立该关联——详见 [身份验证](/security/authentication) 章节。

让我们分解这个示例。`policyHandlers` 是通过 `@CheckPolicies()` 装饰器分配给该方法的处理器数组。接下来，我们使用 `CaslAbilityFactory#create` 方法构建 `Ability` 对象，从而验证用户是否拥有执行特定操作的足够权限。我们将此对象传递给策略处理器，该处理器可以是函数，也可以是实现了 `IPolicyHandler` 接口的类实例，该接口暴露返回布尔值的 `handle()` 方法。最后，我们使用 `Array#every` 方法确保每个处理器都返回 `true` 值。

最后，为了测试此守卫，将其绑定到任意路由处理器，并注册一个内联策略处理器（函数式方法），如下所示：
```typescript
@Get()
@UseGuards(PoliciesGuard)
@CheckPolicies((ability: AppAbility) => ability.can(Action.Read, Article))
findAll() {
  return this.articlesService.findAll();
}
```
或者，我们可以定义一个实现 `IPolicyHandler` 接口的类：
```typescript
export class ReadArticlePolicyHandler implements IPolicyHandler {
  handle(ability: AppAbility) {
    return ability.can(Action.Read, Article);
  }
}
```
并按如下方式使用：
```typescript
@Get()
@UseGuards(PoliciesGuard)
@CheckPolicies(new ReadArticlePolicyHandler())
findAll() {
  return this.articlesService.findAll();
}
```
> warning **注意** 由于我们必须使用 `new` 关键字就地实例化策略处理器，`ReadArticlePolicyHandler` 类无法使用依赖注入。这可以通过 `ModuleRef#get` 方法解决（[详见此处](/fundamentals/module-ref)）。简而言之，不要在 `@CheckPolicies()` 装饰器中注册函数和实例，而是允许传入一个 `Type<IPolicyHandler>`。然后，在你的守卫中，你可以使用类型引用获取实例：`moduleRef.get(YOUR_HANDLER_TYPE)`，甚至可以使用 `ModuleRef#create` 方法动态实例化它。