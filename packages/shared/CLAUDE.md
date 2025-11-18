[根目录](../../CLAUDE.md) > [packages](../) > **shared**

# Shared 模块文档

## 模块职责

ActivePieces Shared 是整个项目的核心共享库，提供统一的类型定义、接口规范、工具函数和常量。它确保了各个模块间的类型安全和一致性。

## 入口与启动

- **主入口**: `src/index.ts`
- **模块类型**: CommonJS 库
- **导出方式**: 命名导出所有公共接口

### 导出结构
```typescript
// 核心模型
export * from './lib/flows/flow'
export * from './lib/flows/flow-version'
export * from './lib/flows/actions/action'
export * from './lib/flows/triggers/trigger'

// 执行相关
export * from './lib/engine'
export * from './lib/flow-run/flow-run'
export * from './lib/workers/job-data'

// 认证和安全
export * from './lib/authentication/model/principal'
export * from './lib/app-connection/app-connection'

// 工具和常量
export * from './lib/common'
export * from './lib/common/base-model'
```

## 核心类型定义

### 1. 流程模型 (Flow Models)

#### Flow - 流程定义
```typescript
interface Flow {
  id: string;
  name: string;
  displayName: string;
  description?: string;
  projectId: string;
  folderId?: string;
  status: FlowStatus;
  created: Date;
  updated: Date;
  version: FlowVersion;
}

enum FlowStatus {
  DRAFT = 'draft',
  PUBLISHED = 'published',
  ARCHIVED = 'archived',
}
```

#### FlowVersion - 流程版本
```typescript
interface FlowVersion {
  id: string;
  displayName: string;
  trigger: Trigger;
  steps: Step[];
  settings: FlowSettings;
  valid: boolean;
  created: Date;
  updated: Date;
}

interface FlowSettings {
  errorHandling: ErrorHandlingSettings;
  retries?: RetrySettings;
  timeout?: number;
}
```

#### Step - 流程步骤
```typescript
interface Step {
  id: string;
  name: string;
  type: StepType;
  displayName: string;
  settings: Record<string, any>;
  nextStep?: string | StepNextAction;
}

enum StepType {
  ACTION = 'action',
  TRIGGER = 'trigger',
  LOOP = 'loop',
  ROUTER = 'router',
  CODE = 'code',
  DELAY = 'delay',
}

interface StepNextAction {
  type: NextActionType;
  stepName?: string;
  conditions?: Condition[];
}
```

### 2. 集成模型 (Piece Models)

#### Action - 动作定义
```typescript
interface Action {
  name: string;
  displayName: string;
  description: string;
  props: Property[];
  requireAuth?: boolean;
  run: (context: ActionContext) => Promise<any>;
  test?: (context: ActionContext) => Promise<any>;
}

interface ActionContext {
  props: Record<string, any>;
  auth?: AuthenticationType;
  connection?: AppConnection;
  files?: Record<string, File>;
  serverUrl?: string;
  webhookUrl?: string;
}
```

#### Trigger - 触发器定义
```typescript
interface Trigger {
  name: string;
  displayName: string;
  description: string;
  props: Property[];
  requireAuth?: boolean;
  type: TriggerType;
  run: (context: TriggerContext) => Promise<void>;
  test?: (context: TriggerContext) => Promise<any>;
  onEnable?: (context: TriggerContext) => Promise<void>;
  onDisable?: (context: TriggerContext) => Promise<void>;
}

enum TriggerType {
  WEBHOOK = 'webhook',
  POLLING = 'polling',
  SCHEDULE = 'schedule',
  MANUAL = 'manual',
}
```

#### Property - 属性定义
```typescript
interface Property {
  displayName: string;
  description?: string;
  required?: boolean;
  type: PropertyType;
  defaultValue?: any;
  options?: PropertyOption[];
  validators?: Validator[];
}

enum PropertyType {
  STRING = 'string',
  NUMBER = 'number',
  BOOLEAN = 'boolean',
  ARRAY = 'array',
  OBJECT = 'object',
  DROPDOWN = 'dropdown',
  MULTI_SELECT = 'multi_select',
  DATE_TIME = 'date_time',
  FILE = 'file',
  JSON = 'json',
  CODE = 'code',
  LONG_TEXT = 'long_text',
  MARKDOWN = 'markdown',
}
```

