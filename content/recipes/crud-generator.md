### CRUD 生成器（仅限 TypeScript）

在一个项目的整个生命周期中，当我们开发新功能时，通常需要向应用程序中添加新的资源。这些资源通常需要执行多个重复的操作，而这些操作在定义新资源时都必须重复进行。

#### 简介

让我们设想一个实际的场景：我们需要为两个实体暴露 CRUD 接口，比如 **User** 和 **Product** 实体。  
按照最佳实践，对于每个实体，我们都需要执行以下多个操作：

- 生成一个模块（`nest g mo`），以组织代码并建立清晰的边界（将相关组件分组）
- 生成一个控制器（`nest g co`），以定义 CRUD 路由（或者 GraphQL 应用中的查询/变更）
- 生成一个服务（`nest g s`），以实现和隔离业务逻辑
- 生成一个实体类/接口，以表示资源的数据结构
- 生成数据传输对象（或者 GraphQL 应用中的输入对象），以定义网络传输的数据格式

这需要很多步骤！

为了帮助加快这个重复的过程，[Nest CLI](/cli/overview) 提供了一个生成器（schematic），它可以自动生成所有样板代码，帮助我们避免手动完成所有步骤，从而极大地简化开发体验。

> info **注意** 该 schematics 支持生成 **HTTP** 控制器、**微服务** 控制器、**GraphQL** 解析器（包括代码优先和模式优先）以及 **WebSocket** 网关。

#### 生成新资源

要创建一个新资源，只需在项目根目录运行以下命令：

```shell
$ nest g resource
```

`nest g resource` 命令不仅会生成所有 NestJS 的基本模块（模块、服务、控制器类），还会生成实体类、DTO 类以及测试文件（`.spec` 文件）。

以下是你将看到的控制器文件示例（针对 REST API）：

```typescript
@Controller('users')
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Post()
  create(@Body() createUserDto: CreateUserDto) {
    return this.usersService.create(createUserDto);
  }

  @Get()
  findAll() {
    return this.usersService.findAll();
  }

  @Get(':id')
  findOne(@Param('id') id: string) {
    return this.usersService.findOne(+id);
  }

  @Patch(':id')
  update(@Param('id') id: string, @Body() updateUserDto: UpdateUserDto) {
    return this.usersService.update(+id, updateUserDto);
  }

  @Delete(':id')
  remove(@Param('id') id: string) {
    return this.usersService.remove(+id);
  }
}
```

它还会自动为所有 CRUD 端点生成占位符（REST API 的路由、GraphQL 的查询与变更、微服务和 WebSocket 网关的消息订阅），这一切都不需要你手动操作。

> warning **注意** 生成的服务类 **不绑定任何特定 ORM（或数据源）**。这使得生成器足够通用，可以满足任何项目的需要。默认情况下，所有方法都包含占位符，你可以根据项目需求填充特定的数据源。

同样地，如果你想为 GraphQL 应用程序生成解析器，只需在提示中选择 `GraphQL (code first)`（或 `GraphQL (schema first)`）作为传输层即可。

在这种情况下，NestJS 将生成一个解析器类，而不是 REST API 控制器：

```shell
$ nest g resource users

> ? What transport layer do you use? GraphQL (code first)
> ? Would you like to generate CRUD entry points? Yes
> CREATE src/users/users.module.ts (224 bytes)
> CREATE src/users/users.resolver.spec.ts (525 bytes)
> CREATE src/users/users.resolver.ts (1109 bytes)
> CREATE src/users/users.service.spec.ts (453 bytes)
> CREATE src/users/users.service.ts (625 bytes)
> CREATE src/users/dto/create-user.input.ts (195 bytes)
> CREATE src/users/dto/update-user.input.ts (281 bytes)
> CREATE src/users/entities/user.entity.ts (187 bytes)
> UPDATE src/app.module.ts (312 bytes)
```

> info **提示** 如果你想避免生成测试文件，可以添加 `--no-spec` 参数，例如：`nest g resource users --no-spec`

如下所示，不仅生成了所有 GraphQL 的查询和变更语句，还自动将所有组件连接了起来。我们使用了 `UsersService`、`User` 实体和 DTO 类。

```typescript
import { Resolver, Query, Mutation, Args, Int } from '@nestjs/graphql';
import { UsersService } from './users.service';
import { User } from './entities/user.entity';
import { CreateUserInput } from './dto/create-user.input';
import { UpdateUserInput } from './dto/update-user.input';

@Resolver(() => User)
export class UsersResolver {
  constructor(private readonly usersService: UsersService) {}

  @Mutation(() => User)
  createUser(@Args('createUserInput') createUserInput: CreateUserInput) {
    return this.usersService.create(createUserInput);
  }

  @Query(() => [User], { name: 'users' })
  findAll() {
    return this.usersService.findAll();
  }

  @Query(() => User, { name: 'user' })
  findOne(@Args('id', { type: () => Int }) id: number) {
    return this.usersService.findOne(id);
  }

  @Mutation(() => User)
  updateUser(@Args('updateUserInput') updateUserInput: UpdateUserInput) {
    return this.usersService.update(updateUserInput.id, updateUserInput);
  }

  @Mutation(() => User)
  removeUser(@Args('id', { type: () => Int }) id: number) {
    return this.usersService.remove(id);
  }
}
```