[根目录](../../CLAUDE.md) > [packages](../) > **ee**

# EE (Enterprise Edition) 模块文档

> **变更记录 (Changelog)**
> - 2025-11-18 15:28:53 - 增量更新：补充企业认证和审计事件的详细分析

## 模块职责

ActivePieces Enterprise Edition (EE) 提供企业级功能和服务，包括高级安全特性、多租户支持、企业级认证、审计日志、API 管理等。它分为共享库和嵌入式 UI 组件两个主要部分。

## 模块结构

```
packages/ee/
├── shared/                # 企业版共享库
│   ├── src/
│   │   └── lib/          # 企业级功能实现
│   ├── package.json
│   └── tsconfig.json
├── ui/
│   └── embed-sdk/        # 嵌入式 SDK
│       ├── src/
│       │   └── index.ts  # SDK 入口
│       ├── package.json
│       └── webpack.config.js
└── README.md
```

## 核心功能深度分析

### 1. 基于角色的访问控制 (RBAC)

#### 权限矩阵 (`access-control-list.ts`)
根据扫描结果，EE 实现了完整的权限控制体系：

```typescript
// 系统权限定义
enum Permission {
  // 流程权限
  READ_FLOW = 'READ_FLOW',
  WRITE_FLOW = 'WRITE_FLOW',
  UPDATE_FLOW_STATUS = 'UPDATE_FLOW_STATUS',

  // 连接权限
  READ_APP_CONNECTION = 'READ_APP_CONNECTION',
  WRITE_APP_CONNECTION = 'WRITE_APP_CONNECTION',

  // 项目管理权限
  READ_PROJECT = 'READ_PROJECT',
  WRITE_PROJECT = 'WRITE_PROJECT',
  READ_PROJECT_MEMBER = 'READ_PROJECT_MEMBER',
  WRITE_PROJECT_MEMBER = 'WRITE_PROJECT_MEMBER',

  // 运行权限
  READ_RUN = 'READ_RUN',
  WRITE_RUN = 'WRITE_RUN',

  // 企业版功能权限
  READ_ALERT = 'READ_ALERT',
  WRITE_ALERT = 'WRITE_ALERT',
  READ_TODOS = 'READ_TODOS',
  WRITE_TODOS = 'WRITE_TODOS',
  READ_TABLE = 'READ_TABLE',
  WRITE_TABLE = 'WRITE_TABLE',
  READ_MCP = 'READ_MCP',
  WRITE_MCP = 'WRITE_MCP',
}

// 角色权限映射
const rolePermissions: Record<DefaultProjectRole, Permission[]> = {
  [DefaultProjectRole.ADMIN]: [
    // 管理员拥有所有权限
    Permission.READ_APP_CONNECTION,
    Permission.WRITE_APP_CONNECTION,
    Permission.READ_FLOW,
    Permission.WRITE_FLOW,
    Permission.UPDATE_FLOW_STATUS,
    Permission.READ_PROJECT_MEMBER,
    Permission.WRITE_PROJECT_MEMBER,
    Permission.WRITE_INVITATION,
    Permission.READ_INVITATION,
    Permission.WRITE_PROJECT_RELEASE,
    Permission.READ_PROJECT_RELEASE,
    Permission.READ_RUN,
    Permission.WRITE_RUN,
    Permission.READ_ISSUES,
    Permission.WRITE_ISSUES,
    Permission.WRITE_ALERT,
    Permission.READ_ALERT,
    Permission.WRITE_PROJECT,
    Permission.READ_PROJECT,
    Permission.WRITE_FOLDER,
    Permission.READ_FOLDER,
    Permission.READ_TODOS,
    Permission.WRITE_TODOS,
    Permission.READ_TABLE,
    Permission.WRITE_TABLE,
    Permission.READ_MCP,
    Permission.WRITE_MCP,
  ],
  [DefaultProjectRole.EDITOR]: [
    // 编辑者权限：可编辑流程和运行，但不能管理成员
    Permission.READ_APP_CONNECTION,
    Permission.WRITE_APP_CONNECTION,
    Permission.READ_FLOW,
    Permission.WRITE_FLOW,
    Permission.UPDATE_FLOW_STATUS,
    Permission.READ_PROJECT_MEMBER,
    Permission.READ_INVITATION,
    Permission.WRITE_PROJECT_RELEASE,
    Permission.READ_PROJECT_RELEASE,
    Permission.READ_RUN,
    Permission.WRITE_RUN,
    Permission.READ_ISSUES,
    Permission.WRITE_ISSUES,
    Permission.READ_PROJECT,
    Permission.WRITE_FOLDER,
    Permission.READ_FOLDER,
    Permission.READ_TODOS,
    Permission.WRITE_TODOS,
    Permission.READ_TABLE,
    Permission.WRITE_TABLE,
    Permission.READ_MCP,
    Permission.WRITE_MCP,
  ],
  [DefaultProjectRole.OPERATOR]: [
    // 操作者权限：可运行流程，但不能编辑
    Permission.READ_APP_CONNECTION,
    Permission.WRITE_APP_CONNECTION,
    Permission.READ_FLOW,
    Permission.UPDATE_FLOW_STATUS,
    Permission.READ_PROJECT_MEMBER,
    Permission.READ_INVITATION,
    Permission.READ_PROJECT_RELEASE,
    Permission.READ_RUN,
    Permission.WRITE_RUN,
    Permission.READ_ISSUES,
    Permission.READ_PROJECT,
    Permission.READ_FOLDER,
    Permission.READ_TODOS,
    Permission.WRITE_TODOS,
    Permission.READ_TABLE,
    Permission.READ_MCP,
  ],
  [DefaultProjectRole.VIEWER]: [
    // 查看者权限：只读访问
    Permission.READ_APP_CONNECTION,
    Permission.READ_FLOW,
    Permission.READ_PROJECT_MEMBER,
    Permission.READ_INVITATION,
    Permission.READ_ISSUES,
    Permission.READ_PROJECT,
    Permission.READ_RUN,
    Permission.READ_FOLDER,
    Permission.READ_TODOS,
    Permission.READ_TABLE,
    Permission.READ_MCP,
  ],
};
```

