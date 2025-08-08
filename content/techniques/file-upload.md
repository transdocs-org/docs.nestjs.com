### 文件上传

为了处理文件上传，Nest 提供了一个基于 [multer](https://github.com/expressjs/multer) 中间件包的内置模块，该包用于 Express。Multer 能够处理以 `multipart/form-data` 格式发布的数据，这种格式主要用于通过 HTTP `POST` 请求上传文件。该模块是完全可配置的，您可以根据应用程序需求调整其行为。

> warning **警告** Multer 无法处理非支持的 multipart 格式（`multipart/form-data`）。同时请注意，此包与 `FastifyAdapter` 不兼容。

为了更好的类型安全，我们安装 Multer 的类型定义包：

```shell
$ npm i -D @types/multer
```

安装此包后，我们现在可以使用 `Express.Multer.File` 类型（您可以按如下方式导入此类型：`import {{ '{' }} Express {{ '}' }} from 'express'`）。

#### 基本示例

要上传单个文件，只需将 `FileInterceptor()` 拦截器绑定到路由处理程序，并使用 `@UploadedFile()` 装饰器从 `request` 中提取 `file`。

```typescript
@@filename()
@Post('upload')
@UseInterceptors(FileInterceptor('file'))
uploadFile(@UploadedFile() file: Express.Multer.File) {
  console.log(file);
}
@@switch
@Post('upload')
@UseInterceptors(FileInterceptor('file'))
@Bind(UploadedFile())
uploadFile(file) {
  console.log(file);
}
```

> info **提示** `FileInterceptor()` 装饰器导自 `@nestjs/platform-express` 包。`@UploadedFile()` 装饰器导自 `@nestjs/common`。

`FileInterceptor()` 装饰器接受两个参数：

- `fieldName`: 提供一个字符串，表示 HTML 表单中保存文件的字段名称
- `options`: 类型为 `MulterOptions` 的可选对象。这与 multer 构造函数使用的对象相同（更多细节[此处](https://github.com/expressjs/multer#multeropts)）。

> warning **警告** `FileInterceptor()` 可能与第三方云提供商（如 Google Firebase 或其他）不兼容。

#### 文件验证

通常，验证传入文件的元数据可能很有用，例如文件大小或文件 MIME 类型。为此，您可以创建自己的 [Pipe](https://docs.nestjs.com/pipes)，并将其绑定到使用 `UploadedFile` 装饰器注解的参数。以下示例演示了如何实现一个基本的文件大小验证器管道：

```typescript
import { PipeTransform, Injectable, ArgumentMetadata } from '@nestjs/common';

@Injectable()
export class FileSizeValidationPipe implements PipeTransform {
  transform(value: any, metadata: ArgumentMetadata) {
    // "value" 是一个包含文件属性和元数据的对象
    const oneKb = 1000;
    return value.size < oneKb;
  }
}
```

这可以在 `FileInterceptor` 中如下使用：

```typescript
@Post('file')
@UseInterceptors(FileInterceptor('file'))
uploadFileAndValidate(@UploadedFile(
  new FileSizeValidationPipe(),
  // 其他管道可以在这里添加
) file: Express.Multer.File, ) {
  return file;
}
```

Nest 提供了一个内置管道来处理常见用例，并促进/标准化新用例的添加。这个管道称为 `ParseFilePipe`，您可以如下使用它：

```typescript
@Post('file')
uploadFileAndPassValidation(
  @Body() body: SampleDto,
  @UploadedFile(
    new ParseFilePipe({
      validators: [
        // ... 这里放置一组文件验证器实例
      ]
    })
  )
  file: Express.Multer.File,
) {
  return {
    body,
    file: file.buffer.toString(),
  };
}
```

如您所见，必须指定由 `ParseFilePipe` 执行的一组文件验证器。我们将讨论验证器的接口，但值得一提的是，这个管道还有两个额外的**可选**选项：

<table>
  <tr>
    <td><code>errorHttpStatusCode</code></td>
    <td>在任何验证器失败的情况下抛出的 HTTP 状态码。默认是 <code>400</code> (BAD REQUEST)</td>
  </tr>
  <tr>
    <td><code>exceptionFactory</code></td>
    <td>一个工厂，接收错误消息并返回一个错误。</td>
  </tr>
</table>

现在，回到 `FileValidator` 接口。要将验证器与此管道集成，您必须使用内置实现或提供自己的自定义 `FileValidator`。请参见以下示例：

```typescript
export abstract class FileValidator<TValidationOptions = Record<string, any>> {
  constructor(protected readonly validationOptions: TValidationOptions) {}

  /**
   * 根据构造函数中传递的选项，指示此文件是否应被视为有效。
   * @param file 请求对象中的文件
   */
  abstract isValid(file?: any): boolean | Promise<boolean>;

  /**
   * 在验证失败时构建错误消息。
   * @param file 请求对象中的文件
   */
  abstract buildErrorMessage(file: any): string;
}
```

> info **提示** `FileValidator` 接口通过其 `isValid` 函数支持异步验证。为了利用类型安全性，如果您使用 express（默认）作为驱动程序，还可以将 `file` 参数类型化为 `Express.Multer.File`。

`FileValidator` 是一个常规类，可以访问文件对象并根据客户端提供的选项对其进行验证。Nest 有两个内置的 `FileValidator` 实现，您可以在项目中使用它们：

- `MaxFileSizeValidator` - 检查给定文件的大小是否小于提供的值（以 `bytes` 为单位）
- `FileTypeValidator` - 检查给定文件的 MIME 类型是否匹配提供的字符串或正则表达式。默认情况下，使用文件内容的 [magic number](https://www.ibm.com/support/pages/what-magic-number) 验证 MIME 类型

为了理解这些如何与前述的 `FileParsePipe` 一起使用，我们将使用最后一个示例的修改片段：

```typescript
@UploadedFile(
  new ParseFilePipe({
    validators: [
      new MaxFileSizeValidator({ maxSize: 1000 }),
      new FileTypeValidator({ fileType: 'image/jpeg' }),
    ],
  }),
)
file: Express.Multer.File,
```

> info **提示** 如果验证器的数量大量增加或其选项使文件变得混乱，您可以将此数组在单独的文件中定义并作为命名常量（如 `fileValidators`）导入此处。

最后，您可以使用特殊的 `ParseFilePipeBuilder` 类，它允许您组合和构建验证器。通过如下使用它可以避免手动实例化每个验证器，只需直接传递其选项即可：

```typescript
@UploadedFile(
  new ParseFilePipeBuilder()
    .addFileTypeValidator({
      fileType: 'jpeg',
    })
    .addMaxSizeValidator({
      maxSize: 1000
    })
    .build({
      errorHttpStatusCode: HttpStatus.UNPROCESSABLE_ENTITY
    }),
)
file: Express.Multer.File,
```

> info **提示** 默认情况下需要文件存在，但您可以通过在 `build` 函数选项中添加 `fileIsRequired: false` 参数（与 `errorHttpStatusCode` 同级）使其可选。

#### 文件数组

要上传一个由单个字段名标识的文件数组，请使用 `FilesInterceptor()` 装饰器（注意装饰器名称中的复数 **Files**）。此装饰器接受三个参数：

- `fieldName`: 如上所述
- `maxCount`: 可选数字，定义接受的最大文件数
- `options`: 如上所述的 `MulterOptions` 对象

使用 `FilesInterceptor()` 时，使用 `@UploadedFiles()` 装饰器从 `request` 中提取文件。

```typescript
@@filename()
@Post('upload')
@UseInterceptors(FilesInterceptor('files'))
uploadFile(@UploadedFiles() files: Array<Express.Multer.File>) {
  console.log(files);
}
@@switch
@Post('upload')
@UseInterceptors(FilesInterceptor('files'))
@Bind(UploadedFiles())
uploadFile(files) {
  console.log(files);
}
```

> info **提示** `FilesInterceptor()` 装饰器导自 `@nestjs/platform-express` 包。`@UploadedFiles()` 装饰器导自 `@nestjs/common`。

#### 多个文件

要上传多个文件（所有文件都有不同的字段名键），请使用 `FileFieldsInterceptor()` 装饰器。此装饰器接受两个参数：

- `uploadedFields`: 一个对象数组，其中每个对象指定一个必需的 `name` 属性（字符串值，表示字段名，如上所述）和一个可选的 `maxCount` 属性（如上所述）
- `options`: 如上所述的 `MulterOptions` 对象

使用 `FileFieldsInterceptor()` 时，使用 `@UploadedFiles()` 装饰器从 `request` 中提取文件。

```typescript
@@filename()
@Post('upload')
@UseInterceptors(FileFieldsInterceptor([
  { name: 'avatar', maxCount: 1 },
  { name: 'background', maxCount: 1 },
]))
uploadFile(@UploadedFiles() files: { avatar?: Express.Multer.File[], background?: Express.Multer.File[] }) {
  console.log(files);
}
@@switch
@Post('upload')
@Bind(UploadedFiles())
@UseInterceptors(FileFieldsInterceptor([
  { name: 'avatar', maxCount: 1 },
  { name: 'background', maxCount: 1 },
]))
uploadFile(files) {
  console.log(files);
}
```

#### 任意文件

要上传所有具有任意字段名键的字段，请使用 `AnyFilesInterceptor()` 装饰器。此装饰器可以接受一个可选的 `options` 对象，如上所述。

使用 `AnyFilesInterceptor()` 时，使用 `@UploadedFiles()` 装饰器从 `request` 中提取文件。

```typescript
@@filename()
@Post('upload')
@UseInterceptors(AnyFilesInterceptor())
uploadFile(@UploadedFiles() files: Array<Express.Multer.File>) {
  console.log(files);
}
@@switch
@Post('upload')
@Bind(UploadedFiles())
@UseInterceptors(AnyFilesInterceptor())
uploadFile(files) {
  console.log(files);
}
```

#### 无文件

要接受 `multipart/form-data` 但不允许上传任何文件，请使用 `NoFilesInterceptor`。这会将 multipart 数据设置为请求正文上的属性。随请求发送的任何文件都会抛出 `BadRequestException`。

```typescript
@Post('upload')
@UseInterceptors(NoFilesInterceptor())
handleMultiPartData(@Body() body) {
  console.log(body)
}
```

#### 默认选项

如上所述，您可以在文件拦截器中指定 multer 选项。要设置默认选项，可以在导入 `MulterModule` 时调用静态 `register()` 方法，并传入支持的选项。您可以使用[此处](https://github.com/expressjs/multer#multeropts)列出的所有选项。

```typescript
MulterModule.register({
  dest: './upload',
});
```

> info **提示** `MulterModule` 类导自 `@nestjs/platform-express` 包。

#### 异步配置

当您需要异步而非静态地设置 `MulterModule` 选项时，请使用 `registerAsync()` 方法。与大多数动态模块一样，Nest 提供了几种处理异步配置的技术。

一种技术是使用工厂函数：

```typescript
MulterModule.registerAsync({
  useFactory: () => ({
    dest: './upload',
  }),
});
```

与其他 [工厂提供者](https://docs.nestjs.com/fundamentals/custom-providers#factory-providers-usefactory) 一样，我们的工厂函数可以是 `async` 的，并且可以通过 `inject` 注入依赖项。

```typescript
MulterModule.registerAsync({
  imports: [ConfigModule],
  useFactory: async (configService: ConfigService) => ({
    dest: configService.get<string>('MULTER_DEST'),
  }),
  inject: [ConfigService],
});
```

或者，您可以使用类而不是工厂来配置 `MulterModule`，如下所示：

```typescript
MulterModule.registerAsync({
  useClass: MulterConfigService,
});
```

上面的构造在 `MulterModule` 中实例化 `MulterConfigService`，使用它来创建所需的选项对象。请注意，在此示例中，`MulterConfigService` 必须实现 `MulterOptionsFactory` 接口，如下所示。`MulterModule` 将在提供的类的实例上调用 `createMulterOptions()` 方法。

```typescript
@Injectable()
class MulterConfigService implements MulterOptionsFactory {
  createMulterOptions(): MulterModuleOptions {
    return {
      dest: './upload',
    };
  }
}
```

如果您想重用现有的选项提供者而不是在 `MulterModule` 中创建私有副本，请使用 `useExisting` 语法。

```typescript
MulterModule.registerAsync({
  imports: [ConfigModule],
  useExisting: ConfigService,
});
```

您还可以将所谓的 `extraProviders` 传递给 `registerAsync()` 方法。这些提供者将与模块提供者合并。

```typescript
MulterModule.registerAsync({
  imports: [ConfigModule],
  useClass: ConfigService,
  extraProviders: [MyAdditionalProvider],
});
```

当您想向工厂函数或类构造函数提供额外的依赖项时，这很有用。

#### 示例

一个工作示例可在[此处](https://github.com/nestjs/nest/tree/master/sample/29-file-upload)找到。