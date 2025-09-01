### 身份验证

身份验证是大多数应用程序中**必不可少**的一部分。处理身份验证的方法和策略有很多，针对不同项目的需求，可以选择不同的方式。本章将介绍几种可以适应多种需求的身份验证方法。

让我们明确一下需求。在这个用例中，客户端首先使用用户名和密码进行身份验证。验证成功后，服务器会签发一个 JWT（JSON Web Token），客户端可以在后续请求的授权头中以 [Bearer Token](https://tools.ietf.org/html/rfc6750) 的形式发送该令牌以证明身份。我们还将创建一个受保护的路由，只有携带有效 JWT 的请求才能访问。

我们首先实现第一个需求：用户身份验证。然后在此基础上签发 JWT。最后，我们创建一个受保护的路由，用于验证请求中的 JWT 是否有效。

#### 创建身份验证模块

我们首先生成一个 `AuthModule`，并在其中创建 `AuthService` 和 `AuthController`。我们将使用 `AuthService` 来实现身份验证逻辑，并通过 `AuthController` 暴露身份验证的端点。

```bash
$ nest g module auth
$ nest g controller auth
$ nest g service auth
```

在实现 `AuthService` 时，我们会发现将用户操作封装在一个 `UsersService` 中会很有帮助。因此我们现在生成该模块和服务：

```bash
$ nest g module users
$ nest g service users
```

将这些生成文件的默认内容替换为如下代码。在我们的示例应用中，`UsersService` 简单地维护了一个硬编码的用户列表，以及一个根据用户名检索用户的查找方法。在实际应用中，这里应该使用你选择的库（例如 TypeORM、Sequelize、Mongoose 等）构建用户模型和持久化层。

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

在 `UsersModule` 中，只需要将 `UsersService` 添加到 `@Module` 装饰器的 `exports` 数组中，以便在模块外部可见（我们将在 `AuthService` 中使用它）。

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

#### 实现“登录”端点

`AuthService` 的职责是检索用户并验证密码。为此我们创建了一个 `signIn()` 方法。在以下代码中，我们使用 ES6 的展开运算符从用户对象中去除密码属性后再返回，这是一种常见的做法，因为你不应暴露像密码或安全密钥这样的敏感字段。

```typescript
@@filename(auth/auth.service)
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { UsersService } from '../users/users.service';

@Injectable()
export class AuthService {
  constructor(private usersService: UsersService) {}

  async signIn(username: string, pass: string): Promise<any> {
    const user = await this.usersService.findOne(username);
    if (user?.password !== pass) {
      throw new UnauthorizedException();
    }
    const { password, ...result } = user;
    // TODO: 在这里生成 JWT 并返回
    // 而不是返回用户对象
    return result;
  }
}
@@switch
import { Injectable, Dependencies, UnauthorizedException } from '@nestjs/common';
import { UsersService } from '../users/users.service';

@Injectable()
@Dependencies(UsersService)
export class AuthService {
  constructor(usersService) {
    this.usersService = usersService;
  }

  async signIn(username: string, pass: string) {
    const user = await this.usersService.findOne(username);
    if (user?.password !== pass) {
      throw new UnauthorizedException();
    }
    const { password, ...result } = user;
    // TODO: 在这里生成 JWT 并返回
    // 而不是返回用户对象
    return result;
  }
}
```

> 警告 **警告** 当然，在实际应用中，你不应该以明文形式存储密码。你应该使用类似 [bcrypt](https://github.com/kelektiv/node.bcrypt.js#readme) 的库，使用带盐的单向哈希算法。这种方式下，你只存储哈希后的密码，并将传入的密码哈希后与存储的哈希值进行比较，从而避免以明文形式存储或暴露用户密码。为了保持示例简单，我们违反了这一绝对准则并使用了明文密码。**在你的实际应用中不要这样做！**

现在，我们更新 `AuthModule` 以导入 `UsersModule`。

```typescript
@@filename(auth/auth.module)
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { AuthController } from './auth.controller';
import { UsersModule } from '../users/users.module';

@Module({
  imports: [UsersModule],
  providers: [AuthService],
  controllers: [AuthController],
})
export class AuthModule {}
@@switch
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { AuthController } from './auth.controller';
import { UsersModule } from '../users/users.module';

@Module({
  imports: [UsersModule],
  providers: [AuthService],
  controllers: [AuthController],
})
export class AuthModule {}
```

完成以上步骤后，打开 `AuthController` 并添加 `signIn()` 方法。此方法将由客户端调用以验证用户身份。它将从请求体中接收用户名和密码，并在验证成功后返回一个 JWT 令牌。

```typescript
@@filename(auth/auth.controller)
import { Body, Controller, Post, HttpCode, HttpStatus } from '@nestjs/common';
import { AuthService } from './auth.service';

@Controller('auth')
export class AuthController {
  constructor(private authService: AuthService) {}

  @HttpCode(HttpStatus.OK)
  @Post('login')
  signIn(@Body() signInDto: Record<string, any>) {
    return this.authService.signIn(signInDto.username, signInDto.password);
  }
}
```

> 提示 **提示** 理想情况下，我们应使用 DTO 类来定义请求体的结构，而不是使用 `Record<string, any>` 类型。更多信息请参阅 [验证](/techniques/validation) 章节。

<app-banner-courses-auth></app-banner-courses-auth>

#### JWT 令牌

我们已准备好进入身份验证系统中 JWT 部分的实现。让我们回顾并完善我们的需求：

- 允许用户使用用户名/密码进行身份验证，并返回一个 JWT，用于后续调用受保护的 API 端点。我们已经完成了大部分工作。要完成这一需求，我们需要编写生成 JWT 的代码。
- 创建受保护的 API 路由，要求请求中包含有效的 JWT 作为 Bearer Token。

为了支持 JWT 的需求，我们需要安装一个额外的包：

```bash
$ npm install --save @nestjs/jwt
```

> 提示 **提示** `@nestjs/jwt` 包（更多信息见 [这里](https://github.com/nestjs/jwt)）是一个用于 JWT 操作的实用包，包括生成和验证 JWT 令牌。

为了保持服务的模块化，我们将在 `AuthService` 中处理 JWT 的生成。打开 `auth.service.ts` 文件，注入 `JwtService`，并更新 `signIn` 方法如下：

```typescript
@@filename(auth/auth.service)
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { UsersService } from '../users/users.service';
import { JwtService } from '@nestjs/jwt';

@Injectable()
export class AuthService {
  constructor(
    private usersService: UsersService,
    private jwtService: JwtService
  ) {}

  async signIn(
    username: string,
    pass: string,
  ): Promise<{ access_token: string }> {
    const user = await this.usersService.findOne(username);
    if (user?.password !== pass) {
      throw new UnauthorizedException();
    }
    const payload = { sub: user.userId, username: user.username };
    return {
      access_token: await this.jwtService.signAsync(payload),
    };
  }
}
@@switch
import { Injectable, Dependencies, UnauthorizedException } from '@nestjs/common';
import { UsersService } from '../users/users.service';
import { JwtService } from '@nestjs/jwt';

@Dependencies(UsersService, JwtService)
@Injectable()
export class AuthService {
  constructor(usersService, jwtService) {
    this.usersService = usersService;
    this.jwtService = jwtService;
  }

  async signIn(username, pass) {
    const user = await this.usersService.findOne(username);
    if (user?.password !== pass) {
      throw new UnauthorizedException();
    }
    const payload = { username: user.username, sub: user.userId };
    return {
      access_token: await this.jwtService.signAsync(payload),
    };
  }
}
```

我们使用 `@nestjs/jwt` 库提供的 `signAsync()` 方法，基于用户对象的一部分属性生成 JWT，并将其作为具有 `access_token` 属性的简单对象返回。注意：我们使用 `sub` 作为 `userId` 的属性名，以遵循 JWT 的标准。

现在我们需要更新 `AuthModule` 以导入新依赖项并配置 `JwtModule`。

首先，在 `auth` 文件夹中创建 `constants.ts` 文件，并添加以下代码：

```typescript
@@filename(auth/constants)
export const jwtConstants = {
  secret: '不要使用此值。相反，应创建一个复杂的密钥并将其安全地保存在源代码之外。',
};
@@switch
export const jwtConstants = {
  secret: '不要使用此值。相反，应创建一个复杂的密钥并将其安全地保存在源代码之外。',
};
```

我们将使用它来共享 JWT 签名和验证步骤中的密钥。

> 警告 **警告** **不要将此密钥公开暴露**。我们在这里这样做是为了清楚展示代码的作用，但在生产系统中，**你必须使用适当的方法保护此密钥**，例如使用密钥库、环境变量或配置服务。

现在，打开 `auth/auth.module.ts` 文件并更新如下：

```typescript
@@filename(auth/auth.module)
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { UsersModule } from '../users/users.module';
import { JwtModule } from '@nestjs/jwt';
import { AuthController } from './auth.controller';
import { jwtConstants } from './constants';

@Module({
  imports: [
    UsersModule,
    JwtModule.register({
      global: true,
      secret: jwtConstants.secret,
      signOptions: { expiresIn: '60s' },
    }),
  ],
  providers: [AuthService],
  controllers: [AuthController],
  exports: [AuthService],
})
export class AuthModule {}
@@switch
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { UsersModule } from '../users/users.module';
import { JwtModule } from '@nestjs/jwt';
import { AuthController } from './auth.controller';
import { jwtConstants } from './constants';

@Module({
  imports: [
    UsersModule,
    JwtModule.register({
      global: true,
      secret: jwtConstants.secret,
      signOptions: { expiresIn: '60s' },
    }),
  ],
  providers: [AuthService],
  controllers: [AuthController],
  exports: [AuthService],
})
export class AuthModule {}
```

> 提示 **提示** 我们将 `JwtModule` 注册为全局模块以简化操作。这意味着我们无需在应用的其他地方再导入 `JwtModule`。

我们通过 `register()` 方法配置 `JwtModule`，并传入配置对象。更多关于 Nest `JwtModule` 的信息请参见 [这里](https://github.com/nestjs/jwt/blob/master/README.md)，关于可用配置选项的详细信息请参见 [这里](https://github.com/auth0/node-jsonwebtoken#usage)。

现在我们使用 cURL 再次测试路由。你可以使用硬编码在 `UsersService` 中的任意 `user` 对象进行测试。

```bash
$ # POST 到 /auth/login
$ curl -X POST http://localhost:3000/auth/login -d '{"username": "john", "password": "changeme"}' -H "Content-Type: application/json"
{"access_token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."}
$ # 注意：上面的 JWT 被截断了
```

#### 实现身份验证守卫

现在我们可以处理最后一个需求：通过要求请求中包含有效的 JWT 来保护端点。我们将通过创建一个 `AuthGuard` 来保护我们的路由。

```typescript
@@filename(auth/auth.guard)
import {
  CanActivate,
  ExecutionContext,
  Injectable,
  UnauthorizedException,
} from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { jwtConstants } from './constants';
import { Request } from 'express';

@Injectable()
export class AuthGuard implements CanActivate {
  constructor(private jwtService: JwtService) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    const token = this.extractTokenFromHeader(request);
    if (!token) {
      throw new UnauthorizedException();
    }
    try {
      const payload = await this.jwtService.verifyAsync(
        token,
        {
          secret: jwtConstants.secret
        }
      );
      // 💡 我们在这里将 payload 赋值给 request 对象
      // 以便在路由处理器中访问
      request['user'] = payload;
    } catch {
      throw new UnauthorizedException();
    }
    return true;
  }

  private extractTokenFromHeader(request: Request): string | undefined {
    const [type, token] = request.headers.authorization?.split(' ') ?? [];
    return type === 'Bearer' ? token : undefined;
  }
}
```

现在我们可以实现受保护的路由并注册 `AuthGuard` 来保护它。

打开 `auth.controller.ts` 文件并更新如下：

```typescript
@@filename(auth.controller)
import {
  Body,
  Controller,
  Get,
  HttpCode,
  HttpStatus,
  Post,
  Request,
  UseGuards
} from '@nestjs/common';
import { AuthGuard } from './auth.guard';
import { AuthService } from './auth.service';

@Controller('auth')
export class AuthController {
  constructor(private authService: AuthService) {}

  @HttpCode(HttpStatus.OK)
  @Post('login')
  signIn(@Body() signInDto: Record<string, any>) {
    return this.authService.signIn(signInDto.username, signInDto.password);
  }

  @UseGuards(AuthGuard)
  @Get('profile')
  getProfile(@Request() req) {
    return req.user;
  }
}
```

我们已将刚刚创建的 `AuthGuard` 应用于 `GET /profile` 路由，因此它将受到保护。

确保应用正在运行，并使用 `cURL` 测试路由。

```bash
$ # GET /profile
$ curl http://localhost:3000/auth/profile
{"statusCode":401,"message":"Unauthorized"}

$ # POST /auth/login
$ curl -X POST http://localhost:3000/auth/login -d '{"username": "john", "password": "changeme"}' -H "Content-Type: application/json"
{"access_token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2Vybm..."}

$ # 使用上一步返回的 access_token 发起 GET /profile 请求
$ curl http://localhost:3000/auth/profile -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2Vybm..."
{"sub":1,"username":"john","iat":...,"exp":...}
```

请注意，在 `AuthModule` 中我们配置 JWT 的过期时间为 `60 秒`。这个时间太短，处理令牌过期和刷新的细节超出了本文的范围。然而，我们选择这个时间是为了演示 JWT 的一个重要特性：如果你在身份验证后等待 60 秒再尝试发起 `GET /auth/profile` 请求，你将收到 `401 Unauthorized` 响应。这是因为 `@nestjs/jwt` 会自动检查 JWT 的过期时间，省去了你在应用中手动处理的麻烦。

至此，我们已完成 JWT 身份验证的实现。JavaScript 客户端（如 Angular/React/Vue）和其他 JavaScript 应用现在可以安全地与我们的 API 服务器进行身份验证和通信。

#### 全局启用身份验证

如果你的大部分端点默认都应受到保护，你可以将身份验证守卫注册为 [全局守卫](/guards#binding-guards)，而不是在每个控制器上使用 `@UseGuards()` 装饰器，而是标记哪些路由是公开的。

首先，使用以下方式将 `AuthGuard` 注册为全局守卫（可以在任意模块中注册，例如 `AuthModule`）：

```typescript
providers: [
  {
    provide: APP_GUARD,
    useClass: AuthGuard,
  },
],
```

这样，Nest 会自动将 `AuthGuard` 绑定到所有端点。

现在我们需要提供一种机制来声明公开路由。为此，我们可以使用 `SetMetadata` 装饰器工厂函数创建一个自定义装饰器。

```typescript
import { SetMetadata } from '@nestjs/common';

export const IS_PUBLIC_KEY = 'isPublic';
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);
```

在上面的文件中，我们导出了两个常量。一个是我们的元数据键 `IS_PUBLIC_KEY`，另一个是我们将调用的自定义装饰器 `Public`（你也可以根据项目需要将其命名为 `SkipAuth` 或 `AllowAnon`）。

有了这个自定义的 `@Public()` 装饰器后，我们可以用它装饰任何方法，如下所示：

```typescript
@Public()
@Get()
findAll() {
  return [];
}
```

最后，我们需要 `AuthGuard` 在检测到 `"isPublic"` 元数据时返回 `true`。为此，我们将使用 `Reflector` 类（更多信息请参见 [这里](/guards#putting-it-all-together)）。

```typescript
@Injectable()
export class AuthGuard implements CanActivate {
  constructor(private jwtService: JwtService, private reflector: Reflector) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (isPublic) {
      // 💡 参见此条件
      return true;
    }

    const request = context.switchToHttp().getRequest();
    const token = this.extractTokenFromHeader(request);
    if (!token) {
      throw new UnauthorizedException();
    }
    try {
      const payload = await this.jwtService.verifyAsync(token, {
        secret: jwtConstants.secret,
      });
      // 💡 我们在这里将 payload 赋值给 request 对象
      // 以便在路由处理器中访问
      request['user'] = payload;
    } catch {
      throw new UnauthorizedException();
    }
    return true;
  }

  private extractTokenFromHeader(request: Request): string | undefined {
    const [type, token] = request.headers.authorization?.split(' ') ?? [];
    return type === 'Bearer' ? token : undefined;
  }
}
```

#### Passport 集成

[Passport](https://github.com/jaredhanson/passport) 是最受欢迎的 node.js 身份验证库，为社区所熟知，并成功用于许多生产应用中。使用 `@nestjs/passport` 模块，可以轻松地将该库集成到 **Nest** 应用中。

如需了解如何将 Passport 与 NestJS 集成，请参阅此 [章节](/recipes/passport)。

#### 示例

你可以在 [这里](https://github.com/nestjs/nest/tree/master/sample/19-auth-jwt) 找到本章代码的完整版本。