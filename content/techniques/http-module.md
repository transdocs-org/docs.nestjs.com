### HTTP 模块

[Axios](https://github.com/axios/axios) 是一个功能丰富的 HTTP 客户端包，被广泛使用。Nest 对 Axios 进行了封装，并通过内置的 `HttpModule` 暴露出来。`HttpModule` 导出了 `HttpService` 类，该类提供了基于 Axios 的方法来执行 HTTP 请求。此外，该库还会将生成的 HTTP 响应转换为 `Observables`。

> info **提示** 你也可以直接使用任何通用的 Node.js HTTP 客户端库，例如 [got](https://github.com/sindresorhus/got) 或 [undici](https://github.com/nodejs/undici)。

#### 安装

要开始使用它，我们首先安装所需的依赖项。

```bash
$ npm i --save @nestjs/axios axios
```

#### 入门

安装完成后，要使用 `HttpService`，首先导入 `HttpModule`。

```typescript
@Module({
  imports: [HttpModule],
  providers: [CatsService],
})
export class CatsModule {}
```

接下来，使用构造函数注入的方式注入 `HttpService`。

> info **提示** `HttpModule` 和 `HttpService` 是从 `@nestjs/axios` 包中导入的。

```typescript
@@filename()
@Injectable()
export class CatsService {
  constructor(private readonly httpService: HttpService) {}

  findAll(): Observable<AxiosResponse<Cat[]>> {
    return this.httpService.get('http://localhost:3000/cats');
  }
}
@@switch
@Injectable()
@Dependencies(HttpService)
export class CatsService {
  constructor(httpService) {
    this.httpService = httpService;
  }

  findAll() {
    return this.httpService.get('http://localhost:3000/cats');
  }
}
```

> info **提示** `AxiosResponse` 是从 `axios` 包导出的接口（`$ npm i axios`）。

所有 `HttpService` 方法都返回一个包裹在 `Observable` 对象中的 `AxiosResponse`。

#### 配置

[Axios](https://github.com/axios/axios) 可以通过多种选项进行配置，以自定义 `HttpService` 的行为。你可以在此处阅读更多相关内容：[这里](https://github.com/axios/axios#request-config)。要配置底层 Axios 实例，请在导入 `HttpModule` 时将可选的配置对象传递给 `register()` 方法。此配置对象将直接传递给底层 Axios 构造函数。

```typescript
@Module({
  imports: [
    HttpModule.register({
      timeout: 5000,
      maxRedirects: 5,
    }),
  ],
  providers: [CatsService],
})
export class CatsModule {}
```

#### 异步配置

当你需要异步而非静态地传递模块选项时，请使用 `registerAsync()` 方法。与大多数动态模块一样，Nest 提供了几种处理异步配置的技术。

一种方法是使用工厂函数：

```typescript
HttpModule.registerAsync({
  useFactory: () => ({
    timeout: 5000,
    maxRedirects: 5,
  }),
});
```

像其他工厂提供者一样，我们的工厂函数可以是 [异步的](https://docs.nestjs.com/fundamentals/custom-providers#factory-providers-usefactory)，并且可以通过 `inject` 注入依赖项。

```typescript
HttpModule.registerAsync({
  imports: [ConfigModule],
  useFactory: async (configService: ConfigService) => ({
    timeout: configService.get('HTTP_TIMEOUT'),
    maxRedirects: configService.get('HTTP_MAX_REDIRECTS'),
  }),
  inject: [ConfigService],
});
```

或者，你可以使用类而不是工厂来配置 `HttpModule`，如下所示：

```typescript
HttpModule.registerAsync({
  useClass: HttpConfigService,
});
```

上述构造在 `HttpModule` 内部实例化 `HttpConfigService`，并使用它创建一个选项对象。请注意，在此示例中，`HttpConfigService` 必须实现 `HttpModuleOptionsFactory` 接口，如下所示。`HttpModule` 将在提供的类实例上调用 `createHttpOptions()` 方法。

```typescript
@Injectable()
class HttpConfigService implements HttpModuleOptionsFactory {
  createHttpOptions(): HttpModuleOptions {
    return {
      timeout: 5000,
      maxRedirects: 5,
    };
  }
}
```

如果你想重用现有的选项提供者而不是在 `HttpModule` 内部创建私有副本，请使用 `useExisting` 语法。

```typescript
HttpModule.registerAsync({
  imports: [ConfigModule],
  useExisting: HttpConfigService,
});
```

你还可以将所谓的 `extraProviders` 传递给 `registerAsync()` 方法。这些提供者将与模块提供者合并。

```typescript
HttpModule.registerAsync({
  imports: [ConfigModule],
  useClass: HttpConfigService,
  extraProviders: [MyAdditionalProvider],
});
```

当你希望向工厂函数或类构造函数提供额外的依赖项时，这非常有用。

#### 直接使用 Axios

如果你认为 `HttpModule.register` 的选项不足以满足需求，或者只是想访问 `@nestjs/axios` 创建的底层 Axios 实例，可以通过 `HttpService#axiosRef` 如下访问：

```typescript
@Injectable()
export class CatsService {
  constructor(private readonly httpService: HttpService) {}

  findAll(): Promise<AxiosResponse<Cat[]>> {
    return this.httpService.axiosRef.get('http://localhost:3000/cats');
    //                      ^ AxiosInstance 接口
  }
}
```

#### 完整示例

由于 `HttpService` 方法的返回值是 `Observable`，我们可以使用 `rxjs` 的 `firstValueFrom` 或 `lastValueFrom` 来以 Promise 的形式获取请求数据。

```typescript
import { catchError, firstValueFrom } from 'rxjs';

@Injectable()
export class CatsService {
  private readonly logger = new Logger(CatsService.name);
  constructor(private readonly httpService: HttpService) {}

  async findAll(): Promise<Cat[]> {
    const { data } = await firstValueFrom(
      this.httpService.get<Cat[]>('http://localhost:3000/cats').pipe(
        catchError((error: AxiosError) => {
          this.logger.error(error.response.data);
          throw 'An error happened!';
        }),
      ),
    );
    return data;
  }
}
```

> info **提示** 有关 `firstValueFrom` 和 `lastValueFrom` 的区别，请访问 RxJS 的文档： [`firstValueFrom`](https://rxjs.dev/api/index/function/firstValueFrom) 和 [`lastValueFrom`](https://rxjs.dev/api/index/function/lastValueFrom)。