#### 权限检查中间件
```typescript
// 权限验证装饰器
@RequirePermission(Permission.WRITE_FLOW)
async updateFlow(request: UpdateFlowRequest): Promise<Flow> {
  // 实现逻辑
}

// 批量权限检查
const hasPermissions = (
  user: User,
  requiredPermissions: Permission[]
): boolean => {
  const userRole = user.projectRole;
  const userPermissions = rolePermissions[userRole];
  return requiredPermissions.every(p => userPermissions.includes(p));
};
```

### 2. 企业级认证系统

#### 本地认证增强 (`enterprise-local-authn/`)
基于扫描的企业认证请求类型：

```typescript
// 邮箱验证请求
export interface VerifyEmailRequestBody {
  identityId: ApId;
  otp: string;
}

// 密码重置请求
export interface ResetPasswordRequestBody {
  identityId: ApId;
  otp: string;
  newPassword: string;
}

// 邀请注册请求
export interface SignUpAndAcceptRequestBody {
  // 继承 SignUpRequest 的字段，但排除 email
  firstName: string;
  lastName: string;
  password: string;
  invitationToken: string;
}

// 认证流程
class EnterpriseAuthService {
  // 邮箱验证
  async verifyEmail(request: VerifyEmailRequestBody): Promise<void> {
    const { identityId, otp } = request;
    // 验证 OTP 码
    // 激活用户账户
    // 发送审计事件
  }

  // 密码重置
  async resetPassword(request: ResetPasswordRequestBody): Promise<void> {
    const { identityId, otp, newPassword } = request;
    // 验证 OTP 码
    // 更新密码
    // 使所有现有会话失效
    // 发送安全通知
  }

  // 邀请注册
  async signUpAndAccept(request: SignUpAndAcceptRequestBody): Promise<User> {
    const { invitationToken, ...signUpData } = request;
    // 验证邀请令牌
    // 创建用户账户
    // 自动加入项目
    // 发送欢迎邮件
  }
}
```

