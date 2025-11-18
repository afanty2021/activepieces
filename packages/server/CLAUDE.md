[根目录](../../CLAUDE.md) > [packages](../) > **server**

# Server 模块文档

> **变更记录 (Changelog)**
> - 2025-11-18 15:59:51 - 第四次增量更新：补充Webhook管理、用户管理、计费管理和引擎服务的深度分析
> - 2025-11-18 15:28:53 - 第三次增量更新：补充流程管理和认证服务的详细分析

## 模块职责

ActivePieces Server 是整个平台的后端核心，提供 RESTful API、WebSocket 服务、任务队列、数据库管理等关键功能。它采用模块化架构，支持多版本部署和企业级功能。

## 模块结构

```
packages/server/
├── api/                    # API 服务
│   ├── src/
│   │   ├── app/           # 应用模块
│   │   ├── database/      # 数据库相关
│   │   └── main.ts        # 服务入口
│   └── package.json
├── worker/                 # 工作进程
│   ├── src/
│   │   ├── lib/          # 核心逻辑
│   │   └── index.ts      # 入口文件
│   └── package.json
├── shared/                # 服务端共享库
│   ├── src/
│   │   └── lib/          # 共享工具
│   └── package.json
└── README.md
```

## 核心架构

### 1. API 服务 (`api/`)

#### 入口与启动
- **主入口**: `src/main.ts`
- **应用配置**: `src/app/app.ts`
- **启动命令**: `nx serve server-api`

#### 技术栈
- **Fastify**: 高性能 HTTP 服务器
- **TypeORM**: 数据库 ORM
- **Socket.IO**: WebSocket 服务
- **BullMQ**: 任务队列
- **Redis**: 缓存和消息代理

#### 核心模块

##### 应用模块 (`src/app/`)
```typescript
// 主要模块结构
app/
├── ai/                    # AI 提供商管理
├── app-connection/        # 应用连接管理
├── authentication/        # 认证授权
├── core/                  # 核心服务
├── database/              # 数据库管理
├── ee/                    # 企业版功能
├── file/                  # 文件管理
├── flags/                 # 功能开关
├── flows/                 # 流程管理
├── pieces/                # 集成管理
├── platform/              # 平台管理
├── project/               # 项目管理
├── trigger/               # 触发器管理
├── user/                  # 用户管理
├── webhooks/              # Webhook 管理
└── workers/               # 工作进程管理
```

##### 数据库模块 (`src/app/database/`)
```typescript
// 迁移文件结构
database/migration/
├── common/                # 通用迁移
└── postgres/              # PostgreSQL 特定迁移
```

### 2. 工作进程 (`worker/`)

#### 职责
- 异步任务处理
- 流程执行调度
- 定时任务管理
- 事件处理

#### 核心功能
```typescript
// 主要处理器
- FlowExecutionWorker    // 流程执行
- TriggerPollingWorker   // 触发器轮询
- ScheduledTaskWorker   // 定时任务
- EventProcessingWorker  // 事件处理
```

### 3. 共享库 (`shared/`)

#### 提供内容
- 数据库实体定义
- 服务接口规范
- 工具函数库
- 配置管理

## 🔗 Webhook管理系统深度分析

ActivePieces 提供完整的 Webhook 管理和自动化处理机制，支持同步和异步两种执行模式，确保外部系统能够可靠地触发自动化流程。

### Webhook服务核心架构

#### Webhook服务 (`webhooks/webhook.service.ts`)
```typescript
export const webhookService = {
  async handleWebhook({
    logger,
    data,
    flowId,
    async,
    saveSampleData,
    flowVersionToRun,
    payload,
    execute,
    onRunCreated,
    parentRunId,
    failParentOnFailure,
  }: HandleWebhookParams): Promise<EngineHttpResponse> {
    // 1. 生成请求ID和日志上下文
    const webhookRequestId = apId();
    const pinoLogger = pinoLogging.createWebhookContextLog({
      log: logger,
      webhookId: webhookRequestId,
      flowId,
    });

    // 2. 验证流程存在性和状态
    const flowExecutionResult = await flowExecutionCache(pinoLogger).get({
      flowId,
      simulate: saveSampleData,
    });

    if (!flowExecutionResult.exists) {
      return {
        status: StatusCodes.GONE,
        body: {},
        headers: { [webhookHeader]: webhookRequestId },
      };
    }

    const { flow } = flowExecutionResult;
    if (flow.status === FlowStatus.DISABLED && !saveSampleData) {
      return {
        status: StatusCodes.NOT_FOUND,
        body: {},
        headers: { [webhookHeader]: webhookRequestId },
      };
    }

    // 3. 获取流程版本
    const flowVersionIdToRun = await webhookHandler.getFlowVersionIdToRun(
      flowVersionToRun,
      flow
    );

    // 4. 处理握手请求
    const response = await handshakeHandler(pinoLogger).handleHandshakeRequest({
      payload: (payload ?? await data(flow.projectId)) as TriggerPayload,
      handshakeConfiguration: flowExecutionResult.handshakeConfiguration ?? null,
      flowId: flow.id,
      flowVersionId: flowVersionIdToRun,
      projectId: flow.projectId,
    });

    if (!isNil(response)) {
      return {
        status: response.status,
        body: response.body,
        headers: response.headers ?? {},
      };
    }

    // 5. 根据模式处理执行
    if (async) {
      return await webhookHandler.handleAsync({
        flow,
        saveSampleData,
        platformId: flowExecutionResult.platformId,
        flowVersionIdToRun,
        payload: payload ?? await data(flow.projectId),
        logger: pinoLogger,
        webhookRequestId,
        runEnvironment: flowVersionToRun === WebhookFlowVersionToRun.LOCKED_FALL_BACK_TO_LATEST
          ? RunEnvironment.PRODUCTION
          : RunEnvironment.TESTING,
        webhookHeader,
        execute: flow.status === FlowStatus.ENABLED && execute,
        parentRunId,
        failParentOnFailure,
      });
    } else {
      // 同步模式处理
      const flowHttpResponse = await webhookHandler.handleSync({
        payload: payload ?? await data(flow.projectId),
        projectId: flow.projectId,
        flow,
        platformId: flowExecutionResult.platformId,
        runEnvironment: flowVersionToRun === WebhookFlowVersionToRun.LOCKED_FALL_BACK_TO_LATEST
          ? RunEnvironment.PRODUCTION
          : RunEnvironment.TESTING,
        logger: pinoLogger,
        webhookRequestId,
        synchronousHandlerId: engineResponseWatcher(pinoLogger).getServerId(),
        flowVersionIdToRun,
        saveSampleData,
        flowVersionToRun,
        onRunCreated,
        parentRunId,
        failParentOnFailure,
      });

      return {
        status: flowHttpResponse.status,
        body: flowHttpResponse.body,
        headers: {
          ...flowHttpResponse.headers,
          [webhookHeader]: webhookRequestId,
        },
      };
    }
  },
};
```

