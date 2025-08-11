### CI/CD 集成

> info **提示** 本章介绍 Nest Devtools 与 Nest 框架的集成。如果您正在寻找 Devtools 应用，请访问 [Devtools](https://devtools.nestjs.com) 网站。

CI/CD 集成功能仅对使用 **[企业版](/settings)** 计划的用户可用。

您可以观看以下视频了解 CI/CD 集成的价值与使用方法：

<figure>
  <iframe
    width="1000"
    height="565"
    src="https://www.youtube.com/embed/r5RXcBrnEQ8"
    title="YouTube 视频播放器"
    frameBorder="0"
    allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
    allowFullScreen
  ></iframe>
</figure>

#### 发布依赖图

首先，我们需要配置应用程序的启动文件 (`main.ts`)，使用 `GraphPublisher` 类（该类导出自 `@nestjs/devtools-integration` 模块 —— 详见前一章节），如下所示：

```typescript
async function bootstrap() {
  const shouldPublishGraph = process.env.PUBLISH_GRAPH === "true";

  const app = await NestFactory.create(AppModule, {
    snapshot: true,
    preview: shouldPublishGraph,
  });

  if (shouldPublishGraph) {
    await app.init();

    const publishOptions = { ... } // 注意：这个 options 对象会根据您使用的 CI/CD 提供商而有所不同
    const graphPublisher = new GraphPublisher(app);
    await graphPublisher.publish(publishOptions);

    await app.close();
  } else {
    await app.listen(process.env.PORT ?? 3000);
  }
}
```

如上所示，我们在这里使用 `GraphPublisher` 将序列化的依赖图发布到中心化注册中心。`PUBLISH_GRAPH` 是一个自定义环境变量，用于控制是否发布依赖图（在 CI/CD 工作流中）或不发布（在常规应用启动时）。此外，我们还设置了 `preview` 属性为 `true`。启用此标志后，我们的应用程序将以预览模式启动 —— 这意味着应用程序中所有控制器、增强器和提供者的构造函数（以及生命周期钩子）将不会被执行。请注意，这并不是**必须的**，但在 CI/CD 流水线中运行应用程序时可以简化流程，因为我们无需连接数据库等外部资源。

`publishOptions` 对象会根据您使用的 CI/CD 提供商不同而有所变化。我们将在后续章节中提供最主流 CI/CD 提供商的配置说明。

一旦依赖图成功发布，您将在工作流视图中看到如下输出：

<figure><img src="/assets/devtools/graph-published-terminal.png" /></figure>

每次发布依赖图后，您应该在对应项目的页面上看到一个新条目：

<figure><img src="/assets/devtools/project.png" /></figure>

#### 报告生成

只有当中心化注册中心中已经存在对应的快照时，Devtools 才会为每次构建生成报告。例如，如果您创建了一个针对 `master` 分支的 PR，而该分支的依赖图已经被发布过，那么应用将能够检测差异并生成报告。否则，报告将不会被生成。

要查看报告，请导航到对应项目的页面（参见组织管理）：

<figure><img src="/assets/devtools/report.png" /></figure>

这一功能在代码审查中特别有用，可以帮助我们发现一些可能被忽略的变更。例如，假设有人更改了某个**深层嵌套提供者**的作用域，这种变化可能不会立刻被审查者注意到。而通过 Devtools，我们可以轻松发现此类变更，并确认是否为有意为之。或者，如果您从某个端点中移除了守卫（Guard），它将在报告中显示为受影响项。如果我们没有为该路由编写集成测试或 E2E 测试，可能不会注意到它已经不再受保护，等到发现问题时可能已经为时已晚。

同样，如果我们正在处理一个**大型代码库**，并修改了某个模块为全局模块（`@Global()`），我们将看到依赖图中新增了多少条边。在大多数情况下，这表明我们可能做错了某些事情。

#### 构建预览

对于每一个已发布的依赖图，我们可以通过点击 **Preview**（预览）按钮，查看它过去的样子。此外，如果报告已生成，我们还可以在依赖图上看到差异高亮显示：

- 绿色节点表示新增的元素
- 浅白色节点表示更新的元素
- 红色节点表示删除的元素

请参见以下截图：

<figure><img src="/assets/devtools/nodes-selection.png" /></figure>

这种“回溯时间”的能力使我们可以通过对比当前依赖图与之前的版本来调查和排查问题。具体取决于您的配置方式，每个 Pull Request（甚至每次提交）都会在注册中心中保留一个对应的快照，因此您可以轻松地回溯并查看变更内容。可以把 Devtools 想象成 Git，但它理解 Nest 是如何构建应用程序依赖图的，并且具备**可视化**展示的能力。

#### 集成：GitHub Actions

首先，让我们在项目中的 `.github/workflows` 目录下创建一个新的 GitHub 工作流文件，并将其命名为 `publish-graph.yml`。在该文件中使用如下定义：

```yaml
name: Devtools

on:
  push:
    branches:
      - master
  pull_request:
    branches:
      - '*'

jobs:
  publish:
    if: github.actor!= 'dependabot[bot]'
    name: Publish graph
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '16'
          cache: 'npm'
      - name: Install dependencies
        run: npm ci
      - name: Setup Environment (PR)
        if: {{ '${{' }} github.event_name == 'pull_request' {{ '}}' }}
        shell: bash
        run: |
          echo "COMMIT_SHA={{ '${{' }} github.event.pull_request.head.sha {{ '}}' }}" >>\${GITHUB_ENV}
      - name: Setup Environment (Push)
        if: {{ '${{' }} github.event_name == 'push' {{ '}}' }}
        shell: bash
        run: |
          echo "COMMIT_SHA=\${GITHUB_SHA}" >> \${GITHUB_ENV}
      - name: Publish
        run: PUBLISH_GRAPH=true npm run start
        env:
          DEVTOOLS_API_KEY: CHANGE_THIS_TO_YOUR_API_KEY
          REPOSITORY_NAME: {{ '${{' }} github.event.repository.name {{ '}}' }}
          BRANCH_NAME: {{ '${{' }} github.head_ref || github.ref_name {{ '}}' }}
          TARGET_SHA: {{ '${{' }} github.event.pull_request.base.sha {{ '}}' }}
```

理想情况下，`DEVTOOLS_API_KEY` 环境变量应从 GitHub Secrets 中获取，请参见 [这里](https://docs.github.com/en/actions/security-guides/encrypted-secrets#creating-encrypted-secrets-for-a-repository) 获取更多信息。

该工作流会在每次针对 `master` 分支的 Pull Request 或直接提交到 `master` 分支的 Commit 时运行。您可以根据项目需求调整此配置。关键在于我们为 `GraphPublisher` 类提供了必要的环境变量（以便其运行）。

但在使用该工作流前，我们需要更新一个变量：`DEVTOOLS_API_KEY`。我们可以在 [此页面](https://devtools.nestjs.com/settings/manage-api-keys) 为项目生成一个专用的 API 密钥。

最后，让我们再次打开 `main.ts` 文件，更新之前留空的 `publishOptions` 对象：

```typescript
const publishOptions = {
  apiKey: process.env.DEVTOOLS_API_KEY,
  repository: process.env.REPOSITORY_NAME,
  owner: process.env.GITHUB_REPOSITORY_OWNER,
  sha: process.env.COMMIT_SHA,
  target: process.env.TARGET_SHA,
  trigger: process.env.GITHUB_BASE_REF ? 'pull' : 'push',
  branch: process.env.BRANCH_NAME,
};
```

为了获得最佳开发体验，建议通过点击“Integrate GitHub app”按钮将 GitHub 应用集成到您的项目中（见下图）。注意：这不是必须的。

<figure><img src="/assets/devtools/integrate-github-app.png" /></figure>

集成后，您将能够在 Pull Request 中直接看到预览/报告生成的状态：

<figure><img src="/assets/devtools/actions-preview.png" /></figure>

#### 集成：GitLab Pipelines

首先，让我们在项目根目录下创建一个新的 GitLab CI 配置文件，并将其命名为 `.gitlab-ci.yml`。在该文件中使用如下定义：

```typescript
const publishOptions = {
  apiKey: process.env.DEVTOOLS_API_KEY,
  repository: process.env.REPOSITORY_NAME,
  owner: process.env.GITHUB_REPOSITORY_OWNER,
  sha: process.env.COMMIT_SHA,
  target: process.env.TARGET_SHA,
  trigger: process.env.GITHUB_BASE_REF ? 'pull' : 'push',
  branch: process.env.BRANCH_NAME,
};
```

> info **提示** 理想情况下，`DEVTOOLS_API_KEY` 环境变量应从密钥管理服务中获取。

该工作流将在每次针对 `master` 分支的 Pull Request 或直接提交到 `master` 分支的 Commit 时运行。您可以根据项目需求调整此配置。关键在于我们为 `GraphPublisher` 类提供了必要的环境变量（以便其运行）。

但在此工作流定义中，有一个变量需要在使用前进行更新：`DEVTOOLS_API_KEY`。我们可以在 [此页面](https://devtools.nestjs.com/settings/manage-api-keys) 为项目生成一个专用的 API 密钥。

最后，让我们再次打开 `main.ts` 文件，更新之前留空的 `publishOptions` 对象：

```yaml
image: node:16

stages:
  - build

cache:
  key:
    files:
      - package-lock.json
  paths:
    - node_modules/

workflow:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      when: always
    - if: $CI_COMMIT_BRANCH == "master" && $CI_PIPELINE_SOURCE == "push"
      when: always
    - when: never

install_dependencies:
  stage: build
  script:
    - npm ci

publish_graph:
  stage: build
  needs:
    - install_dependencies
  script: npm run start
  variables:
    PUBLISH_GRAPH: 'true'
    DEVTOOLS_API_KEY: 'CHANGE_THIS_TO_YOUR_API_KEY'
```

#### 其他 CI/CD 工具

Nest Devtools 的 CI/CD 集成可以与任何您选择的 CI/CD 工具配合使用（例如 [Bitbucket Pipelines](https://bitbucket.org/product/features/pipelines)、[CircleCI](https://circleci.com/) 等），因此请不要局限于我们上面提到的提供商。

查看以下 `publishOptions` 对象配置，以了解发布某个 Commit/Build/PR 的依赖图所需的信息：

```typescript
const publishOptions = {
  apiKey: process.env.DEVTOOLS_API_KEY,
  repository: process.env.CI_PROJECT_NAME,
  owner: process.env.CI_PROJECT_ROOT_NAMESPACE,
  sha: process.env.CI_COMMIT_SHA,
  target: process.env.CI_MERGE_REQUEST_DIFF_BASE_SHA,
  trigger: process.env.CI_MERGE_REQUEST_DIFF_BASE_SHA ? 'pull' : 'push',
  branch: process.env.CI_COMMIT_BRANCH ?? process.env.CI_MERGE_REQUEST_SOURCE_BRANCH_NAME,
};
```

大部分信息可以通过 CI/CD 内置的环境变量获取（例如 [CircleCI 内置变量列表](https://circleci.com/docs/variables/#built-in-environment-variables) 或 [Bitbucket 变量文档](https://support.atlassian.com/bitbucket-cloud/docs/variables-and-secrets/)）。

关于发布依赖图的流水线配置，我们建议使用以下触发器：

- `push` 事件 —— 仅当当前分支代表部署环境时，例如 `master`、`main`、`staging`、`production` 等
- `pull request` 事件 —— 始终触发，或仅当目标分支代表部署环境时触发（参见上文）