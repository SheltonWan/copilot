# Next.js 全栈开发专家指南

> **目标读者**：具有 Windows C++、macOS/iOS Objective-C 及 Flutter 跨平台开发经验的工程师
> **目标**：从零到一成为 Next.js 全栈开发专家
> **版本**：v2.0 | 2026 年 6 月

---

## 目录

1. [前言：为什么 Next.js 值得你投入](#1-前言为什么-nextjs-值得你投入)
2. [环境搭建与工具链](#2-环境搭建与工具链)
3. [TypeScript 深度掌握](#3-typescript-深度掌握)
4. [JavaScript 运行时与异步基础](#4-javascript-运行时与异步基础)
5. [React 核心心智模型](#5-react-核心心智模型)
6. [React Hooks 深度掌握](#6-react-hooks-深度掌握)
7. [Next.js 架构与渲染管线](#7-nextjs-架构与渲染管线)
8. [App Router 与文件系统路由](#8-app-router-与文件系统路由)
9. [Server Components vs Client Components](#9-server-components-vs-client-components)
10. [数据获取与缓存策略](#10-数据获取与缓存策略)
11. [样式解决方案](#11-样式解决方案)
12. [状态管理：从 Context 到 Zustand](#12-状态管理从-context-到-zustand)
13. [鉴权与授权体系](#13-鉴权与授权体系)
14. [数据库与 ORM 集成](#14-数据库与-orm-集成)
15. [表单处理与 Server Actions](#15-表单处理与-server-actions)
16. [API Routes 与后端逻辑](#16-api-routes-与后端逻辑)
17. [测试策略与工程质量](#17-测试策略与工程质量)
18. [性能优化与 Core Web Vitals](#18-性能优化与-core-web-vitals)
19. [架构模式与项目组织](#19-架构模式与项目组织)
20. [全栈集成：tRPC、GraphQL、WebSocket](#20-全栈集成-trpcgraphqlwebsocket)
21. [CI/CD 与部署](#21-cicd-与部署)
22. [进阶专题与生态扩展](#22-进阶专题与生态扩展)
23. [实战项目路线图](#23-实战项目路线图)
24. [附录：C++/ObjC/Flutter → Next.js 速查表](#24-附录cobjcflutter--nextjs-速查表)
25. [大企业面试题精编](#25-大企业面试题精编)

---

## 1. 前言：为什么 Next.js 值得你投入

### 1.1 你的背景优势

作为具备 **C++（Windows）**、**Objective-C（macOS/iOS）** 和 **Flutter（跨平台）** 三栈经验的开发者，你拥有绝大多数前端开发者不具备的深度：

| 已有能力 | 在 Next.js 中的映射 | 迁移难度 |
|---------|-------------------|---------|
| C++ 内存管理 / RAII | React `useEffect` cleanup = 析构函数 | ⭐⭐ |
| C++ 模板元编程 / 泛型 | TypeScript 泛型、条件类型、映射类型 | ⭐ |
| C++ 编译期优化思维 | Next.js SSG/ISR 编译期生成策略 | ⭐⭐ |
| C++ 多线程 / 并发 | Node.js Event Loop + Worker Threads | ⭐⭐⭐ |
| ObjC MVC 架构 | React 组件化 + Custom Hooks = Controller | ⭐⭐ |
| ObjC Block / GCD | JavaScript `Promise` / `async-await` | ⭐⭐ |
| ObjC KVO / Notification | React `useState` / Context / Zustand 订阅 | ⭐⭐ |
| Flutter Widget 树 | React 组件树、组合优于继承 | ⭐ |
| Flutter `setState` / Provider | React `useState`、Context API、Zustand | ⭐ |
| Flutter `Navigator 2.0` | Next.js App Router（文件系统路由） | ⭐⭐ |
| Flutter 渲染管线 | Next.js RSC 流式渲染 / Streaming SSR | ⭐⭐⭐ |
| 跨平台适配思维 | SSR / SSG / ISR / PPR 渲染策略选择 | ⭐⭐⭐ |

**Next.js 不是从零开始，而是你已有知识的自然延伸。**

### 1.2 Web 开发范式的演进与 Next.js 的定位

要理解 Next.js 为什么是当前最优的全栈 Web 框架，需要回顾 Web 渲染范式的三代演进：

**第一代：纯服务端渲染（1995-2010）**
代表：PHP、Ruby on Rails、ASP.NET。服务器生成完整 HTML，浏览器只负责展示。优势是 SEO 友好、首屏快。致命缺陷是每次交互都需要整页刷新——用户体验割裂，服务器压力大。

**第二代：客户端渲染 SPA（2010-2020）**
代表：AngularJS、React（CRA）、Vue.js。浏览器下载 JS Bundle，在客户端渲染所有 UI。优势是交互流畅如原生 App。致命缺陷是首屏白屏时间长（JS 下载+解析+执行）、SEO 几乎不可能（搜索引擎爬虫不执行 JS）、首字节时间（TTFB）到可交互时间（TTI）差距巨大。

**第三代：混合渲染（2020 至今）**
Next.js 开创性地融合了前两代的优势——首屏服务端渲染 + 后续交互客户端接管。但更关键的是 **React Server Components（RSC）**：允许组件在服务端运行，直接访问数据库/文件系统，零 JS 发送到客户端。这是 Web 开发的一次范式转移——不再是"要么服务端渲染，要么客户端渲染"，而是"每个组件独立选择运行环境"。

```
┌─────────────────────────────────────────────────────┐
│                   Web 渲染光谱                         │
│                                                       │
│  纯 SSG  ←──  ISR  ──→  SSR  ←──  CSR  ──→  SPA     │
│  (构建时)   (定时刷新)  (请求时)  (客户端)  (纯前端)    │
│                                                       │
│  Next.js 覆盖全部，按页面/组件粒度混合使用               │
└─────────────────────────────────────────────────────┘
```

而你的 C++ 编译期优化思维让你能深入理解 SSG 和 ISR——它们本质上是"编译期求值"（Compile-time Evaluation），与 C++ `constexpr` 的理念一脉相承。

### 1.3 Next.js 的核心优势

```
┌──────────────────────────────────────────────────────────┐
│                   Next.js 架构全景                          │
├──────────┬──────────┬──────────┬─────────────────────────┤
│  Edge    │  Node.js │  Browser │  CDN / Edge Network      │
├──────────┴──────────┴──────────┴─────────────────────────┤
│              Next.js Framework (React-based)              │
│  ┌────────────┐ ┌──────────┐ ┌──────────────────────┐   │
│  │ App Router │ │ RSC Layer│ │ Server Actions / API  │   │
│  └────────────┘ └──────────┘ └──────────────────────┘   │
├──────────────────────────────────────────────────────────┤
│              React 19 (RSC + Concurrent Features)        │
│  ┌──────────────┐ ┌────────────┐ ┌──────────────────┐   │
│  │ RSC Protocol │ │ Streaming  │ │ Server Actions    │   │
│  └──────────────┘ └────────────┘ └──────────────────┘   │
├──────────────────────────────────────────────────────────┤
│              Platform Layer                               │
│  ┌──────────────┐ ┌────────────┐ ┌──────────────────┐   │
│  │ V8 Engine    │ │ Webpack/   │ │ File System /    │   │
│  │ (Node.js)    │ │ Turbopack  │ │ Database / Cache  │   │
│  └──────────────┘ └────────────┘ └──────────────────┘   │
└──────────────────────────────────────────────────────────┘
```

- **混合渲染**：同一应用内 SSG、SSR、ISR、CSR 按页面/组件混合使用
- **React Server Components**：服务端组件零 JS 体积，直接访问数据库
- **文件系统路由**：`app/` 目录结构即路由，零配置
- **Server Actions**：表单提交无需手动创建 API 端点
- **流式渲染（Streaming）**：边渲染边发送，首字节时间极低
- **Partial Prerendering（PPR）**：静态外壳 + 动态内容的完美平衡
- **Edge Runtime**：全球边缘节点运行，延迟接近 CDN

### 1.4 React 的设计哲学

React 的核心理念可以用一个公式概括：**UI = f(state)**。这与你熟悉的 Flutter 声明式 UI 完全一致，但实现路径不同。

| 设计目标 | React (Next.js) 实现 |
|---------|---------------------|
| **声明式 UI** | JSX 描述 UI 结构，框架负责 DOM 操作 |
| **组件化** | 函数组件 + Hooks，组合优于继承 |
| **单向数据流** | Props 向下传递，事件向上冒泡 |
| **跨平台** | React DOM（Web）、React Native（Mobile）、RSC（Server） |
| **渐进式采用** | 可以只在一个 `<div>` 中使用 React |

**与 Flutter 对比**：React 的 `JSX ≈ Widget tree`，`useState ≈ setState`，`Context ≈ InheritedWidget`，`useEffect ≈ initState + dispose`。最大的区别是 React **不控制渲染管线的每个像素**——它输出 DOM（或 RSC Payload），由浏览器（或 React DOM）完成实际绘制。

---

## 2. 环境搭建与工具链

### 2.0 Node.js 运行时：V8 引擎与事件循环

Node.js 不是一门新语言，而是为 JavaScript 提供了服务端运行时环境。理解其架构的核心是 **V8 引擎 + libuv 事件循环**：

- **V8 引擎**：Google 开发的 JavaScript/Wasm 引擎（C++ 编写，你的主场）。负责将 JS 源码编译为机器码。V8 使用 JIT 编译（Ignition 解释器 → TurboFan 优化编译器），与 Dart VM 的编译策略类似。
- **libuv**：跨平台异步 I/O 库（C 编写）。提供事件循环（Event Loop）、线程池（默认 4 线程，用于文件 I/O 和 DNS）、异步 TCP/UDP。
- **Node.js Bindings**：将 libuv 和 V8 连接起来的 C++ 胶水层。`fs.readFile` 在 JS 侧注册回调，libuv 在线程池中执行阻塞 I/O，完成后将回调推入事件队列。

**事件循环的六阶段**（libuv）：
1. **Timers**：执行 `setTimeout`/`setInterval` 到期回调
2. **Pending Callbacks**：执行延迟到下一轮的 I/O 回调
3. **Idle/Prepare**：内部使用
4. **Poll**：获取新的 I/O 事件，执行 I/O 回调（除 close、timer、setImmediate 外）
5. **Check**：执行 `setImmediate` 回调
6. **Close Callbacks**：执行关闭回调（如 `socket.on('close')`）

每轮之间还穿插 `process.nextTick` 和 Microtask（Promise）队列。理解这个顺序对于处理 `setTimeout(fn, 0)` 和 `setImmediate(fn)` 的微妙差异至关重要。

**与 Dart Event Loop 对比**：Dart 只有两级队列（Microtask + Event），Node.js 更复杂（六阶段 + 两个插队队列）。但核心思想相同：**永远不要阻塞事件循环**。

### 2.1 安装 Node.js

```bash
# 推荐使用 fnm（Fast Node Manager）管理多版本
brew install fnm

# 安装最新 LTS
fnm install --lts
fnm use lts-latest

# 验证
node --version  # v22.x LTS
npm --version   # 10.x

# 安装 pnpm（推荐包管理器）
npm install -g pnpm
```

### 2.2 pnpm vs npm vs yarn：包管理器的本质

包管理器不仅仅是"下载依赖"。理解其架构差异有助于大型项目的依赖管理：

| 特性 | npm | yarn (Classic) | pnpm |
|------|-----|---------------|------|
| 安装策略 | 扁平化 node_modules | 扁平化 + lockfile | 内容寻址存储 + 符号链接 |
| 磁盘效率 | 低（每个项目复制） | 低 | 高（全局缓存+硬链接） |
| 幽灵依赖 | ❌ 存在 | ❌ 存在 | ✅ 严格隔离 |
| Monorepo 支持 | Workspaces | Workspaces | 原生最优 |
| 安装速度 | 慢 | 中 | 快 |

**pnpm 的"幽灵依赖"防护**：npm/yarn 的扁平化意味着你可以 `import` 任何被提升到顶层 `node_modules` 的包——即使你没有在 `package.json` 中声明它。这导致"今天能跑，明天可能不能跑"的非确定性。pnpm 使用符号链接只暴露 `package.json` 中声明的依赖，杜绝此问题。

**推荐 `pnpm`**，原因：磁盘效率最高、Monorepo 支持最好、严格依赖隔离符合工程化需求。

### 2.3 创建第一个 Next.js 项目

```bash
pnpm create next-app@latest my-app --typescript --tailwind --eslint --app --src-dir --import-alias "@/*"

# 进入项目
cd my-app
pnpm dev  # 启动开发服务器 → http://localhost:3000
```

### 2.4 项目结构解析

```
my-app/
├── src/
│   └── app/                    # App Router 路由目录（核心）
│       ├── layout.tsx          # 根布局（必须）
│       ├── page.tsx            # 首页 /
│       ├── globals.css         # 全局样式
│       └── favicon.ico
├── public/                     # 静态资源（图片、字体等）
├── next.config.ts              # Next.js 配置
├── tailwind.config.ts          # Tailwind CSS 配置
├── tsconfig.json               # TypeScript 配置
├── package.json                # 依赖管理
├── postcss.config.mjs          # PostCSS 配置
└── .eslintrc.json              # ESLint 配置
```

### 2.5 VS Code 必装插件

| 插件 | 用途 |
|------|------|
| **ES7+ React/Redux/React-Native snippets** | React 代码片段 |
| **Tailwind CSS IntelliSense** | Tailwind 自动补全、悬停预览 |
| **Prettier - Code formatter** | 代码格式化 |
| **ESLint** | 静态分析 |
| **Pretty TypeScript Errors** | 可读的类型错误 |
| **Error Lens** | 行内错误提示 |
| **Prisma** | Prisma Schema 语法高亮 |
| **Thunder Client** | API 测试（替代 Postman） |
| **GitLens** | Git 增强 |

---

## 3. TypeScript 深度掌握

### 3.0 类型系统的计算机科学基础

TypeScript 的类型系统是一个 **结构化类型系统（Structural Type System）**，而非 C++/Java 的**名义类型系统（Nominal Type System）**。这个区别是一切 TypeScript 类型编程的基石：

- **名义类型**：两个类型即使结构完全相同，只要名字不同就不能互相赋值（C++ 的 `class A {}; class B {};`）
- **结构化类型**：两个类型如果结构兼容就可以互相赋值（"鸭子类型"的类型安全版本）

从编程语言理论（PLT）的角度，TypeScript 属于：
- **渐进类型（Gradual Typing）**：可以任意选择标注类型，未标注的部分自动推断为 `any`（在 `strict: true` 下行为不同）
- **控制流分析（Control Flow Analysis）**：编译器追踪每个变量在每条路径上的类型变化
- **类型收窄（Type Narrowing）**：`typeof`、`instanceof`、`in`、判别联合等将宽类型收窄为窄类型
- **Soundness 权衡**：TypeScript 有意在某些方面不健全（unsound），以换取实用性和开发体验。例如数组索引访问不检查越界，`as` 类型断言不做运行时验证。

**与 C++ 模板的对比**：TypeScript 泛型是"类型级别的函数"——传入类型参数，返回新类型。但 TypeScript 的条件类型 (`T extends U ? X : Y`)、映射类型 (`[K in keyof T]`) 和模板字面量类型是 C++ 模板元编程所没有的高阶能力。而 C++ 模板的 SFINAE 和 `constexpr if` 在 TypeScript 中没有直接对应。

### 3.1 基础类型与类型推断

```typescript
// 类型推断
let name = 'Next.js';           // string
let version = 15;               // number
const isLatest = true;          // true (字面量类型)
let frameworks = ['React'];     // string[]

// 显式标注 (对 C++ 程序员来说很熟悉)
const count: number = 42;
const message: string = 'Hello';
const items: string[] = ['a', 'b'];

// 联合类型 (类比 std::variant)
type Status = 'idle' | 'loading' | 'success' | 'error';

// 交叉类型 (合并多个类型的属性)
type User = { name: string } & { age: number };
// 等价于 { name: string; age: number }

// 字面量类型
type Direction = 'up' | 'down' | 'left' | 'right';
type DiceRoll = 1 | 2 | 3 | 4 | 5 | 6;

// 元组 (定长定类型数组)
type Point = [number, number];
type ApiResult = [number, string]; // [statusCode, message]
```

### 3.2 接口（Interface）vs 类型别名（Type）

```typescript
// interface：描述对象形状，支持声明合并
interface User {
  name: string;
  age: number;
}
interface User {
  email: string; // 声明合并：User 现在有 name, age, email
}

// type：更灵活，支持联合/交叉/映射类型
type ID = string | number;
type PartialUser = Partial<User>;  // 所有字段变可选
type ReadonlyUser = Readonly<User>; // 所有字段变只读

// 选择建议：描述对象形状用 interface，需要联合/映射/条件类型用 type
// 大型项目中推荐统一使用 type（减少声明合并的心智负担）
```

### 3.3 泛型：类型级函数

这是对 C++ 模板程序员最友好的部分。TypeScript 泛型本质上是 **类型级函数**——接受类型参数，产出新类型。

```typescript
// 基础泛型
function identity<T>(arg: T): T {
  return arg;
}

// 泛型约束
function getLength<T extends { length: number }>(arg: T): number {
  return arg.length;
}

// 多个类型参数
function pair<T, U>(first: T, second: U): [T, U] {
  return [first, second];
}

// 泛型默认值
interface ApiResponse<T = unknown> {
  data: T;
  status: number;
}
```

### 3.4 工具类型（Utility Types）

这些是 TypeScript 内置的"类型运算符"——类似于 C++ 的 `<type_traits>` 库：

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  password: string;
  createdAt: Date;
}

// Partial：所有属性变可选（类似 C++ 的 std::optional 用于每个字段）
type UserUpdate = Partial<User>;

// Required：所有属性变必填
type UserRequired = Required<User>;

// Pick：选取部分属性
type UserPublic = Pick<User, 'id' | 'name' | 'email'>;

// Omit：排除部分属性
type UserWithoutPassword = Omit<User, 'password'>;

// Record：构造对象类型
type UserMap = Record<string, User>; // { [key: string]: User }

// ReturnType：获取函数返回值类型
type FetchResult = ReturnType<typeof fetchUser>;

// Parameters：获取函数参数类型
type FetchParams = Parameters<typeof fetchUser>;

// Extract / Exclude：从联合类型中提取/排除
type SuccessStatus = Extract<Status, 'success'>;
type NonErrorStatus = Exclude<Status, 'error'>;

// NonNullable：排除 null/undefined
type NonNullUser = NonNullable<User | null | undefined>;
```

### 3.5 条件类型与 infer

这是 TypeScript 最强大的类型级编程能力——等价于 C++ 模板元编程中的 SFINAE：

```typescript
// 条件类型
type IsString<T> = T extends string ? true : false;
type A = IsString<'hello'>;  // true
type B = IsString<42>;       // false

// infer：在条件类型中推断类型变量
type UnwrapPromise<T> = T extends Promise<infer U> ? U : T;
type Result = UnwrapPromise<Promise<string>>; // string

type ArrayElement<T> = T extends (infer U)[] ? U : T;
type Element = ArrayElement<number[]>; // number

// 获取函数第一个参数类型
type FirstArg<T> = T extends (first: infer F, ...rest: any[]) => any ? F : never;

// 递归条件类型（深度 Partial）
type DeepPartial<T> = {
  [K in keyof T]?: T[K] extends object ? DeepPartial<T[K]> : T[K];
};
```

### 3.6 映射类型与模板字面量类型

```typescript
// 映射类型：遍历对象的 key 生成新类型
type ReadonlyUser = {
  readonly [K in keyof User]: User[K];
};

// keyof 操作符：获取对象所有键的联合类型
type UserKeys = keyof User; // 'id' | 'name' | 'email' | 'password' | 'createdAt'

// 模板字面量类型
type EventName<T extends string> = `on${Capitalize<T>}`;
type ClickEvent = EventName<'click'>; // 'onClick'

type PropEvent<Obj> = {
  [K in keyof Obj & string as `on${Capitalize<K>}Change`]: (value: Obj[K]) => void;
};
```

### 3.7 satisfies 与 as const

```typescript
// satisfies：检查类型但不拓宽类型推断
const config = {
  api: 'https://api.example.com',
  timeout: 5000,
  retry: 3,
} satisfies Record<string, string | number>;

// config.api 仍被推断为 string（而非 string | number）

// as const：收窄为最精确的字面量类型
const colors = ['red', 'green', 'blue'] as const;
// colors 类型：readonly ['red', 'green', 'blue']（而非 string[]）

const routes = {
  home: '/',
  about: '/about',
  contact: '/contact',
} as const;
// routes.home 类型：'/' （而非 string）
```

### 3.8 tsconfig.json 深入配置

```json
{
  "compilerOptions": {
    /* 严格模式（底线，必须全开） */
    "strict": true,
    "noUncheckedIndexedAccess": true,  // 数组/对象索引访问加 undefined
    "exactOptionalPropertyTypes": true, // 可选属性不允许显式赋值 undefined

    /* 模块系统 */
    "module": "esnext",
    "moduleResolution": "bundler",     // Next.js 推荐
    "resolveJsonModule": true,

    /* 路径别名 */
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"],
      "@/components/*": ["./src/components/*"],
      "@/lib/*": ["./src/lib/*"],
      "@/types/*": ["./src/types/*"]
    },

    /* 输出控制 */
    "target": "esnext",
    "lib": ["dom", "dom.iterable", "esnext"],
    "jsx": "preserve",               // 保留 JSX 给 Next.js 编译
    "noEmit": true,                  // Next.js 自己处理编译

    /* 装饰器（Prisma / tRPC 等需要） */
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true,

    /* 路径导入 */
    "allowImportingTsExtensions": true,
    "isolatedModules": true
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"],
  "exclude": ["node_modules"]
}
```

### 3.9 与 C++ 模板 / ObjC 泛型的对比

| 概念 | C++ | ObjC | TypeScript |
|------|-----|------|------------|
| 泛型 | `template<typename T>` | `NSArray<NSString *>` | `<T>` |
| 类型约束 | `requires` (C++20) / SFINAE | `id<Protocol>` | `T extends Constraint` |
| 特化 | 模板特化 / 偏特化 | 无 | 条件类型 |
| 元编程 | `constexpr` / 模板递归 | 无 | 条件类型 + 递归 |
| 类型萃取 | `<type_traits>` | 无 | `keyof` / `infer` / Utility Types |
| SFINAE | 核心机制 | 无 | 条件类型分发 |
| Variadic | `template<typename... Ts>` | 无 | 剩余参数 `...args: T` |

---

## 4. JavaScript 运行时与异步基础

### 4.0 事件循环的计算机科学原理

JavaScript 的异步模型是理解 Next.js 服务端逻辑的基石。它是一个 **单线程事件驱动** 模型，与 GCD 的串行队列（Serial Queue）概念相似但实现不同。

**核心三件套**：
1. **调用栈（Call Stack）**：LIFO 栈，追踪当前执行上下文。同步代码在此栈中执行。
2. **任务队列（Task Queue / Macrotask Queue）**：`setTimeout`、`setInterval`、I/O 回调进入此队列。
3. **微任务队列（Microtask Queue）**：`Promise.then/catch/finally`、`queueMicrotask()`、`MutationObserver` 进入此队列。

**执行规则**（每次事件循环迭代）：
1. 从 Macrotask 队列取出**一个**任务执行（执行完调用栈清空）
2. 清空**所有** Microtask（包括执行过程中新产生的 Microtask）
3. 必要时渲染更新

**关键推论**：
- 如果在 Microtask 中递归添加 Microtask，Macrotask 永远得不到执行（类似 Dart 的 Microtask 饥饿问题）
- `setTimeout(fn, 0)` 将 fn 加入 Macrotask 队列（下个事件循环执行）
- `Promise.resolve().then(fn)` 将 fn 加入 Microtask 队列（当前循环清空阶段执行）
- `process.nextTick`（Node.js）优先级更高，在 Microtask 之前执行

```
┌──────────────────────────────────────────────┐
│              Event Loop 单次迭代               │
│                                                │
│  Macrotask → Microtasks(ALL) → Render(if needed) │
│      ↑_____________________________________↓   │
│              下一轮迭代                         │
└──────────────────────────────────────────────┘
```

### 4.1 Promise 与 async/await

```typescript
// Promise：异步操作的容器（类比 Future<T> in Dart）
const promise = new Promise<string>((resolve, reject) => {
  setTimeout(() => {
    if (Math.random() > 0.5) {
      resolve('Success!');
    } else {
      reject(new Error('Failed!'));
    }
  }, 1000);
});

// 链式调用
fetchUser()
  .then(user => fetchPosts(user.id))
  .then(posts => console.log(posts))
  .catch(error => console.error(error))
  .finally(() => console.log('Done'));

// async/await：语法糖，async 函数返回 Promise
async function getUserPosts(userId: number) {
  try {
    const user = await fetchUser(userId);    // 挂起点
    const posts = await fetchPosts(user.id); // 挂起点
    return posts;
  } catch (error) {
    console.error('Failed to fetch:', error);
    throw error;
  }
}

// Promise 组合器
const [user, config] = await Promise.all([
  fetchUser(),
  fetchConfig(),
]); // 并行执行

const firstResult = await Promise.race([
  fetchFromServer1(),
  fetchFromServer2(),
]); // 超时控制

const results = await Promise.allSettled([
  fetchUser(1),
  fetchUser(2),
  fetchUser(3),
]); // 不因单个失败而中断
```

### 4.2 模块系统：ESM vs CJS

```typescript
// CommonJS (传统 Node.js)
// require + module.exports
const fs = require('fs');
module.exports = { foo: 'bar' };

// ES Modules (现代标准，Next.js 使用)
import fs from 'fs';
export const foo = 'bar';
export default function myFunc() {}

// 关键差异
// CJS：运行时加载、同步、导出的是值的拷贝
// ESM：编译时静态分析（支持 Tree Shaking）、异步、导出的是值的引用（Live Binding）

// Next.js 中统一使用 ESM
// next.config.ts 可能是仅有的使用 CJS 的地方（next.config.mjs 也可以用 ESM）
```

### 4.3 闭包与作用域

```typescript
// 闭包 = 函数 + 其词法环境
// 这是 C++ 程序员需要额外注意的概念，因为与 C++ lambda 的捕获列表不同

function createCounter(initial: number) {
  let count = initial;  // 被闭包捕获的变量
  
  return {
    increment: () => ++count,
    decrement: () => --count,
    getCount: () => count,
  };
}

const counter = createCounter(0);
counter.increment(); // 1
counter.increment(); // 2

// React 中闭包的经典陷阱——与你的 C++ 直觉一致
function Timer() {
  const [count, setCount] = useState(0);
  
  useEffect(() => {
    const id = setInterval(() => {
      setCount(count + 1); // ❌ 闭包捕获的是旧值！
    }, 1000);
    return () => clearInterval(id);
  }, []);

  // ✅ 正确：使用函数式更新
  useEffect(() => {
    const id = setInterval(() => {
      setCount(c => c + 1); // 不依赖闭包中的 count
    }, 1000);
    return () => clearInterval(id);
  }, []);
}
```

### 4.4 与 C++/ObjC/Flutter 的异步模型对比

| 概念 | C++ | ObjC | Flutter (Dart) | JavaScript |
|------|-----|------|----------------|------------|
| 线程模型 | `std::thread` (OS 线程) | GCD (线程池) | Event Loop + Isolate | Event Loop (单线程) |
| 异步原语 | `std::future` / `co_await` | Block / DispatchQueue | `Future` / `async-await` | `Promise` / `async-await` |
| 并发原语 | `std::mutex` / `std::atomic` | `@synchronized` / `dispatch_semaphore` | Isolate (消息传递) | Worker Threads / Atomics |
| 延迟执行 | `std::this_thread::sleep_for` | `dispatch_after` | `Future.delayed` | `setTimeout` |
| 取消操作 | C++20 `stop_token` | `dispatch_block_cancel` | `CancelableOperation` | `AbortController` |
| 内存模型 | 共享内存 + 锁 | ARC + 锁 | GC (独立堆) | GC (共享堆) |

---

## 5. React 核心心智模型

### 5.0 声明式 UI 的理论基础

从计算机科学的角度，React 是 **声明式编程（Declarative Programming）** 在 UI 领域的应用。理解声明式与命令式的本质差异，是你从 ObjC UIKit 顺利过渡的关键。

**命令式 UI（UIKit/AppKit）**：你告诉系统"如何做"。
```objc
// 命令式思维 (ObjC)
UILabel *label = [[UILabel alloc] init];
label.text = @"Hello";
label.frame = CGRectMake(20, 40, 100, 30);
[self.view addSubview:label];
// 状态变化时——手动同步
label.text = @"World";  // 开发者负责追踪每个需要更新的 UI
```
命令式 UI 的核心问题：**状态与 UI 的手动同步**。当应用状态复杂时，开发者需要跟踪所有状态变化点并手动更新对应的 UI。这导致 Bug 的根源：忘记在某处更新某个控件。

**声明式 UI（React/SwiftUI/Flutter）**：你告诉系统"是什么"。
```tsx
// 声明式思维 (React)
function Greeting({ name }: { name: string }) {
  return <h1>Hello, {name}</h1>;
  // name 变化时，React 自动重新渲染
}
```
核心公式：**UI = f(state)**。构建函数是纯函数（给定 Props 和 State，产生 UI 描述），框架负责在状态变化时重新调用函数并高效更新 DOM。

**性能代价与优化**：声明式的挑战在于——每次状态变化都要重新运行组件函数。React 通过以下机制解决：
1. Virtual DOM：先 diff 虚拟 DOM，只更新实际变化的 DOM 节点
2. Fiber 架构：可中断的渲染，优先处理高优先级更新（用户输入）
3. `React.memo`、`useMemo`、`useCallback`：跳过不必要的重渲染
4. RSC：服务端组件完全在服务端运行，零客户端重渲染

### 5.1 组件是函数

React 组件本质上是返回 UI 描述的纯函数。这与 ObjC 的 `UIViewController` 完全不同——没有生命周期的"控制器"，只有输入 Props 输出 JSX 的函数。

```tsx
// 函数组件：接收 Props，返回 JSX
function UserCard({ name, email, avatar }: UserCardProps) {
  return (
    <div className="rounded-lg border p-4">
      <img src={avatar} alt={name} className="h-12 w-12 rounded-full" />
      <h3 className="text-lg font-bold">{name}</h3>
      <p className="text-gray-500">{email}</p>
    </div>
  );
}

// 等价于箭头函数写法
const UserCard = ({ name, email, avatar }: UserCardProps) => (
  <div className="rounded-lg border p-4">
    <img src={avatar} alt={name} className="h-12 w-12 rounded-full" />
    <h3 className="text-lg font-bold">{name}</h3>
    <p className="text-gray-500">{email}</p>
  </div>
);
```

### 5.2 JSX 的本质

JSX 不是 HTML 字符串模板——**它是 `React.createElement()` 的语法糖**。理解这一点可以避免很多困惑。

```tsx
// JSX
<div className="container">
  <h1>Hello</h1>
</div>

// 编译后等价于
React.createElement('div', { className: 'container' },
  React.createElement('h1', null, 'Hello')
);

// JSX 中的表达式用 {} 包裹
const name = 'World';
const element = <h1>Hello, {name.toUpperCase()}</h1>;

// 条件渲染
{isLoggedIn ? <Dashboard /> : <Login />}
{isLoading && <Spinner />}
{error ?? <ErrorMessage message={error} />}

// 列表渲染 (key 必须！）
{users.map(user => (
  <UserCard key={user.id} name={user.name} />
))}
```

### 5.3 Props 与单向数据流

```tsx
// Props 是只读的——子组件不能修改父组件传递的 Props
// 这是 React 单向数据流的核心约束

// 父组件
function Dashboard() {
  const [user, setUser] = useState<User | null>(null);
  
  return (
    <div>
      <Header user={user} />
      <Content user={user} onUpdateUser={setUser} />
      {/*        ↑ Props 向下           ↑ Callback 向上 */}
    </div>
  );
}

// 子组件通过 Callback 向上通知
function Content({ user, onUpdateUser }: ContentProps) {
  const handleSave = async () => {
    const updated = await saveUser(user);
    onUpdateUser(updated); // 通知父组件更新状态
  };
  
  return <button onClick={handleSave}>Save</button>;
}
```

### 5.4 Virtual DOM 与 Reconciliation

React 维护两棵 Fiber 树（current 和 workInProgress），更新过程称为 Reconciliation（协调）：

1. **触发**：`setState` / Props 变化 / Context 变化
2. **Render 阶段**（可中断）：构建新的 Fiber 树，标记需要更新的节点
3. **Diff 算法**（O(n) 启发式）：
   - 不同类型 → 销毁旧树，创建新树
   - 同类型 DOM → 更新属性
   - 同类型组件 → 组件实例保持不变，更新 Props
4. **Commit 阶段**（不可中断）：将变化应用到 DOM

```
原树                    新树
<div>                  <div>
  <h1>Title</h1>  →     <h2>New Title</h2>
  <p>Text</p>            <p>Text</p>
</div>                 </div>

Diff 结果：h1→h2 (类型不同，重建)，p 不变（复用）
```

**与 Flutter 对比**：React 的 Fiber ≈ Flutter 的 Element 树（都是 Widget/Component 的运行时实例）。React 的 Reconciliation ≈ Flutter 的 `canUpdate()` diff。两者都是 O(n) 的线性匹配。

### 5.5 与 Flutter Widget 树的详细对比

| React | Flutter | 说明 |
|-------|---------|------|
| 函数组件 | `StatelessWidget` | 无内部状态，由 Props 驱动 |
| 函数组件 + Hooks | `StatefulWidget` | 有可变状态 |
| JSX | Widget tree | 声明式描述 UI |
| `Prop` | 构造函数参数 | 父传子的只读数据 |
| `<div>` / `<span>` | `Container` / `Text` | 基础 UI 元素 |
| `className` | `decoration` / `style` | 样式 |
| `onClick` | `onPressed` / `GestureDetector` | 事件处理 |
| `key` prop | `Key` parameter | 跨重建匹配元素 |
| `<></>` Fragment | `SizedBox.shrink()` | 不产生 DOM 节点的包装 |
| `React.memo` | `const` Widget | 跳过不必要的重建 |
| Context API | `InheritedWidget` | 依赖注入 |
| Portals | `Overlay` | 脱离父级 DOM 层级的渲染 |

---

## 6. React Hooks 深度掌握

### 6.0 Hooks 的设计动机

Hooks 是为了解决 Class 组件的三大痛点：
1. **状态逻辑难以复用**：Class 组件中共享逻辑需要 HOC（高阶组件）或 Render Props，导致"包装地狱"（Wrapper Hell）
2. **生命周期方法逻辑分散**：`componentDidMount` 中同时包含数据获取和事件监听，`componentWillUnmount` 中混合了多个清理逻辑
3. **`this` 绑定困扰**：需要手动 `.bind(this)` 或使用箭头函数

Hooks 的设计理论：**将状态和副作用按关注点聚合，而非按生命周期分散**。每个 Hook 是独立的逻辑单元，可以在多个组件间复用。

### 6.1 useState

```tsx
function Counter() {
  // [当前值, 更新函数] = useState(初始值)
  const [count, setCount] = useState(0);
  const [name, setName] = useState('');

  // 函数式更新（类比 C++ 的 fetch_add）
  const increment = () => setCount(c => c + 1);
  const incrementBy3 = () => {
    setCount(c => c + 1); // c = 0 → 1
    setCount(c => c + 1); // c = 1 → 2
    setCount(c => c + 1); // c = 2 → 3  ✅ 正确
  };

  // ❌ 如果用 setCount(count + 1)，三个都会用闭包中的 oldCount，只加一次

  // 惰性初始化（昂贵计算只执行一次）
  const [data, setData] = useState(() => {
    const stored = localStorage.getItem('key');
    return stored ? JSON.parse(stored) : [];
  });

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={increment}>+1</button>
    </div>
  );
}
```

### 6.2 useEffect & useLayoutEffect

```tsx
function UserProfile({ userId }: { userId: string }) {
  const [user, setUser] = useState<User | null>(null);
  const [error, setError] = useState<Error | null>(null);

  // useEffect：渲染后异步执行（类比 GCD dispatch_async 到主队列）
  useEffect(() => {
    let cancelled = false; // 竞态条件防护标志

    async function loadUser() {
      try {
        const data = await fetchUser(userId);
        if (!cancelled) setUser(data);
      } catch (e) {
        if (!cancelled) setError(e as Error);
      }
    }

    loadUser();

    // 清理函数 = 析构函数（类比 C++ RAII / ObjC dealloc）
    return () => {
      cancelled = true;
    };
  }, [userId]); // 依赖数组：userId 变化时重新执行

  // useLayoutEffect：DOM 更新后、浏览器绘制前同步执行
  // 用于测量 DOM 尺寸、同步滚动位置等
  useLayoutEffect(() => {
    // 测量元素尺寸
  }, []);

  if (error) return <ErrorDisplay error={error} />;
  if (!user) return <Skeleton />;
  return <div>{user.name}</div>;
}

// useEffect 执行时机：
// 1. 组件渲染到 DOM
// 2. 浏览器完成绘制（用户看到内容）
// 3. useEffect 异步执行 ← 不阻塞渲染

// useLayoutEffect 执行时机：
// 1. 组件渲染到 DOM
// 2. useLayoutEffect 同步执行 ← 阻塞绘制
// 3. 浏览器完成绘制
```

### 6.3 useRef

```tsx
function AutoFocusInput() {
  const inputRef = useRef<HTMLInputElement>(null);

  useEffect(() => {
    // 挂载后自动聚焦
    inputRef.current?.focus();
  }, []);

  // useRef 的两种用途：
  // 1. DOM 引用
  // 2. 保存可变值（不触发重新渲染）

  const renderCount = useRef(0);
  renderCount.current += 1; // 不会触发重渲染

  const prevValue = useRef<string>();
  const [name, setName] = useState('');
  
  useEffect(() => {
    prevValue.current = name; // 追踪上一次的值
  }, [name]);

  return (
    <div>
      <p>Rendered {renderCount.current} times</p>
      <p>Previous: {prevValue.current}</p>
      <input ref={inputRef} value={name} onChange={e => setName(e.target.value)} />
    </div>
  );
}
```

### 6.4 useContext

```tsx
// Context = Flutter 的 InheritedWidget
// 沿组件树向下传递数据，无需逐层 props drilling

// 1. 创建 Context
const ThemeContext = createContext<ThemeContextType>({
  theme: 'light',
  toggleTheme: () => {},
});

// 2. Provider 提供值
function App() {
  const [theme, setTheme] = useState<'light' | 'dark'>('light');
  const toggleTheme = () => setTheme(t => t === 'light' ? 'dark' : 'light');

  return (
    <ThemeContext.Provider value={{ theme, toggleTheme }}>
      <MainContent />
    </ThemeContext.Provider>
  );
}

// 3. 消费者读取值
function ThemedButton() {
  const { theme, toggleTheme } = useContext(ThemeContext);
  
  return (
    <button
      className={theme === 'dark' ? 'bg-gray-800 text-white' : 'bg-white text-black'}
      onClick={toggleTheme}
    >
      Toggle Theme
    </button>
  );
}

// 注意：Context 值变化会导致所有 consumer 重新渲染
// 优化：拆分为多个 Context，或使用 useMemo 缓存值
```

### 6.5 useMemo & useCallback

```tsx
function ExpensiveList({ items, filter }: Props) {
  // useMemo：缓存计算结果
  // 类比：在 build() 外缓存复杂计算，避免每次重建都重新计算
  const filteredItems = useMemo(
    () => items.filter(item => item.name.includes(filter)),
    [items, filter] // 依赖不变，返回缓存值
  );

  // useCallback：缓存函数引用
  // 避免子组件因 Props 中新函数引用而重新渲染
  const handleClick = useCallback((id: number) => {
    console.log('Clicked:', id);
  }, []); // 空依赖 = 函数引用永不变化

  return (
    <ul>
      {filteredItems.map(item => (
        <ListItem key={item.id} item={item} onClick={handleClick} />
      ))}
    </ul>
  );
}

// React.memo：包裹组件，Props 不变则跳过重渲染
const ListItem = React.memo(function ListItem({ 
  item, 
  onClick 
}: ListItemProps) {
  return <li onClick={() => onClick(item.id)}>{item.name}</li>;
});

// 优化原则：
// 1. 先写正确代码，不做过早优化
// 2. 用 React DevTools Profiler 定位真正的性能瓶颈
// 3. useMemo/useCallback 本身也有开销（缓存比较），不要滥用
```

### 6.6 useReducer

```tsx
// useReducer 适合复杂状态逻辑
// 类比：useState 是简单变量赋值，useReducer 是状态机

type Action =
  | { type: 'increment' }
  | { type: 'decrement' }
  | { type: 'reset'; payload: number };

function reducer(state: number, action: Action): number {
  switch (action.type) {
    case 'increment': return state + 1;
    case 'decrement': return state - 1;
    case 'reset':    return action.payload;
  }
}

function Counter() {
  const [count, dispatch] = useReducer(reducer, 0);

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => dispatch({ type: 'increment' })}>+</button>
      <button onClick={() => dispatch({ type: 'decrement' })}>-</button>
      <button onClick={() => dispatch({ type: 'reset', payload: 0 })}>Reset</button>
    </div>
  );
}
```

### 6.7 自定义 Hooks

```tsx
// 提取可复用逻辑到自定义 Hook（类比 C++ 的工具函数模板）

// useDebounce：防抖
function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);

  return debouncedValue;
}

// useLocalStorage：持久化状态
function useLocalStorage<T>(key: string, initialValue: T) {
  const [storedValue, setStoredValue] = useState<T>(() => {
    try {
      const item = window.localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch {
      return initialValue;
    }
  });

  const setValue = (value: T | ((val: T) => T)) => {
    const valueToStore = value instanceof Function ? value(storedValue) : value;
    setStoredValue(valueToStore);
    window.localStorage.setItem(key, JSON.stringify(valueToStore));
  };

  return [storedValue, setValue] as const;
}

// useFetch：数据获取
function useFetch<T>(url: string) {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    const controller = new AbortController();

    async function fetchData() {
      try {
        setLoading(true);
        const res = await fetch(url, { signal: controller.signal });
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        const json = await res.json();
        setData(json);
      } catch (e) {
        if (!controller.signal.aborted) setError(e as Error);
      } finally {
        setLoading(false);
      }
    }

    fetchData();
    return () => controller.abort();
  }, [url]);

  return { data, loading, error };
}
```

### 6.8 Hooks 规则与常见陷阱

| 规则 | 原因 |
|------|------|
| **只在顶层调用 Hooks** | React 依赖调用顺序匹配 Hooks 与状态。条件/循环中调用会打乱顺序 |
| **只在 React 函数中调用** | Hooks 依赖 React 的 Fiber 节点存储状态 |
| **依赖数组必须诚实** | ESLint `react-hooks/exhaustive-deps` 规则帮你检查 |
| **useEffect 中的函数引用** | 如果 effect 依赖函数，用 `useCallback` 稳定引用或移到 effect 内部 |

```tsx
// ❌ 常见错误

// 1. 条件调用 Hook
if (condition) {
  useState(0); // ❌ 可能跳过，打乱顺序
}

// 2. 循环调用 Hook
for (const item of items) {
  useEffect(() => {}, [item]); // ❌ 数量变化
}

// 3. 忘记依赖
useEffect(() => {
  fetchUser(userId); // ❌ userId 变化不会重新 fetch
}, []); // 应该在依赖数组中加上 userId

// 4. 过度使用 useMemo
const doubled = useMemo(() => count * 2, [count]); // ❌ 简单计算无需缓存

// ✅ 修正
const doubled = count * 2; // 内存操作，直接计算即可
```

---

## 7. Next.js 架构与渲染管线

### 7.0 Web 渲染策略的计算机科学分类

从分布式系统视角，Web 渲染是一个 **代码执行位置** 的决策问题。HTML 可以在 Build Time（构建时）、Request Time（请求时）、或 Client Side（客户端）生成。

```
时间维度 →
空间维度 ↓

Server (构建时)    → SSG (Static Site Generation)
Server (请求时)    → SSR (Server-Side Rendering) / ISR (Incremental Static Regeneration)
Server + Client     → PPR (Partial Prerendering)
Client Only         → CSR (Client-Side Rendering)

Next.js 的战略创新：允许在同一应用的同一页面上混合使用所有这些策略
```

**RSC（React Server Components）的理论贡献**：传统 SSR 将整个页面作为 HTML 发送，包含所有组件的渲染结果。RSC 将组件分为两类——Server Components（服务端运行，零 JS 发送到客户端）和 Client Components（客户端运行，需要 JS）。RSC Payload 是服务端组件树序列化后的特殊格式（不是 HTML！），React 在客户端可以将其与 Client Components 合并。

**为什么 RSC 是范式转移**：在此之前，React 的所有组件都在客户端运行。RSC 打破了这一假设——组件可以在服务端运行，直接访问数据库，然后将序列化结果（非 HTML）发送给客户端。这意味着 React 从"客户端 UI 库"进化为"跨服务端和客户端的组件模型"。

### 7.1 请求-响应生命周期

```
┌─────────────────────────────────────────────────────┐
│                 Next.js 请求流程                       │
│                                                       │
│  1. 浏览器发起请求 /dashboard                          │
│         ↓                                             │
│  2. Next.js Server 接收请求                            │
│         ↓                                             │
│  3. Middleware 拦截（鉴权、重定向、A/B 测试）           │
│         ↓                                             │
│  4. 匹配路由 → 找到 page.tsx / layout.tsx             │
│         ↓                                             │
│  5. 渲染 Server Components（可访问数据库）             │
│         ↓                                             │
│  6. 生成 RSC Payload（序列化的组件树）                 │
│         ↓                                             │
│  7. Streaming：边生成边发送 HTML + RSC Payload          │
│         ↓                                             │
│  8. 浏览器渐进渲染 HTML（首屏快速可见）                 │
│         ↓                                             │
│  9. Hydration：Client Components "复活"（绑定事件）    │
│         ↓                                             │
│  10.SPA 导航：后续路由切换只获取 RSC Payload，不刷新页面 │
└─────────────────────────────────────────────────────┘
```

### 7.2 Next.js 的三层运行时

| 层级 | 运行环境 | 能力 | 限制 |
|------|---------|------|------|
| **React Server Components** | Node.js 服务端 | 数据库直连、文件系统、环境变量 | 无交互、无状态、无浏览器 API |
| **Client Components** | 浏览器 | useState/useEffect/事件/浏览器 API | 不能直接访问数据库 |
| **Route Handlers** | Node.js / Edge | 完整 HTTP 请求控制、WebSocket | 不能渲染 React 组件 |

```
Server Component (默认)       →  async function, 直接 await db.query()
Client Component ('use client') →  hooks, onClick, useState
Route Handler (route.ts)        →  GET/POST handler, Web API
```

### 7.3 Server Components 的底层原理

RSC 不是 HTML 渲染——它是一个特殊的 **序列化协议**。当 React 在服务端渲染 Server Components 时：

1. 执行 Server Component 函数（可以 `await` 数据库查询）
2. 遇到 Client Component 占位符时，不执行它，而是插入一个引用（chunk id + props）
3. 将所有结果序列化为 **RSC Payload**（类似 JSON 但支持 Promise、Symbol 等）
4. 通过 HTTP 流（Streaming）逐步发送给客户端

```
RSC Payload 格式（简化）：
M1:{"id":"./src/app/page.tsx","chunks":["@/UserList"],"name":""}
J0:["$","div",null,{"children":["$","h1",null,{"children":"Dashboard"}]}]
M2:{"id":"@/UserList","chunks":[],"name":"default"}
J1:["$","ul",null,{"children":[["$","li","1",{"children":"Alice"}]]}]}
```

### 7.4 Streaming SSR（流式渲染）

```tsx
// page.tsx
import { Suspense } from 'react';
import { UserList, UserListSkeleton } from './UserList';
import { Chart, ChartSkeleton } from './Chart';
import { Notifications, NotificationsSkeleton } from './Notifications';

export default function Dashboard() {
  return (
    <div>
      <h1>Dashboard</h1>
      
      {/* Static：立即渲染，不等待 */}
      <header>Welcome back!</header>
      
      {/* Dynamic：Suspense 包裹，独立流式加载 */}
      <Suspense fallback={<UserListSkeleton />}>
        <UserList />    {/* 数据就绪后流式发送 */}
      </Suspense>
      
      <Suspense fallback={<ChartSkeleton />}>
        <Chart />       {/* 不阻塞 UserList */}
      </Suspense>
      
      <Suspense fallback={<NotificationsSkeleton />}>
        <Notifications /> {/* 三个组件并行加载 */}
      </Suspense>
    </div>
  );
}
```

**流式渲染的工程意义**：传统 SSR 必须等待所有数据就绪才能发送 HTML（瀑布效应）。Streaming 允许先发送静态部分，动态部分在 Suspense 边界处"留空"，数据就绪后以 `<script>` 标签注入 HTML 流中。这等效于 React 的 `renderToPipeableStream`。

### 7.5 Partial Prerendering（PPR）

PPR 是 Next.js 15 的实验性功能，融合了静态和动态渲染的最优特性：

```tsx
// next.config.ts
const nextConfig: NextConfig = {
  experimental: { ppr: 'incremental' },
};

// page.tsx
export const experimental_ppr = true;

export default function Page() {
  return (
    <div>
      {/* Static Shell：构建时预渲染为静态 HTML（CDN 缓存） */}
      <header>
        <Logo />
        <Navigation />
      </header>
      
      {/* Dynamic Hole：请求时动态渲染 */}
      <Suspense fallback={<CartSkeleton />}>
        <ShoppingCart /> {/* 实时数据，不缓存 */}
      </Suspense>
    </div>
  );
}
```

**PPR 的计算机科学类比**：类似于 CPU 的分支预测（Branch Prediction）——静态外壳是"预测的路径"（快速路径），动态内容在运行时填入（类似 speculative execution 失败后回退）。

---

## 8. App Router 与文件系统路由

### 8.0 路由系统设计理论

文件系统路由的核心洞察：**目录结构本身就是一棵树，URL 路径也是一棵树**。将 URL 映射到文件系统，消除了手动维护路由配置表的负担。

对比三种路由范式：
- **配置式路由**（Angular、Vue Router）：`{ path: '/user/:id', component: User }`——灵活但需要维护路由表
- **代码式路由**（React Router）：`<Route path="/user/:id" element={<User />} />`——声明式但嵌套复杂
- **文件系统路由**（Next.js）：`app/user/[id]/page.tsx`——零配置、目录结构即文档

**App Router 的设计模式**：这不是简单的"文件名 = URL"，而是一套约定大于配置的体系。每个文件夹可以包含多个特殊文件（`page.tsx`、`layout.tsx`、`loading.tsx`、`error.tsx`），它们以组合模式协同工作。

### 8.1 路由约定文件

```
app/
├── layout.tsx          # 共享布局（持久化，导航时不重新渲染）
├── page.tsx            # 路由内容（导航时替换）
├── loading.tsx         # Suspense 边界（页面加载时显示）
├── error.tsx           # 错误边界（错误时显示）
├── not-found.tsx       # 404 页面
├── global-error.tsx    # 根错误边界
├── template.tsx        # 类似 layout 但导航时重新挂载
├── default.tsx         # 并行路由的默认页面
└── route.ts            # API 端点（替代 page.tsx）
```

```tsx
// layout.tsx：导航时保持挂载，只替换 children
export default function DashboardLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <div>
      <Sidebar />  {/* 导航到子页面时 Sidebar 不重新渲染 */}
      <main>{children}</main>
    </div>
  );
}

// template.tsx：每次导航都重新挂载（适合需要重置动画/状态的场景）
export default function Template({ children }: { children: React.ReactNode }) {
  return <div className="animate-fade-in">{children}</div>;
}

// loading.tsx：自动作为 Suspense fallback
export default function Loading() {
  return <DashboardSkeleton />;
}

// error.tsx：错误边界（必须是 Client Component）
'use client';
export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  return (
    <div>
      <h2>Something went wrong!</h2>
      <button onClick={() => reset()}>Try again</button>
    </div>
  );
}
```

### 8.2 嵌套布局与组合

```
app/
├── layout.tsx              # 根布局（html/body 标签）
├── page.tsx                # /
├── dashboard/
│   ├── layout.tsx          # /dashboard 布局（Sidebar + Header）
│   ├── page.tsx            # /dashboard
│   ├── settings/
│   │   └── page.tsx        # /dashboard/settings
│   └── analytics/
│       ├── layout.tsx      # /dashboard/analytics 的子布局
│       └── page.tsx        # /dashboard/analytics
```

**渲染时组合**：
```
RootLayout
└── DashboardLayout
    ├── Sidebar (DashboardLayout)
    └── SettingsPage (page.tsx)
```
布局嵌套是自动的——Next.js 根据目录层级组合 layout.tsx，内层 `children` 是下一层的内容。

### 8.3 路由组与私有文件夹

```typescript
// (路由组)：不影响 URL，用于组织代码
app/
├── (marketing)/
│   ├── layout.tsx   # 营销页面的共享布局
│   ├── page.tsx     # /
│   └── about/
│       └── page.tsx # /about
├── (dashboard)/
│   ├── layout.tsx   # 后台的共享布局
│   └── settings/
│       └── page.tsx # /settings

// _私有文件夹：不会成为路由
app/
├── _components/     # 私有：不会映射为 URL
│   └── Button.tsx
└── page.tsx         # /  (可以 import _components/Button)
```

### 8.4 动态路由

```typescript
// [param]：动态参数
app/
├── blog/
│   ├── page.tsx          # /blog
│   └── [slug]/
│       └── page.tsx      # /blog/hello-world

// [...param]：捕获所有
app/
└── docs/
    └── [...slug]/
        └── page.tsx      # /docs/a  /docs/a/b  /docs/a/b/c

// [[...param]]：可选捕获
app/
└── shop/
    └── [[...slug]]/
        └── page.tsx      # /shop  /shop/category/item

// 使用
export default async function BlogPost({
  params,
}: {
  params: Promise<{ slug: string }>; // Next.js 15+: params 是 Promise
}) {
  const { slug } = await params;
  const post = await getPost(slug);
  return <article>{post.content}</article>;
}

// generateStaticParams：预生成静态路径
export async function generateStaticParams() {
  const posts = await getPosts();
  return posts.map(post => ({ slug: post.slug }));
}
```

### 8.5 并行路由与拦截路由

```tsx
// 并行路由：同一页面显示多个子路由内容
app/
├── dashboard/
│   ├── layout.tsx
│   ├── @analytics/       # 插槽
│   │   └── page.tsx
│   ├── @team/            # 插槽
│   │   └── page.tsx
│   └── page.tsx

// dashboard/layout.tsx
export default function Layout({
  children,
  analytics,
  team,
}: {
  children: React.ReactNode;
  analytics: React.ReactNode;
  team: React.ReactNode;
}) {
  return (
    <div>
      {children}
      {analytics}
      {team}
    </div>
  );
}

// 拦截路由：(.)表示匹配同层级，(..)匹配上级
app/
├── photos/
│   └── [id]/
│       └── page.tsx      # /photos/123
└── feed/
    └── (..)photos/        # 拦截 photos 路由
        └── [id]/
            └── page.tsx   # 从 /feed 导航时用模态框而非整页导航
```

### 8.6 中间件（Middleware）

```typescript
// middleware.ts（项目根目录）
import { NextRequest, NextResponse } from 'next/server';

export function middleware(request: NextRequest) {
  const path = request.nextUrl.pathname;
  const token = request.cookies.get('token')?.value;

  // 路由守卫：未登录重定向
  if (path.startsWith('/dashboard') && !token) {
    return NextResponse.redirect(new URL('/login', request.url));
  }

  // 重写 URL（代理）
  if (path.startsWith('/api-proxy/')) {
    const newPath = path.replace('/api-proxy/', '/api/');
    return NextResponse.rewrite(new URL(newPath, 'https://backend.internal'));
  }

  // 添加自定义 Header
  const response = NextResponse.next();
  response.headers.set('x-custom-header', 'hello');
  return response;
}

// 匹配规则
export const config = {
  matcher: ['/dashboard/:path*', '/api/:path*'],
  // 不匹配：静态文件、_next、favicon
};

// 注意：Middleware 运行在 Edge Runtime（受限环境）
// 不能使用 Node.js API（fs、crypto 等部分功能不可用）
```

### 8.7 与 Flutter Navigator 的对比

| Next.js App Router | Flutter GoRouter | 说明 |
|-------------------|-----------------|------|
| `app/dashboard/page.tsx` | `GoRoute(path: '/dashboard')` | 路由页面 |
| `[id]/page.tsx` | `GoRoute(path: '/user/:id')` | 动态参数 |
| `layout.tsx` | `ShellRoute(builder: ...)` | 共享布局 |
| `loading.tsx` | FutureBuilder /骨架屏 | 加载状态 |
| `error.tsx` | ErrorWidget.builder | 错误边界 |
| `middleware.ts` | `GoRouter.redirect` | 路由守卫 |
| `[...slug]` | 无直接类比 | 捕获所有路由 |
| 并行路由 `@modal` | 无直接类比 | 独立子路由插槽 |

---

## 9. Server Components vs Client Components

### 9.0 双组件模型的架构理论

这是 Next.js 最核心也最容易混淆的概念。从架构角度看，RSC 与 Client Components 的分离是一种**关注点分离（Separation of Concerns）**——按"是否需要交互"来划分组件边界，而非按"是否与 UI 相关"。

**决策树**：
```
这个组件需要做以下事情吗？
├── 使用 useState / useEffect / useContext / useReducer？
│   → YES: Client Component（加 'use client'）
│
├── 使用 onClick / onSubmit / onChange 等事件处理器？
│   → YES: Client Component
│
├── 使用浏览器 API（localStorage / window / document）？
│   → YES: Client Component
│
├── 使用自定义 Hooks（内部用了 useState/useEffect）？
│   → YES: Client Component
│
├── 只是读取数据并展示？
│   → NO: 保持 Server Component
│
├── 直接访问数据库 / 文件系统？
│   → NO: 必须是 Server Component
│
└── 作为其他组件的容器（children）？
    → NO: 保持 Server Component（除非自己需要交互）
```

### 9.1 'use client' 边界的精确含义

```tsx
// 'use client' 并不意味"只在客户端运行"
// 它意味"也在客户端运行"——组件在服务端预渲染，客户端 hydration

'use client';

import { useState } from 'react';

export function Counter() {
  const [count, setCount] = useState(0);
  
  return (
    <button onClick={() => setCount(c => c + 1)}>
      Count: {count}
    </button>
  );
}
```

**'use client' 边界规则**：
- 标记 `'use client'` 的文件及其所有导入的依赖都会进入客户端 bundle
- 但 `'use client'` 组件的 children（通过 props 传入）可以是 Server Components
- `'use client'` 组件的父级可以是 Server Components

### 9.2 组合模式：Server 包裹 Client

```tsx
// ✅ 正确模式：Server Component 作为 Client Component 的 children

// app/page.tsx (Server Component)
import { ClientWrapper } from './ClientWrapper';
import { ServerData } from './ServerData';

export default function Page() {
  return (
    <ClientWrapper>
      {/* ServerData 在服务端渲染，作为静态内容传给 ClientWrapper */}
      <ServerData />
    </ClientWrapper>
  );
}

// ClientWrapper.tsx (Client Component)
'use client';

export function ClientWrapper({ children }: { children: React.ReactNode }) {
  const [isOpen, setIsOpen] = useState(false);
  
  return (
    <div>
      <button onClick={() => setIsOpen(!isOpen)}>Toggle</button>
      {isOpen && children}  {/* children 已经是服务端渲染好的静态内容 */}
    </div>
  );
}

// ServerData.tsx (Server Component)
import { db } from '@/db';

export async function ServerData() {
  const users = await db.user.findMany();
  return <ul>{users.map(u => <li key={u.id}>{u.name}</li>)}</ul>;
}
```

### 9.3 数据传递与序列化限制

```tsx
// Server Component → Client Component 的 Props 必须可序列化
// 因为数据通过 RSC Payload（类似 JSON）传递

// ✅ 可序列化
<ClientComponent
  text="hello"
  number={42}
  array={[1, 2, 3]}
  object={{ name: 'Alice' }}
  date={new Date()}  // 特殊处理，自动序列化
/>

// ❌ 不可序列化
<ClientComponent
  onClick={() => {}}       // 函数不能序列化
  component={<OtherComponent />}  // JSX 不能直接作为 prop（除非是 children）
  promise={fetch('/api')}  // Promise 不能序列化
/>

// ✅ 传递 JSX 的正确方式：通过 children
<ClientComponent>
  <ServerSideContent />  {/* children 在服务端渲染好 */}
</ClientComponent>
```

### 9.4 常见误区

| 误区 | 真相 |
|------|------|
| "加了 `'use client'` 组件就只在客户端跑" | 有 SSR 预渲染；只在 CSR 阶段才会有交互 |
| "Server Component 可以做任何事" | 不能处理用户交互；不能使用浏览器 API |
| "应该尽量少用 'use client'" | 把交互尽可能推到叶子节点，但不要牺牲开发体验 |
| "'use client' 子组件都是 Client" | children（通过 props）可以是 Server Components |
| "Server Component 是后端代码" | 它们运行在服务端但属于 React 组件树，不是 API |

---



---

## 10. 数据获取与缓存策略

### 10.0 Web 缓存层次理论

Web 缓存是一个多层次系统，Next.js 让你能在每一层精确控制：

| 缓存层 | 控制方式 | 粒度 | 失效策略 |
|--------|---------|------|---------|
| **CDN 缓存** | Cache-Control headers | 整个响应 | TTL / 手动 purge |
| **Next.js 全路由缓存** | `revalidate` / `dynamic` | 整个路由 | revalidate / revalidatePath |
| **Next.js 数据缓存** | `fetch` options / `unstable_cache` | 单个 fetch | revalidateTag / revalidatePath |
| **Next.js 请求去重** | 自动（React `cache()`） | 同一请求内 | 请求结束 |
| **浏览器缓存** | Cache-Control headers | 单个资源 | TTL / conditional request |
| **TanStack Query** | `staleTime` / `gcTime` | 客户端 query | 时间 / 手动 invalidate |

**单次请求中的缓存层级**：
```
浏览器发起请求
    ↓
CDN (Edge Cache) ← 命中则直接返回
    ↓
Next.js Server
    ↓
全路由缓存 (Full Route Cache) ← 命中则跳过组件渲染
    ↓
组件渲染开始
    ↓
请求级去重 (Request Deduplication) ← 相同 URL 只发一次 fetch
    ↓
数据缓存 (Data Cache) ← 命中则跳过 fetch
    ↓
实际 fetch / 数据库查询
```

### 10.1 fetch API 与 Next.js 扩展

```tsx
// Next.js 扩展了 fetch，增加了缓存控制选项

// force-cache：默认行为，尽可能缓存（类似 SSG）
const data = await fetch('https://api.example.com/data', {
  cache: 'force-cache',
});

// no-store：每次请求都重新获取（类似 SSR）
const data = await fetch('https://api.example.com/data', {
  cache: 'no-store',
});

// 定时重新验证（类似 ISR）
const data = await fetch('https://api.example.com/data', {
  next: { revalidate: 3600 }, // 每小时
});

// 标签化缓存失效
const data = await fetch('https://api.example.com/data', {
  next: { tags: ['posts'] },
});
// 之后可以：
// revalidateTag('posts'); // 清除所有带 posts 标签的缓存
```

### 10.2 静态数据（SSG）：构建时生成

```tsx
// 默认：Next.js 自动尝试静态渲染

// app/posts/[slug]/page.tsx
export const dynamic = 'force-static'; // 强制静态
export const revalidate = false;       // 永不重新验证

export async function generateStaticParams() {
  const posts = await fetch('https://api.example.com/posts', {
    cache: 'force-cache',
  }).then(r => r.json());

  return posts.map((post: { slug: string }) => ({
    slug: post.slug,
  }));
}

export default async function PostPage({
  params,
}: {
  params: Promise<{ slug: string }>;
}) {
  const { slug } = await params;
  const post = await getPost(slug);
  return <article>{post.content}</article>;
}
```

### 10.3 动态数据（SSR）：请求时生成

```tsx
// app/dashboard/page.tsx
export const dynamic = 'force-dynamic'; // 强制动态渲染
export const revalidate = 0;            // 等同于 no-store

export default async function DashboardPage() {
  const stats = await fetch('https://api.example.com/stats', {
    cache: 'no-store', // 每次请求都获取
  }).then(r => r.json());

  return <Dashboard stats={stats} />;
}
```

### 10.4 增量静态再生（ISR）

```tsx
// 页面级 ISR
export const revalidate = 3600; // 1 小时后重新生成

// 或 fetch 级 ISR
const data = await fetch('https://api.example.com/data', {
  next: { revalidate: 3600 },
});

// On-Demand Revalidation（按需重新验证）
// app/api/revalidate/route.ts
import { revalidatePath, revalidateTag } from 'next/cache';
import { NextRequest } from 'next/server';

export async function POST(request: NextRequest) {
  const { path, tag } = await request.json();

  if (path) revalidatePath(path);  // 清除特定路径缓存
  if (tag) revalidateTag(tag);     // 清除特定标签缓存

  return Response.json({ revalidated: true });
}
```

### 10.5 React cache() 与请求去重

```tsx
// React 的 cache() 函数：在同一请求内去重函数调用
import { cache } from 'react';

// ❌ 没有 cache：同一个请求中多次调用会重复查数据库
async function getUser(id: string) {
  return db.user.findUnique({ where: { id } });
}

// ✅ 有 cache：同一请求内多次调用只查一次
const getUser = cache(async (id: string) => {
  return db.user.findUnique({ where: { id } });
});

// 在多个组件中使用
async function Header() {
  const user = await getUser('123'); // 查询数据库
  return <div>{user.name}</div>;
}

async function Sidebar() {
  const user = await getUser('123'); // 命中缓存，不查数据库
  return <div>{user.email}</div>;
}
// 注意：cache() 只在 Server Components / Route Handlers 中工作
// Next.js 15 默认自动对 fetch 做请求去重
```

### 10.6 Server Actions 获取数据

```tsx
// Server Actions 也可以用于数据获取（适合搜索、筛选等交互场景）

// app/actions.ts
'use server';

export async function searchUsers(query: string) {
  const users = await db.user.findMany({
    where: { name: { contains: query } },
    take: 20,
  });
  return users;
}

// app/search/page.tsx (Client Component)
'use client';

import { searchUsers } from './actions';
import { useState, useEffect } from 'react';

export function SearchPage() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState<User[]>([]);

  useEffect(() => {
    if (!query) return;
    const debounce = setTimeout(async () => {
      const users = await searchUsers(query);
      setResults(users);
    }, 300);
    return () => clearTimeout(debounce);
  }, [query]);

  return (
    <div>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      <ul>
        {results.map(user => (
          <li key={user.id}>{user.name}</li>
        ))}
      </ul>
    </div>
  );
}
```

---

## 11. 样式解决方案

### 11.0 CSS 架构的演进

从混淆的全局 CSS → BEM/SMACSS 命名约定 → CSS Modules → Utility-First (Tailwind) → CSS-in-JS → 回归编译时方案。每一代都在解决上一代的痛点：

| 方案 | 原理 | 优势 | 劣势 |
|------|------|------|------|
| **Tailwind CSS** | 原子化工具类 | 快、一致、构建时生成 | 类名长、JSX 膨胀 |
| **CSS Modules** | 编译时哈希隔离 | 零运行时、原生 CSS | 命名问题仍在 |
| **styled-components** | 运行时 CSS-in-JS | 动态样式灵活 | 运行时开销、SSR 坑 |
| **Panda CSS** | 编译时 CSS-in-JS | 类型安全、零运行时 | 生态新、学习曲线 |

### 11.1 Tailwind CSS（推荐首选）

```tsx
// Tailwind 的本质：预定义的原子类映射到 CSS 属性
// w-16 → width: 4rem
// p-4 → padding: 1rem
// bg-blue-500 → background-color: #3b82f6

function Card({ title, description }: { title: string; description: string }) {
  return (
    <div className="
      rounded-xl border border-gray-200 bg-white
      p-6 shadow-sm transition-shadow hover:shadow-md
      dark:border-gray-800 dark:bg-gray-900
    ">
      <h3 className="text-lg font-semibold text-gray-900 dark:text-white">
        {title}
      </h3>
      <p className="mt-2 text-sm text-gray-500 dark:text-gray-400">
        {description}
      </p>
    </div>
  );
}

// 响应式断点
<div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">

// 条件类名管理 (cn helper)
import { clsx, type ClassValue } from 'clsx';
import { twMerge } from 'tailwind-merge';

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}

// 使用 cn
<button className={cn(
  'px-4 py-2 rounded',
  variant === 'primary' && 'bg-blue-500 text-white',
  variant === 'secondary' && 'bg-gray-200 text-gray-800',
  disabled && 'opacity-50 cursor-not-allowed',
  className, // 允许外部覆盖
)} />
```

### 11.2 CSS Modules

```css
/* Card.module.css */
.card {
  border-radius: 12px;
  border: 1px solid var(--border);
  padding: 24px;
}

.card:hover {
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
}

.title {
  font-size: 18px;
  font-weight: 600;
}
```

```tsx
import styles from './Card.module.css';

export function Card() {
  return (
    <div className={styles.card}>
      <h3 className={styles.title}>Hello</h3>
    </div>
  );
}
```

### 11.3 Shadcn/ui：不是组件库，是代码分发平台

Shadcn/ui 的革命性理念：**不安装 npm 包，而是将源码复制到你的项目中**。这意味着：
- 完全控制代码，可以随意修改
- 只打包你实际使用的组件
- 基于 Radix UI（无样式、可访问的行为原语）+ Tailwind

```bash
npx shadcn-ui@latest init
npx shadcn-ui@latest add button card dialog dropdown-menu
```

```tsx
// 使用 Shadcn/ui 组件
import { Button } from '@/components/ui/button';
import {
  Card,
  CardContent,
  CardDescription,
  CardFooter,
  CardHeader,
  CardTitle,
} from '@/components/ui/card';
import { Input } from '@/components/ui/input';

export function SignUpForm() {
  return (
    <Card className="w-[400px]">
      <CardHeader>
        <CardTitle>Create account</CardTitle>
        <CardDescription>Enter your email below.</CardDescription>
      </CardHeader>
      <CardContent>
        <Input placeholder="name@example.com" type="email" />
      </CardContent>
      <CardFooter>
        <Button className="w-full">Sign Up</Button>
      </CardFooter>
    </Card>
  );
}
```

---

## 12. 状态管理：从 Context 到 Zustand

### 12.0 Web 状态管理理论

状态按"持久性"和"作用域"可以分类为：

| 状态类型 | 示例 | 推荐方案 |
|---------|------|---------|
| **URL 状态** | 搜索词、分页、筛选条件 | URL Search Params (`nuqs`) |
| **服务端状态** | API 返回的用户列表 | TanStack Query / SWR |
| **客户端全局状态** | 主题、语言、通知计数 | Zustand / Jotai |
| **表单状态** | 输入值、验证错误 | React Hook Form |
| **UI 瞬态状态** | Modal 开/关、Dropdown 展开 | `useState` |
| **会话状态** | 当前用户、token | Context / Zustand + cookie |

**状态管理选型决策树**：
```
需要跨多个页面/组件共享？
├── NO → useState
└── YES → 数据来源于？  
    ├── URL（搜索/分页）→ URL Search Params (nuqs)
    ├── 服务端 API → TanStack Query / SWR
    └── 纯客户端 → 复杂度？
        ├── 简单（主题/语言）→ Context API
        └── 复杂（购物车/多模块）→ Zustand / Jotai
```

### 12.1 URL 状态（nuqs）

```tsx
// URL 作为唯一数据源，支持后退/前进、书签、分享
import { useQueryState } from 'nuqs';

export function SearchPage() {
  const [query, setQuery] = useQueryState('q', { defaultValue: '' });
  const [page, setPage] = useQueryState('page', { 
    defaultValue: '1',
    parse: v => parseInt(v) || 1,
  });
  const [category, setCategory] = useQueryState('category');

  // URL: /search?q=nextjs&page=2&category=guide
  // 所有这些状态都在 URL 中，可分享、可书签

  return (
    <div>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      <select value={category ?? ''} onChange={e => setCategory(e.target.value || null)}>
        <option value="">All</option>
        <option value="guide">Guide</option>
        <option value="tutorial">Tutorial</option>
      </select>
      {/* 分页组件自动同步 page */}
    </div>
  );
}
```

### 12.2 Context API

```tsx
// 适合：主题、语言、认证状态等低频变化的全局状态

const ThemeContext = createContext<{
  theme: 'light' | 'dark';
  toggleTheme: () => void;
} | null>(null);

export function ThemeProvider({ children }: { children: React.ReactNode }) {
  const [theme, setTheme] = useState<'light' | 'dark'>('light');
  const toggleTheme = () => setTheme(t => t === 'light' ? 'dark' : 'light');

  // 持久化到 localStorage（只在客户端）
  useEffect(() => {
    const stored = localStorage.getItem('theme');
    if (stored === 'dark') setTheme('dark');
  }, []);

  useEffect(() => {
    localStorage.setItem('theme', theme);
    document.documentElement.classList.toggle('dark', theme === 'dark');
  }, [theme]);

  return (
    <ThemeContext.Provider value={{ theme, toggleTheme }}>
      {children}
    </ThemeContext.Provider>
  );
}

export function useTheme() {
  const context = useContext(ThemeContext);
  if (!context) throw new Error('useTheme must be used within ThemeProvider');
  return context;
}
```

### 12.3 Zustand（推荐全局状态管理）

```tsx
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

// 定义 Store
interface CartStore {
  items: CartItem[];
  addItem: (item: CartItem) => void;
  removeItem: (id: string) => void;
  clearCart: () => void;
  totalPrice: () => number;
}

const useCartStore = create<CartStore>()(
  persist(
    (set, get) => ({
      items: [],
      
      addItem: (item) => set(state => ({
        items: [...state.items, item],
      })),
      
      removeItem: (id) => set(state => ({
        items: state.items.filter(i => i.id !== id),
      })),
      
      clearCart: () => set({ items: [] }),
      
      totalPrice: () => get().items.reduce((sum, i) => sum + i.price, 0),
    }),
    { name: 'cart-storage' } // 自动持久化到 localStorage
  )
);

// 在组件中使用（可在 Server Components 外使用）
'use client';

export function CartButton() {
  const items = useCartStore(s => s.items);     // 细粒度订阅
  const addItem = useCartStore(s => s.addItem);
  const totalPrice = useCartStore(s => s.totalPrice);

  return (
    <button onClick={() => addItem({ id: '1', name: 'Widget', price: 9.99 })}>
      Cart ({items.length}) - ${totalPrice()}
    </button>
  );
}

// Zustand 也可以订阅计算值（仅当结果变化时重渲染）
const itemCount = useCartStore(s => s.items.length); // 只在 length 变化时
```

### 12.4 TanStack Query（服务端状态管理）

```tsx
// app/providers.tsx
'use client';

import { QueryClient, QueryClientProvider } from '@tanstack/react-query';

function makeQueryClient() {
  return new QueryClient({
    defaultOptions: {
      queries: {
        staleTime: 60 * 1000,       // 1 分钟内视为新鲜
        gcTime: 5 * 60 * 1000,      // 5 分钟后垃圾回收
        retry: 1,
        refetchOnWindowFocus: false,
      },
    },
  });
}

let browserQueryClient: QueryClient | undefined;

function getQueryClient() {
  if (typeof window === 'undefined') return makeQueryClient(); // 服务端
  if (!browserQueryClient) browserQueryClient = makeQueryClient();
  return browserQueryClient;
}

export function Providers({ children }: { children: React.ReactNode }) {
  const queryClient = getQueryClient();
  return (
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  );
}
```

```tsx
// 在 Client Component 中使用
'use client';

import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';

export function UserList() {
  const queryClient = useQueryClient();

  const { data, isLoading, error } = useQuery({
    queryKey: ['users'],
    queryFn: () => fetch('/api/users').then(r => r.json()),
  });

  const deleteMutation = useMutation({
    mutationFn: (id: string) =>
      fetch(`/api/users/${id}`, { method: 'DELETE' }),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['users'] }); // 重新获取
    },
  });

  if (isLoading) return <Skeleton />;
  if (error) return <ErrorDisplay error={error} />;

  return (
    <ul>
      {data.map((user: User) => (
        <li key={user.id}>
          {user.name}
          <button onClick={() => deleteMutation.mutate(user.id)}>Delete</button>
        </li>
      ))}
    </ul>
  );
}
```

---

## 13. 鉴权与授权体系

### 13.0 Web 鉴权协议理论

Web 鉴权主要包括三种模式：

| 协议 | 原理 | 适用场景 | Token 存储 |
|------|------|---------|-----------|
| **JWT (无状态)** | 自包含 Token，服务端不存储 | 微服务、跨域 API | httpOnly Cookie / localStorage |
| **Session (有状态)** | 服务端存储 Session ID | 传统 Web 应用 | httpOnly Cookie |
| **OAuth 2.0** | 委托第三方授权 | 社交登录、第三方 API | 访问令牌 + 刷新令牌 |

**为什么 JWT 要放 httpOnly Cookie 而非 localStorage？** XSS 攻击可以读取 `localStorage`（`document.localStorage.token`），但无法读取 `httpOnly` Cookie。将 JWT 放入 `httpOnly`、`Secure`、`SameSite=Strict` 的 Cookie 是目前最安全的浏览器存储方案。

**令牌刷新策略**：Access Token 短有效期（15min），Refresh Token 长有效期（7天）。当 Access Token 过期时，用 Refresh Token 静默刷新。Next.js Middleware 可以透明处理此逻辑。

### 13.1 NextAuth.js v5 (Auth.js)

```tsx
// auth.ts
import NextAuth from 'next-auth';
import GitHub from 'next-auth/providers/github';
import Google from 'next-auth/providers/google';
import Credentials from 'next-auth/providers/credentials';
import { PrismaAdapter } from '@auth/prisma-adapter';
import { prisma } from '@/lib/prisma';
import bcrypt from 'bcryptjs';

export const { handlers, auth, signIn, signOut } = NextAuth({
  adapter: PrismaAdapter(prisma),
  session: { strategy: 'jwt' }, // 或 'database'
  pages: {
    signIn: '/login',
    error: '/auth/error',
  },
  callbacks: {
    async jwt({ token, user }) {
      if (user) {
        token.id = user.id;
        token.role = user.role;
      }
      return token;
    },
    async session({ session, token }) {
      if (session.user) {
        session.user.id = token.id as string;
        session.user.role = token.role as string;
      }
      return session;
    },
  },
  providers: [
    GitHub({
      clientId: process.env.AUTH_GITHUB_ID!,
      clientSecret: process.env.AUTH_GITHUB_SECRET!,
    }),
    Google({
      clientId: process.env.AUTH_GOOGLE_ID!,
      clientSecret: process.env.AUTH_GOOGLE_SECRET!,
    }),
    Credentials({
      name: 'credentials',
      credentials: {
        email: { label: 'Email', type: 'email' },
        password: { label: 'Password', type: 'password' },
      },
      async authorize(credentials) {
        const { email, password } = credentials as {
          email: string;
          password: string;
        };
        const user = await prisma.user.findUnique({ where: { email } });
        if (!user) return null;
        const valid = await bcrypt.compare(password, user.password!);
        if (!valid) return null;
        return { id: user.id, email: user.email, name: user.name };
      },
    }),
  ],
});
```

### 13.2 中间件鉴权

```typescript
// middleware.ts
import { auth } from '@/auth';
import { NextResponse } from 'next/server';

export default auth((req) => {
  const { pathname } = req.nextUrl;
  const isLoggedIn = !!req.auth;

  // 保护路由
  const protectedPaths = ['/dashboard', '/settings', '/profile'];
  const isProtected = protectedPaths.some(p => pathname.startsWith(p));

  if (isProtected && !isLoggedIn) {
    return NextResponse.redirect(new URL('/login', req.url));
  }

  // 已登录用户访问 login 页 → 重定向到首页
  if (isLoggedIn && pathname.startsWith('/login')) {
    return NextResponse.redirect(new URL('/dashboard', req.url));
  }

  return NextResponse.next();
});

export const config = {
  matcher: ['/dashboard/:path*', '/settings/:path*', '/profile/:path*', '/login'],
};
```

### 13.3 服务端 & 客户端使用

```tsx
// Server Component 中获取会话
import { auth } from '@/auth';

export default async function DashboardPage() {
  const session = await auth();
  if (!session) return null; // 理论上 middleware 已拦截

  return (
    <div>
      <h1>Welcome, {session.user?.name}</h1>
      <p>Role: {session.user?.role}</p>
    </div>
  );
}

// Server Action 中鉴权
'use server';

import { auth } from '@/auth';

export async function deleteUser(userId: string) {
  const session = await auth();
  if (!session || session.user?.role !== 'admin') {
    throw new Error('Unauthorized');
  }
  await prisma.user.delete({ where: { id: userId } });
}

// Client Component 中使用
'use client';

import { useSession } from 'next-auth/react';

export function UserMenu() {
  const { data: session, status } = useSession();

  if (status === 'loading') return <Skeleton />;
  if (!session) return <LoginButton />;

  return (
    <div>
      <span>{session.user?.name}</span>
      <button onClick={() => signOut()}>Sign Out</button>
    </div>
  );
}
```

### 13.4 RBAC 权限控制

```tsx
// lib/auth.ts
export type Role = 'user' | 'editor' | 'admin';

export const permissions = {
  'posts:create': ['editor', 'admin'],
  'posts:delete': ['admin'],
  'users:manage': ['admin'],
  'settings:edit': ['admin'],
} as const;

type Permission = keyof typeof permissions;

export function hasPermission(role: Role, permission: Permission): boolean {
  return permissions[permission].includes(role);
}

// 权限守卫组件
export async function RequirePermission({
  permission,
  children,
  fallback = null,
}: {
  permission: Permission;
  children: React.ReactNode;
  fallback?: React.ReactNode;
}) {
  const session = await auth();
  const role = (session?.user?.role as Role) || 'user';

  if (!hasPermission(role, permission)) return fallback;
  return <>{children}</>;
}

// 使用
<RequirePermission permission="posts:delete" fallback={<p>Access Denied</p>}>
  <DeletePostButton postId={post.id} />
</RequirePermission>
```

---

## 14. 数据库与 ORM 集成

### 14.0 服务端数据层设计

在 Next.js 中，数据库查询发生在 Server Components 和 Route Handlers 中。这意味着：
- **无需 API 端点**：可以直接在组件中 await 数据库查询
- **类型安全**：ORM 的类型可以直通组件 Props
- **减少序列化**：数据不需要 JSON 序列化/反序列化（除非传给 Client Component）

**ORM 选择**：

| ORM | 理念 | 优势 | 劣势 |
|-----|------|------|------|
| **Prisma** | 声明式 Schema + 类型生成 | DX 极佳、生态最大、迁移工具强 | 启动较慢、查询不够灵活 |
| **Drizzle** | SQL-like 类型安全 | 更接近原生 SQL、Bundle 小、Edge 友好 | 生态较新 |
| **Kysely** | 类型安全 SQL 构建器 | 极接近 SQL、零运行时、Edge 完美 | 无迁移工具 |

### 14.1 Prisma（推荐）

```prisma
// prisma/schema.prisma

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        String    @id @default(cuid())
  email     String    @unique
  name      String?
  password  String?
  role      Role      @default(USER)
  posts     Post[]
  createdAt DateTime  @default(now())
  updatedAt DateTime  @updatedAt
}

model Post {
  id        String    @id @default(cuid())
  title     String
  content   String?
  published Boolean   @default(false)
  author    User      @relation(fields: [authorId], references: [id])
  authorId  String
  tags      Tag[]
  createdAt DateTime  @default(now())
  updatedAt DateTime  @updatedAt
}

model Tag {
  id    String @id @default(cuid())
  name  String @unique
  posts Post[]
}

enum Role {
  USER
  EDITOR
  ADMIN
}
```

```tsx
// lib/prisma.ts — 单例模式（开发环境避免热重载创建多个实例）
import { PrismaClient } from '@prisma/client';

const globalForPrisma = globalThis as unknown as { prisma: PrismaClient };

export const prisma = globalForPrisma.prisma || new PrismaClient({
  log: process.env.NODE_ENV === 'development' ? ['query'] : [],
});

if (process.env.NODE_ENV !== 'production') globalForPrisma.prisma = prisma;
```

```tsx
// Server Component 中直接查询
import { prisma } from '@/lib/prisma';

export default async function PostsPage() {
  const posts = await prisma.post.findMany({
    where: { published: true },
    include: {
      author: { select: { name: true } },
      tags: true,
    },
    orderBy: { createdAt: 'desc' },
    take: 20,
  });

  return (
    <div>
      {posts.map(post => (
        <article key={post.id}>
          <h2>{post.title}</h2>
          <p>By {post.author.name}</p>
          <div>{post.tags.map(t => t.name).join(', ')}</div>
        </article>
      ))}
    </div>
  );
}
```

### 14.2 数据库迁移

```bash
# 开发：创建迁移
npx prisma migrate dev --name add_user_role

# 生产：应用迁移
npx prisma migrate deploy

# 生成 Prisma Client（Schema 变化后）
npx prisma generate

# 数据库管理界面
npx prisma studio

# 种子数据
npx prisma db seed
```

```typescript
// prisma/seed.ts
import { PrismaClient } from '@prisma/client';
import bcrypt from 'bcryptjs';

const prisma = new PrismaClient();

async function main() {
  const password = await bcrypt.hash('admin123', 12);

  const admin = await prisma.user.upsert({
    where: { email: 'admin@example.com' },
    update: {},
    create: {
      email: 'admin@example.com',
      name: 'Admin',
      password,
      role: 'ADMIN',
    },
  });

  console.log({ admin });
}

main()
  .then(async () => await prisma.$disconnect())
  .catch(async (e) => {
    console.error(e);
    await prisma.$disconnect();
    process.exit(1);
  });
```

### 14.3 数据库连接池

```typescript
// 生产环境关注连接池配置
// DATABASE_URL="postgresql://user:pass@host:5432/db?connection_limit=20&pool_timeout=10"

// Serverless 环境需要特别注意：
// 每个 Lambda / Edge Function 实例都会打开自己的连接
// 连接数 = 并发请求数 × 每个请求的平均查询连接数

// 使用 @prisma/client 的 accelerate 或 pgBouncer 管理连接池
// Supabase 默认提供 pgBouncer (Transaction Mode)
```

### 14.4 Redis 缓存集成

```tsx
// lib/redis.ts
import { Redis } from '@upstash/redis';

export const redis = new Redis({
  url: process.env.UPSTASH_REDIS_URL!,
  token: process.env.UPSTASH_REDIS_TOKEN!,
});

// 缓存查询结果
export async function getCachedPost(slug: string) {
  const cached = await redis.get<Post>(`post:${slug}`);
  if (cached) return cached;

  const post = await prisma.post.findUnique({ where: { slug } });
  if (post) {
    await redis.set(`post:${slug}`, post, { ex: 3600 }); // 1 小时过期
  }
  return post;
}

// 限流（Rate Limiting）
export async function checkRateLimit(
  key: string,
  limit: number,
  window: number
): Promise<boolean> {
  const current = await redis.incr(`ratelimit:${key}`);
  if (current === 1) {
    await redis.expire(`ratelimit:${key}`, window);
  }
  return current <= limit;
}
```

---

## 15. 表单处理与 Server Actions

### 15.0 表单理论的演进

Web 表单处理经历了三代范式转变：

1. **传统 HTML Form**：`<form action="/submit" method="POST">`，整页刷新
2. **AJAX 表单**：`e.preventDefault()` + `fetch()` + `setState`
3. **Server Actions**（React 19 / Next.js）：`<form action={serverAction}>`，渐进增强

Server Actions 的独特价值：**既支持客户端 JS（快速交互），又支持无 JS 降级（form POST）**。这是真正的渐进增强——表单在 JS 未加载时仍能工作。

### 15.1 Server Actions 基础

```tsx
// app/actions.ts
'use server';

import { revalidatePath } from 'next/cache';
import { z } from 'zod';
import { prisma } from '@/lib/prisma';

const CreatePostSchema = z.object({
  title: z.string().min(1, 'Title is required').max(100),
  content: z.string().min(10, 'Content must be at least 10 characters'),
});

export async function createPost(formData: FormData) {
  // 1. 验证
  const parsed = CreatePostSchema.safeParse({
    title: formData.get('title'),
    content: formData.get('content'),
  });

  if (!parsed.success) {
    return { error: parsed.error.flatten().fieldErrors };
  }

  // 2. 业务逻辑
  const post = await prisma.post.create({
    data: {
      title: parsed.data.title,
      content: parsed.data.content,
      authorId: '...', // 从 session 获取
    },
  });

  // 3. 缓存失效
  revalidatePath('/posts');

  // 4. 返回结果（或 redirect）
  return { success: true, postId: post.id };
}
```

```tsx
// app/posts/create/page.tsx
'use client';

import { createPost } from '@/app/actions';
import { useActionState } from 'react';

export default function CreatePostPage() {
  const [state, formAction, isPending] = useActionState(createPost, null);

  return (
    <form action={formAction}>
      <div>
        <input name="title" placeholder="Title" />
        {state?.error?.title && (
          <p className="text-red-500">{state.error.title[0]}</p>
        )}
      </div>

      <div>
        <textarea name="content" placeholder="Content" />
        {state?.error?.content && (
          <p className="text-red-500">{state.error.content[0]}</p>
        )}
      </div>

      <button type="submit" disabled={isPending}>
        {isPending ? 'Creating...' : 'Create Post'}
      </button>

      {state?.success && <p className="text-green-500">Post created!</p>}
    </form>
  );
}
```

### 15.2 React Hook Form + Zod（推荐）

```tsx
'use client';

import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';
import { createPost } from '@/app/actions';

const formSchema = z.object({
  title: z.string().min(1, 'Title is required').max(100),
  content: z.string().min(10, 'Content too short'),
  tags: z.array(z.string()).min(1, 'At least one tag'),
});

type FormData = z.infer<typeof formSchema>;

export function CreatePostForm() {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
    setError,
  } = useForm<FormData>({
    resolver: zodResolver(formSchema),
    defaultValues: { tags: [] },
  });

  const onSubmit = async (data: FormData) => {
    const formData = new FormData();
    formData.append('title', data.title);
    formData.append('content', data.content);
    formData.append('tags', data.tags.join(','));

    const result = await createPost(formData);

    if (result?.error) {
      Object.entries(result.error).forEach(([field, messages]) => {
        setError(field as keyof FormData, {
          message: (messages as string[])[0],
        });
      });
    }
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)} className="space-y-4">
      <div>
        <input
          {...register('title')}
          placeholder="Title"
          className="w-full rounded border p-2"
        />
        {errors.title && (
          <p className="text-sm text-red-500">{errors.title.message}</p>
        )}
      </div>

      <div>
        <textarea
          {...register('content')}
          placeholder="Content"
          rows={5}
          className="w-full rounded border p-2"
        />
        {errors.content && (
          <p className="text-sm text-red-500">{errors.content.message}</p>
        )}
      </div>

      <button
        type="submit"
        disabled={isSubmitting}
        className="rounded bg-blue-500 px-4 py-2 text-white disabled:opacity-50"
      >
        {isSubmitting ? 'Submitting...' : 'Submit'}
      </button>
    </form>
  );
}
```

### 15.3 文件上传

```tsx
// Server Action 处理文件上传
'use server';

import { writeFile } from 'fs/promises';
import { join } from 'path';
import { revalidatePath } from 'next/cache';

export async function uploadFile(formData: FormData) {
  const file = formData.get('file') as File;

  if (!file) {
    return { error: 'No file provided' };
  }

  // 验证
  const allowedTypes = ['image/jpeg', 'image/png', 'image/webp'];
  if (!allowedTypes.includes(file.type)) {
    return { error: 'Invalid file type' };
  }

  if (file.size > 5 * 1024 * 1024) {
    return { error: 'File too large (max 5MB)' };
  }

  // 保存到文件系统（或上传到 S3/Cloudflare R2）
  const bytes = await file.arrayBuffer();
  const buffer = Buffer.from(bytes);
  const filename = `${Date.now()}-${file.name}`;
  const filepath = join(process.cwd(), 'public/uploads', filename);

  await writeFile(filepath, buffer);

  revalidatePath('/uploads');
  return { success: true, url: `/uploads/${filename}` };
}
```

---

## 16. API Routes 与后端逻辑

### 16.0 全栈边界设计

Route Handlers 提供标准化 Web API 端点，适用于：
- **Webhook 接收器**（Stripe、GitHub、Resend）
- **外部 API 消费者**（移动 App、第三方服务）
- **非 React 的端点**（RSS feed、sitemap、OG Image 生成）

Server Actions 是 React 生态内的 RPC，Route Handlers 是符合 Web 标准的 REST 端点。

### 16.1 Route Handlers

```typescript
// app/api/users/route.ts

import { NextRequest, NextResponse } from 'next/server';
import { prisma } from '@/lib/prisma';

export async function GET(request: NextRequest) {
  const { searchParams } = request.nextUrl;
  const page = parseInt(searchParams.get('page') || '1');
  const limit = parseInt(searchParams.get('limit') || '10');

  const users = await prisma.user.findMany({
    skip: (page - 1) * limit,
    take: limit,
    select: { id: true, name: true, email: true },
  });

  const total = await prisma.user.count();

  return NextResponse.json({
    data: users,
    pagination: { page, limit, total },
  });
}

export async function POST(request: NextRequest) {
  const body = await request.json();
  const user = await prisma.user.create({ data: body });

  return NextResponse.json(user, { status: 201 });
}
```

```typescript
// app/api/users/[id]/route.ts

export async function GET(
  request: NextRequest,
  { params }: { params: Promise<{ id: string }> }
) {
  const { id } = await params;
  const user = await prisma.user.findUnique({ where: { id } });

  if (!user) {
    return NextResponse.json({ error: 'Not Found' }, { status: 404 });
  }

  return NextResponse.json(user);
}

export async function DELETE(
  request: NextRequest,
  { params }: { params: Promise<{ id: string }> }
) {
  const { id } = await params;
  await prisma.user.delete({ where: { id } });
  return NextResponse.json({ success: true });
}
```

### 16.2 tRPC 全栈类型安全

```tsx
// server/trpc.ts
import { initTRPC, TRPCError } from '@trpc/server';
import { auth } from '@/auth';

const t = initTRPC.create();

// 中间件：添加鉴权
const isAuthed = t.middleware(async ({ next }) => {
  const session = await auth();
  if (!session) throw new TRPCError({ code: 'UNAUTHORIZED' });
  return next({ ctx: { user: session.user } });
});

export const router = t.router;
export const publicProcedure = t.procedure;
export const protectedProcedure = t.procedure.use(isAuthed);

// server/routers/user.ts
import { z } from 'zod';
import { router, publicProcedure, protectedProcedure } from '../trpc';
import { prisma } from '@/lib/prisma';

export const userRouter = router({
  getById: publicProcedure
    .input(z.object({ id: z.string() }))
    .query(async ({ input }) => {
      return prisma.user.findUnique({ where: { id: input.id } });
    }),

  updateProfile: protectedProcedure
    .input(z.object({ name: z.string().min(1) }))
    .mutation(async ({ input, ctx }) => {
      return prisma.user.update({
        where: { id: ctx.user.id },
        data: { name: input.name },
      });
    }),
});

// app/api/trpc/[trpc]/route.ts
import { fetchRequestHandler } from '@trpc/server/adapters/fetch';
import { appRouter } from '@/server/routers';

const handler = (req: Request) =>
  fetchRequestHandler({
    endpoint: '/api/trpc',
    req,
    router: appRouter,
  });

export { handler as GET, handler as POST };
```

```tsx
// Client Component 调用（完全类型安全）
'use client';

import { trpc } from '@/lib/trpc-client';

export function UserProfile({ userId }: { userId: string }) {
  const { data, isLoading } = trpc.user.getById.useQuery({ id: userId });
  const utils = trpc.useUtils();

  const updateMutation = trpc.user.updateProfile.useMutation({
    onSuccess: () => utils.user.getById.invalidate(),
  });

  return (
    <div>
      {isLoading ? <Skeleton /> : <div>{data?.name}</div>}
      <button onClick={() => updateMutation.mutate({ name: 'New Name' })}>
        Update
      </button>
    </div>
  );
}
```

### 16.3 Webhooks

```typescript
// app/api/webhooks/stripe/route.ts
import { headers } from 'next/headers';
import Stripe from 'stripe';
import { prisma } from '@/lib/prisma';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);

export async function POST(request: Request) {
  const body = await request.text();
  const signature = (await headers()).get('stripe-signature')!;

  let event: Stripe.Event;
  try {
    event = stripe.webhooks.constructEvent(
      body,
      signature,
      process.env.STRIPE_WEBHOOK_SECRET!
    );
  } catch (err) {
    return Response.json({ error: 'Invalid signature' }, { status: 400 });
  }

  switch (event.type) {
    case 'checkout.session.completed': {
      const session = event.data.object;
      await prisma.user.update({
        where: { id: session.metadata?.userId },
        data: { subscription: 'pro' },
      });
      break;
    }
    case 'customer.subscription.deleted': {
      const subscription = event.data.object;
      await prisma.user.update({
        where: { stripeCustomerId: subscription.customer as string },
        data: { subscription: 'free' },
      });
      break;
    }
  }

  return Response.json({ received: true });
}
```

---

## 17. 测试策略与工程质量

### 17.0 测试理论

测试的核心目的是"对代码正确性的信心"。从软件工程角度，测试应遵循 **测试奖杯（Testing Trophy）** 而非测试金字塔——强调**集成测试的投资回报率最高**。

```
         ╱ E2E ╲          ← 少量，覆盖关键用户流程
        ╱Integration╲      ← 大量，最高 ROI（组件 + API）
       ╱    Unit      ╲    ← 中量，纯逻辑函数
      ╱   Static       ╲   ← TypeScript / ESLint（零成本）
```

### 17.1 Vitest + React Testing Library

```tsx
// lib/utils.test.ts
import { describe, it, expect } from 'vitest';
import { cn } from './utils';

describe('cn', () => {
  it('merges class names', () => {
    expect(cn('px-4', 'py-2')).toBe('px-4 py-2');
  });

  it('handles conditionals', () => {
    expect(cn('base', false && 'hidden', 'extra')).toBe('base extra');
  });

  it('resolves Tailwind conflicts', () => {
    expect(cn('px-4', 'px-2')).toBe('px-2'); // tailwind-merge
  });
});
```

```tsx
// components/Button.test.tsx
import { describe, it, expect, vi } from 'vitest';
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { Button } from './Button';

describe('Button', () => {
  it('renders children', () => {
    render(<Button>Click me</Button>);
    expect(screen.getByText('Click me')).toBeInTheDocument();
  });

  it('calls onClick when clicked', async () => {
    const onClick = vi.fn();
    const user = userEvent.setup();

    render(<Button onClick={onClick}>Click</Button>);
    await user.click(screen.getByText('Click'));

    expect(onClick).toHaveBeenCalledTimes(1);
  });

  it('disables button when disabled prop is true', () => {
    render(<Button disabled>Disabled</Button>);
    expect(screen.getByText('Disabled')).toBeDisabled();
  });
});
```

### 17.2 Playwright E2E 测试

```typescript
// e2e/auth.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Authentication', () => {
  test('redirects to login when accessing protected page', async ({ page }) => {
    await page.goto('/dashboard');
    await expect(page).toHaveURL('/login');
  });

  test('logs in successfully', async ({ page }) => {
    await page.goto('/login');
    await page.fill('input[name="email"]', 'test@example.com');
    await page.fill('input[name="password"]', 'password123');
    await page.click('button[type="submit"]');
    await expect(page).toHaveURL('/dashboard');
    await expect(page.locator('h1')).toContainText('Welcome');
  });
});
```

### 17.3 测试配置

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';
import tsconfigPaths from 'vite-tsconfig-paths';

export default defineConfig({
  plugins: [react(), tsconfigPaths()],
  test: {
    environment: 'jsdom',
    setupFiles: ['./test/setup.ts'],
    globals: true,
  },
});
```

```typescript
// test/setup.ts
import '@testing-library/jest-dom/vitest';
```

---

## 18. 性能优化与 Core Web Vitals

### 18.0 Web 性能指标体系

Google 的 Core Web Vitals 是衡量用户体验的三个关键指标：

| 指标 | 含义 | 优秀 | 需要改进 |
|------|------|------|---------|
| **LCP** (Largest Contentful Paint) | 最大内容绘制时间 | ≤ 2.5s | ≤ 4.0s |
| **INP** (Interaction to Next Paint) | 交互响应延迟 | ≤ 200ms | ≤ 500ms |
| **CLS** (Cumulative Layout Shift) | 累积布局偏移 | ≤ 0.1 | ≤ 0.25 |

**优化 LCP 的策略链**：
1. CDN 缓存 + 静态生成（减少 TTFB）
2. `<Image />` 组件（自动 WebP、懒加载、尺寸预留）
3. 字体使用 `next/font`（零 CLS、无外部请求）
4. 关键 CSS 内联、非关键 CSS 延迟加载

### 18.1 图片优化

```tsx
import Image from 'next/image';
import heroImage from '@/public/hero.jpg';

// 本地图片：自动 width/height、模糊占位
<Image
  src={heroImage}
  alt="Hero"
  placeholder="blur"
  priority // 首屏图片加 priority
/>

// 远程图片：必须提供 width/height 或 fill
<Image
  src="https://cdn.example.com/photo.jpg"
  alt="Photo"
  width={800}
  height={600}
  // 自动：懒加载、WebP 转换、尺寸优化
/>

// 填充模式
<div className="relative h-64 w-full">
  <Image
    src="/banner.jpg"
    alt="Banner"
    fill
    className="object-cover"
    sizes="(max-width: 768px) 100vw, 1200px"
  />
</div>
```

### 18.2 字体优化

```tsx
// app/layout.tsx
import { Inter, JetBrains_Mono } from 'next/font/google';

const inter = Inter({
  subsets: ['latin'],
  display: 'swap', // FOIT 避免（先用系统字体）
  variable: '--font-inter',
});

const jetbrainsMono = JetBrains_Mono({
  subsets: ['latin'],
  variable: '--font-mono',
});

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html className={`${inter.variable} ${jetbrainsMono.variable}`}>
      <body className="font-sans">{children}</body>
    </html>
  );
}

// tailwind.config.ts 中使用
// fontFamily: { sans: ['var(--font-inter)'], mono: ['var(--font-mono)'] }
```

### 18.3 代码分割与动态导入

```tsx
// 动态导入（懒加载）
import dynamic from 'next/dynamic';

const HeavyChart = dynamic(() => import('./Chart'), {
  loading: () => <ChartSkeleton />,
  ssr: false, // 完全在客户端渲染（适合依赖 window 的库）
});

// 条件动态导入
const ReactQuill = dynamic(
  () => import('react-quill'),
  { ssr: false }
);

// 命名导出动态导入
const { ExportButton } = dynamic(() =>
  import('./ExportTools').then(mod => ({ default: mod.ExportButton }))
);
```

### 18.4 打包分析

```bash
# 分析打包产物
pnpm build

# 使用 @next/bundle-analyzer
# next.config.ts
import withBundleAnalyzer from '@next/bundle-analyzer';

const nextConfig = withBundleAnalyzer({
  enabled: process.env.ANALYZE === 'true',
})({
  // ... your config
});

# ANALYZE=true pnpm build
```

### 18.5 数据库查询优化

```tsx
// ❌ N+1 查询问题
const posts = await prisma.post.findMany();
for (const post of posts) {
  const author = await prisma.user.findUnique({ where: { id: post.authorId } });
}

// ✅ 用 include 批量加载
const posts = await prisma.post.findMany({
  include: { author: true }, // 一次查询解决
});

// ✅ 或用 select 只取需要的字段
const posts = await prisma.post.findMany({
  select: {
    id: true,
    title: true,
    author: { select: { name: true } },
  },
});

// 使用 React cache() 去重
const getUser = cache(async (id: string) => {
  return prisma.user.findUnique({ where: { id } });
});
```

---

## 19. 架构模式与项目组织

### 19.0 前端架构演进

从简单的 pages/ 目录到大型 Monorepo，前端架构的演进反映了 Web 应用复杂度的增长。

**Feature-First vs Layer-First**：
- Layer-First：`components/` `hooks/` `utils/` `types/` —— 按技术角色分层，功能代码分散，不利于删除功能
- Feature-First：`features/auth/` `features/dashboard/` —— 按业务领域分组，每个 feature 自包含

**Colocation 原则**：将相关的代码放在一起。React Server Components 加强了这一原则——数据库查询可以和展示逻辑放在同一个文件中。

### 19.1 推荐项目结构

```
src/
├── app/                           # App Router（路由 + 服务端入口）
│   ├── (marketing)/
│   │   ├── page.tsx
│   │   └── about/page.tsx
│   ├── (dashboard)/
│   │   ├── layout.tsx
│   │   ├── page.tsx
│   │   └── settings/page.tsx
│   ├── api/                       # Route Handlers
│   │   ├── auth/[...nextauth]/
│   │   └── webhooks/
│   ├── layout.tsx
│   └── globals.css
├── components/                    # 共享 UI 组件
│   ├── ui/                        # 基础组件 (Button, Input, Card)
│   └── shared/                    # 业务无关的通用组件
├── features/                      # 业务功能模块
│   ├── auth/
│   │   ├── actions.ts             # Server Actions
│   │   ├── components/            # 功能专用组件
│   │   └── schemas.ts             # Zod 验证 schema
│   ├── posts/
│   │   ├── actions.ts
│   │   ├── components/
│   │   └── queries.ts             # 数据查询函数
│   └── settings/
├── lib/                           # 工具函数与配置
│   ├── prisma.ts
│   ├── auth.ts
│   ├── redis.ts
│   └── utils.ts
├── hooks/                         # 共享自定义 Hooks
├── types/                         # 共享类型定义
├── server/                        # 仅服务端代码
│   ├── db/
│   │   ├── queries/               # 数据库查询
│   │   └── schema.ts              # Drizzle schema
│   └── services/                  # 业务服务层
└── config/                        # 配置
    └── site.ts                    # 站点级配置
```

### 19.2 分层架构

```
┌────────────────────────────────────────────┐
│  UI Layer: app/ + components/              │
│  (Server Components + Client Components)    │
├────────────────────────────────────────────┤
│  Actions Layer: features/*/actions.ts      │
│  (Server Actions — 编排业务逻辑)             │
├────────────────────────────────────────────┤
│  Service Layer: server/services/            │
│  (可复用的业务逻辑单元)                       │
├────────────────────────────────────────────┤
│  Data Layer: server/db/ + lib/prisma.ts    │
│  (数据库查询 + ORM)                         │
└────────────────────────────────────────────┘
```

### 19.3 Monorepo 管理（Turborepo）

```yaml
# turbo.json
{
  "$schema": "https://turbo.build/schema.json",
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": [".next/**", "dist/**"]
    },
    "dev": {
      "cache": false,
      "persistent": true
    },
    "lint": {},
    "test": {},
    "type-check": {}
  }
}
```

```
my-monorepo/
├── apps/
│   ├── web/              # Next.js 主应用
│   └── docs/             # 文档站
├── packages/
│   ├── ui/               # 共享 UI 组件库
│   ├── config/           # ESLint / TSConfig 共享配置
│   └── database/         # Prisma Schema + Client
├── package.json
├── turbo.json
└── pnpm-workspace.yaml
```

---

## 20. 全栈集成：tRPC、GraphQL、WebSocket

### 20.0 BFF（Backend for Frontend）模式

Next.js 天然适合作为 BFF 层——聚合多个后端服务的数据，为前端提供定制化 API。这避免了移动端 / Web 端各自维护一套数据聚合逻辑。

```
浏览器 ← HTTP/2 → Next.js BFF ← gRPC/REST → User Service
                        ↓
                      Post Service
                        ↓
                      Payment Service
```

### 20.1 GraphQL 集成

```tsx
// app/api/graphql/route.ts
import { createYoga } from 'graphql-yoga';
import { makeExecutableSchema } from '@graphql-tools/schema';

const typeDefs = `
  type User {
    id: ID!
    name: String!
    email: String!
    posts: [Post!]!
  }

  type Post {
    id: ID!
    title: String!
    content: String
    author: User!
  }

  type Query {
    users: [User!]!
    user(id: ID!): User
    posts: [Post!]!
  }

  type Mutation {
    createPost(title: String!, content: String): Post!
  }
`;

const resolvers = {
  Query: {
    users: () => prisma.user.findMany(),
    user: (_, { id }) => prisma.user.findUnique({ where: { id } }),
    posts: () => prisma.post.findMany({ include: { author: true } }),
  },
  User: {
    posts: (parent) => prisma.post.findMany({ where: { authorId: parent.id } }),
  },
  Mutation: {
    createPost: (_, { title, content }) =>
      prisma.post.create({ data: { title, content, authorId: '...' } }),
  },
};

const schema = makeExecutableSchema({ typeDefs, resolvers });
const yoga = createYoga({ schema });

export { yoga as GET, yoga as POST };
```

### 20.2 WebSocket 实时通信

```typescript
// server/ws-server.ts (运行在独立的 Node.js 进程中)
import { WebSocketServer } from 'ws';
import { parse } from 'cookie';
import { verifyToken } from '@/lib/jwt';

const wss = new WebSocketServer({ port: 3001 });

wss.on('connection', (ws, req) => {
  const cookies = parse(req.headers.cookie || '');
  const token = cookies['auth-token'];

  try {
    const user = verifyToken(token);
    ws.send(JSON.stringify({ type: 'welcome', userId: user.id }));
  } catch {
    ws.close(1008, 'Unauthorized');
    return;
  }

  ws.on('message', (data) => {
    const message = JSON.parse(data.toString());
    // 广播给所有客户端
    wss.clients.forEach(client => {
      if (client !== ws && client.readyState === WebSocket.OPEN) {
        client.send(JSON.stringify({ type: 'message', ...message }));
      }
    });
  });

  ws.on('close', () => {
    console.log('Client disconnected');
  });
});
```

```tsx
// Client Side WebSocket Hook
'use client';

import { useEffect, useRef, useState } from 'react';

export function useWebSocket(url: string) {
  const wsRef = useRef<WebSocket | null>(null);
  const [lastMessage, setLastMessage] = useState<any>(null);

  useEffect(() => {
    const ws = new WebSocket(url);
    wsRef.current = ws;

    ws.onmessage = (event) => {
      setLastMessage(JSON.parse(event.data));
    };

    ws.onclose = () => {
      // 自动重连
      setTimeout(() => {
        wsRef.current = new WebSocket(url);
      }, 3000);
    };

    return () => ws.close();
  }, [url]);

  const sendMessage = (data: unknown) => {
    wsRef.current?.send(JSON.stringify(data));
  };

  return { lastMessage, sendMessage };
}
```

---

## 21. CI/CD 与部署

### 21.0 DevOps 持续交付理论

CI/CD 的目的是**缩短从代码提交到生产部署的反馈循环**。

**DORA 指标**（衡量团队 DevOps 成熟度）：
- 部署频率（DF）：精英级 = 每天多次
- 变更前置时间（LT）：精英级 < 1 小时
- 变更失败率（CFR）：精英级 0-15%
- 故障恢复时间（MTTR）：精英级 < 1 小时

### 21.1 Vercel 部署（推荐）

```bash
# Git 推送自动部署
git push origin main

# Vercel CLI
npx vercel           # 预览部署
npx vercel --prod    # 生产部署

# 环境变量
npx vercel env add DATABASE_URL
```

### 21.2 Docker 部署

```dockerfile
# Dockerfile
FROM node:22-alpine AS base
RUN corepack enable && corepack prepare pnpm@latest --activate

FROM base AS deps
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN pnpm install --frozen-lockfile

FROM base AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN pnpm build

FROM base AS runner
WORKDIR /app
ENV NODE_ENV=production
COPY --from=builder /app/public ./public
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static

EXPOSE 3000
CMD ["node", "server.js"]
```

```yaml
# docker-compose.yml
services:
  app:
    build: .
    ports:
      - '3000:3000'
    environment:
      - DATABASE_URL=postgresql://user:pass@db:5432/mydb
    depends_on:
      - db

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: mydb
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

### 21.3 GitHub Actions

```yaml
# .github/workflows/ci.yml
name: CI
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: 'pnpm'

      - run: pnpm install --frozen-lockfile
      - run: pnpm type-check
      - run: pnpm lint
      - run: pnpm test
      - run: |
          npx prisma generate
          npx prisma db push --skip-generate
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}

  e2e:
    needs: check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: 'pnpm'

      - run: pnpm install --frozen-lockfile
      - run: npx playwright install --with-deps
      - run: pnpm build
      - run: pnpm e2e
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
```

---

## 22. 进阶专题与生态扩展

### 22.1 国际化（i18n）

```typescript
// lib/i18n.ts (next-intl)
// messages/en.json
{
  "HomePage": {
    "title": "Welcome",
    "description": "This is the home page"
  }
}
// messages/zh.json
{
  "HomePage": {
    "title": "欢迎",
    "description": "这是首页"
  }
}

// app/[locale]/layout.tsx
```

### 22.2 错误监控（Sentry）

```tsx
// sentry.client.config.ts
Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
  tracesSampleRate: 0.1,
  replaysSessionSampleRate: 0.1,
  replaysOnErrorSampleRate: 1.0,
});