#### Webhook处理器 (`webhooks/webhook-handler.ts`)
```typescript
export const webhookHandler = {
  // 获取要运行的流程版本
  async getFlowVersionIdToRun(
    type: WebhookFlowVersionToRun,
    flow: Flow
  ): Promise<FlowVersionId> {
    if (type === WebhookFlowVersionToRun.LOCKED_FALL_BACK_TO_LATEST &&
        !isNil(flow.publishedVersionId)) {
      return flow.publishedVersionId;
    }

    const flowVersionSchema = await flowVersionRepo().createQueryBuilder()
      .select('id')
      .where({ flowId: flow.id })
      .orderBy('created', 'DESC')
      .getRawOne();

    assertNotNullOrUndefined(flowVersionSchema, 'Flow version not found');
    return flowVersionSchema.id;
  },

  // 异步处理模式
  async handleAsync(params: AsyncWebhookParams): Promise<EngineHttpResponse> {
    const { flow, logger, webhookRequestId, payload, flowVersionIdToRun,
            webhookHeader, saveSampleData, execute, runEnvironment,
            parentRunId, failParentOnFailure, platformId } = params;

    // 注入追踪上下文
    const traceContext: Record<string, string> = {};
    propagation.inject(context.active(), traceContext);

    // 添加到任务队列
    await jobQueue(logger).add({
      id: webhookRequestId,
      type: JobType.ONE_TIME,
      data: {
        platformId,
        projectId: flow.projectId,
        schemaVersion: LATEST_JOB_DATA_SCHEMA_VERSION,
        requestId: webhookRequestId,
        payload,
        jobType: WorkerJobType.EXECUTE_WEBHOOK,
        flowId: flow.id,
        saveSampleData,
        flowVersionIdToRun,
        runEnvironment,
        execute,
        parentRunId,
        failParentOnFailure,
        traceContext,
      },
    });

    return {
      status: StatusCodes.OK,
      body: {},
      headers: { [webhookHeader]: webhookRequestId },
    };
  },

  // 同步处理模式
  async handleSync(params: SyncWebhookParams): Promise<EngineHttpResponse> {
    const { payload, projectId, flow, logger, webhookRequestId,
            synchronousHandlerId, flowVersionIdToRun, runEnvironment,
            saveSampleData, flowVersionToRun, parentRunId, failParentOnFailure,
            platformId } = params;

    // 保存示例数据（如果需要）
    if (saveSampleData) {
      rejectedPromiseHandler(savePayload({
        flow,
        logger,
        webhookRequestId,
        payload,
        platformId,
        flowVersionIdToRun,
        runEnvironment,
        parentRunId,
        failParentOnFailure,
      }), logger);
    }

    // 检查流程状态
    const disabledFlow = flow.status !== FlowStatus.ENABLED &&
                        flowVersionToRun === WebhookFlowVersionToRun.LOCKED_FALL_BACK_TO_LATEST;

    if (disabledFlow) {
      return {
        status: StatusCodes.NOT_FOUND,
        body: {},
        headers: {},
      };
    }

    // 创建流程运行
    const createdRun = await flowRunService(logger).start({
      platformId,
      environment: runEnvironment,
      flowId: flow.id,
      flowVersionId: flowVersionIdToRun,
      payload,
      synchronousHandlerId,
      projectId,
      executeTrigger: true,
      httpRequestId: webhookRequestId,
      executionType: ExecutionType.BEGIN,
      progressUpdateType: ProgressUpdateType.WEBHOOK_RESPONSE,
      parentRunId,
      failParentOnFailure,
    });

    params.onRunCreated?.(createdRun);

    // 等待执行结果
    return await engineResponseWatcher(logger).oneTimeListener<EngineHttpResponse>(
      webhookRequestId,
      true,
      WEBHOOK_TIMEOUT_MS,
      {
        status: StatusCodes.NO_CONTENT,
        body: {},
        headers: {},
      }
    );
  },
};
```

### Webhook握手机制

#### 握手处理器 (`webhooks/handshake-handler.ts`)
```typescript
export const handshakeHandler = (log: FastifyBaseLogger) => ({
  async handleHandshakeRequest(
    params: HandleHandshakeRequestParams
  ): Promise<WebhookHandshakeResponse | null> {
    const { payload, handshakeConfiguration } = params;

    // 检查是否为握手请求
    if (!isHandshakeRequest({ payload, handshakeConfiguration })) {
      return null;
    }

    // 获取流程版本
    const flowVersion = await flowVersionService(log).getFlowVersionOrThrow({
      flowId: params.flowId,
      versionId: params.flowVersionId,
    });

    const platformId = await projectService.getPlatformId(params.projectId);

    // 执行握手触发器
    const engineHelperResponse = await userInteractionWatcher(log)
      .submitAndWaitForResponse<EngineHelperResponse<EngineHelperTriggerResult<TriggerHookType.HANDSHAKE>>>({
        jobType: WorkerJobType.EXECUTE_TRIGGER_HOOK,
        hookType: TriggerHookType.HANDSHAKE,
        flowVersion,
        projectId: params.projectId,
        test: false,
        platformId,
        triggerPayload: payload,
      });

    if (engineHelperResponse.status !== EngineResponseStatus.OK) {
      return null;
    }

    return engineHelperResponse.result?.response ?? null;
  },

  async getWebhookHandshakeConfiguration(
    triggerSource: TriggerSource | null
  ): Promise<WebhookHandshakeConfiguration | null> {
    if (isNil(triggerSource) ||
        isNil(triggerSource.pieceName) ||
        isNil(triggerSource.pieceVersion) ||
        isNil(triggerSource.triggerName) ||
        isNil(triggerSource.projectId)) {
      return null;
    }

    const pieceTrigger = await triggerUtils(log).getPieceTriggerByName({
      pieceName: triggerSource.pieceName,
      pieceVersion: triggerSource.pieceVersion,
      triggerName: triggerSource.triggerName,
      projectId: triggerSource.projectId,
    });

    return pieceTrigger?.handshakeConfiguration ?? null;
  },
});
```

#### 握手策略检测
```typescript
function isHandshakeRequest(params: IsHandshakeRequestParams): boolean {
  const { payload, handshakeConfiguration } = params;

  if (isNil(handshakeConfiguration) ||
      isNil(handshakeConfiguration.strategy) ||
      isNil(handshakeConfiguration.paramName)) {
    return false;
  }

  const { strategy, paramName } = handshakeConfiguration;

  switch (strategy) {
    case WebhookHandshakeStrategy.HEADER_PRESENT:
      return paramName.toLowerCase() in payload.headers;

    case WebhookHandshakeStrategy.QUERY_PRESENT:
      return paramName in payload.queryParams;

    case WebhookHandshakeStrategy.BODY_PARAM_PRESENT:
      return typeof payload.body === 'object' &&
             payload.body !== null &&
             paramName in payload.body;

    default:
      return false;
  }
}
```

### Webhook配置与常量

#### 流程版本运行策略
```typescript
export enum WebhookFlowVersionToRun {
  LOCKED_FALL_BACK_TO_LATEST = 'locked_fall_back_to_latest',
  LATEST = 'latest',
}
```

#### 握手策略
```typescript
enum WebhookHandshakeStrategy {
  HEADER_PRESENT = 'header_present',
  QUERY_PRESENT = 'query_present',
  BODY_PARAM_PRESENT = 'body_param_present',
}
```

#### 握手配置
```typescript
interface WebhookHandshakeConfiguration {
  strategy: WebhookHandshakeStrategy;
  paramName: string;
}

type WebhookHandshakeResponse = {
  status: number;
  body?: unknown;
  headers?: Record<string, string>;
};
```

## 👥 用户管理系统深度分析

ActivePieces 提供完整的用户生命周期管理，支持企业级的用户权限控制和多租户架构。

### 用户实体与数据模型

#### 用户实体 (`user/user-entity.ts`)
```typescript
export type UserSchema = User & {
  projects: Project[]
  identity: UserIdentity
}

export const UserEntity = new EntitySchema<UserSchema>({
  name: 'user',
  columns: {
    ...BaseColumnSchemaPart,
    status: {
      type: String,
    },
    platformRole: {
      type: String,
      nullable: false,
    },
    identityId: {
      type: String,
      nullable: false,
    },
    externalId: {
      type: String,
      nullable: true,
    },
    platformId: {
      type: String,
      nullable: true,
    },
  },
  indices: [
    {
      name: 'idx_user_platform_id_email',
      columns: ['platformId', 'identityId'],
      unique: true,
    },
    {
      name: 'idx_user_platform_id_external_id',
      columns: ['platformId', 'externalId'],
      unique: true,
    },
  ],
  relations: {
    projects: {
      type: 'one-to-many',
      target: 'project',
      inverseSide: 'owner',
    },
    identity: {
      type: 'many-to-one',
      target: 'user_identity',
      joinColumn: {
        name: 'identityId',
        referencedColumnName: 'id',
      },
    },
  },
});
```

