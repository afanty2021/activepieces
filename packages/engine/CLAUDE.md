[根目录](../../CLAUDE.md) > [packages](../) > **engine**

# Engine 模块文档

## 模块职责

ActivePieces Engine 是流程执行的核心引擎，负责安全、高效地执行用户创建的自动化流程。它采用沙箱隔离架构，确保代码执行的安全性和稳定性。

## 入口与启动

- **主入口**: `src/main.ts`
- **启动方式**: 通过环境变量 `WORKER_ID` 启动工作进程
- **通信协议**: WebSocket 与主进程通信

### 启动流程
```typescript
// 环境变量
WORKER_ID=<worker-id> node dist/main.js

// WebSocket 连接
WS_URL = 'ws://127.0.0.1:12345/worker/ws'
```

## 对外接口

### 核心操作类型
```typescript
enum EngineOperationType {
  TRIGGER_HOOK = 'trigger-hook',
  EXECUTE_FLOW = 'execute-flow',
  EXECUTE_STEP = 'execute-step',
  PIECE_METADATA = 'piece-metadata',
  PROPERTY_OPERATION = 'property-operation',
  TOOL_OPERATION = 'tool-operation',
  AUTH_VALIDATION = 'auth-validation',
}
```

### WebSocket 事件
```typescript
enum EngineSocketEvent {
  ENGINE_OPERATION = 'engine-operation',
  ENGINE_RESPONSE = 'engine-response',
  ENGINE_STDOUT = 'engine-stdout',
  ENGINE_STDERR = 'engine-stderr',
}
```

### 关键依赖与配置
- **@activepieces/pieces-framework**: 集成框架
- **@activepieces/shared**: 共享类型
- **isolated-vm**: V8 沙箱执行环境
- **ws**: WebSocket 客户端
- **@sinclair/typebox**: 类型验证

## 核心架构

### 1. 沙箱执行系统
```typescript
interface CodeSandbox {
  execute(code: string, context: ExecutionContext): Promise<any>;
  destroy(): void;
}
```

支持的沙箱类型：
- **V8 Isolate**: 生产环境，高性能安全隔离
- **No-Op**: 测试环境，直接执行

### 2. 流程执行器
```typescript
class FlowExecutor {
  async execute(flow: Flow, input: FlowExecuteInput): Promise<FlowExecutionOutput>;
  async executeStep(step: Step, context: StepExecutionContext): Promise<StepOutput>;
  async handleLoop(loop: Loop, context: LoopContext): Promise<LoopResult>;
}
```

### 3. 集成执行器
```typescript
class PieceExecutor {
  async executeAction(action: Action, props: Record<string, any>): Promise<any>;
  async executeTrigger(trigger: Trigger, payload: any): Promise<void>;
  async validateAuth(auth: AuthenticationType): Promise<boolean>;
}
```

### 4. 变量处理器
```typescript
class PropsProcessor {
  resolve(props: Record<string, any>, context: ExecutionContext): Promise<Record<string, any>>;
  processVariable(variable: string, context: ExecutionContext): Promise<any>;
}
```

## 数据模型

### 执行上下文
```typescript
interface ExecutionContext {
  flow: Flow;
  flowVersion: FlowVersion;
  step: Step;
  runId: string;
  projectId: string;
  variables: Record<string, any>;
  connections: AppConnection[];
}
```

### 执行结果
```typescript
interface EngineResponse {
  response?: any;
  status: EngineResponseStatus;
  error?: EngineError;
}

enum EngineResponseStatus {
  OK = 'ok',
  INTERNAL_ERROR = 'internal-error',
  PREMATURE_END = 'premature-end',
}
```

### 错误处理
```typescript
class EngineGenericError extends Error {
  constructor(type: string, message: string);
}

enum ErrorType {
  WORKER_ID_NOT_SET = 'WorkerIdNotSetError',
  STEP_TIMEOUT = 'StepTimeoutError',
  CODE_EXECUTION = 'CodeExecutionError',
}
```

## 关键特性

### 1. 安全沙箱
- **进程隔离**: 每个执行在独立的 V8 Isolate 中运行
- **资源限制**: 内存、CPU 使用限制
- **API 限制**: 只允许安全的内置 API
- **网络控制**: 可配置的网络访问策略

### 2. 实时日志
- **Stdout 重定向**: 执行日志实时传输
- **错误过滤**: 敏感信息自动脱敏
- **日志分级**: 支持 debug、info、error 等级别