#### SSO 集成支持
```typescript
// SAML 配置接口
interface SamlConfiguration {
  enabled: boolean;
  entryPoint: string;
  issuer: string;
  cert: string;
  privateKey: string;
  attributeMapping: {
    email: string;
    firstName: string;
    lastName: string;
    role?: string;
  };
  defaultRole: DefaultProjectRole;
}

// OAuth2 配置接口
interface OAuth2Configuration {
  enabled: boolean;
  provider: 'google' | 'microsoft' | 'github' | 'custom';
  clientId: string;
  clientSecret: string;
  scopes: string[];
  redirectUri: string;
  attributeMapping: Record<string, string>;
}

// SSO 认证服务
class SsoAuthService {
  async configureSaml(config: SamlConfiguration): Promise<void> {
    // 配置 SAML 提供商
    // 设置元数据端点
    // 测试连接
  }

  async configureOAuth2(config: OAuth2Configuration): Promise<void> {
    // 配置 OAuth2 提供商
    // 注册重定向 URI
    // 测试连接
  }

  async handleSamlResponse(samlResponse: string): Promise<AuthResult> {
    // 解析 SAML 响应
    // 验证签名
    // 提取用户属性
    // 创建或更新用户
    // 生成本地会话
  }
}
```

### 3. 全面审计日志系统

#### 审计事件类型 (`audit-events/index.ts`)
根据扫描结果，系统定义了丰富的审计事件：

```typescript
// 核心审计事件类型
enum ApplicationEventName {
  // 流程事件
  FLOW_CREATED = 'flow.created',
  FLOW_DELETED = 'flow.deleted',
  FLOW_UPDATED = 'flow.updated',
  FLOW_RUN_RESUMED = 'flow.run.resumed',
  FLOW_RUN_STARTED = 'flow.run.started',
  FLOW_RUN_FINISHED = 'flow.run.finished',

  // 文件夹事件
  FOLDER_CREATED = 'folder.created',
  FOLDER_UPDATED = 'folder.updated',
  FOLDER_DELETED = 'folder.deleted',

  // 连接事件
  CONNECTION_UPSERTED = 'connection.upserted',
  CONNECTION_DELETED = 'connection.deleted',

  // 用户事件
  USER_SIGNED_UP = 'user.signed.up',
  USER_SIGNED_IN = 'user.signed.in',
  USER_PASSWORD_RESET = 'user.password.reset',
  USER_EMAIL_VERIFIED = 'user.email.verified',

  // 安全事件
  SIGNING_KEY_CREATED = 'signing.key.created',
  PROJECT_ROLE_CREATED = 'project.role.created',
  PROJECT_ROLE_DELETED = 'project.role.deleted',
  PROJECT_ROLE_UPDATED = 'project.role.updated',

  // 项目事件
  PROJECT_RELEASE_CREATED = 'project.release.created',
}
```