#### 用户服务 (`user/user-service.ts`)
```typescript
export const userService = {
  // 创建用户
  async create(params: CreateParams): Promise<User> {
    const user: NewUser = {
      id: apId(),
      identityId: params.identityId,
      platformRole: params.platformRole,
      status: UserStatus.ACTIVE,
      externalId: params.externalId,
      platformId: params.platformId,
    }
    return userRepo().save(user);
  },

  // 更新用户
  async update({
    id,
    status,
    platformId,
    platformRole,
    externalId
  }: UpdateParams): Promise<UserWithMetaInformation> {
    const user = await this.getOrThrow({ id });
    const platform = await platformService.getOneOrThrow(user.platformId!);

    // 防止管理员停用自己
    if (platform.ownerId === user.id && status === UserStatus.INACTIVE) {
      throw new ActivepiecesError({
        code: ErrorCode.VALIDATION,
        params: {
          message: 'Admin cannot be deactivated',
        },
      });
    }

    const updateResult = await userRepo().update({
      id,
      platformId,
    }, {
      ...spreadIfDefined('status', status),
      ...spreadIfDefined('platformRole', platformRole),
      ...spreadIfDefined('externalId', externalId),
    });

    if (updateResult.affected !== 1) {
      throw new ActivepiecesError({
        code: ErrorCode.ENTITY_NOT_FOUND,
        params: {
          entityType: 'user',
          entityId: id,
        },
      });
    }

    return this.getMetaInformation({ id });
  },

  // 用户列表查询（支持分页）
  async list({
    platformId,
    externalId,
    cursorRequest,
    limit
  }: ListParams): Promise<SeekPage<UserWithMetaInformation>> {
    const decodedCursor = paginationHelper.decodeCursor(cursorRequest);
    const paginator = buildPaginator({
      entity: UserEntity,
      query: {
        limit,
        afterCursor: decodedCursor.nextCursor,
        beforeCursor: decodedCursor.previousCursor,
      },
    });

    const { data, cursor } = await paginator.paginate(
      userRepo().createQueryBuilder('user').where({
        platformId,
        ...spreadIfDefined('externalId', externalId),
      })
    );

    const usersWithMetaInformation = await Promise.all(
      data.map(this.getMetaInformation)
    );

    return paginationHelper.createPage<UserWithMetaInformation>(
      usersWithMetaInformation,
      cursor
    );
  },

  // 按身份ID查询
  async getOneByIdentityIdOnly({ identityId }: GetOneByIdentityIdOnlyParams): Promise<User | null> {
    return userRepo().findOneBy({ identityId });
  },

  // 按平台和身份查询
  async getOneByIdentityAndPlatform({
    identityId,
    platformId
  }: GetOneByIdentityIdParams): Promise<User | null> {
    return userRepo().findOneBy({ identityId, platformId });
  },

  // 按外部ID查询
  async getByPlatformAndExternalId({
    platformId,
    externalId,
  }: GetByPlatformAndExternalIdParams): Promise<User | null> {
    return userRepo().findOneBy({
      platformId,
      externalId,
    });
  },

  // 获取用户元信息
  async getMetaInformation({ id }: IdParams): Promise<UserWithMetaInformation> {
    const user = await userRepo().findOneByOrFail({ id });
    const identity = await userIdentityService(system.globalLogger())
      .getBasicInformation(user.identityId);

    return {
      id: user.id,
      email: identity.email,
      firstName: identity.firstName,
      lastName: identity.lastName,
      platformId: user.platformId,
      platformRole: user.platformRole,
      status: user.status,
      externalId: user.externalId,
      created: user.created,
      updated: user.updated,
    };
  },

  // 项目用户查询
  async listProjectUsers({
    platformId,
    projectId
  }: ListUsersForProjectParams): Promise<UserWithMetaInformation[]> {
    const users = await getUsersForProject(platformId, projectId);
    const usersWithMetaInformation = await userRepo()
      .find({
        where: { platformId, id: In(users) },
        relations: { identity: true }
      })
      .then((users) => users.map(this.getMetaInformation));

    return Promise.all(usersWithMetaInformation);
  },

  // 添加平台所有者
  async addOwnerToPlatform({
    id,
    platformId,
  }: UpdatePlatformIdParams): Promise<void> {
    await userRepo().update(id, {
      updated: dayjs().toISOString(),
      platformRole: PlatformRole.ADMIN,
      platformId,
    });
  },

  // 删除用户
  async delete({ id, platformId }: DeleteParams): Promise<void> {
    await userRepo().delete({
      id,
      platformId,
    });
  },
};
```

### 平台级用户控制器

#### 平台用户控制器 (`user/platform/platform-user-controller.ts`)
```typescript
export const platformUserController: FastifyPluginAsyncTypebox = async (app) => {
  // 创建平台用户
  app.post('/', CreateUserRequest, async (request, reply) => {
    const user = await userService(request.log).create({
      identityId: request.body.identityId,
      platformId: request.principal.platformId,
      platformRole: request.body.platformRole,
      externalId: request.body.externalId,
    });

    return user;
  });

  // 获取平台用户列表
  app.get('/', ListPlatformUsersRequest, async (request) => {
    return await userService(request.log).list({
      platformId: request.principal.platformId,
      externalId: request.query.externalId,
      cursorRequest: request.query.cursor,
      limit: request.query.limit,
    });
  });

  // 更新平台用户
  app.post('/:id', UpdatePlatformUserRequest, async (request) => {
    return await userService(request.log).update({
      id: request.params.id,
      platformId: request.principal.platformId,
      platformRole: request.body.platformRole,
      status: request.body.status,
      externalId: request.body.externalId,
    });
  });

  // 获取平台用户详情
  app.get('/:id', GetPlatformUserRequest, async (request) => {
    return await userService(request.log).getMetaInformation({
      id: request.params.id,
    });
  });

  // 删除平台用户
  app.delete('/:id', DeletePlatformUserRequest, async (request, reply) => {
    await userService(request.log).delete({
      id: request.params.id,
      platformId: request.principal.platformId,
    });

    reply.status(StatusCodes.NO_CONTENT).send();
  });
};
```

### 项目用户获取逻辑

#### 获取项目用户
```typescript
async function getUsersForProject(platformId: PlatformId, projectId: string) {
  // 获取平台管理员
  const platformAdmins = await userRepo()
    .find({
      where: { platformId, platformRole: PlatformRole.ADMIN }
    })
    .then((users) => users.map((user) => user.id));

  const edition = system.getEdition();

  // 社区版只返回平台管理员
  if (edition === ApEdition.COMMUNITY) {
    return platformAdmins;
  }

  // 企业版还包括项目成员
  const projectMembers = await projectMemberRepo()
    .find({ where: { projectId, platformId } })
    .then((members) => members.map((member) => member.userId));

  return [...platformAdmins, ...projectMembers];
}
```

### 用户类型定义

#### 用户状态枚举
```typescript
enum UserStatus {
  ACTIVE = 'ACTIVE',
  INACTIVE = 'INACTIVE',
  PENDING_VERIFICATION = 'PENDING_VERIFICATION',
}
```

#### 平台角色枚举
```typescript
enum PlatformRole {
  ADMIN = 'ADMIN',
  MEMBER = 'MEMBER',
}
```

#### 用户元信息类型
```typescript
interface UserWithMetaInformation {
  id: string;
  email: string;
  firstName: string;
  lastName: string;
  platformId: string;
  platformRole: PlatformRole;
  status: UserStatus;
  externalId?: string;
  created: string;
  updated: string;
}
```

## 💰 计费管理系统深度分析

ActivePieces 提供完整的商业化计费管理功能，支持多种订阅模式、资源配额管理和使用量计费。

### 订阅计划体系

#### 计划层次结构
```typescript
enum PlanName {
  FREE = 'free',
  PLUS = 'plus',
  BUSINESS = 'business',
  ENTERPRISE = 'enterprise',
  APPSUMO_ACTIVEPIECES_TIER1 = 'appsumo_activepieces_tier1',
  APPSUMO_ACTIVEPIECES_TIER2 = 'appsumo_activepieces_tier2',
  APPSUMO_ACTIVEPIECES_TIER3 = 'appsumo_activepieces_tier3',
  APPSUMO_ACTIVEPIECES_TIER4 = 'appsumo_activepieces_tier4',
  APPSUMO_ACTIVEPIECES_TIER5 = 'appsumo_activepieces_tier5',
  APPSUMO_ACTIVEPIECES_TIER6 = 'appsumo_activepieces_tier6',
}

const PLAN_HIERARCHY = {
  [PlanName.FREE]: 0,
  [PlanName.PLUS]: 1,
  [PlanName.BUSINESS]: 2,
  [PlanName.ENTERPRISE]: 3,
  [PlanName.APPSUMO_ACTIVEPIECES_TIER1]: 0,
  [PlanName.APPSUMO_ACTIVEPIECES_TIER2]: 0,
  [PlanName.APPSUMO_ACTIVEPIECES_TIER3]: 1,
  [PlanName.APPSUMO_ACTIVEPIECES_TIER4]: 2,
  [PlanName.APPSUMO_ACTIVEPIECES_TIER5]: 3,
  [PlanName.APPSUMO_ACTIVEPIECES_TIER6]: 4,
} as const;
```

