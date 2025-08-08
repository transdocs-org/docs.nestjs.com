### 部署

当你准备将 NestJS 应用程序部署到生产环境时，可以采取一些关键步骤来确保它尽可能高效地运行。在本指南中，我们将探讨一些重要的技巧和最佳实践，以帮助你成功部署 NestJS 应用程序。

#### 前提条件

在部署 NestJS 应用程序之前，请确保你已完成以下准备：

- 一个已准备好部署的 NestJS 应用程序。
- 可用于托管应用程序的部署平台或服务器。
- 已配置好应用程序所需的所有环境变量。
- 所需的服务（如数据库）已设置并准备就绪。
- 部署平台上已安装至少 LTS 版本的 Node.js。

> info **提示** 如果你正在寻找一个基于云的平台来部署你的 NestJS 应用程序，请查看 [Mau](https://mau.nestjs.com/ 'Deploy Nest')，这是我们官方提供的在 AWS 上部署 NestJS 应用程序的平台。使用 Mau，只需点击几个按钮并运行一条命令即可轻松部署你的 NestJS 应用程序：
>
> ```bash
> $ npm install -g @nestjs/mau
> $ mau deploy
> ```
>
> 部署完成后，你的 NestJS 应用程序将在几秒钟内运行在 AWS 上！

#### 构建你的应用程序

要构建你的 NestJS 应用程序，你需要将 TypeScript 代码编译为 JavaScript。此过程会生成一个包含编译文件的 `dist` 目录。你可以通过运行以下命令来构建你的应用程序：

```bash
$ npm run build
```

此命令通常会在底层运行 `nest build` 命令，它本质上是对 TypeScript 编译器的封装，并具有一些额外功能（如资源文件复制等）。如果你有自定义的构建脚本，可以直接运行它。此外，对于使用 NestJS CLI 的单仓库项目，请确保将要构建的项目名称作为参数传递（例如 `npm run build my-app`）。

编译成功后，你应该会在项目根目录下看到一个 `dist` 目录，其中包含编译后的文件，入口文件为 `main.js`。如果你在项目根目录中有一些 `.ts` 文件（并且你的 `tsconfig.json` 配置为编译这些文件），它们也会被复制到 `dist` 目录中，从而略微改变目录结构（例如从 `dist/main.js` 变为 `dist/src/main.js`，因此在配置服务器时请注意这一点）。

#### 生产环境

你的生产环境是应用程序对外提供服务的地方。这可以是基于云的平台，如 [AWS](https://aws.amazon.com/)（包括 EC2、ECS 等）、[Azure](https://azure.microsoft.com/) 或 [Google Cloud](https://cloud.google.com/)，也可以是你自己管理的专用服务器，如 [Hetzner](https://www.hetzner.com/)。

为了简化部署流程并避免手动配置，你可以使用 [Mau](https://mau.nestjs.com/ 'Deploy Nest') 这样的服务，这是我们官方提供的在 AWS 上部署 NestJS 应用程序的平台。更多详情请查看 [这一节](todo)。

使用像 [Mau](https://mau.nestjs.com/ 'Deploy Nest') 这样的**云平台**或服务的一些优点包括：

- **可扩展性**：随着用户数量增长，可以轻松扩展你的应用程序。
- **安全性**：享受内置的安全功能和合规性认证。
- **监控**：实时监控应用程序的性能和健康状况。
- **可靠性**：确保应用程序始终可用，并提供高可用性保证。

另一方面，云平台通常比自托管更昂贵，并且你对底层基础设施的控制可能较少。如果你正在寻找更具成本效益的解决方案并且具备自行管理服务器的技术能力，简单的 VPS 可能是一个不错的选择，但请注意你需要手动处理服务器维护、安全和备份等任务。

#### NODE_ENV=production

虽然在 Node.js 和 NestJS 中开发环境和生产环境之间没有技术上的区别，但在生产环境中运行应用程序时，最好将 `NODE_ENV` 环境变量设置为 `production`，因为生态系统中的一些库可能会根据此变量表现出不同的行为（例如启用或禁用调试输出等）。

你可以在启动应用程序时设置 `NODE_ENV` 环境变量，如下所示：

```bash
$ NODE_ENV=production node dist/main.js
```

或者在你的云平台/Mau 仪表板中设置。

#### 运行你的应用程序

要在生产环境中运行你的 NestJS 应用程序，只需使用以下命令：

```bash
$ node dist/main.js # 请根据你的入口文件位置进行调整
```

此命令将启动你的应用程序，并监听指定的端口（通常默认为 `3000`）。请确保这与你在应用程序中配置的端口一致。

另外，你也可以使用 `nest start` 命令。这个命令本质上是 `node dist/main.js` 的封装，但它有一个关键区别：它会在启动应用程序之前自动运行 `nest build`，因此你无需手动执行 `npm run build`。

#### 健康检查

健康检查对于监控你的 NestJS 应用程序在生产环境中的健康状态至关重要。通过设置一个健康检查端点，你可以定期验证应用程序是否正常运行，并在问题变得严重之前作出响应。

在 NestJS 中，你可以使用 **@nestjs/terminus** 包轻松实现健康检查，该包提供了强大的工具用于添加健康检查，包括数据库连接、外部服务以及自定义检查。

请查看 [本指南](/recipes/terminus) 了解如何在你的 NestJS 应用程序中实现健康检查，并确保你的应用程序始终处于被监控和响应状态。

#### 日志记录

日志记录对于任何生产就绪的应用程序都是必不可少的。它有助于跟踪错误、监控行为并进行故障排查。在 NestJS 中，你可以使用内置的日志记录器，或者选择使用外部库以获得更高级的功能。

日志记录的最佳实践：

- **记录错误，而非异常**：专注于记录详细的错误信息，以加快调试和问题解决速度。
- **避免记录敏感数据**：不要记录密码或令牌等敏感信息，以保护安全性。
- **使用关联 ID**：在分布式系统中，应在日志中包含唯一标识符（如关联 ID），以便追踪跨不同服务的请求。
- **使用日志级别**：按严重程度对日志进行分类（例如 `info`、`warn`、`error`），并在生产环境中禁用调试或详细日志以减少噪音。

> info **提示** 如果你正在使用 [AWS](https://aws.amazon.com/)（通过 [Mau](https://mau.nestjs.com/ 'Deploy Nest') 或直接使用），建议使用 JSON 格式日志，以便更轻松地解析和分析日志。

对于分布式应用程序，使用像 ElasticSearch、Loggly 或 Datadog 这样的集中式日志服务可能非常有用。这些工具提供了日志聚合、搜索和可视化等强大功能，使你更容易监控和分析应用程序的性能和行为。

#### 横向或纵向扩展

有效地扩展你的 NestJS 应用程序对于处理增加的流量并确保最佳性能至关重要。扩展的两种主要策略是 **纵向扩展** 和 **横向扩展**。理解这些方法将帮助你设计应用程序以高效管理负载。

**纵向扩展**，通常称为“向上扩展”，是指通过增加单个服务器的资源来提高其性能。这可能意味着为现有机器增加更多的 CPU、内存或存储空间。以下是一些需要考虑的关键点：

- **简单性**：纵向扩展通常更容易实现，因为你只需升级现有服务器，而无需管理多个实例。
- **限制**：单台机器的扩展存在物理限制。一旦达到最大容量，你可能需要考虑其他选项。
- **成本效益**：对于流量适中的应用程序，纵向扩展具有成本效益，因为它减少了对额外基础设施的需求。

示例：如果你的 NestJS 应用程序托管在虚拟机上，并且在高峰时段运行缓慢，你可以将虚拟机升级为具有更多资源的更大实例。要升级虚拟机，只需导航到当前提供商的仪表板并选择更大的实例类型。

**横向扩展**，或称为“向外扩展”，是指通过添加更多服务器或实例来分担负载。这种策略在云环境中广泛使用，对于预期流量较高的应用程序至关重要。以下是其优点和注意事项：

- **增加容量**：通过添加更多应用程序实例，你可以处理更多并发用户，而不会降低性能。
- **冗余性**：横向扩展提供了冗余性，因为一个服务器的故障不会导致整个应用程序宕机。流量可以在剩余服务器之间重新分配。
- **负载均衡**：为了有效管理多个实例，请使用负载均衡器（如 Nginx 或 AWS Elastic Load Balancing）将传入流量均匀分配到你的服务器。

示例：对于流量较高的 NestJS 应用程序，你可以在云环境中部署多个应用程序实例，并使用负载均衡器路由请求，确保没有单个实例成为瓶颈。

使用像 [Docker](https://www.docker.com/) 这样的容器化技术以及 [Kubernetes](https://kubernetes.io/) 等容器编排平台，可以轻松实现这一过程。此外，你还可以利用云平台特定的负载均衡器，如 [AWS Elastic Load Balancing](https://aws.amazon.com/elasticloadbalancing/) 或 [Azure Load Balancer](https://azure.microsoft.com/en-us/services/load-balancer/)，将流量分配到你的应用程序实例。

> info **提示** [Mau](https://mau.nestjs.com/ 'Deploy Nest') 在 AWS 上提供了对横向扩展的内置支持，允许你轻松部署多个 NestJS 应用程序实例，并通过几次点击即可管理它们。

#### 其他一些建议

在部署 NestJS 应用程序时，还有一些其他提示需要注意：

- **安全性**：确保你的应用程序安全，防止 SQL 注入、XSS 等常见威胁。详见“安全性”类别。
- **监控**：使用 [Prometheus](https://prometheus.io/) 或 [New Relic](https://newrelic.com/) 等监控工具来跟踪应用程序的性能和健康状况。如果你使用的是云平台/Mau，它们可能提供内置的监控服务（如 [AWS CloudWatch](https://aws.amazon.com/cloudwatch/) 等）。
- **不要硬编码环境变量**：避免在代码中硬编码 API 密钥、密码或令牌等敏感信息。使用环境变量或密钥管理服务来安全地存储和访问这些值。
- **备份**：定期备份数据，以防止发生意外事件导致数据丢失。
- **自动化部署**：使用 CI/CD 管道自动化部署流程，确保环境之间的一致性。
- **限流**：实施限流以防止滥用并保护应用程序免受 DDoS 攻击。详见 [限流章节](/security/rate-limiting)，或使用 [AWS WAF](https://aws.amazon.com/waf/) 等服务以获得更高级的保护。

#### 使用 Docker 容器化你的应用程序

[Docker](https://www.docker.com/) 是一个使用容器化技术的平台，允许开发人员将应用程序及其依赖项打包到一个称为容器的标准单元中。容器轻量、便携且隔离，非常适合在各种环境中部署应用程序，从本地开发到生产环境。

将 NestJS 应用程序容器化的优点包括：

- **一致性**：Docker 确保你的应用程序在任何机器上都以相同方式运行，消除了“在我机器上能运行”的问题。
- **隔离性**：每个容器在隔离的环境中运行，防止依赖项之间的冲突。
- **可扩展性**：Docker 使得通过在不同机器或云实例上运行多个容器来轻松扩展应用程序。
- **便携性**：容器可以在不同环境之间轻松移动，使你在不同平台上部署应用程序变得简单。

要安装 Docker，请按照 [官方网站](https://www.docker.com/get-started) 上的说明进行操作。安装 Docker 后，你可以在你的 NestJS 项目中创建一个 `Dockerfile` 来定义构建容器镜像的步骤。

`Dockerfile` 是一个文本文件，包含 Docker 用于构建容器镜像的指令。

下面是一个 NestJS 应用程序的示例 Dockerfile：

```bash
# 使用官方的 Node.js 镜像作为基础镜像
FROM node:20

# 设置容器内的工作目录
WORKDIR /usr/src/app

# 将 package.json 和 package-lock.json 复制到工作目录
COPY package*.json ./

# 安装应用程序依赖
RUN npm install

# 复制其余的应用程序文件
COPY . .

# 构建 NestJS 应用程序
RUN npm run build

# 暴露应用程序端口
EXPOSE 3000

# 运行应用程序的命令
CMD ["node", "dist/main"]
```

> info **提示** 请确保将 `node:20` 替换为你在项目中使用的适当的 Node.js 版本。你可以在 [Docker Hub 官方仓库](https://hub.docker.com/_/node) 中找到可用的 Node.js Docker 镜像。

这是一个基本的 Dockerfile，它设置了 Node.js 环境，安装了应用程序依赖项，构建了 NestJS 应用程序并运行它。你可以根据项目需求自定义此文件（例如使用不同的基础镜像、优化构建流程、仅安装生产依赖等）。

我们还应该创建一个 `.dockerignore` 文件，以指定 Docker 在构建镜像时应忽略哪些文件和目录。在你的项目根目录中创建一个 `.dockerignore` 文件：

```bash
node_modules
dist
*.log
*.md
.git
```

此文件确保不必要的文件不会包含在容器镜像中，使其保持轻量。现在你已经配置好了 Dockerfile，可以构建你的 Docker 镜像。打开终端，导航到你的项目目录，并运行以下命令：

```bash
docker build -t my-nestjs-app .
```

在此命令中：

- `-t my-nestjs-app`：将镜像标记为 `my-nestjs-app`。
- `.`：表示当前目录作为构建上下文。

构建完成后，你可以将其作为容器运行。执行以下命令：

```bash
docker run -p 3000:3000 my-nestjs-app
```

在此命令中：

- `-p 3000:3000`：将主机上的 3000 端口映射到容器内的 3000 端口。
- `my-nestjs-app`：指定要运行的镜像。

你的 NestJS 应用程序现在应该在 Docker 容器中运行。

如果你想将 Docker 镜像部署到云平台或与他人共享，则需要将其推送到 Docker 仓库（如 [Docker Hub](https://hub.docker.com/)、[AWS ECR](https://aws.amazon.com/ecr/) 或 [Google Container Registry](https://cloud.google.com/container-registry)）。

确定仓库后，你可以按照以下步骤推送镜像：

```bash
docker login # 登录到你的 Docker 仓库
docker tag my-nestjs-app your-dockerhub-username/my-nestjs-app # 标记你的镜像
docker push your-dockerhub-username/my-nestjs-app # 推送你的镜像
```

将 `your-dockerhub-username` 替换为你的 Docker Hub 用户名或相应的仓库 URL。推送镜像后，你可以在任何机器上拉取并运行它作为容器。

像 AWS、Azure 和 Google Cloud 这样的云平台提供了托管容器服务，简化了大规模部署和管理容器的过程。这些服务提供自动扩展、负载均衡和监控等功能，使你更容易在生产环境中运行你的 NestJS 应用程序。

#### 使用 Mau 简化部署

[Mau](https://mau.nestjs.com/ 'Deploy Nest') 是我们官方提供的在 [AWS](https://aws.amazon.com/) 上部署 NestJS 应用程序的平台。如果你尚未准备好手动管理基础设施（或只是想节省时间），Mau 是你的理想解决方案。

使用 Mau，配置和维护基础设施就像点击几个按钮一样简单。Mau 设计得简单直观，使你可以专注于构建应用程序，而无需担心底层基础设施。在底层，我们使用 **Amazon Web Services** 为你提供强大且可靠的平台，同时隐藏了 AWS 的所有复杂性。我们为你处理所有繁重的工作，使你可以专注于构建应用程序并发展业务。

[Mau](https://mau.nestjs.com/ 'Deploy Nest') 非常适合初创公司、中小型企业和大型企业，以及希望快速上手而不必花大量时间学习和管理基础设施的开发人员。它非常易于使用，几分钟内即可让基础设施运行起来。同时，它还利用了 AWS 的幕后支持，为你提供 AWS 的所有优势，而无需处理其复杂性。

<figure><img src="/assets/mau-metrics.png" /></figure>

使用 [Mau](https://mau.nestjs.com/ 'Deploy Nest')，你可以：

- 仅需几次点击即可部署你的 NestJS 应用程序（如 API、微服务等）。
- 配置数据库，例如：
  - PostgreSQL
  - MySQL
  - MongoDB (DocumentDB)
  - Redis
  - 更多
- 配置消息代理服务，例如：
  - RabbitMQ
  - Kafka
  - NATS
- 部署定时任务（CRON 作业）和后台工作者。
- 部署 Lambda 函数和无服务器应用程序。
- 配置用于自动部署的 **CI/CD 管道**。
- 以及更多功能！

要使用 Mau 部署你的 NestJS 应用程序，只需运行以下命令：

```bash
$ npm install -g @nestjs/mau
$ mau deploy
```

立即注册并 [使用 Mau 部署](https://mau.nestjs.com/ 'Deploy Nest')，让你的 NestJS 应用程序在几分钟内运行在 AWS 上！