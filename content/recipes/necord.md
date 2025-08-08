### Necord

Necord 是一个强大的模块，它简化了 [Discord](https://discord.com) 机器人的创建，使您能够无缝地将其集成到 NestJS 应用程序中。

> info **注意** Necord 是一个第三方包，并非由 NestJS 核心团队官方维护。如果您遇到任何问题，请在[官方仓库](https://github.com/necordjs/necord)中报告。

#### 安装

要开始使用，请安装 Necord 及其依赖项 [`Discord.js`](https://discord.js.org)。

```bash
$ npm install necord discord.js
```

#### 使用方法

要在您的项目中使用 Necord，请导入 `NecordModule` 并使用必要的选项进行配置。

```typescript
@@filename(app.module)
import { Module } from '@nestjs/common';
import { NecordModule } from 'necord';
import { IntentsBitField } from 'discord.js';
import { AppService } from './app.service';

@Module({
  imports: [
    NecordModule.forRoot({
      token: process.env.DISCORD_TOKEN,
      intents: [IntentsBitField.Flags.Guilds],
      development: [process.env.DISCORD_DEVELOPMENT_GUILD_ID],
    }),
  ],
  providers: [AppService],
})
export class AppModule {}
```

> info **提示** 您可以在此处找到可用意图的完整列表：[这里](https://discord.com/developers/docs/topics/gateway#gateway-intents)。

通过此设置，您可以将 `AppService` 注入到您的提供者中，以轻松注册命令、事件等。

```typescript
@@filename(app.service)
import { Injectable, Logger } from '@nestjs/common';
import { Context, On, Once, ContextOf } from 'necord';
import { Client } from 'discord.js';

@Injectable()
export class AppService {
  private readonly logger = new Logger(AppService.name);

  @Once('ready')
  public onReady(@Context() [client]: ContextOf<'ready'>) {
    this.logger.log(`Bot logged in as ${client.user.username}`);
  }

  @On('warn')
  public onWarn(@Context() [message]: ContextOf<'warn'>) {
    this.logger.warn(message);
  }
}
```

##### 理解上下文

您可能已经注意到上面示例中的 `@Context` 装饰器。此装饰器将事件上下文注入到您的方法中，使您可以访问各种特定于事件的数据。由于存在多种类型的事件，因此使用 `ContextOf<type: string>` 类型推断上下文类型。您可以使用 `@Context()` 装饰器轻松访问上下文变量，该装饰器会用与事件相关的参数数组填充变量。

#### 文本命令

> warning **警告** 文本命令依赖于消息内容，而消息内容对于经过验证的机器人和拥有超过 100 个服务器的应用程序来说将被弃用。这意味着如果您的机器人无法访问消息内容，则文本命令将无法正常工作。有关此更改的更多信息，请参阅[此处](https://support-dev.discord.com/hc/en-us/articles/4404772028055-Message-Content-Access-Deprecation-for-Verified-Bots)。

以下是使用 `@TextCommand` 装饰器为消息创建简单命令处理程序的方法。

```typescript
@@filename(app.commands)
import { Injectable } from '@nestjs/common';
import { Context, TextCommand, TextCommandContext, Arguments } from 'necord';

@Injectable()
export class AppCommands {
  @TextCommand({
    name: 'ping',
    description: 'Responds with pong!',
  })
  public onPing(
    @Context() [message]: TextCommandContext,
    @Arguments() args: string[],
  ) {
    return message.reply('pong!');
  }
}
```

#### 应用程序命令

应用程序命令为用户提供了一种在 Discord 客户端内与您的应用程序进行交互的原生方式。有三种类型的应用程序命令，可以通过不同的界面访问：聊天输入、消息上下文菜单（通过右键单击消息访问）和用户上下文菜单（通过右键单击用户访问）。

<figure><img class="illustrative-image" src="https://i.imgur.com/4EmG8G8.png" /></figure>

#### 斜杠命令

斜杠命令是一种以结构化方式与用户互动的绝佳方式。它们允许您创建带有精确参数和选项的命令，从而显著提升用户体验。

要使用 Necord 定义斜杠命令，您可以使用 `SlashCommand` 装饰器。

```typescript
@@filename(app.commands)
import { Injectable } from '@nestjs/common';
import { Context, SlashCommand, SlashCommandContext } from 'necord';

@Injectable()
export class AppCommands {
  @SlashCommand({
    name: 'ping',
    description: 'Responds with pong!',
  })
  public async onPing(@Context() [interaction]: SlashCommandContext) {
    return interaction.reply({ content: 'Pong!' });
  }
}
```

> info **提示** 当您的机器人客户端登录时，它将自动注册所有定义的命令。请注意，全局命令的缓存时间最长为一小时。为了避免全局缓存带来的问题，请在 Necord 模块中使用 `development` 参数，该参数会将命令的可见性限制为单个服务器。

##### 选项

您可以使用选项装饰器为斜杠命令定义参数。为此，我们创建一个 `TextDto` 类：

```typescript
@@filename(text.dto)
import { StringOption } from 'necord';

export class TextDto {
  @StringOption({
    name: 'text',
    description: 'Input your text here',
    required: true,
  })
  text: string;
}
```

然后可以在 `AppCommands` 类中使用此 DTO：

```typescript
@@filename(app.commands)
import { Injectable } from '@nestjs/common';
import { Context, SlashCommand, Options, SlashCommandContext } from 'necord';
import { TextDto } from './length.dto';

@Injectable()
export class AppCommands {
  @SlashCommand({
    name: 'length',
    description: 'Calculate the length of your text',
  })
  public async onLength(
    @Context() [interaction]: SlashCommandContext,
    @Options() { text }: TextDto,
  ) {
    return interaction.reply({
      content: `The length of your text is: ${text.length}`,
    });
  }
}
```

有关内置选项装饰器的完整列表，请查看[此文档](https://necord.org/interactions/slash-commands#options)。

##### 自动补全

要为斜杠命令实现自动补全功能，您需要创建一个拦截器。此拦截器将在用户在自动补全字段中键入时处理请求。

```typescript
@@filename(cats-autocomplete.interceptor)
import { Injectable } from '@nestjs/common';
import { AutocompleteInteraction } from 'discord.js';
import { AutocompleteInterceptor } from 'necord';

@Injectable()
class CatsAutocompleteInterceptor extends AutocompleteInterceptor {
  public transformOptions(interaction: AutocompleteInteraction) {
    const focused = interaction.options.getFocused(true);
    let choices: string[];

    if (focused.name === 'cat') {
      choices = ['Siamese', 'Persian', 'Maine Coon'];
    }

    return interaction.respond(
      choices
        .filter((choice) => choice.startsWith(focused.value.toString()))
        .map((choice) => ({ name: choice, value: choice })),
    );
  }
}
```

您还需要在选项类上标记 `autocomplete: true`：

```typescript
@@filename(cat.dto)
import { StringOption } from 'necord';

export class CatDto {
  @StringOption({
    name: 'cat',
    description: 'Choose a cat breed',
    autocomplete: true,
    required: true,
  })
  cat: string;
}
```

最后，将拦截器应用到您的斜杠命令：

```typescript
@@filename(cats.commands)
import { Injectable, UseInterceptors } from '@nestjs/common';
import { Context, SlashCommand, Options, SlashCommandContext } from 'necord';
import { CatDto } from '/cat.dto';
import { CatsAutocompleteInterceptor } from './cats-autocomplete.interceptor';

@Injectable()
export class CatsCommands {
  @UseInterceptors(CatsAutocompleteInterceptor)
  @SlashCommand({
    name: 'cat',
    description: 'Retrieve information about a specific cat breed',
  })
  public async onSearch(
    @Context() [interaction]: SlashCommandContext,
    @Options() { cat }: CatDto,
  ) {
    return interaction.reply({
      content: `I found information on the breed of ${cat} cat!`,
    });
  }
}
```

#### 用户上下文菜单

用户命令出现在右键单击（或点击）用户时出现的上下文菜单中。这些命令提供了直接针对用户的快速操作。

```typescript
@@filename(app.commands)
import { Injectable } from '@nestjs/common';
import { Context, UserCommand, UserCommandContext, TargetUser } from 'necord';
import { User } from 'discord.js';

@Injectable()
export class AppCommands {
  @UserCommand({ name: 'Get avatar' })
  public async getUserAvatar(
    @Context() [interaction]: UserCommandContext,
    @TargetUser() user: User,
  ) {
    return interaction.reply({
      embeds: [
        new MessageEmbed()
          .setTitle(`Avatar of ${user.username}`)
          .setImage(user.displayAvatarURL({ size: 4096, dynamic: true })),
      ],
    });
  }
}
```

#### 消息上下文菜单

消息命令出现在右键单击消息时出现的上下文菜单中，允许对这些消息执行快速操作。

```typescript
@@filename(app.commands)
import { Injectable } from '@nestjs/common';
import { Context, MessageCommand, MessageCommandContext, TargetMessage } from 'necord';
import { Message } from 'discord.js';

@Injectable()
export class AppCommands {
  @MessageCommand({ name: 'Copy Message' })
  public async copyMessage(
    @Context() [interaction]: MessageCommandContext,
    @TargetMessage() message: Message,
  ) {
    return interaction.reply({ content: message.content });
  }
}
```

#### 按钮

[按钮](https://discord.com/developers/docs/interactions/message-components#buttons)是可以包含在消息中的交互式元素。当单击时，它们会向您的应用程序发送一个[交互](https://discord.com/developers/docs/interactions/receiving-and-responding#interaction-object)。

```typescript
@@filename(app.components)
import { Injectable } from '@nestjs/common';
import { Context, Button, ButtonContext } from 'necord';

@Injectable()
export class AppComponents {
  @Button('BUTTON')
  public onButtonClick(@Context() [interaction]: ButtonContext) {
    return interaction.reply({ content: 'Button clicked!' });
  }
}
```

#### 选择菜单

[选择菜单](https://discord.com/developers/docs/interactions/message-components#select-menus)是出现在消息上的另一种交互式组件。它们为用户提供了一个类似下拉菜单的界面以选择选项。

```typescript
@@filename(app.components)
import { Injectable } from '@nestjs/common';
import { Context, StringSelect, StringSelectContext, SelectedStrings } from 'necord';

@Injectable()
export class AppComponents {
  @StringSelect('SELECT_MENU')
  public onSelectMenu(
    @Context() [interaction]: StringSelectContext,
    @SelectedStrings() values: string[],
  ) {
    return interaction.reply({ content: `You selected: ${values.join(', ')}` });
  }
}
```

有关内置选择菜单组件的完整列表，请访问[此链接](https://necord.org/interactions/message-components#select-menu)。

#### 模态框

模态框是弹出表单，允许用户提交格式化的输入。以下是如何使用 Necord 创建和处理模态框：

```typescript
@@filename(app.modals)
import { Injectable } from '@nestjs/common';
import { Context, Modal, ModalContext } from 'necord';

@Injectable()
export class AppModals {
  @Modal('pizza')
  public onModal(@Context() [interaction]: ModalContext) {
    return interaction.reply({
      content: `Your fav pizza : ${interaction.fields.getTextInputValue('pizza')}`
    });
  }
}
```

#### 更多信息

请访问 [Necord](https://necord.org) 网站以获取更多信息。