### 免费计划配置

```typescript
export const FREE_CLOUD_PLAN: PlatformPlanWithOnlyLimits = {
  plan: 'free',
  includedAiCredits: 200,
  aiCreditsOverageLimit: undefined,
  aiCreditsOverageState: AiOverageState.NOT_ALLOWED,
  activeFlowsLimit: 10,
  userSeatsLimit: 1,
  projectsLimit: 1,
  tablesLimit: 1,
  mcpLimit: 1,

  // 基础功能
  agentsEnabled: true,
  tablesEnabled: true,
  todosEnabled: true,
  mcpsEnabled: true,

  // 高级功能禁用
  embeddingEnabled: false,
  globalConnectionsEnabled: false,
  customRolesEnabled: false,
  environmentsEnabled: false,
  analyticsEnabled: false,
  showPoweredBy: false,
  auditLogEnabled: false,
  managePiecesEnabled: false,
  manageTemplatesEnabled: false,
  customAppearanceEnabled: false,
  manageProjectsEnabled: false,
  projectRolesEnabled: false,
  customDomainsEnabled: false,
  apiKeysEnabled: false,
  ssoEnabled: false,
};
```

### Plus计划配置

```typescript
export const PLUS_CLOUD_PLAN: PlatformPlanWithOnlyLimits = {
  plan: 'plus',
  includedAiCredits: 500,
  aiCreditsOverageLimit: undefined,
  aiCreditsOverageState: AiOverageState.ALLOWED_BUT_OFF,
  activeFlowsLimit: 10,
  userSeatsLimit: 1,
  projectsLimit: 1,
  mcpLimit: undefined,
  tablesLimit: undefined,

  // 基础功能
  agentsEnabled: true,
  tablesEnabled: true,
  todosEnabled: true,
  mcpsEnabled: true,

  // 高级功能仍然禁用
  embeddingEnabled: false,
  globalConnectionsEnabled: false,
  customRolesEnabled: false,
  environmentsEnabled: false,
  analyticsEnabled: false,
  managePiecesEnabled: false,
  manageTemplatesEnabled: false,
  customAppearanceEnabled: false,
  manageProjectsEnabled: false,
  projectRolesEnabled: false,
  customDomainsEnabled: false,
  apiKeysEnabled: false,
  ssoEnabled: false,
  showPoweredBy: false,
  auditLogEnabled: false,
};
```

### 商业计划配置

```typescript
export const BUSINESS_CLOUD_PLAN: PlatformPlanWithOnlyLimits = {
  plan: 'business',
  includedAiCredits: 1000,
  aiCreditsOverageLimit: undefined,
  aiCreditsOverageState: AiOverageState.ALLOWED_BUT_OFF,
  activeFlowsLimit: 50,
  userSeatsLimit: 5,
  projectsLimit: 10,
  mcpLimit: undefined,
  tablesLimit: undefined,

  // 基础功能
  agentsEnabled: true,
  tablesEnabled: true,
  todosEnabled: true,
  mcpsEnabled: true,

  // 企业级功能启用
  embeddingEnabled: false,
  globalConnectionsEnabled: false,
  customRolesEnabled: false,
  environmentsEnabled: false,
  analyticsEnabled: true,                // 启用分析
  managePiecesEnabled: false,
  manageTemplatesEnabled: false,
  customAppearanceEnabled: false,
  manageProjectsEnabled: true,           // 启用项目管理
  projectRolesEnabled: true,             // 启用项目角色
  customDomainsEnabled: false,
  apiKeysEnabled: true,                  // 启用API密钥
  ssoEnabled: true,                      // 启用SSO
  showPoweredBy: false,
  auditLogEnabled: false,
};
```

### AppSumo计划配置

```typescript
export const APPSUMO_PLAN = ({
  planName,
  userSeatsLimit,
  tablesLimit,
  mcpLimit
}: {
  planName: string;
  userSeatsLimit: number;
  tablesLimit: number;
  mcpLimit: number;
}): PlatformPlanWithOnlyLimits => {
  return {
    plan: planName,
    userSeatsLimit,
    includedAiCredits: 200,
    aiCreditsOverageState: AiOverageState.ALLOWED_BUT_OFF,
    aiCreditsOverageLimit: undefined,
    activeFlowsLimit: undefined,
    projectsLimit: 1,
    mcpLimit,
    tablesLimit,

    // 基础功能
    agentsEnabled: true,
    tablesEnabled: true,
    todosEnabled: true,
    mcpsEnabled: true,

    // AppSumo特色功能
    embeddingEnabled: false,
    globalConnectionsEnabled: false,
    customRolesEnabled: false,
    environmentsEnabled: false,
    analyticsEnabled: false,
    showPoweredBy: false,
    auditLogEnabled: false,
    managePiecesEnabled: false,
    manageTemplatesEnabled: false,
    customAppearanceEnabled: false,
    manageProjectsEnabled: false,
    projectRolesEnabled: true,             // AppSumo特色：项目角色
    customDomainsEnabled: false,
    apiKeysEnabled: false,
    ssoEnabled: false,
  };
};
```

### 计费周期管理

#### 计费周期枚举
```typescript
export enum BillingCycle {
  MONTHLY = 'monthly',
  ANNUAL = 'annual',
}

const BILLING_CYCLE_HIERARCHY = {
  [BillingCycle.MONTHLY]: 0,
  [BillingCycle.ANNUAL]: 1,
} as const;
```

#### 定价策略
```typescript
// 附加资源定价映射
export const PRICE_PER_EXTRA_USER_MAP = {
  [BillingCycle.ANNUAL]: 11.4,    // 年付：$11.4/用户/月
  [BillingCycle.MONTHLY]: 15,     // 月付：$15/用户/月
};

export const PRICE_PER_EXTRA_PROJECT_MAP = {
  [BillingCycle.ANNUAL]: 7.6,     // 年付：$7.6/项目/月
  [BillingCycle.MONTHLY]: 10,     // 月付：$10/项目/月
};

export const PRICE_PER_EXTRA_5_ACTIVE_FLOWS_MAP = {
  [BillingCycle.ANNUAL]: 11.4,    // 年付：$11.4/5个活跃流程/月
  [BillingCycle.MONTHLY]: 15,     // 月付：$15/5个活跃流程/月
};
```

### 资源配额管理

#### 配额指标映射
```typescript
export const METRIC_TO_LIMIT_MAPPING = {
  [PlatformUsageMetric.ACTIVE_FLOWS]: 'activeFlowsLimit',
  [PlatformUsageMetric.USER_SEATS]: 'userSeatsLimit',
  [PlatformUsageMetric.PROJECTS]: 'projectsLimit',
  [PlatformUsageMetric.TABLES]: 'tablesLimit',
  [PlatformUsageMetric.MCPS]: 'mcpLimit',
} as const;

export const METRIC_TO_USAGE_MAPPING = {
  [PlatformUsageMetric.ACTIVE_FLOWS]: 'activeFlows',
  [PlatformUsageMetric.USER_SEATS]: 'seats',
  [PlatformUsageMetric.PROJECTS]: 'projects',
  [PlatformUsageMetric.TABLES]: 'tables',
  [PlatformUsageMetric.MCPS]: 'mcps',
} as const;
```

#### 资源限制消息映射
```typescript
export const RESOURCE_TO_MESSAGE_MAPPING = {
  [PlatformUsageMetric.PROJECTS]: 'Project limit reached. Delete old projects or upgrade to create new ones.',
  [PlatformUsageMetric.TABLES]: 'Table limit reached. Please delete tables or upgrade to restore access.',
  [PlatformUsageMetric.MCPS]: 'MCP server limit reached. Delete unused MCPs or upgrade your plan to continue.',
};
```

### AI积分管理

