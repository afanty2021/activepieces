[根目录](../../CLAUDE.md) > [packages](../) > **react-ui**

# React UI 模块文档

## 模块职责

ActivePieces React UI 是平台的现代化前端界面，提供直观的可视化流程编辑器、实时协作、多语言支持等功能。它采用 React 18 + TypeScript + Vite 技术栈，为用户提供流畅的自动化工作流构建体验。

## 入口与启动

- **主入口**: `src/main.tsx`
- **应用根组件**: `src/app/app.tsx`
- **启动命令**: `nx serve react-ui`

### 启动流程
```typescript
// React 18 Strict Mode
root.render(
  <StrictMode>
    <App />
  </StrictMode>,
);
```

## 技术栈

### 核心框架
- **React 18**: 前端框架，支持并发特性
- **TypeScript**: 类型安全的 JavaScript
- **Vite**: 快速构建工具
- **React Router v6**: 路由管理

### UI 组件库
- **Radix UI**: 无样式组件库
- **Tailwind CSS**: 实用优先的 CSS 框架
- **Lucide React**: 图标库
- **React Hook Form**: 表单处理
- **TanStack React Query**: 服务端状态管理

### 可视化编辑器
- **@xyflow/react**: 流程图可视化
- **CodeMirror**: 代码编辑器
- **React DnD Kit**: 拖拽功能
- **Framer Motion**: 动画库

### 国际化与工具
- **React i18next**: 国际化框架
- **Zustand**: 客户端状态管理
- **Axios**: HTTP 客户端
- **Socket.io Client**: WebSocket 客户端

## 应用架构

### 目录结构
```
src/
├── app/                    # 应用组件
│   ├── app.tsx            # 应用根组件
│   ├── builder/           # 流程构建器
│   ├── platform/          # 平台管理
│   ├── settings/          # 设置页面
│   └── shared/            # 共享组件
├── components/             # 通用组件
│   ├── ui/               # UI 基础组件
│   ├── forms/            # 表单组件
│   └── layout/           # 布局组件
├── hooks/                # 自定义 Hooks
├── lib/                  # 工具库
├── stores/               # 状态管理
├── i18n/                 # 国际化配置
└── styles/               # 样式文件
```

### 核心模块

#### 1. 流程构建器 (`app/builder/`)
```typescript
builder/
├── builder-state-provider.tsx    # 构建器状态管理
├── builder-header.tsx            # 构建器头部
├── data-selector/               # 数据选择器
├── flow-canvas/                 # 流程画布
├── pieces-selector/             # 集成选择器
├── piece-properties/            # 属性配置面板
├── step-settings/               # 步骤设置
├── test-step/                   # 测试面板
├── run-details/                 # 执行详情
└── run-list/                    # 执行历史
```

##### 流程画布 (`flow-canvas/`)
- **可视化编辑器**: 基于 React Flow 的拖拽式流程编辑
- **节点系统**: 支持多种节点类型（动作、触发器、路由、循环）
- **连接线**: 智能连接线和路由
- **上下文菜单**: 右键菜单和快捷操作

##### 属性配置 (`piece-properties/`)
- **动态表单**: 根据集成配置动态生成表单
- **属性验证**: 实时验证和错误提示
- **代码编辑器**: 代码片段编辑支持
- **文件上传**: 文件选择和上传组件

##### 测试面板 (`test-step/`)
- **触发器测试**: 手动触发和模拟测试
- **动作测试**: 单步执行和调试
- **数据预览**: 输入输出数据预览
- **错误调试**: 详细错误信息和堆栈

#### 2. 平台管理 (`app/platform/`)
```typescript
platform/
├── dashboard/           # 仪表板
├── flows/              # 流程管理
├── connections/        # 连接管理
├── pieces/             # 集成管理
├── projects/           # 项目管理
├── users/              # 用户管理
└── settings/           # 平台设置
```

#### 3. 共享组件 (`app/shared/`)
- **错误边界**: 错误捕获和处理
- **加载状态**: 统一的加载组件
- **通知系统**: 消息提示和通知
- **路由保护**: 权限验证和重定向

## 状态管理

### 1. Zustand 状态
```typescript
// 构建器状态
interface BuilderState {
  flow: Flow;
  selectedStep: string | null;
  readonly: boolean;
  setFlow: (flow: Flow) => void;
  selectStep: (stepId: string | null) => void;
  updateStep: (stepId: string, updates: Partial<Step>) => void;
}

// 应用状态
interface AppState {
  user: User | null;
  currentProject: Project | null;
  theme: 'light' | 'dark';
  language: string;
  setUser: (user: User | null) => void;
  setCurrentProject: (project: Project | null) => void;
  setTheme: (theme: 'light' | 'dark') => void;
  setLanguage: (language: string) => void;
}
```

### 2. React Query 状态
```typescript
// 服务端状态
const useFlowsQuery = (projectId: string) => {
  return useQuery({
    queryKey: ['flows', projectId],
    queryFn: () => flowService.list({ projectId }),
    staleTime: 5 * 60 * 1000, // 5 minutes
  });
};

const useFlowMutation = () => {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: flowService.create,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['flows'] });
    },
  });
};
```

