[根目录](../../CLAUDE.md) > [packages](../) > **pieces**

# Pieces 集成模块文档

[根目录](../../../CLAUDE.md) > [packages](../../) > **pieces** > **community** (280+)

## 模块职责

ActivePieces Pieces 模块是整个平台的核心生态系统，提供了 280+ 个第三方服务集成，支持各种 SaaS 应用、数据库、API、通信工具等。每个集成都是一个独立的 npm 包，采用标准化的开发框架，支持热重载开发和自动化测试。

## 模块结构

```
packages/pieces/
├── community/             # 社区集成 (280+)
│   ├── activecampaign/    # ActiveCampaign 集成
│   ├── slack/            # Slack 集成
│   ├── google-sheets/    # Google Sheets 集成
│   ├── openai/           # OpenAI 集成
│   └── ...               # 更多集成
├── framework/            # 集成开发框架
│   ├── src/
│   │   ├── index.ts      # 框架入口
│   │   ├── src/
│   │   │   ├── pieces/   # 集成基类
│   │   │   ├── triggers/ # 触发器基类
│   │   │   ├── actions/  # 动作基类
│   │   │   ├── props/    # 属性系统
│   │   │   └── auth/     # 认证系统
│   ├── package.json
│   └── README.md
└── README.md
```

## 集成开发框架

### 核心概念

#### 1. Piece - 集成定义
```typescript
// 集成基本结构
const myIntegration: Piece = {
  name: 'my-integration',
  displayName: 'My Integration',
  description: 'Integration with My Service',
  logoUrl: 'https://example.com/logo.png',
  authors: ['author@example.com'],
  categories: [PieceCategory.COMMUNICATION],

  // 认证配置
  auth: {
    type: AuthenticationType.BASIC_AUTH,
    required: true,
  },

  // 动作定义
  actions: {
    sendMessage: sendMessageAction,
    createChannel: createChannelAction,
  },

  // 触发器定义
  triggers: {
    onNewMessage: onNewMessageTrigger,
    onUserJoined: onUserJoinedTrigger,
  },
};
```

#### 2. Action - 动作定义
```typescript
// 动作定义
export const sendMessageAction: Action = {
  name: 'send_message',
  displayName: 'Send Message',
  description: 'Send a message to a channel',
  description: 'Send a message to a channel',

  // 输入属性
  props: {
    channel: Property.Dropdown({
      displayName: 'Channel',
      description: 'Select a channel',
      required: true,
      options: async (context) => {
        return await getChannels(context.auth);
      },
    }),

    message: Property.LongText({
      displayName: 'Message',
      description: 'Message to send',
      required: true,
    }),

    attachments: Property.Array({
      displayName: 'Attachments',
      description: 'Files to attach',
      required: false,
    }),
  },

  // 执行逻辑
  async run(context) {
    const { channel, message, attachments } = context.propsValue;
    const auth = context.auth;

    try {
      const response = await sendMessage(auth, {
        channel,
        message,
        attachments,
      });

      return response;
    } catch (error) {
      throw new Error(`Failed to send message: ${error.message}`);
    }
  },

  // 测试逻辑
  async test(context) {
    const { channel, message } = context.propsValue;

    // 验证连接和基本参数
    await validateChannel(context.auth, channel);

    return {
      channel,
      message: `Test message: ${message}`,
      timestamp: new Date().toISOString(),
    };
  },
};
```

#### 3. Trigger - 触发器定义
```typescript
// 触发器定义
export const onNewMessageTrigger: Trigger = {
  name: 'on_new_message',
  displayName: 'On New Message',
  description: 'Trigger when a new message is posted',

  // 输入属性
  props: {
    channel: Property.Dropdown({
      displayName: 'Channel',
      description: 'Select channel to monitor',
      required: true,
      options: async (context) => {
        return await getChannels(context.auth);
      },
    }),

    keywords: Property.Array({
      displayName: 'Keywords',
      description: 'Filter messages by keywords',
      required: false,
    }),
  },

  // 启用触发器
  async onEnable(context) {
    const { channel, keywords } = context.propsValue;
    const webhookUrl = context.webhookUrl;

    // 注册 webhook
    await registerWebhook(context.auth, {
      channel,
      webhookUrl,
      keywords,
    });
  },

  // 禁用触发器
  async onDisable(context) {
    await unregisterWebhook(context.auth, context.webhookUrl);
  },

  // 手动测试
  async test(context) {
    const { channel } = context.propsValue;

    // 模拟测试数据
    return {
      channel,
      message: 'Test message',
      user: 'test-user',
      timestamp: new Date().toISOString(),
    };
  },

  // 处理 webhook
  async onExecute(context) {
    const payload = context.payload;

    // 验证和转换数据
    if (!isValidMessage(payload)) {
      return [];
    }

    return [{
      channel: payload.channel,
      message: payload.text,
      user: payload.user,
      timestamp: payload.ts,
    }];
  },
};
```

