### Passport（身份验证）

[Passport](https://github.com/jaredhanson/passport) 是最受欢迎的 Node.js 身份验证库，被社区广泛熟知，并成功用于许多生产环境应用中。使用 `@nestjs/passport` 模块可以轻松地将此库集成到 **Nest** 应用中。从高层次来看，Passport 会执行以下一系列步骤：

- 通过验证用户的“凭证”（如用户名/密码、JSON Web Token ([JWT](https://jwt.io/)) 或身份提供商的身份令牌）来对用户进行身份验证
- 管理已认证状态（通过签发可移植令牌，如 JWT，或创建 [Express 会话](https://github.com/expressjs/session)）
- 将认证用户的信息附加到 `Request` 对象上，以便在路由处理程序中进一步使用

Passport 拥有丰富的 [策略](http://www.passportjs.org/) 生态系统，这些策略实现了各种身份验证机制。虽然概念上很简单，但你可以选择的 Passport 策略种类繁多且变化多样。Passport 将这些不同的步骤抽象成一个标准模式，而 `@nestjs/passport` 模块则将这些模式封装并标准化为熟悉的 Nest 构造。

在本章中，我们将使用这些强大且灵活的模块，为 RESTful API 服务器实现一个完整的端到端身份验证解决方案。你可以使用此处描述的概念来实现任何 Passport 策略，以自定义你的身份验证方案。你也可以按照本章中的步骤构建完整的示例。

#### 身份验证需求

让我们明确我们的需求。对于此用例，客户端将首先使用用户名和密码进行身份验证。身份验证成功后，服务器将签发一个 JWT，该 JWT 可以在后续请求中作为 [授权头中的承载令牌](https://tools.ietf.org/html/rfc6750) 发送，以证明身份验证。我们还将创建一个受保护的路由，该路由仅允许携带有效 JWT 的请求访问。

我们将从第一个需求开始：用户身份验证。然后我们将通过签发 JWT 来扩展它。最后，我们将创建一个受保护的路由，检查请求中是否包含有效的 JWT。

首先，我们需要安装所需的包。Passport 提供了一个名为 [passport-local](https://github.com/jaredhanson/passport-local) 的策略，它实现了用户名/密码的身份验证机制，这正好符合我们当前用例的需求。

```bash
$ npm install --save @nestjs/passport passport passport-local
$ npm install --save-dev @types/passport-local
```

> warning **注意** 对于你选择的 **任何** Passport 策略，你始终需要 `@nestjs/passport` 和 `passport` 包。然后，你需要安装实现特定身份验证策略的包（例如 `passport-jwt` 或 `passport-local`）。此外，你还可以安装任何 Passport 策略的类型定义，如上面的 `@types/passport-local`，这将帮助你在编写 TypeScript 代码时获得更好的支持。

#### 实现 Passport 策略

我们现在准备实现身份验证功能。我们将从 **任何** Passport 策略的使用流程概述开始。将 Passport 视为一个迷你框架是很有帮助的。该框架的优雅之处在于，它将身份验证过程抽象为几个你可以根据所实现的策略进行自定义的基本步骤。它的设计类似于框架，你可以通过提供定制参数（作为普通 JSON 对象）和自定义代码（作为回调函数）来配置它，Passport 会在适当的时候调用这些回调函数。`@nestjs/passport` 模块将这个框架封装为 Nest 风格的包，使其易于集成到 Nest 应用中。我们将在下面使用 `@nestjs/passport`，但首先让我们了解 **原生 Passport** 是如何工作的。

在原生 Passport 中，你需要通过提供以下两样东西来配置一个策略：

1. 该策略特有的配置选项。例如，在 JWT 策略中，你可能会提供一个用于签名令牌的密钥。
2. 一个“验证回调”函数，你在这里告诉 Passport 如何与你的用户存储库（你管理用户账户的地方）交互。在这里，你验证用户是否存在（和/或创建新用户），以及其凭证是否有效。Passport 库期望这个回调函数在验证成功时返回完整的用户对象，如果失败则返回 null（失败定义为未找到用户，或者在 passport-local 的情况下，密码不匹配）。

在 `@nestjs/passport` 中，你通过扩展 `PassportStrategy` 类来配置 Passport 策略。你通过在子类中调用 `super()` 方法传递策略选项（即上面的第 1 项），可以选择性地传入一个选项对象。你通过在子类中实现 `validate()` 方法来提供验证回调（即上面的第 2 项）。

我们首先生成一个 `AuthModule`，并在其中生成一个 `AuthService`：

```bash
$ nest g module auth
$ nest g service auth
```

在实现 `AuthService` 的过程中，我们会发现将用户操作封装到一个 `UsersService` 中很有用，因此我们现在生成该模块和服务：

```bash
$ nest g module users
$ nest g service users
```

替换生成文件的默认内容如下所示。对于我们的示例应用，`UsersService` 简单地维护一个硬编码的内存用户列表，并提供一个按用户名检索的方法。在真实应用中，这里是你构建用户模型和持久化层的地方，使用你选择的库（例如 TypeORM、Sequelize、Mongoose 等）。

```typescript
@@filename(users/users.service)
import { Injectable } from '@nestjs/common';

// 这应该是一个表示用户实体的真实类/接口
export type User = any;

@Injectable()
export class UsersService {
  private readonly users = [
    {
      userId: 1,
      username: 'john',
      password: 'changeme',
    },
    {
      userId: 2,
      username: 'maria',
      password: 'guess',
    },
  ];

  async findOne(username: string): Promise<User | undefined> {
    return this.users.find(user => user.username === username);
  }
}
@@switch
import { Injectable } from '@nestjs/common';

@Injectable()
export class UsersService {
  constructor() {
    this.users = [
      {
        userId: 1,
        username: 'john',
        password: 'changeme',
      },
      {
        userId: 2,
        username: 'maria',
        password: 'guess',
      },
    ];
  }

  async findOne(username) {
    return this.users.find(user => user.username === username);
  }
}
```

在 `UsersModule` 中，唯一需要做的更改是将 `UsersService` 添加到 `@Module` 装饰器的 exports 数组中，以便它在模块外部可见（我们将在 `AuthService` 中使用它）。

```typescript
@@filename(users/users.module)
import { Module } from '@nestjs/common';
import { UsersService } from './users.service';

@Module({
  providers: [UsersService],
  exports: [UsersService],
})
export class UsersModule {}
@@switch
import { Module } from '@nestjs/common';
import { UsersService } from './users.service';

@Module({
  providers: [UsersService],
  exports: [UsersService],
})
export class UsersModule {}
```

我们的 `AuthService` 负责检索用户并验证密码。我们为此目的创建了一个 `validateUser()` 方法。在下面的代码中，我们使用了一个方便的 ES6 扩展运算符来在返回用户对象之前从对象中删除密码属性。稍后我们将在 Passport 本地策略中调用 `validateUser()` 方法。

```typescript
@@filename(auth/auth.service)
import { Injectable } from '@nestjs/common';
import { UsersService } from '../users/users.service';

@Injectable()
export class AuthService {
  constructor(private usersService: UsersService) {}

  async validateUser(username: string, pass: string): Promise<any> {
    const user = await this.usersService.findOne(username);
    if (user && user.password === pass) {
      const { password, ...result } = user;
      return result;
    }
    return null;
  }
}
@@switch
import { Injectable, Dependencies } from '@nestjs/common';
import { UsersService } from '../users/users.service';

@Injectable()
@Dependencies(UsersService)
export class AuthService {
  constructor(usersService) {
    this.usersService = usersService;
  }

  async validateUser(username, pass) {
    const user = await this.usersService.findOne(username);
    if (user && user.password === pass) {
      const { password, ...result } = user;
      return result;
    }
    return null;
  }
}
```

> Warning **Warning** 当然，在真实应用中，你不应该以明文形式存储密码。你应该使用像 [bcrypt](https://github.com/kelektiv/node.bcrypt.js#readme) 这样的库，使用加盐的单向哈希算法。使用这种方法，你只存储哈希后的密码，然后将存储的密码与传入密码的哈希版本进行比较，从而永远不会以明文形式存储或暴露用户密码。为了简化我们的示例应用，我们违反了这一绝对要求，使用了明文密码。**不要在你的真实应用中这样做！**

现在，我们更新 `AuthModule` 以导入 `UsersModule`。

```typescript
@@filename(auth/auth.module)
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { UsersModule } from '../users/users.module';

@Module({
  imports: [UsersModule],
  providers: [AuthService],
})
export class AuthModule {}
@@switch
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { UsersModule } from '../users/users.module';

@Module({
  imports: [UsersModule],
  providers: [AuthService],
})
export class AuthModule {}
```

#### 实现 Passport 本地身份验证

现在我们可以实现我们的 Passport **本地身份验证策略**。在 `auth` 文件夹中创建一个名为 `local.strategy.ts` 的文件，并添加以下代码：

```typescript
@@filename(auth/local.strategy)
import { Strategy } from 'passport-local';
import { PassportStrategy } from '@nestjs/passport';
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { AuthService } from './auth.service';

@Injectable()
export class LocalStrategy extends PassportStrategy(Strategy) {
  constructor(private authService: AuthService) {
    super();
  }

  async validate(username: string, password: string): Promise<any> {
    const user = await this.authService.validateUser(username, password);
    if (!user) {
      throw new UnauthorizedException();
    }
    return user;
  }
}
@@switch
import { Strategy } from 'passport-local';
import { PassportStrategy } from '@nestjs/passport';
import { Injectable, UnauthorizedException, Dependencies } from '@nestjs/common';
import { AuthService } from './auth.service';

@Injectable()
@Dependencies(AuthService)
export class LocalStrategy extends PassportStrategy(Strategy) {
  constructor(authService) {
    super();
    this.authService = authService;
  }

  async validate(username, password) {
    const user = await this.authService.validateUser(username, password);
    if (!user) {
      throw new UnauthorizedException();
    }
    return user;
  }
}
```

我们遵循了前面描述的所有 Passport 策略的实现步骤。在此用例中，passport-local 没有配置选项，因此我们的构造函数仅调用 `super()`，不传递选项对象。

> info **提示** 我们可以在调用 `super()` 时传递一个选项对象来定制 Passport 策略的行为。在此示例中，默认情况下，passport-local 策略期望请求体中包含名为 `username` 和 `password` 的属性。传递一个选项对象以指定不同的属性名，例如：`super({{ '{' }} usernameField: 'email' {{ '}' }})`。有关更多信息，请参阅 [Passport 文档](http://www.passportjs.org/docs/configure/)。

我们还实现了 `validate()` 方法。对于每种策略，Passport 都会使用适当的策略特定参数集调用验证函数（在 `@nestjs/passport` 中通过 `validate()` 方法实现）。对于本地策略，Passport 期望一个具有以下签名的 `validate()` 方法：`validate(username: string, password: string): any`。

大多数验证工作是在我们的 `AuthService`（借助 `UsersService`）中完成的，因此此方法非常直接。**任何** Passport 策略的 `validate()` 方法都将遵循类似的模式，唯一的区别在于凭证的表示方式。例如，在 JWT 策略中，根据需求，我们可能会评估解码令牌中携带的 `userId` 是否与我们用户数据库中的记录匹配，或者是否匹配一组已撤销的令牌列表。因此，这种通过子类化并实现特定策略验证的模式是统一、优雅且可扩展的。

通常，每种策略的 `validate()` 方法中唯一的显著区别在于**如何**确定用户是否存在且有效。例如，在 JWT 策略中，根据需求，我们可能会评估解码后的令牌中携带的 `userId` 是否与我们的用户数据库中的记录匹配，或者是否与撤销的令牌列表匹配。因此，这种子类化并实现特定于策略的验证模式是一致的、优雅的和可扩展的。

我们需要配置 `AuthModule` 以使用我们刚刚定义的 Passport 功能。更新 `auth.module.ts` 使其看起来像这样：

```typescript
@@filename(auth/auth.module)
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { UsersModule } from '../users/users.module';
import { PassportModule } from '@nestjs/passport';
import { LocalStrategy } from './local.strategy';

@Module({
  imports: [UsersModule, PassportModule],
  providers: [AuthService, LocalStrategy],
})
export class AuthModule {}
@@switch
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { UsersModule } from '../users/users.module';
import { PassportModule } from '@nestjs/passport';
import { LocalStrategy } from './local.strategy';

@Module({
  imports: [UsersModule, PassportModule],
  providers: [AuthService, LocalStrategy],
})
export class AuthModule {}
```

#### 内置的 Passport 守卫

<a href="guards">守卫</a> 章节描述了守卫的主要功能：确定请求是否将由路由处理程序处理。这仍然适用，我们很快将使用这种标准能力。然而，在使用 `@nestjs/passport` 模块的上下文中，我们还将介绍一种略微不同的新机制，这可能一开始会让人感到困惑，因此我们现在讨论它。从身份验证的角度来看，你的应用可以处于两种状态：

1. 用户/客户端 **未** 登录（未认证）
2. 用户/客户端 **已** 登录（已认证）

在第一种情况（用户未登录）中，我们需要执行两个不同的功能：

- 限制未认证用户可以访问的路由（即拒绝访问受保护的路由）。我们将使用熟悉的守卫功能来处理此功能，通过在受保护的路由上放置一个守卫。正如你可能预料的那样，我们将在该守卫中检查是否存在有效的 JWT，因此我们将在成功签发 JWT 后再处理此守卫。

- 当之前未认证的用户尝试登录时，启动 **身份验证步骤** 本身。这是我们将 **签发** JWT 给有效用户的步骤。稍作思考，我们知道我们需要 `POST` 用户名/密码凭证来启动身份验证，因此我们将设置一个 `POST /auth/login` 路由来处理该请求。这引发了一个问题：我们如何在这个路由中调用 passport-local 策略？

答案很简单：通过使用另一种略微不同的守卫类型。`@nestjs/passport` 模块为我们提供了内置的守卫来为我们完成这项工作。这个守卫调用 Passport 策略并启动上述步骤（提取凭证、运行验证函数、创建 `user` 属性等）。

第二种情况（已登录用户）则只需依赖我们已经讨论过的标准守卫类型，以允许已登录用户访问受保护的路由。

<app-banner-courses-auth></app-banner-courses-auth>

#### 登录路由

有了策略之后，我们现在可以实现一个简单的 `/auth/login` 路由，并应用内置的守卫来启动 passport-local 流程。

打开 `app.controller.ts` 文件并将其内容替换为以下内容：

```typescript
@@filename(app.controller)
import { Controller, Request, Post, UseGuards } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Controller()
export class AppController {
  @UseGuards(AuthGuard('local'))
  @Post('auth/login')
  async login(@Request() req) {
    return req.user;
  }
}
@@switch
import { Controller, Bind, Request, Post, UseGuards } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Controller()
export class AppController {
  @UseGuards(AuthGuard('local'))
  @Post('auth/login')
  @Bind(Request())
  async login(req) {
    return req.user;
  }
}
```

通过 `@UseGuards(AuthGuard('local'))`，我们使用了 `@nestjs/passport` **自动为我们配置** 的 `AuthGuard`。让我们分解一下。我们的 Passport 本地策略有一个默认名称为 `'local'`。我们在 `@UseGuards()` 装饰器中引用该名称，以将其与 `passport-local` 包提供的代码关联起来。这用于在我们的应用中存在多个 Passport 策略（每个策略可能提供特定策略的 `AuthGuard`）时消除歧义。虽然目前我们只有一个这样的策略，但我们很快将添加第二个，因此需要此名称用于消除歧义。

为了测试我们的路由，我们将 `/auth/login` 路由暂时返回用户对象。这还让我们演示了另一个 Passport 特性：Passport 会自动创建一个 `user` 对象，基于我们从 `validate()` 方法返回的值，并将其作为 `req.user` 分配给 `Request` 对象。稍后，我们将用生成并返回 JWT 的代码替换它。

由于这些是 API 路由，我们将使用常用的 [cURL](https://curl.haxx.se/) 库对其进行测试。你可以使用硬编码在 `UsersService` 中的任何 `user` 对象进行测试。

```bash
$ # POST to /auth/login
$ curl -X POST http://localhost:3000/auth/login -d '{"username": "john", "password": "changeme"}' -H "Content-Type: application/json"
$ # result -> {"userId":1,"username":"john"}
```

虽然这有效，但直接将策略名称传递给 `AuthGuard()` 会在代码库中引入魔法字符串。相反，我们建议创建自己的类，如下所示：

```typescript
@@filename(auth/local-auth.guard)
import { Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class LocalAuthGuard extends AuthGuard('local') {}
```

现在，我们可以更新 `/auth/login` 路由处理程序并使用 `LocalAuthGuard`：

```typescript
@UseGuards(LocalAuthGuard)
@Post('auth/login')
async login(@Request() req) {
  return req.user;
}
```

#### 注销路由

要注销，我们可以创建一个额外的路由，调用 `req.logout()` 来清除用户的会话。这是基于会话的身份验证中常用的方法，但不适用于 JWT。

```typescript
@UseGuards(LocalAuthGuard)
@Post('auth/logout')
async logout(@Request() req) {
  return req.logout();
}
```

#### JWT 功能

我们已准备好进入我们身份验证系统的 JWT 部分。让我们回顾并完善我们的需求：

- 允许用户使用用户名/密码进行身份验证，并返回一个 JWT 用于后续调用受保护的 API 端点。我们已经很好地实现了这一需求。要完成它，我们需要编写签发 JWT 的代码。
- 创建基于是否存在有效 JWT 作为承载令牌的受保护 API 路由

我们需要安装几个更多包以支持我们的 JWT 需求：

```bash
$ npm install --save @nestjs/jwt passport-jwt
$ npm install --save-dev @types/passport-jwt
```

`@nestjs/jwt` 包（详见[此处](https://github.com/nestjs/jwt)）是一个用于 JWT 操作的实用包。`passport-jwt` 包是 Passport 的包，它实现了 JWT 策略，`@types/passport-jwt` 提供了 TypeScript 类型定义。

让我们更仔细地看看 `POST /auth/login` 请求是如何处理的。我们使用了 passport-local 策略提供的内置 `AuthGuard` 装饰了该路由。这意味着：

1. 只有在用户被验证后才会调用路由处理程序
2. `req` 参数将包含一个 `user` 属性（由 Passport 在 passport-local 身份验证流程中填充）

考虑到这一点，我们现在终于可以在该路由中生成一个真实的 JWT 并返回它。为了保持我们服务的清晰模块化，我们将在 `authService` 中处理生成 JWT。打开 `auth.service.ts` 文件并在 `auth` 文件夹中添加 `login()` 方法，并按如下方式导入 `JwtService`：

```typescript
@@filename(auth/auth.service)
import { Injectable } from '@nestjs/common';
import { UsersService } from '../users/users.service';
import { JwtService } from '@nestjs/jwt';

@Injectable()
export class AuthService {
  constructor(
    private usersService: UsersService,
    private jwtService: JwtService
  ) {}

  async validateUser(username: string, pass: string): Promise<any> {
    const user = await this.usersService.findOne(username);
    if (user && user.password === pass) {
      const { password, ...result } = user;
      return result;
    }
    return null;
  }

  async login(user: any) {
    const payload = { username: user.username, sub: user.userId };
    return {
      access_token: this.jwtService.sign(payload),
    };
  }
}

@@switch
import { Injectable, Dependencies } from '@nestjs/common';
import { UsersService } from '../users/users.service';
import { JwtService } from '@nestjs/jwt';

@Dependencies(UsersService, JwtService)
@Injectable()
export class AuthService {
  constructor(usersService, jwtService) {
    this.usersService = usersService;
    this.jwtService = jwtService;
  }

  async validateUser(username, pass) {
    const user = await this.usersService.findOne(username);
    if (user && user.password === pass) {
      const { password, ...result } = user;
      return result;
    }
    return null;
  }

  async login(user) {
    const payload = { username: user.username, sub: user.userId };
    return {
      access_token: this.jwtService.sign(payload),
    };
  }
}
```

我们使用 `@nestjs/jwt` 库，它提供了一个 `sign()` 函数来从 `user` 对象属性的子集生成我们的 JWT，然后将其作为一个具有单个 `access_token` 属性的简单对象返回。注意：我们选择使用 `sub` 属性来保存我们的 `userId` 值，以符合 JWT 标准。不要忘记将 JwtService 提供者注入到 `AuthService` 中。

我们现在需要更新 `AuthModule` 以导入新的依赖项并配置 `JwtModule`。

首先，在 `auth` 文件夹中创建 `constants.ts` 文件，并添加以下代码：

```typescript
@@filename(auth/constants)
export const jwtConstants = {
  secret: 'DO NOT USE THIS VALUE. INSTEAD, CREATE A COMPLEX SECRET AND KEEP IT SAFE OUTSIDE OF THE SOURCE CODE.',
};
@@switch
export const jwtConstants = {
  secret: 'DO NOT USE THIS VALUE. INSTEAD, CREATE A COMPLEX SECRET AND KEEP IT SAFE OUTSIDE OF THE SOURCE CODE.',
};
```

我们将使用它在 JWT 签名和验证步骤之间共享我们的密钥。

> Warning **警告** **不要公开此密钥**。我们在这里这样做是为了清楚地展示代码在做什么，但在生产系统中 **你必须使用适当的措施保护此密钥**，例如使用密钥保险库、环境变量或配置服务。

现在，打开 `auth.module.ts` 文件并在 `auth` 文件夹中更新它如下所示：

```typescript
@@filename(auth/auth.module)
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { LocalStrategy } from './local.strategy';
import { UsersModule } from '../users/users.module';
import { PassportModule } from '@nestjs/passport';
import { JwtModule } from '@nestjs/jwt';
import { jwtConstants } from './constants';

@Module({
  imports: [
    UsersModule,
    PassportModule,
    JwtModule.register({
      secret: jwtConstants.secret,
      signOptions: { expiresIn: '60s' },
    }),
  ],
  providers: [AuthService, LocalStrategy],
  exports: [AuthService],
})
export class AuthModule {}
@@switch
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { LocalStrategy } from './local.strategy';
import { UsersModule } from '../users/users.module';
import { PassportModule } from '@nestjs/passport';
import { JwtModule } from '@nestjs/jwt';
import { jwtConstants } from './constants';

@Module({
  imports: [
    UsersModule,
    PassportModule,
    JwtModule.register({
      secret: jwtConstants.secret,
      signOptions: { expiresIn: '60s' },
    }),
  ],
  providers: [AuthService, LocalStrategy],
  exports: [AuthService],
})
export class AuthModule {}
```

我们使用 `register()` 配置 `JwtModule`，并传入一个配置对象。有关 Nest `JwtModule` 的更多信息，请参见 [此处](https://github.com/nestjs/jwt/blob/master/README.md)，有关可用配置选项的更多详细信息，请参见 [此处](https://github.com/auth0/node-jsonwebtoken#usage)。

现在我们可以更新 `/auth/login` 路由以返回 JWT。

```typescript
@@filename(app.controller)
import { Controller, Request, Post, UseGuards } from '@nestjs/common';
import { LocalAuthGuard } from './auth/local-auth.guard';
import { AuthService } from './auth/auth.service';

@Controller()
export class AppController {
  constructor(private authService: AuthService) {}

  @UseGuards(LocalAuthGuard)
  @Post('auth/login')
  async login(@Request() req) {
    return this.authService.login(req.user);
  }
}
@@switch
import { Controller, Bind, Request, Post, UseGuards } from '@nestjs/common';
import { LocalAuthGuard } from './auth/local-auth.guard';
import { AuthService } from './auth/auth.service';

@Controller()
export class AppController {
  constructor(private authService: AuthService) {}

  @UseGuards(LocalAuthGuard)
  @Post('auth/login')
  @Bind(Request())
  async login(req) {
    return this.authService.login(req.user);
  }
}
```

让我们继续使用 cURL 测试我们的路由。你可以使用硬编码在 `UsersService` 中的任何 `user` 对象进行测试。

```bash
$ # POST to /auth/login
$ curl -X POST http://localhost:3000/auth/login -d '{"username": "john", "password": "changeme"}' -H "Content-Type: application/json"
$ # result -> {"access_token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."}
$ # Note: above JWT truncated
```

#### 实现 Passport JWT

我们现在可以解决我们的最终需求：通过要求请求中存在有效的 JWT 来保护端点。Passport 在这里也能帮助我们。它提供了 [passport-jwt](https://github.com/mikenicholson/passport-jwt) 策略，用于使用 JSON Web Tokens 保护 RESTful 端点。首先在 `auth` 文件夹中创建一个名为 `jwt.strategy.ts` 的文件，并添加以下代码：

```typescript
@@filename(auth/jwt.strategy)
import { ExtractJwt, Strategy } from 'passport-jwt';
import { PassportStrategy } from '@nestjs/passport';
import { Injectable } from '@nestjs/common';
import { jwtConstants } from './constants';

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor() {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,
      secretOrKey: jwtConstants.secret,
    });
  }

  async validate(payload: any) {
    return { userId: payload.sub, username: payload.username };
  }
}
@@switch
import { ExtractJwt, Strategy } from 'passport-jwt';
import { PassportStrategy } from '@nestjs/passport';
import { Injectable } from '@nestjs/common';
import { jwtConstants } from './constants';

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor() {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,
      secretOrKey: jwtConstants.secret,
    });
  }

  async validate(payload) {
    return { userId: payload.sub, username: payload.username };
  }
}
```

通过我们的 `JwtStrategy`，我们遵循了前面描述的所有 Passport 策略的相同配方。此策略需要一些初始化，因此我们通过在 `super()` 调用中传递一个选项对象来进行此操作。你可以在 [此处](https://github.com/mikenicholson/passport-jwt#configure-strategy) 阅读有关可用选项的更多信息。在我们的情况下，这些选项是：

- `jwtFromRequest`: 提供从 `Request` 中提取 JWT 的方法。我们将使用在 API 请求的 Authorization 头中提供承载令牌的标准方法。其他选项在 [此处](https://github.com/mikenicholson/passport-jwt#extracting-the-jwt-from-the-request) 描述。
- `ignoreExpiration`: 为了明确起见，我们选择默认的 `false` 设置，这将确保 JWT 是否已过期的责任委托给 Passport 模块。这意味着如果我们的路由收到一个过期的 JWT，请求将被拒绝并发送一个 `401 Unauthorized` 响应。Passport 会自动为我们处理这一点，非常方便。
- `secretOrKey`: 我们使用提供对称密钥来签名令牌的便捷选项。在生产应用中，可能更适合使用 PEM 编码的公钥等其他选项（参见 [此处](https://github.com/mikenicholson/passport-jwt#configure-strategy) 了解更多）。无论如何，如前所述，**不要公开此密钥**。

`validate()` 方法值得讨论。对于 jwt-strategy，Passport 首先验证 JWT 的签名并解码 JSON。然后，它通过将解码的 JSON 作为其唯一参数调用我们的 `validate()` 方法。根据 JWT 签名的工作方式，**我们保证收到的是一个我们之前签署并颁发给有效用户的令牌**。

因此，我们对 `validate()` 回调的响应很简单：我们只需返回一个包含 `userId` 和 `username` 属性的对象。再次回想一下，Passport 将基于我们 `validate()` 方法的返回值构建一个 `user` 对象，并将其作为 `Request` 对象的属性附加。

此外，你还可以返回一个数组，其中第一个值用于创建 `user` 对象，第二个值用于创建 `authInfo` 对象。

也值得一提的是，这种方法为我们提供了注入其他业务逻辑的空间（可以理解为“钩子”）。例如，我们可以在 `validate()` 方法中进行数据库查找，以提取更多关于用户的信息，从而在我们的 `Request` 中提供一个更丰富的 `user` 对象。这也是我们可能决定进行进一步令牌验证的地方，例如在撤销令牌列表中查找 `userId`，从而实现令牌撤销。我们在此示例代码中实现的模型是一个快速的“无状态 JWT”模型，其中每个 API 调用都立即基于是否存在有效的 JWT 授权，并且请求者的一小部分信息（其 `userId` 和 `username`）在我们的请求管道中可用。

在 `AuthModule` 中将新的 `JwtStrategy` 添加为提供者：

```typescript
@@filename(auth/auth.module)
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { LocalStrategy } from './local.strategy';
import { JwtStrategy } from './jwt.strategy';
import { UsersModule } from '../users/users.module';
import { PassportModule } from '@nestjs/passport';
import { JwtModule } from '@nestjs/jwt';
import { jwtConstants } from './constants';

@Module({
  imports: [
    UsersModule,
    PassportModule,
    JwtModule.register({
      secret: jwtConstants.secret,
      signOptions: { expiresIn: '60s' },
    }),
  ],
  providers: [AuthService, LocalStrategy, JwtStrategy],
  exports: [AuthService],
})
export class AuthModule {}
@@switch
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { LocalStrategy } from './local.strategy';
import { JwtStrategy } from './jwt.strategy';
import { UsersModule } from '../users/users.module';
import { PassportModule } from '@nestjs/passport';
import { JwtModule } from '@nestjs/jwt';
import { jwtConstants } from './constants';

@Module({
  imports: [
    UsersModule,
    PassportModule,
    JwtModule.register({
      secret: jwtConstants.secret,
      signOptions: { expiresIn: '60s' },
    }),
  ],
  providers: [AuthService, LocalStrategy, JwtStrategy],
  exports: [AuthService],
})
export class AuthModule {}
```

通过导入我们用于签名 JWT 的相同密钥，我们确保了由 Passport 执行的 **验证** 阶段和我们在 AuthService 中执行的 **签名** 阶段使用相同的密钥。

最后，我们定义 `JwtAuthGuard` 类，它扩展了内置的 `AuthGuard`：

```typescript
@@filename(auth/jwt-auth.guard)
import { Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {}
```

#### 实现受保护的路由和 JWT 策略守卫

我们现在可以实现我们的受保护路由及其相关的守卫。

打开 `app.controller.ts` 文件并按如下所示更新它：

```typescript
@@filename(app.controller)
import { Controller, Get, Request, Post, UseGuards } from '@nestjs/common';
import { JwtAuthGuard } from './auth/jwt-auth.guard';
import { LocalAuthGuard } from './auth/local-auth.guard';
import { AuthService } from './auth/auth.service';

@Controller()
export class AppController {
  constructor(private authService: AuthService) {}

  @UseGuards(LocalAuthGuard)
  @Post('auth/login')
  async login(@Request() req) {
    return this.authService.login(req.user);
  }

  @UseGuards(JwtAuthGuard)
  @Get('profile')
  getProfile(@Request() req) {
    return req.user;
  }
}
@@switch
import { Controller, Dependencies, Bind, Get, Request, Post, UseGuards } from '@nestjs/common';
import { JwtAuthGuard } from './auth/jwt-auth.guard';
import { LocalAuthGuard } from './auth/local-auth.guard';
import { AuthService } from './auth/auth.service';

@Dependencies(AuthService)
@Controller()
export class AppController {
  constructor(authService) {
    this.authService = authService;
  }

  @UseGuards(LocalAuthGuard)
  @Post('auth/login')
  @Bind(Request())
  async login(req) {
    return this.authService.login(req.user);
  }

  @UseGuards(JwtAuthGuard)
  @Get('profile')
  @Bind(Request())
  getProfile(req) {
    return req.user;
  }
}
```

再次，我们使用了 `@nestjs/passport` 模块在我们配置 passport-jwt 模块时自动为我们提供的 `AuthGuard`。这个守卫通过其默认名称 `jwt` 被引用。当我们的 `GET /profile` 路由被触发时，守卫将自动调用我们自定义配置的 passport-jwt 策略，验证 JWT，并将 `user` 属性分配给 `Request` 对象。

确保应用正在运行，并使用 `cURL` 测试路由。

```bash
$ # GET /profile
$ curl http://localhost:3000/profile
$ # result -> {"statusCode":401,"message":"Unauthorized"}

$ # POST /auth/login
$ curl -X POST http://localhost:3000/auth/login -d '{"username": "john", "password": "changeme"}' -H "Content-Type: application/json"
$ # result -> {"access_token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2Vybm... }

$ # GET /profile using access_token returned from previous step as bearer code
$ curl http://localhost:3000/profile -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2Vybm..."
$ # result -> {"userId":1,"username":"john"}
```

请注意，在 `AuthModule` 中，我们将 JWT 配置为 60 秒的过期时间。这可能太短了，处理令牌过期和刷新的细节超出了本文的范围。然而，我们选择它是为了演示 JWT 和 passport-jwt 策略的一个重要特性。如果你在认证后等待 60 秒再尝试执行 `GET /profile` 请求，你将收到一个 `401 Unauthorized` 响应。这是因为 Passport 自动检查 JWT 的过期时间，节省了你在应用程序中自行检查的麻烦。

我们现在完成了 JWT 身份验证的实现。JavaScript 客户端（如 Angular/React/Vue）和其他 JavaScript 应用现在可以安全地与我们的 API 服务器进行身份验证和通信。

#### 扩展守卫

在大多数情况下，使用提供的 `AuthGuard` 类就足够了。然而，在某些用例中，你可能希望扩展默认的错误处理或身份验证逻辑。为此，你可以扩展内置类并在子类中覆盖方法。

```typescript
import {
  ExecutionContext,
  Injectable,
  UnauthorizedException,
} from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {
  canActivate(context: ExecutionContext) {
    // 在这里添加你的自定义身份验证逻辑
    // 例如，调用 super.logIn(request) 以建立会话。
    return super.canActivate(context);
  }

  handleRequest(err, user, info) {
    // 你可以基于 "info" 或 "err" 参数抛出异常
    if (err || !user) {
      throw err || new UnauthorizedException();
    }
    return user;
  }
}
```

除了扩展默认的错误处理和身份验证逻辑外，我们还可以通过一系列策略进行身份验证。第一个成功、重定向或出错的策略将停止链。身份验证失败将按顺序通过每个策略，如果所有策略都失败，则最终失败。

```typescript
export class JwtAuthGuard extends AuthGuard(['strategy_jwt_1', 'strategy_jwt_2', '...']) { ... }
```

#### 全局启用身份验证

如果你的大部分端点默认应受保护，你可以将身份验证守卫注册为 [全局守卫](/guards#binding-guards)，并使用 `@UseGuards()` 装饰器在每个控制器上标注，而是标记哪些路由应为公共路由。

首先，使用以下构造将 `JwtAuthGuard` 注册为全局守卫（在任何模块中）：

```typescript
providers: [
  {
    provide: APP_GUARD,
    useClass: JwtAuthGuard,
  },
],
```

这样，Nest 将自动将 `JwtAuthGuard` 绑定到所有端点。

现在我们必须提供一种机制来声明路由为公共路由。为此，我们可以使用 `SetMetadata` 装饰器工厂函数创建一个自定义装饰器。

```typescript
import { SetMetadata } from '@nestjs/common';

export const IS_PUBLIC_KEY = 'isPublic';
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);
```

在上面的文件中，我们导出了两个常量。一个是名为 `IS_PUBLIC_KEY` 的元数据键，另一个是我们新的装饰器本身，我们将其命名为 `Public`（你也可以根据项目需要将其命名为 `SkipAuth` 或 `AllowAnon`）。

现在我们有了一个自定义的 `@Public()` 装饰器，我们可以使用它来装饰任何方法，如下所示：

```typescript
@Public()
@Get()
findAll() {
  return [];
}
```

最后，我们需要 `JwtAuthGuard` 在找到 `"isPublic"` 元数据时返回 `true`。为此，我们将使用 `Reflector` 类（更多信息请参见 [此处](/guards#putting-it-all-together)）。

```typescript
@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {
  constructor(private reflector: Reflector) {
    super();
  }

  canActivate(context: ExecutionContext) {
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (isPublic) {
      return true;
    }
    return super.canActivate(context);
  }
}
```

#### 请求作用域策略

Passport API 基于将策略注册到库的全局实例。因此，策略不是为每个请求设计的，也不是为每个请求动态实例化的（有关 [请求作用域](/fundamentals/injection-scopes) 提供者的更多信息）。当你将策略配置为请求作用域时，Nest 永远不会实例化它，因为它不绑定到任何特定的路由。没有物理方法来确定每个请求应执行哪些“请求作用域”的策略。

然而，我们可以在策略中动态解析请求作用域提供者。为此，我们利用 [模块引用](/fundamentals/module-ref) 特性。

首先，打开 `local.strategy.ts` 文件，并以正常方式注入 `ModuleRef`：

```typescript
constructor(private moduleRef: ModuleRef) {
  super({
    passReqToCallback: true,
  });
}
```

> info **提示** `ModuleRef` 类从 `@nestjs/core` 包中导入。

确保将 `passReqToCallback` 配置属性设置为 `true`，如上所示。

在下一步中，请求实例将用于获取当前上下文标识符，而不是生成一个新的（有关请求上下文的更多信息，请参见 [此处](/fundamentals/module-ref#getting-current-sub-tree)）。

现在，在 `LocalStrategy` 类的 `validate()` 方法中，使用 `ContextIdFactory` 类的 `getByRequest()` 方法基于请求对象创建一个上下文 ID，并将其传递给 `resolve()` 调用：

```typescript
async validate(
  request: Request,
  username: string,
  password: string,
) {
  const contextId = ContextIdFactory.getByRequest(request);
  // "AuthService" 是一个请求作用域提供者
  const authService = await this.moduleRef.resolve(AuthService, contextId);
  ...
}
```

在上面的示例中，`resolve()` 方法将异步返回 `AuthService` 提供者的请求作用域实例（我们假设 `AuthService` 被标记为请求作用域提供者）。

#### 自定义 Passport

任何标准的 Passport 自定义选项都可以通过 `register()` 方法以相同的方式传递。可用选项取决于所实现的策略。例如：

```typescript
PassportModule.register({ session: true });
```

你也可以在构造函数中传递一个选项对象给策略以配置它们。
对于本地策略，你可以传递例如：

```typescript
constructor(private authService: AuthService) {
  super({
    usernameField: 'email',
    passwordField: 'password',
  });
}
```

查看官方 [Passport 网站](http://www.passportjs.org/docs/oauth/) 了解属性名称。

#### 命名策略

在实现策略时，你可以通过向 `PassportStrategy` 函数传递第二个参数来为其提供一个名称。如果不这样做，每个策略将有一个默认名称（例如，jwt 策略的默认名称为 'jwt'）：

```typescript
export class JwtStrategy extends PassportStrategy(Strategy, 'myjwt')
```

然后，你可以通过装饰器如 `@UseGuards(AuthGuard('myjwt'))` 来引用它。

#### GraphQL

要在 [GraphQL](https://docs.nestjs.com/graphql/quick-start) 中使用 AuthGuard，扩展内置的 `AuthGuard` 类并覆盖 `getRequest()` 方法。

```typescript
@Injectable()
export class GqlAuthGuard extends AuthGuard('jwt') {
  getRequest(context: ExecutionContext) {
    const ctx = GqlExecutionContext.create(context);
    return ctx.getContext().req;
  }
}
```

要在你的 GraphQL 解析器中获取当前经过身份验证的用户，你可以定义一个 `@CurrentUser()` 装饰器：

```typescript
import { createParamDecorator, ExecutionContext } from '@nestjs/common';
import { GqlExecutionContext } from '@nestjs/graphql';

export const CurrentUser = createParamDecorator(
  (data: unknown, context: ExecutionContext) => {
    const ctx = GqlExecutionContext.create(context);
    return ctx.getContext().req.user;
  },
);
```

要在解析器中使用上述装饰器，请确保将其作为查询或突变的参数包含进去：

```typescript
@Query(() => User)
@UseGuards(GqlAuthGuard)
whoAmI(@CurrentUser() user: User) {
  return this.usersService.findById(user.id);
}
```

对于 passport-local 策略，你还需要将 GraphQL 上下文的参数添加到请求体中，以便 Passport 可以访问它们进行验证。否则，你会收到一个 Unauthorized 错误。

```typescript
@Injectable()
export class GqlLocalAuthGuard extends AuthGuard('local') {
  getRequest(context: ExecutionContext) {
    const gqlExecutionContext = GqlExecutionContext.create(context);
    const gqlContext = gqlExecutionContext.getContext();
    const gqlArgs = gqlExecutionContext.getArgs();

    gqlContext.req.body = { ...gqlContext.req.body, ...gqlArgs };
    return gqlContext.req;
  }
}
```