### 3. 执行模型 (Execution Models)

#### FlowRun - 流程执行
```typescript
interface FlowRun {
  id: string;
  flowVersionId: string;
  status: FlowRunStatus;
  created: Date;
  updated: Date;
  startedAt?: Date;
  completedAt?: Date;
  duration?: number;
  tags?: string[];
  output?: any;
  error?: ExecutionError;
  stepRuns: StepRun[];
}

enum FlowRunStatus {
  RUNNING = 'running',
  SUCCEEDED = 'succeeded',
  FAILED = 'failed',
  CANCELLED = 'cancelled',
  PAUSED = 'paused',
  TIMEOUT = 'timeout',
}
```

#### StepRun - 步骤执行
```typescript
interface StepRun {
  id: string;
  flowRunId: string;
  stepName: string;
  status: StepRunStatus;
  input: any;
  output?: any;
  error?: ExecutionError;
  duration?: number;
  created: Date;
  updated: Date;
}

enum StepRunStatus {
  RUNNING = 'running',
  SUCCEEDED = 'succeeded',
  FAILED = 'failed',
  CANCELLED = 'cancelled',
}
```

### 4. 认证模型 (Authentication Models)

#### Principal - 主体信息
```typescript
interface Principal {
  id: string;
  type: PrincipalType;
  projectId?: string;
  collectionId?: string;
}

enum PrincipalType {
  USER = 'user',
  SERVICE = 'service',
  API_KEY = 'api_key',
  WEBHOOK = 'webhook',
}
```

#### AppConnection - 应用连接
```typescript
interface AppConnection {
  id: string;
  name: string;
  projectId: string;
  pieceName: string;
  pieceVersion: string;
  status: ConnectionStatus;
  created: Date;
  updated: Date;
  value: AuthenticationValue;
}

enum ConnectionStatus {
  ACTIVE = 'active',
  INACTIVE = 'inactive',
  ERROR = 'error',
}
```

### 5. 通用模型 (Common Models)

#### Base Model
```typescript
interface BaseModel {
  id: string;
  created: Date;
  updated: Date;
}

interface Id {
  id: string;
}
```

#### Pagination
```typescript
interface SeekPage {
  next?: string;
  previous?: string;
  data: any[];
}

interface Cursor {
  limit?: number;
  cursor?: string;
}
```

#### Error Handling
```typescript
interface ActivepiecesError {
  code: string;
  message: string;
  details?: any;
}

interface ExecutionError {
  code: string;
  message: string;
  stack?: string;
  details?: any;
}
```

## 核心工具函数

### 1. ID 生成器
```typescript
export function idGenerator(): string;
export function generateId(): string;
export function generateApiKey(): string;
export function generateWebhookId(): string;
```

### 2. 验证器
```typescript
export function isValidEmail(email: string): boolean;
export function isValidUrl(url: string): boolean;
export function validateRequired(value: any): boolean;
export function validateProperty(property: Property, value: any): ValidationResult;
```

### 3. 转换器
```typescript
export function toJson(value: any): string;
export function parseJson(json: string): any;
export function sanitizeObject(obj: any): any;
export function deepClone<T>(obj: T): T;
```

### 4. 错误处理
```typescript
export function isNil(value: any): boolean;
export function isEmpty(value: any): boolean;
export function createError(code: string, message: string): ActivepiecesError;
export function formatError(error: Error): ExecutionError;
```

## 常量定义

### 1. 系统属性
```typescript
export enum AppSystemProp {
  ENVIRONMENT = 'AP_ENVIRONMENT',
  FRONTEND_URL = 'AP_FRONTEND_URL',
  API_URL = 'AP_API_URL',
  DATABASE_URL = 'AP_DATABASE_URL',
  REDIS_URL = 'AP_REDIS_URL',
  PIECES_SOURCE = 'AP_PIECES_SOURCE',
  SENTRY_DSN = 'AP_SENTRY_DSN',
}
```

### 2. 环境类型
```typescript
export enum ApEnvironment {
  PRODUCTION = 'production',
  STAGING = 'staging',
  DEVELOPMENT = 'development',
  TESTING = 'testing',
}
```

### 3. 版本类型
```typescript
export enum ApEdition {
  COMMUNITY = 'community',
  ENTERPRISE = 'enterprise',
  CLOUD = 'cloud',
}
```