#### 结构化审计事件
```typescript
// 基础审计事件属性
interface BaseAuditEventProps {
  id: string;
  created: Date;
  updated: Date;
  platformId: string;
  projectId?: string;
  projectDisplayName?: string;
  userId?: string;
  userEmail?: string;
  ip?: string;
}

// 流程更新事件
export const FlowUpdatedEvent = Type.Object({
  ...BaseAuditEventProps,
  action: Type.Literal(ApplicationEventName.FLOW_UPDATED),
  data: Type.Object({
    flowVersion: Type.Pick(FlowVersion, [
      'id',
      'displayName',
      'flowId',
      'created',
      'updated',
    ]),
    request: FlowOperationRequest,
    project: Type.Optional(Type.Pick(Project, ['displayName'])),
  }),
});

// 连接事件
export const ConnectionEvent = Type.Object({
  ...BaseAuditEventProps,
  action: Type.Union([
    Type.Literal(ApplicationEventName.CONNECTION_DELETED),
    Type.Literal(ApplicationEventName.CONNECTION_UPSERTED),
  ]),
  data: Type.Object({
    connection: Type.Pick(AppConnectionWithoutSensitiveData, [
      'displayName',
      'externalId',
      'pieceName',
      'status',
      'type',
      'id',
      'created',
      'updated',
    ]),
    project: Type.Optional(Type.Pick(Project, ['displayName'])),
  }),
});

// 认证事件
export const AuthenticationEvent = Type.Object({
  ...BaseAuditEventProps,
  action: Type.Union([
    Type.Literal(ApplicationEventName.USER_SIGNED_IN),
    Type.Literal(ApplicationEventName.USER_PASSWORD_RESET),
    Type.Literal(ApplicationEventName.USER_EMAIL_VERIFIED),
  ]),
  data: Type.Object({
    user: Type.Optional(UserMeta),
  }),
});
```

#### 智能事件摘要
```typescript
// 自动生成可读的事件摘要
export function summarizeApplicationEvent(event: ApplicationEvent): string {
  switch (event.action) {
    case ApplicationEventName.FLOW_UPDATED:
      return convertUpdateActionToDetails(event);
    case ApplicationEventName.FLOW_RUN_STARTED:
      return `Flow run ${event.data.flowRun.id} is started`;
    case ApplicationEventName.FLOW_RUN_FINISHED:
      return `Flow run ${event.data.flowRun.id} is finished`;
    case ApplicationEventName.FLOW_CREATED:
      return `Flow ${event.data.flow.id} is created`;
    case ApplicationEventName.CONNECTION_UPSERTED:
      return `${event.data.connection.displayName} (${event.data.connection.externalId}) is updated`;
    case ApplicationEventName.USER_SIGNED_IN:
      return `User ${event.userEmail} signed in`;
    // ... 更多事件类型
  }
}

// 详细的流程操作描述
function convertUpdateActionToDetails(event: FlowUpdatedEvent): string {
  const { request, flowVersion } = event.data;

  switch (request.type) {
    case FlowOperationType.ADD_ACTION:
      return `Added action "${request.request.action.displayName}" to "${flowVersion.displayName}" Flow.`;
    case FlowOperationType.UPDATE_ACTION:
      return `Updated action "${request.request.displayName}" in "${flowVersion.displayName}" Flow.`;
    case FlowOperationType.DELETE_ACTION:
      return `Deleted actions "${request.request.names.join(', ')}" from "${flowVersion.displayName}" Flow.`;
    case FlowOperationType.CHANGE_NAME:
      return `Renamed flow "${flowVersion.displayName}" to "${request.request.displayName}".`;
    // ... 更多操作类型
  }
}
```

#### 审计查询接口
```typescript
// 审计事件查询请求
export interface ListAuditEventsRequest {
  limit?: number;
  cursor?: string;
  action?: string[];
  projectId?: string[];
  userId?: string;
  createdBefore?: string;
  createdAfter?: string;
}

// 审计服务
class AuditService {
  async listEvents(request: ListAuditEventsRequest): Promise<SeekPage<ApplicationEvent>> {
    // 分页查询审计事件
    // 应用过滤条件
    // 按时间倒序排列
  }

  async exportEvents(filter: AuditFilter): Promise<Buffer> {
    // 导出审计数据
    // 支持多种格式 (JSON, CSV, PDF)
    // 压缩大型数据集
  }

  async searchEvents(query: AuditSearchQuery): Promise<ApplicationEvent[]> {
    // 全文搜索审计事件
    // 支持复杂查询条件
  }

  async getEventStatistics(filter: AuditFilter): Promise<AuditStatistics> {
    // 统计分析
    // 生成图表数据
  }
}
```