### 认证系统

#### 1. 基础认证
```typescript
// API Key 认证
export const apiKeyAuth = PieceAuth.ApiKey({
  description: 'Enter your API key',
  required: true,
  displayName: 'API Key',
});

// Basic Auth 认证
export const basicAuth = PieceAuth.BasicAuth({
  description: 'Enter your username and password',
  required: true,
  username: {
    displayName: 'Username',
    description: 'Your username',
  },
  password: {
    displayName: 'Password',
    description: 'Your password',
  },
});
```

#### 2. OAuth2 认证
```typescript
// OAuth2 认证
export const oauth2Auth = PieceAuth.OAuth2({
  description: 'Authenticate using OAuth2',
  authUrl: 'https://api.example.com/oauth/authorize',
  tokenUrl: 'https://api.example.com/oauth/token',
  required: true,
  scope: ['read', 'write'],
});
```

#### 3. 自定义认证
```typescript
// 自定义认证
export const customAuth = PieceAuth.CustomAuth({
  description: 'Enter your credentials',
  required: true,
  props: {
    apiKey: Property.ShortText({
      displayName: 'API Key',
      description: 'Your API key',
      required: true,
    }),
    environment: Property.StaticDropdown({
      displayName: 'Environment',
      description: 'API environment',
      required: true,
      options: {
        disabled: false,
        options: [
          { label: 'Production', value: 'prod' },
          { label: 'Sandbox', value: 'sandbox' },
        ],
      },
    }),
  },
  validate: async ({ auth }) => {
    // 验证认证信息
    try {
      await validateCredentials(auth);
      return {
        valid: true,
      };
    } catch (error) {
      return {
        valid: false,
        error: 'Invalid credentials',
      };
    }
  },
});
```

### 属性系统

#### 1. 基础属性
```typescript
// 文本输入
Property.ShortText({
  displayName: 'Name',
  description: 'Enter your name',
  required: true,
  validators: [Validators.maxLength(50)],
});

// 长文本
Property.LongText({
  displayName: 'Description',
  description: 'Enter description',
  required: false,
});

// 数字输入
Property.Number({
  displayName: 'Count',
  description: 'Enter count',
  required: true,
  validators: [Validators.min(1), Validators.max(100)],
});

// 布尔值
Property.Checkbox({
  displayName: 'Enable notifications',
  description: 'Turn on notifications',
  required: false,
});
```

#### 2. 选择器属性
```typescript
// 下拉选择
Property.StaticDropdown({
  displayName: 'Priority',
  description: 'Select priority',
  required: true,
  options: {
    disabled: false,
    options: [
      { label: 'Low', value: 'low' },
      { label: 'Medium', value: 'medium' },
      { label: 'High', value: 'high' },
    ],
  },
});

// 动态下拉
Property.Dropdown({
  displayName: 'Project',
  description: 'Select a project',
  required: true,
  refreshers: ['auth'], // 当认证改变时刷新选项
  options: async ({ auth }) => {
    const projects = await getProjects(auth);
    return projects.map(project => ({
      label: project.name,
      value: project.id,
    }));
  },
});

// 多选
Property.MultiSelectDropdown({
  displayName: 'Tags',
  description: 'Select tags',
  required: false,
  options: async ({ auth }) => {
    return await getTags(auth);
  },
});
```

#### 3. 高级属性
```typescript
// 日期时间
Property.DateTime({
  displayName: 'Scheduled Time',
  description: 'Select date and time',
  required: true,
});

// 文件上传
Property.File({
  displayName: 'Attachment',
  description: 'Upload a file',
  required: false,
});

// JSON 输入
Property.Json({
  displayName: 'Configuration',
  description: 'Enter JSON configuration',
  required: false,
});

// 代码编辑
Property.Code({
  displayName: 'Script',
  description: 'Enter JavaScript code',
  required: false,
  language: 'javascript',
});
```

## 社区集成

### 集成分类

#### 1. 通信工具
- **Slack**: 消息发送、频道管理、文件上传
- **Microsoft Teams**: 频道消息、会议管理
- **Discord**: 服务器管理、消息发送
- **Telegram**: 机器人消息、命令处理
- **Email (SMTP/SendGrid)**: 邮件发送、模板管理

#### 2. CRM 和销售
- **Salesforce**: 客户管理、销售记录
- **HubSpot**: 联系人管理、营销自动化
- **Pipedrive**: 交易管理、活动跟踪
- **Zoho CRM**: 客户关系管理

#### 3. 项目管理
- **Asana**: 任务管理、项目跟踪
- **Trello**: 卡片管理、列表操作
- **Jira**: 问题跟踪、项目管理
- **Monday.com**: 工作流自动化
- **ClickUp**: 任务和项目管理

