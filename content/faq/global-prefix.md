### 全局前缀

若要为 HTTP 应用程序中注册的 **所有路由** 设置前缀，请使用 `INestApplication` 实例的 `setGlobalPrefix()` 方法。

```typescript
const app = await NestFactory.create(AppModule);
app.setGlobalPrefix('v1');
```

你可以通过以下方式将某些路由排除在全局前缀之外：

```typescript
app.setGlobalPrefix('v1', {
  exclude: [{ path: 'health', method: RequestMethod.GET }],
});
```

或者，你也可以将路由指定为字符串（它将应用于所有请求方法）：

```typescript
app.setGlobalPrefix('v1', { exclude: ['cats'] });
```

> info **提示** `path` 属性支持使用 [path-to-regexp](https://github.com/pillarjs/path-to-regexp#parameters) 包的通配符参数。注意：这里不接受通配符星号 `*`，而必须使用参数 (`:param`) 或命名通配符 (`*splat`)。