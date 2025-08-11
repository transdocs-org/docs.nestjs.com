### Prisma

[Prisma](https://www.prisma.io) 是一个用于 Node.js 和 TypeScript 的 [开源](https://github.com/prisma/prisma) ORM。它可以作为编写原始 SQL 或使用其他数据库访问工具（例如 SQL 查询构建器（如 [knex.js](https://knexjs.org/)）或 ORM（如 [TypeORM](https://typeorm.io/) 和 [Sequelize](https://sequelize.org/)））的替代方案。目前 Prisma 支持 PostgreSQL、MySQL、SQL Server、SQLite、MongoDB 和 CockroachDB（[预览版](https://www.prisma.io/docs/reference/database-reference/supported-databases)）。

虽然 Prisma 可以与纯 JavaScript 一起使用，但它拥抱 TypeScript，并提供超越 TypeScript 生态系统中其他 ORM 的类型安全保证。您可以在此处找到 Prisma 和 TypeORM 类型安全保证的深入比较 [这里](https://www.prisma.io/docs/concepts/more/comparisons/prisma-and-typeorm#type-safety)。

> info **注意** 如果您想快速了解 Prisma 的工作原理，可以按照 [快速入门](https://www.prisma.io/docs/getting-started/quickstart) 或阅读 [简介](https://www.prisma.io/docs/understand-prisma/introduction) 部分，这些内容都在 [文档](https://www.prisma.io/docs/) 中。在 [`prisma-examples`](https://github.com/prisma/prisma-examples/) 仓库中还提供了可用于 [REST](https://github.com/prisma/prisma-examples/tree/b53fad046a6d55f0090ddce9fd17ec3f9b95cab3/orm/nest) 和 [GraphQL](https://github.com/prisma/prisma-examples/tree/b53fad046a6d55f0090ddce9fd17ec3f9b95cab3/orm/nest-graphql) 的即用示例。

#### 入门

在本指南中，您将学习如何从零开始使用 NestJS 和 Prisma。您将构建一个示例 NestJS 应用程序，该应用程序具有一个可以读写数据库数据的 REST API。

为了简化操作，本指南中您将使用 [SQLite](https://sqlite.org/) 数据库，以避免设置数据库服务器的开销。请注意，即使您使用的是 PostgreSQL 或 MySQL，也可以继续按照本指南操作 — 您将在适当的位置获得使用这些数据库的额外说明。

> info **注意** 如果您已有现有项目并考虑迁移到 Prisma，可以按照 [将 Prisma 添加到现有项目](https://www.prisma.io/docs/getting-started/setup-prisma/add-to-existing-project-typescript-postgres) 的指南进行操作。如果您是从 TypeORM 迁移过来的，可以阅读 [从 TypeORM 迁移到 Prisma](https://www.prisma.io/docs/guides/migrate-to-prisma/migrate-from-typeorm) 的指南。

#### 创建您的 NestJS 项目

首先，安装 NestJS CLI 并使用以下命令创建您的应用骨架：

```bash
$ npm install -g @nestjs/cli
$ nest new hello-prisma
```

有关此命令创建的项目文件的更多信息，请参见 [入门指南](https://docs.nestjs.com/first-steps) 页面。请注意，您现在可以运行 `npm start` 来启动您的应用程序。运行在 `http://localhost:3000/` 的 REST API 当前提供了一个由 `src/app.controller.ts` 实现的单一路由。在本指南的过程中，您将实现额外的路由来存储和检索关于 _用户_ 和 _文章_ 的数据。

#### 设置 Prisma

首先在您的项目中将 Prisma CLI 安装为开发依赖：

```bash
$ cd hello-prisma
$ npm install prisma --save-dev
```

在接下来的步骤中，我们将使用 [Prisma CLI](https://www.prisma.io/docs/reference/tools-and-interfaces/prisma-cli)。作为最佳实践，建议通过在其前面加上 `npx` 来本地调用 CLI：

```bash
$ npx prisma
```

<details><summary>如果您使用的是 Yarn，请展开此部分</summary>

如果您使用的是 Yarn，则可以按以下方式安装 Prisma CLI：

```bash
$ yarn add prisma --dev
```

安装完成后，可以通过在其前面加上 `yarn` 来调用它：

```bash
$ yarn prisma
```

</details>

现在使用 Prisma CLI 的 `init` 命令创建您的初始 Prisma 设置：

```bash
$ npx prisma init
```

此命令将在 `prisma` 目录下创建以下内容：

- `schema.prisma`：指定您的数据库连接并包含数据库模式
- `.env`：一个 [dotenv](https://github.com/motdotla/dotenv) 文件，通常用于将数据库凭据存储在一组环境变量中

#### 设置数据库连接

您的数据库连接在 `schema.prisma` 文件中的 `datasource` 块中配置。默认情况下，它被设置为 `postgresql`，但由于在本指南中您使用的是 SQLite 数据库，因此需要将 `datasource` 块中的 `provider` 字段更改为 `sqlite`：

```groovy
datasource db {
  provider = "sqlite"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}
```

现在，打开 `.env` 并将 `DATABASE_URL` 环境变量调整为如下所示：

```bash
DATABASE_URL="file:./dev.db"
```

请确保您已配置 [ConfigModule](https://docs.nestjs.com/techniques/configuration)，否则 `.env` 中的 `DATABASE_URL` 变量将不会被拾取。

SQLite 数据库是简单的文件；使用 SQLite 数据库不需要服务器。因此，您无需配置包含 _host_ 和 _port_ 的连接 URL，只需将其指向本地文件即可，本例中文件名为 `dev.db`。此文件将在下一步中创建。

<details><summary>如果您使用的是 PostgreSQL、MySQL、MsSQL 或 Azure SQL，请展开此部分</summary>

对于 PostgreSQL 和 MySQL，您需要配置连接 URL 以指向 _数据库服务器_。您可以在此处了解所需的连接 URL 格式 [这里](https://www.prisma.io/docs/reference/database-reference/connection-urls)。

**PostgreSQL**

如果您使用的是 PostgreSQL，则需要按如下方式调整 `schema.prisma` 和 `.env` 文件：

**`schema.prisma`**

```groovy
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}
```

**`.env`**

```bash
DATABASE_URL="postgresql://USER:PASSWORD@HOST:PORT/DATABASE?schema=SCHEMA"
```

将全大写字母的占位符替换为您的数据库凭据。请注意，如果您不确定 `SCHEMA` 占位符应该填写什么内容，它很可能就是默认值 `public`：

```bash
DATABASE_URL="postgresql://USER:PASSWORD@HOST:PORT/DATABASE?schema=public"
```

如果您想了解如何设置 PostgreSQL 数据库，请按照此指南 [在 Heroku 上设置免费 PostgreSQL 数据库](https://dev.to/prisma/how-to-setup-a-free-postgresql-database-on-heroku-1dc1)。

**MySQL**

如果您使用的是 MySQL，则需要按如下方式调整 `schema.prisma` 和 `.env` 文件：

**`schema.prisma`**

```groovy
datasource db {
  provider = "mysql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}
```

**`.env`**

```bash
DATABASE_URL="mysql://USER:PASSWORD@HOST:PORT/DATABASE"
```

将全大写字母的占位符替换为您的数据库凭据。

**Microsoft SQL Server / Azure SQL Server**

如果您使用的是 Microsoft SQL Server 或 Azure SQL Server，则需要按如下方式调整 `schema.prisma` 和 `.env` 文件：

**`schema.prisma`**

```groovy
datasource db {
  provider = "sqlserver"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}
```

**`.env`**

将全大写字母的占位符替换为您的数据库凭据。请注意，如果您不确定 `encrypt` 占位符应该填写什么内容，它很可能就是默认值 `true`：

```bash
DATABASE_URL="sqlserver://HOST:PORT;database=DATABASE;user=USER;password=PASSWORD;encrypt=true"
```

</details>

#### 使用 Prisma Migrate 创建两个数据库表

在本节中，您将使用 [Prisma Migrate](https://www.prisma.io/docs/concepts/components/prisma-migrate) 在数据库中创建两个新表。Prisma Migrate 会根据 Prisma 模式中的声明性数据模型定义生成 SQL 迁移文件。这些迁移文件完全可自定义，以便您可以配置底层数据库的任何附加功能或包含附加命令，例如用于种子数据。

将以下两个模型添加到您的 `schema.prisma` 文件中：

```groovy
model User {
  id    Int     @default(autoincrement()) @id
  email String  @unique
  name  String?
  posts Post[]
}

model Post {
  id        Int      @default(autoincrement()) @id
  title     String
  content   String?
  published Boolean? @default(false)
  author    User?    @relation(fields: [authorId], references: [id])
  authorId  Int?
}
```

有了 Prisma 模型后，您可以生成 SQL 迁移文件并将其运行到数据库中。在终端中运行以下命令：

```bash
$ npx prisma migrate dev --name init
```

此 `prisma migrate dev` 命令会生成 SQL 文件并直接将其运行到数据库中。在这种情况下，在现有的 `prisma` 目录中创建了以下迁移文件：

```bash
$ tree prisma
prisma
├── dev.db
├── migrations
│   └── 20201207100915_init
│       └── migration.sql
└── schema.prisma
```

<details><summary>展开以查看生成的 SQL 语句</summary>

以下表在您的 SQLite 数据库中被创建：

```sql
-- CreateTable
CREATE TABLE "User" (
    "id" INTEGER NOT NULL PRIMARY KEY AUTOINCREMENT,
    "email" TEXT NOT NULL,
    "name" TEXT
);

-- CreateTable
CREATE TABLE "Post" (
    "id" INTEGER NOT NULL PRIMARY KEY AUTOINCREMENT,
    "title" TEXT NOT NULL,
    "content" TEXT,
    "published" BOOLEAN DEFAULT false,
    "authorId" INTEGER,

    FOREIGN KEY ("authorId") REFERENCES "User"("id") ON DELETE SET NULL ON UPDATE CASCADE
);

-- CreateIndex
CREATE UNIQUE INDEX "User.email_unique" ON "User"("email");
```

</details>

#### 安装并生成 Prisma Client

Prisma Client 是一个类型安全的数据库客户端，它根据您的 Prisma 模型定义 _生成_。由于这种方法，Prisma Client 可以暴露专门针对您的模型的 [CRUD](https://www.prisma.io/docs/concepts/components/prisma-client/crud) 操作。

要在您的项目中安装 Prisma Client，请在终端中运行以下命令：

```bash
$ npm install @prisma/client
```

请注意，在安装过程中，Prisma 会自动为您调用 `prisma generate` 命令。将来，每次更改 Prisma 模型后，都需要运行此命令来更新生成的 Prisma Client。

> info **注意** `prisma generate` 命令会读取您的 Prisma 模式并更新 `node_modules/@prisma/client` 中生成的 Prisma Client 库。

#### 在您的 NestJS 服务中使用 Prisma Client

现在您可以使用 Prisma Client 发送数据库查询。如果您想了解更多关于使用 Prisma Client 构建查询的信息，请查看 [API 文档](https://www.prisma.io/docs/reference/tools-and-interfaces/prisma-client/crud)。

当设置您的 NestJS 应用程序时，您希望将数据库查询的 Prisma Client API 抽象到一个服务中。为了开始，您可以创建一个新的 `PrismaService`，它负责实例化 `PrismaClient` 并连接到您的数据库。

在 `src` 目录中，创建一个名为 `prisma.service.ts` 的新文件，并添加以下代码：

```typescript
import { Injectable, OnModuleInit } from '@nestjs/common';
import { PrismaClient } from '@prisma/client';

@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit {
  async onModuleInit() {
    await this.$connect();
  }
}
```

> info **注意** `onModuleInit` 是可选的 — 如果省略它，Prisma 将在第一次调用数据库时懒惰地连接。

接下来，您可以编写服务，以便从 Prisma 模式中的 `User` 和 `Post` 模型进行数据库调用。

仍在 `src` 目录中，创建一个名为 `user.service.ts` 的新文件，并添加以下代码：

```typescript
import { Injectable } from '@nestjs/common';
import { PrismaService } from './prisma.service';
import { User, Prisma } from '@prisma/client';

@Injectable()
export class UsersService {
  constructor(private prisma: PrismaService) {}

  async user(
    userWhereUniqueInput: Prisma.UserWhereUniqueInput,
  ): Promise<User | null> {
    return this.prisma.user.findUnique({
      where: userWhereUniqueInput,
    });
  }

  async users(params: {
    skip?: number;
    take?: number;
    cursor?: Prisma.UserWhereUniqueInput;
    where?: Prisma.UserWhereInput;
    orderBy?: Prisma.UserOrderByWithRelationInput;
  }): Promise<User[]> {
    const { skip, take, cursor, where, orderBy } = params;
    return this.prisma.user.findMany({
      skip,
      take,
      cursor,
      where,
      orderBy,
    });
  }

  async createUser(data: Prisma.UserCreateInput): Promise<User> {
    return this.prisma.user.create({
      data,
    });
  }

  async updateUser(params: {
    where: Prisma.UserWhereUniqueInput;
    data: Prisma.UserUpdateInput;
  }): Promise<User> {
    const { where, data } = params;
    return this.prisma.user.update({
      data,
      where,
    });
  }

  async deleteUser(where: Prisma.UserWhereUniqueInput): Promise<User> {
    return this.prisma.user.delete({
      where,
    });
  }
}
```

请注意，您是如何使用 Prisma Client 生成的类型来确保服务暴露的方法具有正确的类型。因此，您可以节省输入模型和创建额外接口或 DTO 文件的样板代码。

现在对 `Post` 模型执行相同的操作。

仍在 `src` 目录中，创建一个名为 `post.service.ts` 的新文件，并添加以下代码：

```typescript
import { Injectable } from '@nestjs/common';
import { PrismaService } from './prisma.service';
import { Post, Prisma } from '@prisma/client';

@Injectable()
export class PostsService {
  constructor(private prisma: PrismaService) {}

  async post(
    postWhereUniqueInput: Prisma.PostWhereUniqueInput,
  ): Promise<Post | null> {
    return this.prisma.post.findUnique({
      where: postWhereUniqueInput,
    });
  }

  async posts(params: {
    skip?: number;
    take?: number;
    cursor?: Prisma.PostWhereUniqueInput;
    where?: Prisma.PostWhereInput;
    orderBy?: Prisma.PostOrderByWithRelationInput;
  }): Promise<Post[]> {
    const { skip, take, cursor, where, orderBy } = params;
    return this.prisma.post.findMany({
      skip,
      take,
      cursor,
      where,
      orderBy,
    });
  }

  async createPost(data: Prisma.PostCreateInput): Promise<Post> {
    return this.prisma.post.create({
      data,
    });
  }

  async updatePost(params: {
    where: Prisma.PostWhereUniqueInput;
    data: Prisma.PostUpdateInput;
  }): Promise<Post> {
    const { data, where } = params;
    return this.prisma.post.update({
      data,
      where,
    });
  }

  async deletePost(where: Prisma.PostWhereUniqueInput): Promise<Post> {
    return this.prisma.post.delete({
      where,
    });
  }
}
```

您的 `UsersService` 和 `PostsService` 当前封装了 Prisma Client 中可用的 CRUD 查询。在实际应用程序中，服务也是添加应用程序业务逻辑的地方。例如，您可以在 `UsersService` 中有一个名为 `updatePassword` 的方法，负责更新用户的密码。

请记得在应用程序模块中注册新服务。

##### 在主应用程序控制器中实现 REST API 路由

最后，您将使用在前面部分中创建的服务来实现应用程序的不同路由。为了简化操作，本指南中您将把所有路由放入现有的 `AppController` 类中。

将 `app.controller.ts` 文件的内容替换为以下代码：

```typescript
import {
  Controller,
  Get,
  Param,
  Post,
  Body,
  Put,
  Delete,
} from '@nestjs/common';
import { UsersService } from './user.service';
import { PostsService } from './post.service';
import { User as UserModel, Post as PostModel } from '@prisma/client';

@Controller()
export class AppController {
  constructor(
    private readonly userService: UsersService,
    private readonly postService: PostsService,
  ) {}

  @Get('post/:id')
  async getPostById(@Param('id') id: string): Promise<PostModel> {
    return this.postService.post({ id: Number(id) });
  }

  @Get('feed')
  async getPublishedPosts(): Promise<PostModel[]> {
    return this.postService.posts({
      where: { published: true },
    });
  }

  @Get('filtered-posts/:searchString')
  async getFilteredPosts(
    @Param('searchString') searchString: string,
  ): Promise<PostModel[]> {
    return this.postService.posts({
      where: {
        OR: [
          {
            title: { contains: searchString },
          },
          {
            content: { contains: searchString },
          },
        ],
      },
    });
  }

  @Post('post')
  async createDraft(
    @Body() postData: { title: string; content?: string; authorEmail: string },
  ): Promise<PostModel> {
    const { title, content, authorEmail } = postData;
    return this.postService.createPost({
      title,
      content,
      author: {
        connect: { email: authorEmail },
      },
    });
  }

  @Post('user')
  async signupUser(
    @Body() userData: { name?: string; email: string },
  ): Promise<UserModel> {
    return this.userService.createUser(userData);
  }

  @Put('publish/:id')
  async publishPost(@Param('id') id: string): Promise<PostModel> {
    return this.postService.updatePost({
      where: { id: Number(id) },
      data: { published: true },
    });
  }

  @Delete('post/:id')
  async deletePost(@Param('id') id: string): Promise<PostModel> {
    return this.postService.deletePost({ id: Number(id) });
  }
}
```

此控制器实现了以下路由：

###### `GET`

- `/post/:id`：根据 `id` 获取单个文章
- `/feed`：获取所有 _已发布_ 的文章
- `/filtered-posts/:searchString`：根据 `title` 或 `content` 过滤文章

###### `POST`

- `/post`：创建一篇新文章
  - 请求体：
    - `title: String`（必填）：文章的标题
    - `content: String`（可选）：文章的内容
    - `authorEmail: String`（必填）：创建文章的用户的邮箱
- `/user`：创建一个新用户
  - 请求体：
    - `email: String`（必填）：用户的邮箱地址
    - `name: String`（可选）：用户的名称

###### `PUT`

- `/publish/:id`：根据 `id` 发布一篇文章

###### `DELETE`

- `/post/:id`：根据 `id` 删除一篇文章

#### 总结

在本指南中，您学习了如何将 Prisma 与 NestJS 一起使用来实现 REST API。实现 API 路由的控制器调用了一个 `PrismaService`，该服务反过来使用 Prisma Client 向数据库发送查询，以满足传入请求的数据需求。

如果您想了解更多关于在 NestJS 中使用 Prisma 的信息，请务必查看以下资源：

- [NestJS & Prisma](https://www.prisma.io/nestjs)
- [REST 和 GraphQL 的即用示例项目](https://github.com/prisma/prisma-examples/)
- [生产就绪的启动套件](https://github.com/notiz-dev/nestjs-prisma-starter#instructions)
- [视频：使用 NestJS 和 Prisma 访问数据库（5分钟）](https://www.youtube.com/watch?v=UlVJ340UEuk&ab_channel=Prisma) by [Marc Stammerjohann](https://github.com/marcjulian)