## 组件设计

### 1. 基础组件 (`components/ui/`)
```typescript
// 按钮
interface ButtonProps {
  variant?: 'primary' | 'secondary' | 'outline' | 'ghost';
  size?: 'sm' | 'md' | 'lg';
  loading?: boolean;
  disabled?: boolean;
  leftIcon?: React.ReactNode;
  rightIcon?: React.ReactNode;
  children: React.ReactNode;
}

// 输入框
interface InputProps {
  label?: string;
  placeholder?: string;
  error?: string;
  helperText?: string;
  required?: boolean;
  disabled?: boolean;
  value?: string;
  onChange: (value: string) => void;
}
```

### 2. 表单组件 (`components/forms/`)
- **动态表单**: 根据配置生成表单
- **验证集成**: 集成 react-hook-form 验证
- **错误处理**: 统一的错误显示
- **文件上传**: 支持拖拽上传

### 3. 布局组件 (`components/layout/`)
- **侧边栏**: 可折叠的导航栏
- **顶部栏**: 用户信息和操作
- **面包屑**: 路径导航
- **模态框**: 统一的弹窗组件

## 主题与样式

### 1. Tailwind CSS 配置
```javascript
module.exports = {
  theme: {
    extend: {
      colors: {
        primary: {
          50: '#eff6ff',
          500: '#3b82f6',
          900: '#1e3a8a',
        },
        // 自定义颜色方案
      },
      fontFamily: {
        sans: ['Inter', 'sans-serif'],
        mono: ['JetBrains Mono', 'monospace'],
      },
    },
  },
  plugins: [
    require('@tailwindcss/forms'),
    require('@tailwindcss/typography'),
  ],
};
```

### 2. 暗色模式
```typescript
// 主题切换
const useTheme = () => {
  const [theme, setTheme] = useState<'light' | 'dark'>('light');

  useEffect(() => {
    const root = document.documentElement;
    root.classList.remove('light', 'dark');
    root.classList.add(theme);
  }, [theme]);

  return { theme, setTheme };
};
```

### 3. 组件样式
- **CSS-in-JS**: 使用 Tailwind CSS 类
- **响应式设计**: 移动优先设计
- **动画效果**: Framer Motion 动画
- **自定义组件**: 特定样式覆盖

## 国际化

### 1. 语言支持
```typescript
// 支持的语言
enum Language {
  EN = 'en',
  ZH = 'zh',
  ES = 'es',
  FR = 'fr',
  DE = 'de',
  JA = 'ja',
  PT = 'pt',
  RU = 'ru',
}
```

### 2. 翻译配置
```javascript
// i18next 配置
import i18n from 'i18next';
import { initReactI18next } from 'react-i18next';

i18n
  .use(initReactI18next)
  .init({
    lng: 'en',
    fallbackLng: 'en',
    interpolation: {
      escapeValue: false,
    },
    resources: {
      en: {
        translation: require('./locales/en/translation.json'),
      },
      zh: {
        translation: require('./locales/zh/translation.json'),
      },
    },
  });
```

### 3. 使用方式
```typescript
// 在组件中使用
import { useTranslation } from 'react-i18next';

const Component = () => {
  const { t } = useTranslation();

  return (
    <div>
      <h1>{t('title')}</h1>
      <p>{t('description')}</p>
    </div>
  );
};
```

## 实时功能

### 1. WebSocket 连接
```typescript
// WebSocket 客户端
const useWebSocket = (projectId: string) => {
  const [socket, setSocket] = useState<Socket | null>(null);

  useEffect(() => {
    const newSocket = io(`${API_URL}/project/${projectId}`, {
      auth: {
        token: getAuthToken(),
      },
    });

    newSocket.on('connect', () => {
      console.log('Connected to server');
    });

    newSocket.on('flow:run:updated', (data) => {
      // 更新流程执行状态
    });

    setSocket(newSocket);

    return () => {
      newSocket.close();
    };
  }, [projectId]);

  return socket;
};
```

### 2. 实时更新
- **流程执行状态**: 实时显示执行进度
- **步骤日志**: 实时显示执行日志
- **通知消息**: 实时接收系统通知
- **协作功能**: 多用户实时协作（未来功能）

## 性能优化

### 1. 代码分割
```typescript
// 路由级别的代码分割
const Builder = lazy(() => import('./app/builder'));
const Dashboard = lazy(() => import('./app/platform/dashboard'));
const Settings = lazy(() => import('./app/platform/settings'));

// 组件级别的代码分割
const HeavyComponent = lazy(() => import('./components/HeavyComponent'));
```

### 2. 状态优化
```typescript
// React Query 缓存
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000,
      cacheTime: 10 * 60 * 1000,
      retry: 3,
    },
  },
});

// 组件记忆化
const MemoizedComponent = memo(Component, (prevProps, nextProps) => {
  return prevProps.id === nextProps.id;
});
```