#### AI积分系统配置
```typescript
export enum AiOverageState {
  NOT_ALLOWED = 'not_allowed',      // 不允许超额
  ALLOWED_BUT_OFF = 'allowed_but_off', // 允许但未开启
  ENABLED = 'enabled',              // 已开启超额计费
}

export const AI_CREDITS_USAGE_THRESHOLD = 15000;  // 使用量阈值
```

#### AI积分配置类型
```typescript
export type ProjectPlanLimits = {
  nickname?: string;
  locked?: boolean;
  pieces?: string[];
  aiCredits?: number | null;
  piecesFilterType?: PiecesFilterType;
}
```

### Stripe集成配置

#### 价格名称枚举
```typescript
export enum PRICE_NAMES {
  PLUS_PLAN = 'plus-plan',
  BUSINESS_PLAN = 'business-plan',
  AI_CREDITS = 'ai-credit',
  ACTIVE_FLOWS = 'active-flow',
  USER_SEAT = 'user-seat',
  PROJECT = 'project',
}
```

#### 价格ID映射配置
```typescript
export const PRICE_ID_MAP = {
  [PRICE_NAMES.PLUS_PLAN]: {
    [BillingCycle.MONTHLY]: {
      dev: 'price_1RTRd4QN93Aoq4f8E22qF5JU',
      prod: 'price_1RflgUKZ0dZRqLEK5COq9Kn8',
    },
    [BillingCycle.ANNUAL]: {
      dev: 'price_1RtZrSQN93Aoq4f8KLZq4yif',
      prod: 'price_1RtZwlKZ0dZRqLEKBiPradv4',
    },
  },
  [PRICE_NAMES.BUSINESS_PLAN]: {
    [BillingCycle.MONTHLY]: {
      dev: 'price_1RTReBQN93Aoq4f8v9CnMTFT',
      prod: 'price_1RflgbKZ0dZRqLEKaW4Nlt0P',
    },
    [BillingCycle.ANNUAL]: {
      dev: 'price_1RtZpuQN93Aoq4f8mNgEjs0b',
      prod: 'price_1RtZxNKZ0dZRqLEKqTYawR8q',
    },
  },
  [PRICE_NAMES.AI_CREDITS]: {
    [BillingCycle.MONTHLY]: {
      dev: 'price_1RnbNPQN93Aoq4f8GLiZbJFj',
      prod: 'price_1Rnj5bKZ0dZRqLEKQx2gwL7s',
    },
    [BillingCycle.ANNUAL]: {
      dev: 'price_1RtPc0QN93Aoq4f8JAPe5HbG',
      prod: 'price_1RtZziKZ0dZRqLEKiWU2iAz8',
    },
  },
  // ... 更多价格配置
};
```

### 订阅管理请求类型

#### 创建订阅请求
```typescript
export const CreateSubscriptionParamsSchema = Type.Object({
  plan: Type.Union([Type.Literal(PlanName.PLUS), Type.Literal(PlanName.BUSINESS)]),
  cycle: Type.Enum(BillingCycle),
  addons: Type.Object({
    userSeats: Type.Optional(Type.Number()),
    activeFlows: Type.Optional(Type.Number()),
    projects: Type.Optional(Type.Number()),
  }),
});

export type CreateSubscriptionParams = Static<typeof CreateSubscriptionParamsSchema>;
```

#### 更新订阅请求
```typescript
export const UpdateSubscriptionParamsSchema = Type.Object({
  plan: Type.Union([
    Type.Literal(PlanName.FREE),
    Type.Literal(PlanName.PLUS),
    Type.Literal(PlanName.BUSINESS)
  ]),
  addons: Type.Object({
    userSeats: Type.Optional(Type.Number()),
    activeFlows: Type.Optional(Type.Number()),
    projects: Type.Optional(Type.Number()),
  }),
  cycle: Type.Enum(BillingCycle),
});

export type UpdateSubscriptionParams = Static<typeof UpdateSubscriptionParamsSchema>;
```

#### AI超额配置请求
```typescript
export const SetAiCreditsOverageLimitParamsSchema = Type.Object({
  limit: Type.Number({ minimum: 10 }),
});

export type SetAiCreditsOverageLimitParams = Static<typeof SetAiCreditsOverageLimitParamsSchema>;

export const ToggleAiCreditsOverageEnabledParamsSchema = Type.Object({
  state: Type.Enum(AiOverageState),
});

export type ToggleAiCreditsOverageEnabledParams = Static<typeof ToggleAiCreditsOverageEnabledParamsSchema>;
```

## ⚙️ 触发器管理系统深度分析

ActivePieces 提供强大的触发器管理系统，支持多种触发类型、调度管理和事件处理机制。

### 触发器模块架构

#### 触发器模块注册 (`trigger/trigger.module.ts`)
```typescript
export const triggerModule: FastifyPluginAsyncTypebox = async (app) => {
  await app.register(testTriggerController, { prefix: '/v1/test-trigger' });
  await app.register(triggerEventController, { prefix: '/v1/trigger-events' });
  await app.register(triggerRunController, { prefix: '/v1/trigger-runs' });
};
```

### 触发器源管理

#### 触发器源服务 (`trigger/trigger-source/trigger-source-service.ts`)
```typescript
export const triggerSourceService = (log: FastifyBaseLogger) => {
  return {
    // 启用触发器
    async enable(params: EnableTriggerParams): Promise<TriggerSource> {
      const { flowVersion, projectId, simulate } = params;

      // 1. 获取Piece触发器定义
      const pieceTrigger = await triggerUtils(log)
        .getPieceTriggerOrThrow({ flowVersion, projectId });

      // 2. 清理现有触发器
      await triggerSourceRepo().softDelete({
        flowId: flowVersion.flowId,
        projectId,
        simulate,
      });

      // 3. 创建新触发器源
      const triggerSourceWithoutSchedule: Omit<TriggerSource, 'created' | 'updated' | 'schedule'> = {
        id: apId(),
        type: pieceTrigger.type,
        projectId,
        flowId: flowVersion.flowId,
        triggerName: pieceTrigger.name,
        flowVersionId: flowVersion.id,
        pieceName: flowVersion.trigger.settings.pieceName,
        pieceVersion: flowVersion.trigger.settings.pieceVersion,
        simulate,
      };

      const triggerSource = await triggerSourceRepo().save(triggerSourceWithoutSchedule);

      // 4. 设置触发器副作用（调度、轮询等）
      const { scheduleOptions } = await flowTriggerSideEffect(log).enable({
        flowVersion,
        projectId,
        pieceName: flowVersion.trigger.settings.pieceName,
        pieceTrigger,
        simulate,
      });

      // 5. 保存完整配置
      return triggerSourceRepo().save({
        ...triggerSource,
        schedule: scheduleOptions,
      });
    },

    // 获取触发器
    async get(params: GetTriggerParams): Promise<TriggerSource | null> {
      const { projectId, id } = params;
      return triggerSourceRepo().findOne({
        where: { id, projectId },
      });
    },

    // 按流程ID获取触发器
    async getByFlowId(params: GetFlowIdParamsWithProjectId): Promise<TriggerSource | null> {
      const { flowId, simulate, projectId } = params;
      return triggerSourceRepo().findOne({
        where: { flowId, simulate, ...(projectId ? { projectId } : {}) },
      });
    },

    // 按流程ID获取带关联数据的触发器
    async getByFlowIdPopulated(params: GetByFlowIdParams): Promise<PopulatedTriggerSource | null> {
      const { flowId, simulate } = params;
      return triggerSourceRepo().findOne({
        where: { flowId, simulate },
        relations: { flow: true },
      });
    },

    // 获取或抛出异常
    async getOrThrow({ projectId, id }: GetTriggerParams): Promise<TriggerSource> {
      const triggerSource = await triggerSourceRepo().findOne({
        where: { id, projectId },
      });

      if (isNil(triggerSource)) {
        throw new ActivepiecesError({
          code: ErrorCode.ENTITY_NOT_FOUND,
          params: { entityType: 'trigger', entityId: id },
        });
      }

      return triggerSource;
    },

    // 检查触发器是否存在
    async existsByFlowId(params: ExistsByFlowIdParams): Promise<boolean> {
      const { flowId, simulate } = params;
      return triggerSourceRepo().existsBy({ flowId, simulate });
    },

    // 禁用触发器
    async disable(params: DisableTriggerParams): Promise<void> {
      const { projectId, flowId, simulate } = params;

      const triggerSource = await triggerSourceRepo().findOneBy({
        flowId,
        projectId,
        simulate,
      });

      if (isNil(triggerSource)) {
        return;
      }

      const flowVersion = await flowVersionService(log)
        .getOneOrThrow(triggerSource.flowVersionId);

      const pieceTrigger = await triggerUtils(log)
        .getPieceTrigger({ flowVersion, projectId });

      if (!isNil(pieceTrigger)) {
        await flowTriggerSideEffect(log).disable({
          flowVersion,
          projectId,
          pieceName: triggerSource.pieceName,
          pieceTrigger,
          simulate,
          ignoreError: params.ignoreError,
        });
      }

      await triggerSourceRepo().softDelete({
        id: triggerSource.id,
        projectId,
      });
    },
  };
};
```