#### 4. 数据库和存储
- **Google Sheets**: 表格操作、数据管理
- **Airtable**: 数据库操作、记录管理
- **Notion**: 页面管理、数据库操作
- **MongoDB**: 文档数据库操作
- **PostgreSQL**: 关系型数据库操作
- **MySQL**: 关系型数据库操作

#### 5. AI 和机器学习
- **OpenAI**: GPT 模型调用、图像生成
- **Anthropic Claude**: AI 对话、文本生成
- **Google AI**: 各种 AI 模型调用
- **Replicate**: 机器学习模型调用

#### 6. 电商和支付
- **Shopify**: 产品管理、订单处理
- **Stripe**: 支付处理、客户管理
- **WooCommerce**: 电商操作、订单管理
- **BigCommerce**: 产品和订单管理

#### 7. 生产力工具
- **Google Workspace**: 文档、表格、邮件
- **Microsoft 365**: Office 文件、邮件、日历
- **Dropbox**: 文件管理、共享
- **Box**: 云存储、文件操作

#### 8. 开发工具
- **GitHub**: 仓库管理、Issue 处理
- **GitLab**: 项目管理、CI/CD
- **Jenkins**: 构建管理、任务触发

### 热门集成示例

#### 1. Google Sheets
```typescript
export const googleSheetsCreateRowAction: Action = {
  name: 'create_row',
  displayName: 'Create Row',
  description: 'Create a new row in a spreadsheet',

  props: {
    spreadsheet_id: Property.Dropdown({
      displayName: 'Spreadsheet',
      description: 'Select spreadsheet',
      required: true,
      options: async ({ auth }) => {
        return await getSpreadsheets(auth);
      },
    }),

    worksheet_id: Property.Dropdown({
      displayName: 'Worksheet',
      description: 'Select worksheet',
      required: true,
      refreshers: ['spreadsheet_id'],
      options: async ({ auth, propsValue }) => {
        return await getWorksheets(auth, propsValue.spreadsheet_id);
      },
    }),

    values: Property.Array({
      displayName: 'Row Values',
      description: 'Values to insert in the row',
      required: true,
    }),
  },

  async run(context) {
    const { spreadsheet_id, worksheet_id, values } = context.propsValue;

    const response = await googleSheets.spreadsheets().values().append({
      spreadsheetId: spreadsheet_id,
      range: `${worksheet_id}!A:Z`,
      valueInputOption: 'USER_ENTERED',
      resource: {
        values: [values],
      },
    });

    return response.data;
  },
};
```

#### 2. Slack
```typescript
export const slackSendMessageAction: Action = {
  name: 'send_message',
  displayName: 'Send Message',
  description: 'Send a message to a channel',

  props: {
    channel: Property.Dropdown({
      displayName: 'Channel',
      description: 'Select channel',
      required: true,
      options: async ({ auth }) => {
        const client = new SlackClient(auth);
        const channels = await client.conversations.list();
        return channels.channels.map(channel => ({
          label: channel.name,
          value: channel.id,
        }));
      },
    }),

    text: Property.LongText({
      displayName: 'Message',
      description: 'Message to send',
      required: true,
    }),
  },

  async run(context) {
    const { channel, text } = context.propsValue;
    const client = new SlackClient(context.auth);

    const result = await client.chat.postMessage({
      channel,
      text,
    });

    return result;
  },
};
```

## 开发指南

### 1. 创建新集成
```bash
# 使用 CLI 创建集成模板
npm run cli pieces create my-integration

# 或者手动创建目录
mkdir packages/pieces/community/my-integration
cd packages/pieces/community/my-integration
```

### 2. 目录结构
```
my-integration/
├── src/
│   ├── index.ts          # 集成入口
│   ├── actions/          # 动作实现
│   │   ├── create-item.ts
│   │   └── update-item.ts
│   ├── triggers/         # 触发器实现
│   │   └── on-item-created.ts
│   ├── lib/              # 工具函数
│   │   ├── client.ts     # API 客户端
│   │   └── types.ts      # 类型定义
│   └── i18n/             # 国际化文件
│       ├── en.json
│       └── zh.json
├── package.json
├── tsconfig.json
└── README.md
```

### 3. 集成入口文件
```typescript
// src/index.ts
import { createPiece } from '@activepieces/pieces-framework';
import { PieceCategory } from '@activepieces/shared';
import { myAuth } from './lib/auth';
import { createItemAction } from './actions/create-item';
import { updateItemAction } from './actions/update-item';
import { onItemCreatedTrigger } from './triggers/on-item-created';

export const myIntegration = createPiece({
  name: 'my-integration',
  displayName: 'My Integration',
  description: 'Integration with My Service',
  logoUrl: 'https://example.com/logo.png',
  categories: [PieceCategory.PRODUCTIVITY],
  authors: ['author@example.com'],
  auth: myAuth,
  actions: [createItemAction, updateItemAction],
  triggers: [onItemCreatedTrigger],
});
```

