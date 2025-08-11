### 平台无关性

Nest 是一个平台无关的框架。这意味着你可以开发 **可重用的逻辑组件**，这些组件可以在不同种类的应用程序中使用。例如，大多数组件可以在不同的底层 HTTP 服务器框架（如 Express 和 Fastify）之间无需修改即可重用，甚至可以在不同 _类型_ 的应用程序之间重用（如 HTTP 服务器框架、使用不同传输层的微服务以及 Web Sockets）。

#### 一次构建，到处使用

文档中的 **概览** 部分主要展示了基于 HTTP 服务器框架的编码技术（例如提供 REST API 或提供 MVC 风格服务器端渲染页面的应用程序）。然而，所有这些构建模块都可以在不同传输层之上使用（[微服务](/microservices/basics) 或 [WebSockets](/websockets/gateways)）。

此外，Nest 还提供了专门的 [GraphQL](/graphql/quick-start) 模块。你可以将 GraphQL 作为 API 层与 REST API 互换使用。

另外，[应用上下文](/application-context) 功能帮助你在 Nest 的基础上创建各种类型的 Node.js 应用程序——包括 CRON 任务和命令行工具等。

Nest 旨在成为一个功能完善的 Node.js 应用平台，为你的应用程序带来更高层次的模块化和可重用性。一次构建，到处使用！