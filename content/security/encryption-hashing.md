### 加密和哈希

**加密** 是对信息进行编码的过程。该过程将信息的原始表示形式（称为明文）转换为另一种形式（称为密文）。理想情况下，只有授权方才能将密文解密回明文并访问原始信息。加密本身并不能防止信息被截取，但可以使得潜在的拦截者无法理解其内容。加密是一种双向函数；被加密的内容可以使用正确的密钥进行解密。

**哈希** 是将给定的键转换为另一个值的过程。哈希函数根据特定的数学算法生成新的值。一旦完成哈希操作，应该无法从输出反推出输入。

#### 加密

Node.js 提供了一个内置的 [crypto 模块](https://nodejs.org/api/crypto.html)，可用于加密和解密字符串、数字、缓冲区、流等。Nest 本身并未在此模块之上提供任何额外的封装，以避免引入不必要的抽象。

例如，我们将使用 AES（高级加密系统）的 `'aes-256-ctr'` 算法和 CTR 加密模式。

```typescript
import { createCipheriv, randomBytes, scrypt } from 'crypto';
import { promisify } from 'util';

const iv = randomBytes(16);
const password = '用于生成密钥的密码';

// 密钥长度取决于算法。
// 在本例中，对于 aes256，密钥长度为 32 字节。
const key = (await promisify(scrypt)(password, 'salt', 32)) as Buffer;
const cipher = createCipheriv('aes-256-ctr', key, iv);

const textToEncrypt = 'Nest';
const encryptedText = Buffer.concat([
  cipher.update(textToEncrypt),
  cipher.final(),
]);
```

现在要解密 `encryptedText` 值：

```typescript
import { createDecipheriv } from 'crypto';

const decipher = createDecipheriv('aes-256-ctr', key, iv);
const decryptedText = Buffer.concat([
  decipher.update(encryptedText),
  decipher.final(),
]);
```

#### 哈希

对于哈希处理，我们建议使用 [bcrypt](https://www.npmjs.com/package/bcrypt) 或 [argon2](https://www.npmjs.com/package/argon2) 包。Nest 本身并未对这些模块进行额外的封装，以避免引入不必要的抽象（从而缩短学习曲线）。

例如，我们使用 `bcrypt` 对一个随机密码进行哈希处理。

首先安装所需的包：

```shell
$ npm i bcrypt
$ npm i -D @types/bcrypt
```

安装完成后，可以使用 `hash` 函数，如下所示：

```typescript
import * as bcrypt from 'bcrypt';

const saltOrRounds = 10;
const password = 'random_password';
const hash = await bcrypt.hash(password, saltOrRounds);
```

要生成盐值，请使用 `genSalt` 函数：

```typescript
const salt = await bcrypt.genSalt();
```

要比较/验证密码，请使用 `compare` 函数：

```typescript
const isMatch = await bcrypt.compare(password, hash);
```

你可以在此处阅读有关可用函数的[更多信息](https://www.npmjs.com/package/bcrypt)。