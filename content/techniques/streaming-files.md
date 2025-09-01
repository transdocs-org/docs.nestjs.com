### 流式传输文件

> info **注意** 本章展示了如何从你的 **HTTP 应用程序** 流式传输文件。下面的示例不适用于 GraphQL 或微服务应用程序。

有时你可能希望将文件从 REST API 发送给客户端。使用 Nest，通常你可以这样做：

```ts
@Controller('file')
export class FileController {
  @Get()
  getFile(@Res() res: Response) {
    const file = createReadStream(join(process.cwd(), 'package.json'));
    file.pipe(res);
  }
}
```

但这样做会使你失去对控制器之后的拦截器逻辑的访问权限。为了解决这个问题，你可以返回一个 `StreamableFile` 实例，框架会在底层自动处理将流写入响应。

#### 可流式传输的文件类

`StreamableFile` 是一个用于持有待返回流的类。要创建一个新的 `StreamableFile` 实例，你可以将一个 `Buffer` 或 `Stream` 传入 `StreamableFile` 的构造函数。

> info **提示** `StreamableFile` 类可以从 `@nestjs/common` 中导入。

#### 跨平台支持

默认情况下，Fastify 可以直接支持发送文件而无需调用 `stream.pipe(res)`，因此你完全不需要使用 `StreamableFile` 类。然而，Nest 支持在两种平台类型中使用 `StreamableFile`，因此即使你在 Express 和 Fastify 之间切换，也无需担心两个引擎之间的兼容性问题。

#### 示例

下面是一个将 `package.json` 作为文件而非 JSON 数据返回的简单示例，但这个思路可以自然地扩展到图片、文档和其他任意类型的文件：

```ts
import { Controller, Get, StreamableFile } from '@nestjs/common';
import { createReadStream } from 'fs';
import { join } from 'path';

@Controller('file')
export class FileController {
  @Get()
  getFile(): StreamableFile {
    const file = createReadStream(join(process.cwd(), 'package.json'));
    return new StreamableFile(file);
  }
}
```

默认的 MIME 类型（即 HTTP 响应头 `Content-Type` 的值）是 `application/octet-stream`。如果你需要自定义这个值，可以使用 `StreamableFile` 的 `type` 选项，或者使用 `res.set` 方法或 [`@Header()`](/controllers#response-headers) 装饰器，如下所示：

```ts
import { Controller, Get, StreamableFile, Res } from '@nestjs/common';
import { createReadStream } from 'fs';
import { join } from 'path';
import type { Response } from 'express'; // 假设你使用的是 ExpressJS HTTP 适配器

@Controller('file')
export class FileController {
  @Get()
  getFile(): StreamableFile {
    const file = createReadStream(join(process.cwd(), 'package.json'));
    return new StreamableFile(file, {
      type: 'application/json',
      disposition: 'attachment; filename="package.json"',
      // 如果你想将 Content-Length 设置为不同于文件大小的值：
      // length: 123,
    });
  }

  // 或者：
  @Get()
  getFileChangingResponseObjDirectly(@Res({ passthrough: true }) res: Response): StreamableFile {
    const file = createReadStream(join(process.cwd(), 'package.json'));
    res.set({
      'Content-Type': 'application/json',
      'Content-Disposition': 'attachment; filename="package.json"',
    });
    return new StreamableFile(file);
  }

  // 或者：
  @Get()
  @Header('Content-Type', 'application/json')
  @Header('Content-Disposition', 'attachment; filename="package.json"')
  getFileUsingStaticValues(): StreamableFile {
    const file = createReadStream(join(process.cwd(), 'package.json'));
    return new StreamableFile(file);
  }  
}
```