### 3. 虚拟滚动
```typescript
// 大列表虚拟滚动
import { FixedSizeList as List } from 'react-window';

const VirtualizedList = ({ items }: { items: any[] }) => {
  const Row = ({ index, style }) => (
    <div style={style}>
      {items[index].name}
    </div>
  );

  return (
    <List
      height={600}
      itemCount={items.length}
      itemSize={50}
    >
      {Row}
    </List>
  );
};
```

## 测试策略

### 1. 单元测试
```typescript
// Jest + React Testing Library
import { render, screen, fireEvent } from '@testing-library/react';
import { Button } from './Button';

describe('Button', () => {
  it('renders with correct text', () => {
    render(<Button>Click me</Button>);
    expect(screen.getByText('Click me')).toBeInTheDocument();
  });

  it('calls onClick when clicked', () => {
    const handleClick = jest.fn();
    render(<Button onClick={handleClick}>Click me</Button>);

    fireEvent.click(screen.getByText('Click me'));
    expect(handleClick).toHaveBeenCalledTimes(1);
  });
});
```

### 2. 集成测试
```typescript
// 组件集成测试
import { render, screen } from '@testing-library/react';
import { Builder } from './builder';

describe('Builder', () => {
  it('loads flow data correctly', async () => {
    render(<Builder flowId="test-flow-id" />);

    expect(await screen.findByText('Flow Builder')).toBeInTheDocument();
    expect(await screen.findByTestId('flow-canvas')).toBeInTheDocument();
  });
});
```

### 3. E2E 测试
```typescript
// Playwright E2E 测试
import { test, expect } from '@playwright/test';

test('can create and run a flow', async ({ page }) => {
  await page.goto('/flows/new');

  // 添加触发器
  await page.click('[data-testid="add-trigger"]');
  await page.click('[data-testid="trigger-manual"]');

  // 添加动作
  await page.click('[data-testid="add-step"]');
  await page.click('[data-testid="action-http"]');

  // 配置动作
  await page.fill('[data-testid="url-input"]', 'https://api.example.com');

  // 保存并运行
  await page.click('[data-testid="save-flow"]');
  await page.click('[data-testid="run-flow"]');

  // 验证执行结果
  await expect(page.locator('[data-testid="run-status"]')).toHaveText('Succeeded');
});
```

## 构建与部署

### 1. 构建配置
```javascript
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import { resolve } from 'path';

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      '@': resolve(__dirname, 'src'),
    },
  },
  build: {
    outDir: 'dist',
    sourcemap: true,
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ['react', 'react-dom'],
          router: ['react-router-dom'],
          query: ['@tanstack/react-query'],
        },
      },
    },
  },
  server: {
    port: 3000,
    proxy: {
      '/api': {
        target: 'http://localhost:3001',
        changeOrigin: true,
      },
    },
  },
});
```

### 2. 环境配置
```bash
# .env.development
VITE_API_URL=http://localhost:3001
VITE_WS_URL=ws://localhost:3001
VITE_APP_NAME=ActivePieces
VITE_SENTRY_DSN=

# .env.production
VITE_API_URL=https://api.activepieces.com
VITE_WS_URL=wss://api.activepieces.com
VITE_APP_NAME=ActivePieces
VITE_SENTRY_DSN=...
```

### 3. 部署配置
```dockerfile
# Dockerfile
FROM node:18-alpine AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/nginx.conf

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

## 常见问题 (FAQ)

### Q: 如何添加新页面？
A: 在 `app/` 目录下创建新组件，在路由配置中添加路由，遵循现有的文件组织结构。

### Q: 如何处理大文件上传？
A: 使用分片上传、进度显示、断点续传等策略，优化用户体验。

### Q: 如何优化首屏加载速度？
A: 使用代码分割、预加载关键资源、优化图片、启用缓存等技术。

### Q: 如何处理国际化？
A: 使用 react-i18next，保持翻译文件同步，支持动态语言切换。

## 相关文件清单

### 核心应用
- `src/main.tsx` - 应用入口
- `src/app/app.tsx` - 应用根组件
- `src/app/builder/` - 流程构建器
- `src/app/platform/` - 平台管理界面

### 组件库
- `src/components/ui/` - 基础 UI 组件
- `src/components/forms/` - 表单组件
- `src/components/layout/` - 布局组件
- `src/app/shared/` - 共享应用组件

### 状态管理
- `src/stores/` - Zustand 状态
- `src/hooks/` - 自定义 Hooks
- `src/lib/` - 工具函数库

### 配置文件
- `src/i18n/` - 国际化配置
- `src/styles/` - 样式文件
- `package.json` - 包配置
- `vite.config.ts` - Vite 配置
- `tailwind.config.js` - Tailwind 配置
- `i18next-parser.config.js` - 翻译解析配置

### 测试文件
- `jest.config.ts` - Jest 配置
- `playwright.config.ts` - E2E 测试配置

---

*模块文档版本: 1.0.0*
*最后更新: 2025-11-18 15:10:43*