### 触发器工具函数

#### 触发器工具 (`trigger/trigger-source/trigger-utils.ts`)
```typescript
// 获取Piece触发器定义
async getPieceTriggerOrThrow(params: GetPieceTriggerParams): Promise<PieceTrigger> {
  const pieceTrigger = await this.getPieceTrigger(params);

  if (isNil(pieceTrigger)) {
    throw new ActivepiecesError({
      code: ErrorCode.PIECE_TRIGGER_NOT_FOUND,
      params: {
        pieceName: params.flowVersion.trigger.settings.pieceName,
        pieceVersion: params.flowVersion.trigger.settings.pieceVersion,
        triggerName: params.flowVersion.trigger.settings.triggerName,
      },
    });
  }

  return pieceTrigger;
}

// 按名称获取Piece触发器
async getPieceTriggerByName(params: GetPieceTriggerByNameParams): Promise<PieceTrigger | null> {
  const piece = await pieceUtils.loadPiece(
    params.projectId,
    params.pieceName,
    params.pieceVersion,
  );

  const trigger = piece.triggers(params.triggerName);

  if (isNil(trigger)) {
    return null;
  }

  return {
    ...trigger,
    name: params.triggerName,
  };
}
```

## 📁 文件管理系统深度分析

ActivePieces 提供完整的文件管理功能，支持多种存储后端、文件压缩和生命周期管理。

### 文件服务核心架构

#### 文件服务 (`file/file.service.ts`)
```typescript
export const fileService = (log: FastifyBaseLogger) => ({
  // 保存文件
  async save(params: SaveParams): Promise<File> {
    const baseFile: BaseFile = {
      id: params.fileId ?? apId(),
      projectId: params.projectId,
      platformId: params.platformId,
      type: params.type,
      fileName: params.fileName,
      compression: params.compression,
      size: params.size,
      metadata: params.metadata,
      created: dayjs().toISOString(),
      updated: dayjs().toISOString(),
    };

    const location = getLocationForFile(params.type);

    switch (location) {
      case FileLocation.DB:
        return saveFileToDb(baseFile, params.data);

      case FileLocation.S3:
        try {
          // 构建S3键
          const s3Key = await s3Helper(log).constructS3Key(
            params.platformId,
            params.projectId,
            params.type,
            baseFile.id
          );

          // 上传文件到S3
          if (!isNil(params.data)) {
            await s3Helper(log).uploadFile(s3Key, params.data);
          }

          // 保存文件记录
          const savedFile = await fileRepo().save({
            ...baseFile,
            location: FileLocation.S3,
            s3Key,
          });

          return savedFile;
        } catch (error) {
          // S3失败时回退到数据库存储
          exceptionHandler.handle(error, log);
          return saveFileToDb(baseFile, params.data);
        }
    }
  },

  // 检查文件是否存在
  async exists(params: GetOneParams): Promise<boolean> {
    const file = await fileRepo().findOneBy({
      projectId: params.projectId,
      id: params.fileId,
      type: params.type,
    });
    return !isNil(file);
  },

  // 获取文件
  async getFile({ projectId, fileId, type }: GetOneParams): Promise<File | null> {
    const file = await fileRepo().findOneBy({ projectId, fileId, type });
    return file;
  },

  // 获取文件或抛出异常
  async getFileOrThrow(params: GetOneParams): Promise<File> {
    const file = await this.getFile(params);
    if (isNil(file)) {
      throw new ActivepiecesError({
        code: ErrorCode.FILE_NOT_FOUND,
        params: { id: params.fileId },
      });
    }
    return file;
  },

  // 获取文件数据或返回undefined
  async getDataOrUndefined({ projectId, fileId, type }: GetOneParams): Promise<GetDataResponse | undefined> {
    try {
      return await this.getDataOrThrow({ projectId, fileId, type });
    } catch (error) {
      log.error({ error }, '[FileService#getData] error');
      return undefined;
    }
  },

  // 获取文件数据或抛出异常
  async getDataOrThrow({ projectId, fileId, type }: GetOneParams): Promise<GetDataResponse> {
    const file = await fileRepo().findOneBy({ projectId, fileId, type });

    if (isNil(file)) {
      throw new ActivepiecesError({
        code: ErrorCode.FILE_NOT_FOUND,
        params: { id: fileId },
      });
    }

    // 根据存储位置获取数据
    const data = await fileCompressor.decompress({
      data: file.location === FileLocation.DB
        ? file.data
        : await s3Helper(log).getFile(file.s3Key!),
      compression: file.compression,
    });

    return {
      metadata: file.metadata,
      data,
      fileName: file.fileName,
    };
  },

  // 批量删除过期文件
  async deleteStaleBulk(types: FileType[]): Promise<void> {
    const retentionDateBoundary = dayjs()
      .subtract(EXECUTION_DATA_RETENTION_DAYS, 'days')
      .toISOString();

    const maximumFilesToDeletePerIteration = 4000;
    let totalAffected = 0;

    // 分批处理大量文件
    let affected: undefined | number = undefined;
    while (isNil(affected) || affected === maximumFilesToDeletePerIteration) {
      const staleFiles = await fileRepo().find({
        select: ['id', 'created', 's3Key'],
        where: {
          type: In(types),
          created: LessThanOrEqual(retentionDateBoundary),
        },
        take: maximumFilesToDeletePerIteration,
      });

      // 删除S3文件
      const s3Keys = staleFiles.filter(f => !isNil(f.s3Key)).map(f => f.s3Key!);
      await s3Helper(log).deleteFiles(s3Keys);

      // 删除数据库记录
      const result = await fileRepo().delete({
        type: In(types),
        created: LessThanOrEqual(retentionDateBoundary),
        id: In(staleFiles.map(file => file.id)),
      });

      affected = result.affected || 0;
      totalAffected += affected;

      log.info({
        counts: affected,
        types,
      }, '[FileService#deleteStaleBulk] iteration completed');
    }

    log.info({
      totalAffected,
      types,
    }, '[FileService#deleteStaleBulk] completed');
  },
});
```

### 文件存储策略

#### 存储位置选择逻辑
```typescript
function getLocationForFile(type: FileType): FileLocation {
  const FILE_LOCATION = system.getOrThrow<FileLocation>(
    AppSystemProp.FILE_STORAGE_LOCATION
  );

  // 过期文件使用配置的存储位置
  if (isExecutionDataFileThatExpires(type)) {
    return FILE_LOCATION;
  }

  // 永久文件存储在数据库
  return FileLocation.DB;
}
```

#### 文件类型分类
```typescript
function isExecutionDataFileThatExpires(type: FileType): boolean {
  switch (type) {
    case FileType.FLOW_RUN_LOG:
    case FileType.FLOW_STEP_FILE:
    case FileType.TRIGGER_PAYLOAD:
    case FileType.TRIGGER_EVENT_FILE:
      return true;    // 临时文件，会过期

    case FileType.SAMPLE_DATA:
    case FileType.SAMPLE_DATA_INPUT:
    case FileType.PACKAGE_ARCHIVE:
    case FileType.PROJECT_RELEASE:
    case FileType.FLOW_VERSION_BACKUP:
      return false;   // 永久文件
    default:
      throw new Error(`File type ${type} is not supported`);
  }
}
```

### 数据库文件存储