### 4. 高级安全特性

#### 数字签名系统
```typescript
// 签名密钥管理
interface SigningKey {
  id: string;
  keyId: string;
  publicKey: string;
  privateKey: string;
  algorithm: 'RS256' | 'ES256';
  projectId: string;
  created: Date;
  expiresAt: Date;
}

// 签名服务
class SigningService {
  async createSigningKey(projectId: string): Promise<SigningKey> {
    // 生成密钥对
    // 安全存储私钥
    // 返回公钥信息
  }

  async signData(keyId: string, data: any): Promise<string> {
    // 加载私钥
    // 签名数据
    // 返回签名结果
  }

  async verifySignature(keyId: string, data: any, signature: string): Promise<boolean> {
    // 加载公钥
    // 验证签名
    // 返回验证结果
  }
}
```

#### 连接密钥管理
```typescript
// 连接密钥
interface ConnectionKey {
  id: string;
  name: string;
  keyId: string;
  publicKey: string;
  expiresAt: Date;
  projectId: string;
  created: Date;
}

// 密钥轮换策略
interface KeyRotationPolicy {
  autoRotate: boolean;
  rotationInterval: number; // days
  expirationWarning: number; // days before expiration
}
```

### 5. 合规与数据保护

#### 数据保留策略
```typescript
// 数据保留配置
interface DataRetentionPolicy {
  auditLogs: {
    retentionDays: number;
    autoArchive: boolean;
    archiveLocation?: string;
  };
  flowRuns: {
    retentionDays: number;
    autoDelete: boolean;
  };
  userActivity: {
    retentionDays: number;
    anonymizationEnabled: boolean;
  };
}

// 数据脱敏
interface DataMaskingRule {
  id: string;
  name: string;
  pattern: RegExp;
  replacement: string;
  enabled: boolean;
  scopes: string[];
}

// 脱敏服务
class DataMaskingService {
  async maskSensitiveData(data: any, context: MaskingContext): Promise<any> {
    // 检测敏感信息
    // 应用脱敏规则
    // 保持数据结构完整性
  }

  async addMaskingRule(rule: DataMaskingRule): Promise<void> {
    // 验证规则
    // 保存规则
    // 更新脱敏引擎
  }
}
```

### 6. 企业监控与告警

#### 告警系统
```typescript
// 告警类型
enum AlertType {
  FLOW_FAILURE = 'flow_failure',
  QUOTA_EXCEEDED = 'quota_exceeded',
  SECURITY_BREACH = 'security_breach',
  SYSTEM_ERROR = 'system_error',
  LICENSE_EXPIRATION = 'license_expiration',
}

// 告警严重程度
enum AlertSeverity {
  LOW = 'low',
  MEDIUM = 'medium',
  HIGH = 'high',
  CRITICAL = 'critical',
}

// 告警定义
interface Alert {
  id: string;
  type: AlertType;
  severity: AlertSeverity;
  title: string;
  message: string;
  projectId: string;
  userId?: string;
  created: Date;
  acknowledged: boolean;
  acknowledgedBy?: string;
  acknowledgedAt?: Date;
}

// 告警服务
class AlertService {
  async createAlert(alert: CreateAlertRequest): Promise<Alert> {
    // 创建告警
    // 发送通知
    // 记录审计事件
  }

  async acknowledgeAlert(alertId: string, userId: string): Promise<void> {
    // 确认告警
    // 更新状态
    // 发送确认通知
  }

  async resolveAlert(alertId: string, resolution: string): Promise<void> {
    // 解决告警
    // 记录解决方案
    // 生成报告
  }
}
```

### 7. 多租户增强