### 3. 变量系统
```typescript
// 支持的变量类型
{{ stepName.output.property }}     // 步骤输出
{{ flowVariable.name }}            // 流程变量
{{ connection.apiKey }}            // 连接信息
{{ env.ENVIRONMENT_VARIABLE }}     // 环境变量
{{ 'static value' }}               // 静态值
```

### 4. 数据转换
- **数组处理**: map、filter、zip 等操作
- **对象操作**: merge、pick、omit 等操作
- **类型转换**: 自动类型推断和转换
- **函数调用**: 支持自定义转换函数

## 性能优化

### 1. 缓存机制
- **集成缓存**: 热加载集成代码
- **变量缓存**: 缓存复杂计算结果
- **连接缓存**: 复用网络连接

### 2. 资源管理
- **内存限制**: 单个执行最大内存使用
- **超时控制**: 执行超时自动终止
- **并发控制**: 限制同时执行的进程数

### 3. 错误恢复
- **自动重试**: 网络错误自动重试
- **断点续传**: 长时间执行的流程支持暂停/恢复
- **状态持久化**: 执行状态定期保存

## 测试与质量

### 测试策略
- **单元测试**: 核心逻辑覆盖
- **集成测试**: 完整流程执行测试
- **性能测试**: 执行性能基准
- **安全测试**: 沙箱安全验证

### 测试资源
```
test/
├── handler/                     # 执行器测试
├── resources/codes/            # 测试代码样本
└── resources/flows/            # 测试流程样本
```

## 监控与调试

### 1. 执行监控
- **执行时间**: 每个步骤的执行时间统计
- **资源使用**: 内存、CPU 使用情况
- **错误率**: 执行成功率统计
- **吞吐量**: 并发执行能力

### 2. 调试工具
- **详细日志**: 分级日志输出
- **执行追踪**: 步骤执行路径
- **变量检查**: 执行过程中的变量值
- **错误堆栈**: 详细的错误信息

### 3. 性能调优
- **沙箱配置**: 根据场景调整沙箱参数
- **缓存策略**: 优化缓存命中率
- **并发设置**: 调整执行并发数
- **资源配额**: 合理分配资源限制

## 常见问题 (FAQ)

### Q: 如何调试沙箱中的代码？
A: 使用 `console.log` 输出会通过 WebSocket 传输到主进程日志中。

### Q: 执行超时如何处理？
A: 检查代码逻辑、增加超时时间、优化算法复杂度。

### Q: 如何提高执行性能？
A: 使用缓存、减少网络请求、优化算法、调整并发数。

### Q: 沙箱安全性如何保证？
A: V8 Isolate 提供进程级隔离，限制系统 API 访问，过滤敏感信息。

## 相关文件清单

### 核心文件
- `src/main.ts` - 引擎主入口和 WebSocket 通信
- `src/lib/handler/` - 各种执行器实现
- `src/lib/core/code/` - 沙箱代码执行
- `src/lib/operations/` - 操作处理器
- `src/lib/services/` - 核心服务实现

### 执行器
- `src/lib/handler/flow-executor.ts` - 流程执行器
- `src/lib/handler/step-executor.ts` - 步骤执行器
- `src/lib/handler/piece-executor.ts` - 集成执行器
- `src/lib/handler/router-executor.ts` - 路由执行器
- `src/lib/handler/loop-executor.ts` - 循环执行器

### 沙箱实现
- `src/lib/core/code/code-sandbox.ts` - 沙箱接口
- `src/lib/core/code/v8-isolate-code-sandbox.ts` - V8 沙箱实现
- `src/lib/core/code/no-op-code-sandbox.ts` - 测试沙箱

### 变量处理
- `src/lib/variables/props-processor.ts` - 属性处理器
- `src/lib/variables/props-resolver.ts` - 属性解析器
- `src/lib/variables/processors/` - 各种数据类型处理器

### 工具函数
- `src/lib/utils.ts` - 通用工具函数
- `src/lib/helper/` - 辅助函数库
- `test/` - 测试文件和资源

### 配置文件
- `package.json` - 包配置
- `tsconfig.json` - TypeScript 配置
- `jest.config.ts` - 测试配置
- `webpack.config.js` - Webpack 打包配置

---

*模块文档版本: 1.0.0*
*最后更新: 2025-11-18 15:10:43*