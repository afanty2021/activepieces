[根目录](../../CLAUDE.md) > [packages](../) > **cli**

# CLI 模块文档

## 模块职责

ActivePieces CLI 是开发者和用户的命令行接口，提供集成开发、构建发布、同步管理等核心功能。它是开发 ActivePieces 集成的官方工具。

## 入口与启动

- **主入口**: `src/index.ts`
- **启动方式**:
  ```bash
  npx activepieces-cli <command>
  # 或
  npm run cli <command>
  ```

### 主要命令组
- `pieces` - 集成管理命令
- `actions` - 动作创建命令
- `triggers` - 触发器创建命令
- `workers` - 工作进程管理命令

## 对外接口

### 核心命令
```typescript
// 集成开发
pieces create          # 创建新集成
pieces sync            # 同步集成到注册表
pieces build           # 构建集成包
pieces publish         # 发布集成到 npm

// 动作和触发器
actions create         # 创建新动作
triggers create        # 创建新触发器

// 工作进程
workers generate-token # 生成工作进程令牌
```

### 关键依赖与配置
- **Commander.js**: 命令行框架
- **@activepieces/pieces-framework**: 集成开发框架
- **@activepieces/shared**: 共享类型和工具
- **TypeScript**: 开发语言

### 配置文件
- `package.json` - 包配置和依赖
- `tsconfig.json` - TypeScript 配置
- `jest.config.ts` - 测试配置

## 核心功能

### 1. 集成开发 (`create-piece`)
```typescript
// 创建新集成的基本结构
pieces create <name>
```
生成的目录结构：
```
<name>/
├── src/
│   ├── index.ts          # 主入口
│   ├── actions/          # 动作定义
│   ├── triggers/         # 触发器定义
│   └── lib/             # 工具函数
├── package.json         # 包配置
├── tsconfig.json        # TS 配置
└── README.md           # 文档
```

### 2. 集成构建 (`build-piece`)
- TypeScript 编译
- 依赖打包
- 元数据生成
- 多平台支持

### 3. 集成发布 (`publish-piece`)
- npm 包发布
- 版本管理
- 注册表同步
- 质量检查

### 4. 工作进程管理
- 生成安全的工作进程令牌
- 支持分布式执行
- 权限控制

## 数据模型

### 集成元数据
```typescript
interface PieceMetadata {
  name: string;
  version: string;
  displayName: string;
  description: string;
  logoUrl: string;
  authors: string[];
  categories: PieceCategory[];
  actions: Action[];
  triggers: Trigger[];
}
```

### 动作定义
```typescript
interface Action {
  name: string;
  displayName: string;
  description: string;
  props: Property[];
  run: (context: ActionContext) => Promise<any>;
}
```

## 测试与质量

### 测试框架
- **Jest**: 单元测试框架
- **测试覆盖率**: 最低 80% 要求
- **测试文件**: `*.spec.ts` 格式

### 质量检查
- TypeScript 严格模式
- ESLint 代码检查
- Prettier 代码格式化
- 自动化测试流水线

## 开发工作流

### 1. 创建新集成
```bash
# 创建集成模板
npm run cli pieces create my-integration

# 开发集成代码
cd packages/pieces/community/my-integration

# 本地测试
npm run cli pieces build my-integration
```

### 2. 开发动作/触发器
```bash
# 创建新动作
npm run cli actions create

# 创建新触发器
npm run cli triggers create
```

### 3. 发布集成
```bash
# 构建集成
npm run cli pieces build my-integration

# 发布到注册表
npm run cli pieces publish my-integration
```

## 常见问题 (FAQ)

### Q: 如何调试 CLI 工具？
A: 使用 `DEBUG=activepieces:*` 环境变量启用调试日志。

### Q: 集成发布失败怎么办？
A: 检查 package.json 配置、npm 登录状态、网络连接等。

### Q: 如何自定义集成模板？
A: 修改 `templates/` 目录下的模板文件。

### Q: 支持哪些包管理器？
A: 支持 npm、yarn、pnpm、bun。

## 相关文件清单

### 核心文件
- `src/index.ts` - CLI 入口和命令注册
- `src/lib/commands/` - 各命令实现
- `src/lib/utils/` - 工具函数库

### 命令实现
- `src/lib/commands/create-piece.ts` - 创建集成命令
- `src/lib/commands/build-piece.ts` - 构建集成命令
- `src/lib/commands/publish-piece.ts` - 发布集成命令
- `src/lib/commands/sync-pieces.ts` - 同步集成命令

### 工具函数
- `src/lib/utils/files.ts` - 文件操作工具
- `src/lib/utils/piece-utils.ts` - 集成相关工具
- `src/lib/utils/exec.ts` - 命令执行工具

### 配置文件
- `package.json` - 包配置
- `tsconfig.json` - TypeScript 配置
- `jest.config.ts` - 测试配置
- `project.json` - Nx 项目配置

---

*模块文档版本: 1.0.0*
*最后更新: 2025-11-18 15:10:43*