### 4. 测试集成
```typescript
// 测试文件
import { testPiece } from '@activepieces/pieces-framework';
import { myIntegration } from '../src';

describe('My Integration', () => {
  testPiece({
    piece: myIntegration,
    auth: {
      apiKey: 'test-api-key',
    },
    tests: {
      'create-item': {
        input: {
          name: 'Test Item',
          description: 'Test Description',
        },
        output: {
          id: expect.any(String),
          name: 'Test Item',
        },
      },
    },
  });
});
```

### 5. 构建和发布
```bash
# 构建集成
npm run cli pieces build my-integration

# 测试集成
npm test

# 发布到 npm
npm run cli pieces publish my-integration

# 同步到注册表
npm run cli pieces sync
```

## 国际化支持

### 1. 翻译文件
```json
// src/i18n/en.json
{
  "actions": {
    "create_item": {
      "displayName": "Create Item",
      "description": "Create a new item"
    }
  },
  "triggers": {
    "on_item_created": {
      "displayName": "On Item Created",
      "description": "Trigger when a new item is created"
    }
  }
}
```

```json
// src/i18n/zh.json
{
  "actions": {
    "create_item": {
      "displayName": "创建项目",
      "description": "创建新项目"
    }
  },
  "triggers": {
    "on_item_created": {
      "displayName": "项目创建时",
      "description": "当新项目创建时触发"
    }
  }
}
```

### 2. 使用翻译
```typescript
// 在组件中使用
export const createItemAction: Action = {
  name: 'create_item',
  displayName: 'actions.create_item.displayName', // 使用翻译键
  description: 'actions.create_item.description',
  // ...
};
```

## 最佳实践

### 1. 代码规范
- 使用 TypeScript 严格模式
- 遵循 ESLint 和 Prettier 配置
- 编写完整的 JSDoc 注释
- 保持函数简洁和单一职责

### 2. 错误处理
```typescript
// 统一错误处理
export class MyIntegrationError extends Error {
  constructor(message: string, public code: string) {
    super(message);
    this.name = 'MyIntegrationError';
  }
}

// 在动作中使用
async run(context) {
  try {
    const result = await apiCall(context.auth, context.propsValue);
    return result;
  } catch (error) {
    if (error.response?.status === 401) {
      throw new Error('Authentication failed. Please check your credentials.');
    }
    if (error.response?.status === 404) {
      throw new Error('Resource not found. Please check the input parameters.');
    }
    throw new Error(`API request failed: ${error.message}`);
  }
}
```

### 3. 性能优化
```typescript
// 缓存 API 客户端
const getClient = memoize((auth: MyAuth) => {
  return new MyClient(auth);
});

// 批量操作
export const createMultipleItemsAction: Action = {
  name: 'create_multiple_items',
  // ...
  async run(context) {
    const items = context.propsValue.items;
    const client = getClient(context.auth);

    // 批量创建
    const results = await Promise.allSettled(
      items.map(item => client.create(item))
    );

    return results.map(result =>
      result.status === 'fulfilled' ? result.value : null
    ).filter(Boolean);
  },
};
```

### 4. 安全考虑
- 不要记录敏感信息
- 验证所有输入参数
- 使用 HTTPS 连接
- 实现适当的超时和重试机制

## 常见问题 (FAQ)

### Q: 如何调试集成？
A: 使用 `console.log` 输出调试信息，在测试面板中查看执行结果。

### Q: 如何处理分页？
A: 在属性中添加分页选项，循环获取所有数据。

### Q: 如何实现文件上传？
A: 使用 `Property.File` 属性，处理文件数据并上传到目标服务。

### Q: 如何支持多个 API 版本？
A: 在认证或属性中添加版本选择，根据版本调用不同的 API 端点。

## 相关文件清单

### 集成框架
- `framework/src/index.ts` - 框架入口
- `framework/src/src/pieces/` - 集成基类
- `framework/src/src/actions/` - 动作基类
- `framework/src/src/triggers/` - 触发器基类
- `framework/src/src/props/` - 属性系统
- `framework/src/src/auth/` - 认证系统

### 社区集成
- `community/*/src/index.ts` - 各集成入口
- `community/*/src/actions/` - 动作实现
- `community/*/src/triggers/` - 触发器实现
- `community/*/src/lib/` - 工具函数
- `community/*/src/i18n/` - 翻译文件

### 配置文件
- `community/*/package.json` - 包配置
- `community/*/tsconfig.json` - TypeScript 配置
- `framework/package.json` - 框架包配置

---

*模块文档版本: 1.0.0*
*最后更新: 2025-11-18 15:10:43*