## 项目特定类型

### 1. 项目相关
```typescript
interface Project {
  id: string;
  name: string;
  displayName: string;
  status: ProjectStatus;
  created: Date;
  updated: Date;
}

interface ProjectWithLimits extends Project {
  limits: ProjectLimits;
}

interface ProjectLimits {
  flows: number;
  teamMembers: number;
  executionTime: number;
  storage: number;
}
```

### 2. 用户相关
```typescript
interface User {
  id: string;
  email: string;
  firstName: string;
  lastName: string;
  avatarUrl?: string;
  status: UserStatus;
  created: Date;
  updated: Date;
}

interface UserWithMetaInformation extends User {
  meta: UserMeta;
}

interface UserMeta {
  onboardingCompleted: boolean;
  lastLoginAt?: Date;
  loginCount: number;
}
```

### 3. 文件相关
```typescript
interface File {
  id: string;
  name: string;
  size: number;
  mimeType: string;
  data: Buffer;
  projectId: string;
  created: Date;
  updated: Date;
}
```

## 使用指南

### 1. 导入方式
```typescript
// 推荐的导入方式
import { Flow, FlowVersion, Step, Action } from '@activepieces/shared';

// 或者按模块导入
import { Flow } from '@activepieces/shared/lib/flows/flow';
import { Action } from '@activepieces/shared/lib/flows/actions/action';
```

### 2. 类型使用
```typescript
// 定义函数参数类型
function executeFlow(flow: Flow, input: any): Promise<FlowRun> {
  // 实现逻辑
}

// 定义接口返回类型
interface ApiResponse<T> {
  data: T;
  status: 'success' | 'error';
}
```

### 3. 常量使用
```typescript
import { AppSystemProp, ApEnvironment } from '@activepieces/shared';

// 获取环境变量
const environment = process.env[AppSystemProp.ENVIRONMENT];

// 检查环境类型
if (environment === ApEnvironment.PRODUCTION) {
  // 生产环境逻辑
}
```

## 扩展和自定义

### 1. 扩展接口
```typescript
// 扩展现有接口
interface CustomAction extends Action {
  customProperty: string;
  customMethod(): void;
}

// 扩展枚举
enum CustomStepType extends StepType {
  CUSTOM_STEP = 'custom_step',
}
```

### 2. 自定义工具函数
```typescript
// 添加自定义工具函数
export function customValidator(value: any): boolean {
  // 自定义验证逻辑
  return true;
}
```

## 版本兼容性

### 1. 语义化版本
- 主版本：破坏性变更
- 次版本：新功能添加
- 修订版本：bug 修复和优化

### 2. 向后兼容
- 新增字段：向后兼容
- 修改字段：谨慎处理，可能影响下游
- 删除字段：需要主版本升级

### 3. 迁移指南
提供详细的迁移文档和工具，帮助用户平滑升级。

## 常见问题 (FAQ)

### Q: 如何添加新的类型定义？
A: 在相应模块目录下创建类型文件，并在 index.ts 中导出。

### Q: 如何确保类型安全？
A: 使用 TypeScript 严格模式，禁用隐式 any，完整覆盖。

### Q: 如何处理版本升级？
A: 遵循语义化版本规范，提供迁移指南和废弃警告。

### Q: 如何优化包大小？
A: 使用 tree-shaking，按需导入，避免循环依赖。

## 相关文件清单

### 核心模块
- `src/lib/flows/` - 流程相关类型
- `src/lib/flow-run/` - 执行相关类型
- `src/lib/flows/actions/` - 动作类型
- `src/lib/flows/triggers/` - 触发器类型
- `src/lib/app-connection/` - 连接相关类型
- `src/lib/authentication/` - 认证相关类型
- `src/lib/workers/` - 工作进程相关类型
- `src/lib/common/` - 通用类型和工具

### 具体文件
- `src/index.ts` - 主导出文件
- `src/lib/common/base-model.ts` - 基础模型
- `src/lib/common/id-generator.ts` - ID 生成器
- `src/lib/common/telemetry.ts` - 遥测相关
- `package.json` - 包配置
- `tsconfig.json` - TypeScript 配置

---

*模块文档版本: 1.0.0*
*最后更新: 2025-11-18 15:10:43*