#### 项目级隔离
```typescript
// 项目配置
interface ProjectConfiguration {
  id: string;
  name: string;
  domain?: string;
  branding: ProjectBranding;
  limits: ResourceLimits;
  security: SecuritySettings;
  compliance: ComplianceSettings;
}

// 资源限制
interface ResourceLimits {
  flows: number;
  teamMembers: number;
  executionTime: number; // minutes per month
  storage: number; // GB
  apiCalls: number;
  customPieces: number;
}

// 配额管理
class QuotaManagementService {
  async checkQuota(
    projectId: string,
    resource: ResourceType,
    amount: number
  ): Promise<QuotaCheckResult> {
    // 检查当前使用量
    // 计算剩余配额
    // 返回检查结果
  }

  async updateUsage(
    projectId: string,
    resource: ResourceType,
    amount: number
  ): Promise<void> {
    // 更新使用量
    // 检查是否超限
    // 触发告警（如需要）
  }

  async getUsageReport(projectId: string, period: ReportingPeriod): Promise<UsageReport> {
    // 生成使用报告
    // 包含趋势分析
    // 提供优化建议
  }
}
```

## 集成与部署

### 1. 环境配置
```bash
# 企业版基础配置
AP_EDITION=enterprise
AP_LICENSE_KEY=your-enterprise-license-key

# 审计配置
AP_AUDIT_ENABLED=true
AP_AUDIT_RETENTION_DAYS=2555
AP_AUDIT_EXPORT_PATH=/var/log/activepieces/audit

# RBAC 配置
AP_RBAC_ENABLED=true
AP_DEFAULT_ROLE=viewer

# SSO 配置
AP_SAML_ENABLED=true
AP_SAML_ENTRY_POINT=https://your-idp.com/saml
AP_SAML_ISSUER=activepieces
AP_SAML_CERT=-----BEGIN CERTIFICATE-----...

# OAuth2 配置
AP_OAUTH2_GOOGLE_ENABLED=true
AP_OAUTH2_GOOGLE_CLIENT_ID=your-google-client-id
AP_OAUTH2_GOOGLE_CLIENT_SECRET=your-google-client-secret

# 数据保护配置
AP_DATA_ENCRYPTION_KEY=your-encryption-key
AP_DATA_MASKING_ENABLED=true
AP_DATA_RETENTION_DAYS=365

# 监控配置
AP_ALERTS_ENABLED=true
AP_ALERT_WEBHOOK_URL=https://your-webhook.com/alerts
```

### 2. 许可证管理
```typescript
// 企业许可证验证
interface EnterpriseLicense {
  key: string;
  edition: ApEdition.ENTERPRISE;
  expiresAt: Date;
  features: LicenseFeature[];
  limits: LicenseLimits;
  signature: string;
}

enum LicenseFeature {
  SSO = 'sso',
  AUDIT_LOGS = 'audit_logs',
  RBAC = 'rbac',
  API_MANAGEMENT = 'api_management',
  ADVANCED_SECURITY = 'advanced_security',
  MULTI_TENANT = 'multi_tenant',
}

class LicenseManager {
  async validateLicense(key: string): Promise<EnterpriseLicense> {
    // 验证许可证签名
    // 检查有效期
    // 验证功能授权
  }

  async checkFeature(feature: LicenseFeature): Promise<boolean> {
    // 检查功能是否授权
  }

  async getLicenseInfo(): Promise<LicenseInfo> {
    // 获取许可证信息
    // 包含剩余天数
    // 功能列表和使用限制
  }
}
```

