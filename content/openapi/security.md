### 安全性

要定义特定操作应使用的安全机制，请使用 `@ApiSecurity()` 装饰器。

```typescript
@ApiSecurity('basic')
@Controller('cats')
export class CatsController {}
```

在运行应用程序之前，请记得使用 `DocumentBuilder` 将安全定义添加到基础文档中：

```typescript
const options = new DocumentBuilder().addSecurity('basic', {
  type: 'http',
  scheme: 'basic',
});
```

一些最流行的身份验证技术是内置的（例如，`basic` 和 `bearer`），因此无需像上面那样手动定义安全机制。

#### 基本身份验证

要启用基本身份验证，请使用 `@ApiBasicAuth()`。

```typescript
@ApiBasicAuth()
@Controller('cats')
export class CatsController {}
```

在运行应用程序之前，请记得使用 `DocumentBuilder` 将安全定义添加到基础文档中：

```typescript
const options = new DocumentBuilder().addBasicAuth();
```

#### Bearer 身份验证

要启用 Bearer 身份验证，请使用 `@ApiBearerAuth()`。

```typescript
@ApiBearerAuth()
@Controller('cats')
export class CatsController {}
```

在运行应用程序之前，请记得使用 `DocumentBuilder` 将安全定义添加到基础文档中：

```typescript
const options = new DocumentBuilder().addBearerAuth();
```

#### OAuth2 身份验证

要启用 OAuth2，请使用 `@ApiOAuth2()`。

```typescript
@ApiOAuth2(['pets:write'])
@Controller('cats')
export class CatsController {}
```

在运行应用程序之前，请记得使用 `DocumentBuilder` 将安全定义添加到基础文档中：

```typescript
const options = new DocumentBuilder().addOAuth2();
```

#### Cookie 身份验证

要启用 Cookie 身份验证，请使用 `@ApiCookieAuth()`。

```typescript
@ApiCookieAuth()
@Controller('cats')
export class CatsController {}
```

在运行应用程序之前，请记得使用 `DocumentBuilder` 将安全定义添加到基础文档中：

```typescript
const options = new DocumentBuilder().addCookieAuth('optional-session-id');
```