#### 数据库存储实现
```typescript
type BaseFile = Pick<File, 'id' | 'projectId' | 'platformId' | 'type' | 'fileName' | 'compression' | 'size' | 'metadata' | 'created' | 'updated'>;

const saveFileToDb = async (baseFile: BaseFile, data: SaveParams['data']) => {
  assertNotNullOrUndefined(data, 'data is required');
  return fileRepo().save({
    ...baseFile,
    location: FileLocation.DB,
    data,
  });
};
```

### S3集成

#### S3辅助服务 (`file/s3-helper.ts`)
```typescript
export const s3Helper = (log: FastifyBaseLogger) => ({
  // 构建S3键
  async constructS3Key(
    platformId: string,
    projectId: string,
    type: FileType,
    fileId: string
  ): Promise<string> {
    // S3键格式：{platformId}/{projectId}/{type}/{fileId}
    return `${platformId}/${projectId}/${type}/${fileId}`;
  },

  // 上传文件
  async uploadFile(key: string, data: Buffer): Promise<void> {
    const s3Client = this.getS3Client();

    const params = {
      Bucket: system.get(AppSystemProp.S3_BUCKET),
      Key: key,
      Body: data,
    };

    await s3Client.putObject(params).promise();
  },

  // 获取文件
  async getFile(key: string): Promise<Buffer> {
    const s3Client = this.getS3Client();

    const params = {
      Bucket: system.get(AppSystemProp.S3_BUCKET),
      Key: key,
    };

    const result = await s3Client.getObject(params).promise();
    return result.Body as Buffer;
  },

  // 删除文件
  async deleteFile(key: string): Promise<void> {
    const s3Client = this.getS3Client();

    const params = {
      Bucket: system.get(AppSystemProp.S3_BUCKET),
      Key: key,
    };

    await s3Client.deleteObject(params).promise();
  },

  // 批量删除文件
  async deleteFiles(keys: string[]): Promise<void> {
    if (keys.length === 0) {
      return;
    }

    const s3Client = this.getS3Client();

    const params = {
      Bucket: system.get(AppSystemProp.S3_BUCKET),
      Delete: {
        Objects: keys.map(key => ({ Key: key })),
      },
    };

    await s3Client.deleteObjects(params).promise();
  },

  // 获取S3客户端
  private getS3Client(): AWS.S3 {
    return new AWS.S3({
      accessKeyId: system.get(AppSystemProp.S3_ACCESS_KEY_ID),
      secretAccessKey: system.get(AppSystemProp.S3_SECRET_ACCESS_KEY),
      region: system.get(AppSystemProp.S3_REGION),
      endpoint: system.get(AppSystemProp.S3_ENDPOINT),
    });
  },
});
```

### 文件压缩管理

#### 文件压缩配置
```typescript
enum FileCompression {
  NONE = 'none',
  GZIP = 'gzip',
}

interface SaveParams {
  fileId?: FileId | undefined;
  projectId?: ProjectId;
  data: Buffer | null;
  size: number;
  type: FileType;
  platformId?: string;
  fileName?: string;
  compression: FileCompression;
  metadata?: Record<string, string>;
}
```

### 步骤文件管理

#### 步骤文件控制器 (`file/step-file/step-file.controller.ts`)
```typescript
export const stepFileController: FastifyPluginAsyncTypebox = async (app) => {
  // 上传步骤文件
  app.post('/', UploadStepFileRequest, async (request) => {
    const file = await stepFileService(request.log).upload({
      projectId: request.principal.projectId,
      flowId: request.body.flowId,
      stepName: request.body.stepName,
      runId: request.body.runId,
      file: await request.file(),
    });

    return file;
  });

  // 下载步骤文件
  app.get('/:stepFileId', DownloadStepFileRequest, async (request, reply) => {
    const file = await stepFileService(request.log).download({
      projectId: request.principal.projectId,
      stepFileId: request.params.stepFileId,
    });

    return reply
      .type(file.fileName)
      .header('Content-Disposition', `attachment; filename="${file.fileName}"`)
      .send(file.data);
  });
};
```

#### 步骤文件服务 (`file/step-file/step-file.service.ts`)
```typescript
export const stepFileService = (log: FastifyBaseLogger) => ({
  // 上传步骤文件
  async upload(params: UploadStepFileParams): Promise<File> {
    const file = params.file;
    const fileBuffer = await file.toBuffer();

    return await fileService(log).save({
      projectId: params.projectId,
      fileName: file.filename,
      data: fileBuffer,
      size: fileBuffer.length,
      type: FileType.FLOW_STEP_FILE,
      platformId: undefined,
      compression: FileCompression.GZIP,
      metadata: {
        flowId: params.flowId,
        stepName: params.stepName,
        runId: params.runId,
        originalName: file.filename,
        mimeType: file.mimetype,
      },
    });
  },

  // 下载步骤文件
  async download(params: DownloadStepFileParams): Promise<DownloadStepFileResponse> {
    const file = await fileService(log).getFileOrThrow({
      projectId: params.projectId,
      fileId: params.stepFileId,
      type: FileType.FLOW_STEP_FILE,
    });

    const fileData = await fileService(log).getDataOrThrow({
      projectId: params.projectId,
      fileId: params.stepFileId,
      type: FileType.FLOW_STEP_FILE,
    });

    return {
      fileName: file.fileName,
      data: fileData.data,
    };
  },
});
```

## 对外接口

### RESTful API
```typescript
// 核心 API 端点
GET    /api/v1/flows                 // 获取流程列表
POST   /api/v1/flows                 // 创建新流程
GET    /api/v1/flows/:id             // 获取流程详情
PUT    /api/v1/flows/:id             // 更新流程
DELETE /api/v1/flows/:id             // 删除流程
POST   /api/v1/flows/:id/publish     // 发布流程
POST   /api/v1/flows/:id/duplicate   // 复制流程

POST   /api/v1/flow-runs             // 执行流程
GET    /api/v1/flow-runs             // 获取执行历史
GET    /api/v1/flow-runs/:id         // 获取执行详情
POST   /api/v1/flow-runs/:id/retry   // 重试执行
POST   /api/v1/flow-runs/:id/stop    // 停止执行

POST   /api/v1/authentication/sign-up     // 用户注册
POST   /api/v1/authentication/sign-in     // 用户登录
POST   /api/v1/authentication/sign-out    // 用户登出
POST   /api/v1/authentication/verify-email // 邮箱验证
POST   /api/v1/authentication/reset-password // 密码重置

GET    /api/v1/folders               // 获取文件夹列表
POST   /api/v1/folders               // 创建文件夹
PUT    /api/v1/folders/:id           // 更新文件夹
DELETE /api/v1/folders/:id           // 删除文件夹

// Webhook API
POST   /api/v1/webhooks/:flowId      // Webhook端点

// 用户管理 API
GET    /api/v1/platform-users        // 获取平台用户列表
POST   /api/v1/platform-users        // 创建平台用户
PUT    /api/v1/platform-users/:id    // 更新平台用户
DELETE /api/v1/platform-users/:id    // 删除平台用户

// 触发器 API
POST   /api/v1/test-trigger          // 测试触发器
GET    /api/v1/trigger-events        // 获取触发事件
POST   /api/v1/trigger-runs          // 创建触发运行

// 文件管理 API
POST   /api/v1/step-files            // 上传步骤文件
GET    /api/v1/step-files/:id        // 下载步骤文件
```

### WebSocket 事件
```typescript
// WebSocket 事件类型
interface WebSocketEvents {
  // 流程执行事件
  'flow:run:started': FlowRunStartedEvent;
  'flow:run:updated': FlowRunUpdatedEvent;
  'flow:run:completed': FlowRunCompletedEvent;
  'flow:run:failed': FlowRunFailedEvent;

  // 实时日志
  'flow:run:log': FlowRunLogEvent;
  'flow:run:step:updated': StepRunUpdatedEvent;

  // 系统事件
  'system:notification': SystemNotificationEvent;
  'system:maintenance': SystemMaintenanceEvent;
}
```

## 数据模型

### 核心实体