// next.config.ts 中使用 @sentry/nextjs 插件
```

### 22.3 PWA 支持

```tsx
// app/manifest.ts
import type { MetadataRoute } from 'next';

export default function manifest(): MetadataRoute.Manifest {
  return {
    name: 'My Next.js App',
    short_name: 'MyApp',
    start_url: '/',
    display: 'standalone',
    background_color: '#ffffff',
    theme_color: '#000000',
    icons: [
      { src: '/icon-192.png', sizes: '192x192', type: 'image/png' },
      { src: '/icon-512.png', sizes: '512x512', type: 'image/png' },
    ],
  };
}
```

### 22.4 分析（Analytics）

```tsx
// 使用 Vercel Analytics (零配置)
import { Analytics } from '@vercel/analytics/react';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html>
      <body>
        {children}
        <Analytics />
      </body>
    </html>
  );
}
```

### 22.5 Feature Flags

```tsx
// 使用 Vercel Flags 或自建
import { unstable_flag as flag } from '@vercel/flags/next';

const newDashboard = flag({
  key: 'new-dashboard',
  decide: () => Math.random() > 0.5, // 50% 流量
});

export default async function Dashboard() {
  const showNewUI = await newDashboard();

  if (showNewUI) {
    return <DashboardV2 />;
  }
  return <DashboardV1 />;
}
```

---

## 23. 实战项目路线图

### 23.0 螺旋式学习理论

基于认知负荷理论（Cognitive Load Theory），工作记忆只能同时处理约 4 个信息块。分阶段学习——每阶段只引入 1-2 个新概念：

### 23.1 阶段一：静态网站（第 1 周）

**项目：个人博客**
- 技术：App Router + Tailwind CSS + MDX
- 概念：layout/page、SSG、ISR、动态路由
- 功能：文章列表、分类、暗色模式
- 你的迁移优势：Flutter 路由 → App Router；ObjC MVC → 组件拆分

### 23.2 阶段二：CRUD 应用（第 2-3 周）

**项目：任务管理应用**
- 技术：Prisma + PostgreSQL + Server Actions + React Hook Form + Zod
- 概念：数据库 CRUD、表单验证、Server Actions、缓存失效
- 功能：任务增删改查、优先级、截止日期、搜索过滤
- 你的迁移优势：数据库操作 ≈ 文件 I/O 思维

### 23.3 阶段三：全栈 SaaS（第 4-6 周）

**项目：团队仪表盘**
- 技术：NextAuth.js + Prisma + tRPC + TanStack Query + Zustand
- 概念：OAuth 鉴权、RBAC、乐观更新、实时数据
- 功能：团队管理、数据可视化、文件上传、通知
- 你的迁移优势：权限控制 ≈ C++ 访问控制；tRPC ≈ FFI 类型安全

### 23.4 阶段四：电商应用（第 7-10 周）

**项目：在线商店**
- 技术：完整技术栈 + 支付集成 + Redis 缓存
- 概念：购物车状态、Stripe 支付、库存管理、搜索优化
- 功能：商品浏览、购物车、结算、订单管理
- 你的迁移优势：复杂状态管理 ≈ C++ 状态机设计

### 23.5 阶段五：实时协作（第 11-14 周）

**项目：协作文档编辑器**
- 技术：WebSocket + CRDT / Yjs + TipTap + 自定义 Server
- 概念：实时同步、并发编辑、OT/CRDT
- 你的迁移优势：并发控制 ≈ C++ 多线程/锁机制；CRDT ≈ 分布式一致性

### 23.6 学习资源推荐

| 资源 | 类型 | 适合阶段 |
|------|------|---------|
| Next.js 官方文档 | 文档 | 全阶段 |
| Next.js Learn | 交互教程 | 阶段一/二 |
| React 官方文档（beta） | 文档 | 阶段一/二 |
| Josh Comeau CSS 课程 | 课程 | 阶段一/二 |
| Prisma 文档 | 文档 | 阶段二+ |
| Vercel Templates | 模板 | 阶段三+ |
| Lee Robinson (Vercel VP) 博客 | 博客 | 阶段四+ |
| Build UI | 动画技巧 | 阶段三+ |
| TypeScript 手册 | 文档 | 全阶段 |

---

## 24. 附录：C++/ObjC/Flutter → Next.js 速查表

### 24.0 跨语言知识迁移的科学

从认知科学角度，你已掌握的 C++/ObjC/Flutter 概念大部分可以 1:1 映射。只有少数核心范式需要重新建立神经通路。

### 24.1 语法对比

| 特性 | C++ | ObjC | Flutter (Dart) | Next.js (TS/React) |
|------|-----|------|----------------|-------------------|
| 变量 | `int x = 5;` | `NSInteger x = 5;` | `int x = 5;` | `let x = 5;` / `const x = 5;` |
| 常量 | `const int x = 5;` | — | `const x = 5;` | `const x = 5;` (不可变引用) |
| 函数 | `int add(int a, int b)` | `- (int)add:(int)a` | `int add(int a, int b)` | `function add(a: number, b: number): number` |
| 字符串 | `std::string s = "hi";` | `@"hi"` | `'hi'` | `'hi'` / `"hi"` / `` `hi ${name}` `` |
| 数组 | `std::vector<int>{1,2}` | `@[@1,@2]` | `[1, 2]` | `[1, 2]` |
| 字典 | `std::map<string,int>` | `@{@"a":@1}` | `{'a': 1}` | `{ a: 1 }` / `Record<string, number>` |
| 类 | `class Foo {};` | `@interface Foo` | `class Foo {}` | `class Foo {}` / `interface Foo {}` |
| 继承 | `class B : public A` | `@interface B : A` | `class B extends A` | `class B extends A` / `extends` |
| 泛型 | `template<typename T>` | 轻量泛型 | `class Box<T>` | `<T>` / `function foo<T>(x: T)` |
| Lambda | `[&](int x){ return x; }` | `^(int x){ return x; }` | `(int x) => x` | `(x: number) => x` / `x => x` |
| async | C++20 coroutine | GCD completion | `async/await` | `async/await` |
| 空安全 | `std::optional<T>` | `nil` 消息静默 | `T?` (NNBD) | `T | null` / `T | undefined` |

### 24.2 架构概念映射

| 概念 | Flutter | Next.js (React) |
|------|---------|-----------------|
| 声明式 UI | `Widget build(BuildContext)` | `function Component(props)` |
| 有状态组件 | `StatefulWidget` + `setState` | `useState` / `useReducer` |
| 依赖注入 | `InheritedWidget` / Riverpod | Context API / Zustand |
| 列表 | `ListView.builder` | `array.map(item => <Item key={item.id} />)` |
| 导航 | GoRouter | App Router (文件系统) |
| 路由参数 | `path: '/user/:id'` | `app/user/[id]/page.tsx` |
| 布局 | `Row` / `Column` / `Stack` | Flexbox / Grid / `absolute` |
| 动画 | `AnimationController` / `AnimatedContainer` | CSS Transitions / Framer Motion |
| 数据获取 | `FutureBuilder` / Riverpod | `async` Server Component / TanStack Query |
| 本地存储 | `SharedPreferences` / Isar | `localStorage` / IndexedDB |

### 24.3 开发流程对比

| 活动 | Flutter | Next.js |
|------|---------|---------|
| 新建项目 | `flutter create` | `pnpm create next-app` |
| 依赖管理 | `pubspec.yaml` + `flutter pub get` | `package.json` + `pnpm install` |
| 运行 | `flutter run` | `pnpm dev` |
| 热重载 | `r` (Hot Reload) | 自动 Fast Refresh |
| 调试 | Dart DevTools | Chrome DevTools / React DevTools |
| 测试 | `flutter test` | `pnpm test` (Vitest) |
| 构建 | `flutter build` | `pnpm build` |
| Lint | `flutter analyze` | `pnpm lint` |

---

## 25. 大企业面试题精编

### 25.0 面试策略

Next.js 面试通常分为三个层次：
1. **基础层**（15-20%）：React Hooks、TypeScript、App Router
2. **架构层**（40-50%）：RSC、渲染策略、状态管理、性能优化
3. **深度层**（30-40%）：RSC Payload 协议、Streaming 原理、缓存层级

你的 C++/ObjC/Flutter 背景在深度层是巨大优势。

### 25.1 React 与 Hooks

#### Q1：详细解释 `useEffect` 的执行时机。它和 `useLayoutEffect` 的区别？

**考察点**：渲染管线理解

**回答要点**：
- `useEffect`：DOM 更新 + 浏览器绘制**之后**异步执行。不阻塞渲染。
- `useLayoutEffect`：DOM 更新后、浏览器绘制**之前**同步执行。阻塞渲染。
- 为什么需要两个？`useLayoutEffect` 适合需要同步测量 DOM 并修改以避免闪烁的场景（如滚动位置恢复、Tooltip 定位）。
- 类比：`useEffect` = `dispatch_async`（异步），`useLayoutEffect` = `dispatch_sync`（同步）。

#### Q2：React 的 Fiber 架构解决了什么问题？

**考察点**：React 内部机制

**回答要点**：
- 在 Fiber 之前，React 使用 Stack Reconciler——同步递归遍历 Virtual DOM，无法中断。大型组件树的更新会阻塞主线程，导致掉帧。
- Fiber 将渲染工作分解为小的工作单元（Fiber Node），每个单元执行完后检查是否有更高优先级的任务（用户输入、动画）。
- 可中断渲染使得 Concurrent Features（Suspense、Transitions）成为可能。
- 类比：Fiber 类似操作系统的抢占式调度——高优先级任务可以打断低优先级任务。

### 25.2 Next.js 核心

#### Q3：RSC Payload 的格式是什么？它和 HTML 有什么区别？

**考察点**：对 RSC 协议的深入理解

**回答要点**：
- RSC Payload 是一种特殊的序列化格式（不是 HTML、不是 JSON），包含：
  - `M` 行：模块引用（Client Component 的 chunk ID）
  - `J` 行：序列化的 React Element 树（类似 JSON 但支持更多类型）
  - `S` 行：Suspense 边界标记
- HTML 是给浏览器看的（展示），RSC Payload 是给 React 看的（用于构建 React 树）。
- 客户端拿到 RSC Payload 后，React 将其与 Client Components 合并，在内存中构建完整的 React 树。

#### Q4：解释 Next.js 的四级缓存体系。`revalidatePath` 和 `revalidateTag` 的区别？

**考察点**：缓存策略

**回答要点**：
- 四级缓存：CDN 缓存 → 全路由缓存（Full Route Cache）→ 数据缓存（Data Cache）→ 请求去重
- `revalidatePath('/posts')` 清除指定路径的**全路由缓存**（下次请求重新渲染整个页面）
- `revalidateTag('posts')` 清除所有带 `posts` 标签的**数据缓存**（fetch 和 `unstable_cache`）
- 关键区别：`revalidatePath` 是按路由失效，`revalidateTag` 是按数据标签失效。后者更精确。

#### Q5：Server Actions 的底层实现原理是什么？它如何支持无 JS 降级？

**考察点**：Server Actions 机制

**回答要点**：
- Server Actions 本质是自动生成的 POST API 端点。
- 有 JS 时：`<form action={serverAction}>`，React 拦截 submit 事件，通过 fetch POST 调用。
- 无 JS 时：浏览器原生 form POST 到同一个端点，Next.js 解析后重定向回原页面（或指定 redirect）。
- 每个 Server Action 在构建时获得一个唯一的 Action ID，用于路由到正确的函数。
- 类比：类似于 Flutter 的 Platform Channel——通过 ID 路由调用。

### 25.3 性能与架构

#### Q6：一个页面有 5 个独立的数据源（每个查询耗时 200ms），如何优化到最短的首屏时间？

**考察点**：Streaming + 并行数据获取

**回答要点**：
1. 在 Server Component 中用 `Promise.all` 并行发起所有 5 个请求（200ms 而非 1000ms）
2. 用 `<Suspense>` 包裹每个数据区域，设置独立的骨架屏
3. 关键数据（Above the Fold）不包 Suspense 或设置 `priority`
4. 非关键数据用 Suspense 包裹，流式加载
5. 最终效果：200ms 首字节（并行）+ 静态外壳立即显示 + 各数据区域独立流式注入

#### Q7：如何设计一个大型 Next.js 项目的目录结构？

**考察点**：架构设计能力

**回答要点**：
- 采用 Feature-First 组织：`features/auth/`、`features/posts/`、`features/settings/`
- Colocation 原则：Server Actions 和组件放在同一个 feature 下
- `components/ui/` 放基础组件，`components/shared/` 放业务无关通用组件
- `lib/` 放工具函数和基础设施（prisma、auth、redis）
- `server/` 放仅服务端代码（查询函数、业务服务层）
- 每个 feature 自包含：actions、components、schemas、queries

#### Q8：100 万行级别的 Next.js 项目，如何保证代码质量和构建速度？

**考察点**：大型项目工程化

**回答要点**：
- 构建速度：Turbopack（开发）+ Turborepo（Monorepo 缓存）+ 增量构建
- 代码质量：ESLint + Prettier + TypeScript strict + Husky pre-commit
- 类型检查：`tsc --noEmit` 在 CI 中运行，不在开发期阻塞
- Monorepo 拆分：按功能域分成独立 package，独立构建和测试
- CI 缓存：缓存 `.next/cache`、`node_modules`、Turborepo 缓存
- 模块边界：用 ESLint `import/no-restricted-paths` 强制依赖方向

### 25.4 综合设计题

#### Q9：设计一个支持离线模式的博客应用架构。

**考察点**：全栈思维

**回答要点**：
- 构建时 SSG 所有页面（静态 = 天然离线友好）
- Service Worker 缓存策略：Cache-First（静态资源）+ Network-First（动态内容）
- IndexedDB 存储阅读历史和草稿（客户端）
- 后台同步：用户离线点赞/评论存 IndexedDB，联网后批量提交
- ISR 定时刷新缓存内容

#### Q10：如何为一个已有的 Express 后端项目逐步引入 Next.js？

**考察点**：渐进式迁移

**回答要点**：
- **Strangler Fig Pattern**：不重写，逐步替换
- 阶段 1：将 Express API 保留，Next.js 只做前端渲染层（BFF 模式）
- 阶段 2：将 Express 路由逐个迁移到 Next.js Route Handlers / Server Actions
- 阶段 3：将数据库查询从 Express 迁移到 Next.js Server Components
- 使用 `next.config.ts` 的 `rewrites` 将旧 API 路径代理到 Express 服务
- 在迁移期间，Express 和 Next.js 共存

### 25.5 高频快问快答

| 问题 | 一句话答案 |
|------|----------|
| SSG vs SSR vs ISR | SSG 构建时生成，SSR 请求时生成，ISR SSG+定时/按需刷新 |
| `'use client'` 意味着？ | 组件也在客户端渲染（服务端预渲染 + 客户端 hydration） |
| Server Action vs API Route | SA 是 React 生态内的 RPC；Route Handler 是标准 HTTP 端点 |
| `useMemo` vs `React.memo` | useMemo 缓存值，React.memo 缓存组件（Props 不变则跳过渲染） |
| `revalidate` vs `dynamic` | revalidate 控制缓存时长，dynamic 控制渲染模式 |
| `layout.tsx` vs `template.tsx` | layout 导航时保持挂载；template 每次都重新挂载 |
| Turbopack 是什么？ | Rust 编写的增量打包器，替代 Webpack，开发速度 10x+ |
| `next/image` vs `<img>` | Image 组件自动 WebP、懒加载、模糊占位、CLS 防护 |
| Middleware 运行在哪？ | Edge Runtime（受限 V8 环境，不能用 Node.js API） |
| Next.js 可以不用 Vercel 吗？ | 可以，支持 Docker、Node.js Server、任何支持 Node.js 的平台 |
| `cache()` vs `unstable_cache` | cache 去重同一请求内的调用；unstable_cache 跨请求持久化缓存 |
| 如何在 RSC 中使用 Context？ | 不能。RSC 没有 React Context。改用 Props drilling 或在 Client Component 包裹 |
| 为什么 `params` 是 Promise？ | Next.js 15+ 的异步 API，支持 PPR 等需要动态获取路由参数的功能 |

---

## 写在最后

从 C++/ObjC/Flutter 转到 Next.js，你不是从零开始——你带来的系统编程能力、内存管理直觉、跨平台架构思维，是绝大多数前端开发者不具备的深度优势。

**记住四件事**：
1. **TypeScript 只是语法，你的编程思维不变**——数据结构和算法、设计模式、架构原则完全通用
2. **你的 C++ 编译期思维让你天然理解 SSG/ISR**——构建时求值、增量更新，与 `constexpr` 一脉相承
3. **你的 ObjC MVC 经验让你快速掌握组件拆分**——UIViewController ≈ 函数组件 + Custom Hooks
4. **你的 Flutter 声明式 UI 经验直接映射到 React**——Widget tree ≈ JSX，`setState` ≈ `useState`

**行动建议**：从今天开始，用 Next.js 重写你之前用 Flutter 做过的简单 App。两周后你会惊讶于自己的进步。

> *"Next.js is not about learning web development from scratch — it's about extending what you already know to a new platform."*

---

*本指南持续更新中。如有问题或建议，欢迎提交 Issue。*