### 3. 嵌入式 SDK 更新
```typescript
// 企业版嵌入式 SDK 功能增强
interface EnterpriseEmbedConfig extends EmbedConfig {
  // 企业功能
  enableSso?: boolean;
  customTheme?: EnterpriseTheme;
  branding?: BrandingConfig;
  complianceMode?: ComplianceMode;
}

// 品牌定制
interface BrandingConfig {
  logo?: string;
  primaryColor?: string;
  fontFamily?: string;
  customCSS?: string;
  favicon?: string;
  domain?: string;
}

// 合规模式
enum ComplianceMode {
  NONE = 'none',
  GDPR = 'gdpr',
  HIPAA = 'hipaa',
  SOX = 'sox',
}

// SDK 企业功能
class EnterpriseEmbedSDK extends ActivePiecesEmbed {
  constructor(config: EnterpriseEmbedConfig) {
    super(config);
    this.configureEnterpriseFeatures(config);
  }

  private configureEnterpriseFeatures(config: EnterpriseEmbedConfig): void {
    // 配置 SSO 登录
    // 应用品牌定制
    // 启用合规模式
    // 设置数据保护
  }

  // SSO 集成
  async signInWithSso(provider: SsoProvider): Promise<AuthResult> {
    // 处理 SSO 登录流程
  }

  // 合规报告
  async generateComplianceReport(type: ComplianceReportType): Promise<ComplianceReport> {
    // 生成合规报告
  }
}
```

## 安全最佳实践

### 1. 认证安全
- **多因子认证**: 支持 TOTP、邮件、短信验证
- **会话管理**: 安全的会话令牌和自动过期
- **密码策略**: 强密码要求和定期更换
- **异常检测**: 登录异常行为监控和阻止

### 2. 数据安全
- **端到端加密**: 敏感数据传输和存储加密
- **访问控制**: 细粒度的数据访问控制
- **审计追踪**: 完整的数据访问审计日志
- **备份恢复**: 安全的数据备份和恢复机制

### 3. 网络安全
- **HTTPS 强制**: 所有通信强制使用 HTTPS
- **CORS 配置**: 严格的跨域资源共享配置
- **速率限制**: API 访问速率限制
- **DDoS 防护**: 分布式拒绝服务攻击防护

### 4. 合规支持
- **GDPR 合规**: 欧盟数据保护法规合规
- **SOC2 认证**: 服务组织控制 2 型认证
- **HIPAA 合规**: 医疗保险便携性和责任法案合规
- **数据本地化**: 数据存储地域限制支持

## 常见问题 (FAQ)

### Q: 如何配置企业级 SSO？
A: 在企业设置中配置 SAML 或 OAuth2 提供商，设置属性映射，启用 SSO 登录。

### Q: 审计日志保留多久？
A: 默认保留 7 年（2555 天），可根据合规要求调整。

### Q: 如何自定义用户角色和权限？
A: 使用 RBAC 系统定义自定义角色，分配特定权限组合。

### Q: 数据脱敏如何工作？
A: 系统自动检测敏感数据（如邮箱、电话、身份证），应用预定义脱敏规则。

### Q: 如何监控企业版使用情况？
A: 使用内置的使用报告和配额管理功能，实时监控资源使用情况。

## 相关文件清单

### 企业认证 (`authn/`)
- `src/lib/authn/access-control-list.ts` - RBAC 权限矩阵
- `src/lib/authn/enterprise-local-authn/index.ts` - 企业本地认证
- `src/lib/authn/enterprise-local-authn/requests.ts` - 认证请求类型

### 审计事件 (`audit-events/`)
- `src/lib/audit-events/index.ts` - 审计事件定义和摘要生成

### 其他企业功能
- `src/lib/alerts/` - 告警系统
- `src/lib/billing/` - 计费管理
- `src/lib/project-members/` - 项目成员管理
- `src/lib/oauth-apps/` - OAuth 应用管理
- `src/lib/signing-key/` - 数字签名
- `src/lib/connection-keys/` - 连接密钥管理

### 嵌入式 SDK (`ui/embed-sdk/`)
- `src/index.ts` - SDK 入口文件
- `webpack.config.js` - 构建配置

---

*模块文档版本: 2.0.0*
*最后更新: 2025-11-18 15:28:53*