#### Flow - 流程
```typescript
@Entity('flow')
export class Flow {
  @PrimaryColumn()
  id: string;

  @Column()
  name: string;

  @Column()
  displayName: string;

  @Column({ nullable: true })
  description: string;

  @Column()
  projectId: string;

  @Column({ nullable: true })
  folderId: string;

  @Column({ default: FlowStatus.DRAFT })
  status: FlowStatus;

  @Column({ default: false })
  locked: boolean;

  @Column({ nullable: true })
  lockedBy: string;

  @Column({ nullable: true })
  lockedAt: Date;

  @CreateDateColumn()
  created: Date;

  @UpdateDateColumn()
  updated: Date;

  @ManyToOne(() => Project, project => project.flows)
  project: Project;

  @OneToOne(() => FlowVersion, version => version.flow)
  version: FlowVersion;

  @OneToMany(() => FlowRun, run => run.flow)
  runs: FlowRun[];
}
```

#### FlowRun - 流程执行
```typescript
@Entity('flow_run')
export class FlowRun {
  @PrimaryColumn()
  id: string;

  @Column()
  flowVersionId: string;

  @Column({ default: FlowRunStatus.RUNNING })
  status: FlowRunStatus;

  @Column({ type: 'jsonb', nullable: true })
  output: any;

  @Column({ nullable: true })
  error: string;

  @Column({ nullable: true })
  startedAt: Date;

  @Column({ nullable: true })
  completedAt: Date;

  @Column({ nullable: true })
  duration: number;

  @Column({ type: 'simple-array', nullable: true })
  tags: string[];

  @Column({ nullable: true })
  environment: ExecutionType;

  @Column({ nullable: true })
  triggerType: string;

  @CreateDateColumn()
  created: Date;

  @UpdateDateColumn()
  updated: Date;

  @ManyToOne(() => FlowVersion, version => version.runs)
  flowVersion: FlowVersion;

  @OneToMany(() => StepRun, stepRun => stepRun.flowRun)
  stepRuns: StepRun[];
}
```

#### Folder - 文件夹
```typescript
@Entity('folder')
export class Folder {
  @PrimaryColumn()
  id: string;

  @Column()
  displayName: string;

  @Column()
  projectId: string;

  @CreateDateColumn()
  created: Date;

  @UpdateDateColumn()
  updated: Date;

  @ManyToOne(() => Project, project => project.folders)
  project: Project;

  @OneToMany(() => Flow, flow => flow.folder)
  flows: Flow[];
}
```

#### User - 用户
```typescript
@Entity('user')
export class User {
  @PrimaryColumn()
  id: string;

  @Column()
  identityId: string;

  @Column({ nullable: true })
  platformId: string;

  @Column()
  platformRole: PlatformRole;

  @Column()
  status: UserStatus;

  @Column({ nullable: true })
  externalId: string;

  @CreateDateColumn()
  created: Date;

  @UpdateDateColumn()
  updated: Date;

  @ManyToOne(() => UserIdentity, identity => identity.users)
  identity: UserIdentity;

  @ManyToOne(() => Platform, platform => platform.users)
  platform: Platform;
}
```

#### TriggerSource - 触发器源
```typescript
@Entity('trigger_source')
export class TriggerSource {
  @PrimaryColumn()
  id: string;

  @Column()
  type: string;

  @Column()
  projectId: string;

  @Column()
  flowId: string;

  @Column()
  triggerName: string;

  @Column()
  flowVersionId: string;

  @Column()
  pieceName: string;

  @Column()
  pieceVersion: string;

  @Column()
  simulate: boolean;

  @Column({ type: 'jsonb', nullable: true })
  schedule?: ScheduleOptions;

  @CreateDateColumn()
  created: Date;

  @UpdateDateColumn()
  updated: Date;

  @ManyToOne(() => Flow, flow => flow.triggerSources)
  flow: Flow;
}
```

#### File - 文件
```typescript
@Entity('file')
export class File {
  @PrimaryColumn()
  id: string;

  @Column({ nullable: true })
  projectId: string;

  @Column({ nullable: true })
  platformId: string;

  @Column()
  type: FileType;

  @Column()
  location: FileLocation;

  @Column({ nullable: true })
  s3Key: string;

  @Column({ nullable: true })
  fileName: string;

  @Column()
  compression: FileCompression;

  @Column()
  size: number;

  @Column({ type: 'jsonb', nullable: true })
  metadata: Record<string, string>;

  @Column({ type: 'bytea', nullable: true })
  data: Buffer;

  @CreateDateColumn()
  created: Date;

  @UpdateDateColumn()
  updated: Date;
}
```

## 安全机制

### 1. 认证
- **JWT 令牌**: 无状态的身份验证
- **刷新令牌**: 安全的令牌续期机制
- **令牌黑名单**: 支持令牌撤销
- **多因子认证**: 企业级安全支持

### 2. 授权
- **RBAC**: 基于角色的访问控制
- **权限检查**: 细粒度的权限验证
- **实体所有权**: 确保数据访问安全
- **API 密钥**: 服务间认证支持

### 3. 安全中间件
```typescript
// 安全处理链
app.addHook('preHandler', securityMiddleware);
app.addHook('preHandler', rateLimitMiddleware);
app.addHook('preHandler', auditMiddleware);
app.addHook('onSend', dataMaskingMiddleware);
```

## 性能优化

### 1. 数据库优化
- **连接池**: 优化数据库连接管理
- **查询优化**: 避免 N+1 查询问题
- **索引策略**: 关键字段建立索引
- **分页查询**: 大数据集分页处理

### 2. 缓存策略
```typescript
// 多层缓存
- Redis Cache            // 分布式缓存
- Memory Cache           // 内存缓存
- Database Cache         // 查询缓存
- CDN Cache              // 静态资源缓存
```

### 3. 队列优化
- **任务分片**: 大任务分片处理
- **优先级队列**: 重要任务优先处理
- **重试机制**: 失败任务自动重试
- **死信队列**: 无法处理的任务处理

## 监控与日志

### 1. 应用监控
```typescript
// 监控指标
interface ServerMetrics {
  requestCount: number;
  responseTime: number;
  errorRate: number;
  activeConnections: number;
  queueSize: number;
  memoryUsage: number;
  cpuUsage: number;
  databaseConnections: number;
}
```

### 2. 结构化日志
```typescript
// 日志上下文
interface LogContext {
  requestId: string;
  userId: string;
  projectId: string;
  ip: string;
  userAgent: string;
  method: string;
  url: string;
  statusCode: number;
  duration: number;
}
```

### 3. 健康检查
```typescript
// 健康检查端点
GET /api/v1/health           // 基础健康检查
GET /api/v1/health/detailed  // 详细健康检查
GET /api/v1/ready           // 就绪状态检查
GET /api/v1/metrics         // Prometheus 指标
```

## 常见问题 (FAQ)

### Q: 如何扩展新的 API 端点？
A: 创建新的控制器文件，定义路由处理器，在相应模块中注册路由。

### Q: 如何处理大文件上传？
A: 使用流式处理、分块上传、进度显示，限制文件大小和类型。

### Q: 如何优化数据库查询性能？
A: 使用索引、优化查询语句、避免 N+1 查询、使用连接池。

### Q: 如何实现实时通信？
A: 使用 WebSocket 连接，定义事件类型，处理连接管理和错误恢复。

### Q: Webhook处理失败如何处理？
A: 系统提供自动重试机制，支持错误日志记录，可以配置重试次数和间隔。

### Q: 用户权限如何控制？
A: 使用基于角色的访问控制(RBAC)，支持平台级和项目级权限管理。

### Q: 文件存储如何选择？
A: 系统支持数据库和S3两种存储方式，会根据文件类型和配置自动选择最佳存储策略。

## 相关文件清单

### API 服务 (`api/`)
- `src/main.ts` - 服务入口
- `src/app/app.ts` - 应用配置
- `src/app/flows/` - 流程管理模块
- `src/app/authentication/` - 认证授权模块
- `src/app/webhooks/` - Webhook管理模块
- `src/app/user/` - 用户管理模块
- `src/app/trigger/` - 触发器管理模块
- `src/app/file/` - 文件管理模块
- `src/app/database/` - 数据库模块
- `src/app/**/*` - 其他业务模块

### 工作进程 (`worker/`)
- `src/index.ts` - 工作进程入口
- `src/lib/` - 核心处理逻辑

### 共享库 (`shared/`)
- `src/lib/` - 共享工具和服务

### 配置文件
- `package.json` - 包配置
- `tsconfig.json` - TypeScript 配置
- `jest.setup.js` - 测试配置

---

*模块文档版本: 4.0.0*
*最后更新: 2025-11-18 15:59:51*