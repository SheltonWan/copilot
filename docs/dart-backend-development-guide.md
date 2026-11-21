# Dart 后端开发知识全景指南

| 项目 | 说明 |
|---|---|
| **目标读者** | 掌握或正在学习 Dart、希望使用 Dart 构建 API、服务端应用或云函数的开发者 |
| **学习目标** | 建立从语言基础到生产部署的完整后端知识体系，并给出循序渐进的学习路线 |
| **适用范围** | HTTP API、业务服务、微服务、实时服务、后台任务和 Flutter 配套后端 |
| **更新日期** | 2026 年 11 月 |

## 目录

1. [先建立正确的后端心智模型](#1-先建立正确的后端心智模型)
2. [Dart 服务端运行环境与工具链](#2-dart-服务端运行环境与工具链)
3. [后端必需的 Dart 语言能力](#3-后端必需的-dart-语言能力)
4. [异步、并发与运行时](#4-异步并发与运行时)
5. [HTTP 与网络基础](#5-http-与网络基础)
6. [Web 框架与服务端分层](#6-web-框架与服务端分层)
7. [API 设计与契约](#7-api-设计与契约)
8. [输入校验、序列化与错误处理](#8-输入校验序列化与错误处理)
9. [数据库、SQL 与数据建模](#9-数据库sql-与数据建模)
10. [缓存、搜索与文件存储](#10-缓存搜索与文件存储)
11. [身份认证、授权与应用安全](#11-身份认证授权与应用安全)
12. [架构、领域建模与代码组织](#12-架构领域建模与代码组织)
13. [微服务架构与落地](#13-微服务架构与落地)
14. [分布式系统专题](#14-分布式系统专题)
15. [消息队列、实时通信与后台任务](#15-消息队列实时通信与后台任务)
16. [测试策略与质量保障](#16-测试策略与质量保障)
17. [性能、可靠性与可观测性](#17-性能可靠性与可观测性)
18. [部署、云平台与交付流程](#18-部署云平台与交付流程)
19. [后端开发协作与日常工程实践](#19-后端开发协作与日常工程实践)
20. [学习路线、实战与面试](#20-学习路线实战与面试)
21. [上线前检查清单](#21-上线前检查清单)
22. [常见误区](#22-常见误区)
23. [官方资料与延伸阅读](#23-官方资料与延伸阅读)

---

## 1. 先建立正确的后端心智模型

后端不是“写几个接口”或“把数据库增删改查包一层”。后端工程的核心命题是：**在不可靠的网络、会失败的依赖、会重启的进程和不断变化的业务规则之上，持续地给出正确、可解释、可运维的结果。**

### 1.1 一个服务的三段生命周期

把服务想成一个长期运行的“生物”，它有三个阶段，每个阶段都有独立的工程要求：

```text
[启动阶段]                     [运行阶段]                     [关闭阶段]
读取配置                      接收连接                       停止接收新请求
校验依赖                    解析请求                       等待在途请求到截止时间
预加载必要资源              认证与授权                     停止消费者拉取
建立连接池                  执行业务用例                   冲刷缓冲区/提交或回滚事务
注册信号处理器              访问依赖                       关闭连接池、HTTP 客户端
通过健康检查                返回响应                       上报最终指标
                            记录日志/指标/追踪              释放文件、订阅与 isolate
```

**启动阶段最常见的生产事故**：配置缺失却用默认值静默启动、启动时连接数据库失败后无限重试导致一直不 ready、启动即做全量数据加载导致扩容缓慢。

**运行阶段最常见的生产事故**：无超时的外部调用拖垮全部并发、单请求内存暴涨、慢查询把所有连接池占满、日志里写了令牌。

**关闭阶段最常见的生产事故**：滚动发布时直接杀死进程，正在处理中的写请求被中断，客户端看到大量 502；消费者被强杀导致任务处于半完成状态。

### 1.2 一次请求的完整旅程

不要只看见 `handler`。一个真实请求会穿过十几层边界，每一层都可能失败：

| # | 阶段 | 典型动作 | 失败表现 | 谁负责 |
|---|---|---|---|---|
| 1 | DNS / TCP / TLS | 连接建立、证书校验 | 连接超时、证书错误 | 平台 / 网关 |
| 2 | 反向代理 | 路由、TLS 终止、限流 | 413、429、502 | 网关 |
| 3 | 请求读取 | 限制 body 大小、解析头部 | 400、413 | 服务器层 |
| 4 | 请求上下文 | 生成 requestId、记录开始时间 | — | 中间件 |
| 5 | 安全中间件 | CORS、安全响应头、body 限制 | 403、431 | 中间件 |
| 6 | 认证 | 校验令牌/会话、构建 Principal | 401 | 认证中间件 |
| 7 | 路由 | 方法 + 路径匹配、提取路径参数 | 404、405 | 路由器 |
| 8 | 解析与校验 | JSON 解析、字段校验、类型转换 | 400、422 | DTO + 校验器 |
| 9 | 授权 | 资源归属、角色、租户校验 | 403、404（隐藏存在性） | 用例 / 策略层 |
| 10 | 用例执行 | 业务规则、状态机、事务编排 | 409、422 | 应用服务 |
| 11 | 数据访问 | SQL、索引、事务、缓存 | 500、503、超时 | 仓储层 |
| 12 | 响应映射 | DTO 转换、序列化、状态码 | 500 | 控制器 |
| 13 | 观测 | 日志、指标、追踪、审计 | — | 中间件 |
| 14 | 连接归还 | 连接复用、响应冲刷 | — | 服务器层 |

一个成熟的后端工程师能在脑内按这张表定位问题：“用户说偶发 500，且只在某个接口” → 先看第 11 层（超时/连接池）与第 10 层（并发竞态），再看第 12 层（序列化失败），而不是先怀疑框架。

对应的骨架代码（Shelf 风格，后面章节会展开细节）：

```dart
Future<Response> handleRequest(Request request, AppDeps deps) async {
  final requestId = request.headers['x-request-id'] ?? newRequestId();
  final stopwatch = Stopwatch()..start();
  try {
    final principal = await deps.auth.authenticate(request);   // 6 认证
    final route = deps.router.match(request);                  // 7 路由
    final input = CreateOrderInput.fromJson(route.body);       // 8 解析
    validateCreateOrder(input);                                // 8 校验
    final result = await deps.createOrder.execute(             // 9-11 用例
      principal,
      input,
      requestId: requestId,
    );
    return Response.ok(jsonEncode(result.toJson()));           // 12 响应
  } on ValidationException catch (e) {
    return problemResponse(400, 'invalid_request', e.message);
  } on DomainConflict catch (e) {
    return problemResponse(409, e.code, e.message);
  } catch (e, st) {
    deps.logger.error('unhandled', error: e, stackTrace: st, requestId: requestId);
    return problemResponse(500, 'internal_error', '服务暂时不可用');
  } finally {
    deps.metrics.observeLatency(route.name, stopwatch.elapsed);  // 13 观测
  }
}
```

### 1.3 必须内化的非功能需求

功能正确只是及格线。生产系统靠以下指标说话：

**可用性（Availability）**

| 年度可用性 | 年停机时间 | 适用场景 |
|---|---|---|
| 99% | 3.65 天 | 内部工具 |
| 99.9% | 8.77 小时 | 一般商业服务 |
| 99.95% | 4.38 小时 | 有付费 SLA 的核心业务 |
| 99.99% | 52.6 分钟 | 支付、通信基础设施 |

注意：可用性是端到端概念。你的服务 99.99%，但依赖的数据库 99.9%，端到端上限仍受最弱一环（及串并联关系）约束。

**延迟**

- 区分 P50 / P95 / P99 / P999。平均值会掩盖长尾，用户感知的是 P99。
- 建立延迟预算：网关 5ms + 认证 2ms + 业务 20ms + 数据库 30ms + 序列化 3ms = 60ms（P95）。
- 每个外部调用都要有超时，且超时之和不能超过上游给本服务的时间预算。

**吞吐**：QPS 与单请求资源成本决定容量。要看的是“DB 连接数 × 每连接承载 QPS”这类约束，而不是只压测出一个漂亮数字。

**一致性**：强一致、读己之写、最终一致是不同承诺。缓存、只读副本、异步事件都会削弱一致性，必须显式设计并告知调用方。

**成本**：CPU、内存、出口流量、数据库 IOPS、日志存储都是钱。序列化开销、日志量、指标标签基数会直接体现在账单上。

**安全与隐私**：机密性、完整性、可用性、可审计性。任何“先上线再补安全”的计划都应视为高风险。

### 1.4 Dart 作为后端语言的客观评估

**优势**

| 维度 | 说明 |
|---|---|
| 单一语言栈 | Flutter 客户端与后端共享 DTO、校验规则、错误码枚举，减少契约漂移 |
| 异步模型清晰 | `Future`/`Stream` + event loop，天然适配 I/O 密集服务 |
| AOT 原生编译 | `dart compile exe` 产出无需 VM 的原生二进制，冷启动优于 JIT 方案 |
| 静态类型 + 空安全 | 编译期拦截大量空值与类型错误，重构安全 |
| isolate 模型 | 无共享内存，避免数据竞争；消息传递语义明确 |
| 开发体验 | 热重载（Flutter）、`dart analyze` 快速、格式化统一 |

**短板（必须正视）**

| 维度 | 说明 | 缓解方式 |
|---|---|---|
| 服务端生态较小 | 相比 Java/Go/Node，ORM、消息队列客户端、可观测 SDK 的成熟度与选择面更窄 | 优先选用维护活跃的包，必要时用 HTTP/gRPC 连接成熟中间件 |
| 招聘市场 | 后端 Dart 人才少于主流后端语言 | 团队已有 Flutter 背景时优势明显；否则需评估 |
| 部分中间件缺少官方 SDK | 例如某些云厂商服务、特定数据库 | 走标准协议（PostgreSQL 协议、AMQP、S3 API、OTel OTLP） |
| CPU 密集场景 | 单 isolate 单线程，CPU 密集需要 isolate 或独立服务 | 拆分工作进程、使用原生扩展 |
| 反射能力受限 | 无通用运行期反射，序列化多依赖代码生成 | 接受生成步骤，纳入 CI |
| 平台运行时支持 | Serverless 平台对 Dart 运行时支持不一致 | 优先容器部署或 AOT 二进制 |

**结论**：Dart 后端在“Flutter 全栈 + I/O 密集型业务 API”这个场景下性价比很高；在“需要大量现成企业中间件 SDK、超大团队、强招聘供给”的场景下需要更谨慎的论证。技术选型要用**团队能力、生态匹配度、运维成本**三把尺子，而不是语言偏好。

### 1.5 选型决策框架

```text
团队已用 Flutter/Dart？
├─ 是 → 是否需要类型安全的客户端-服务端一体开发？
│        ├─ 是 → Serverpod（或自建 Shelf + 生成客户端）
│        └─ 否 → REST/微服务 → Dart Frog 或 Shelf
└─ 否 → 后端人才以 Java/Go/Node 为主？
         ├─ 是 → 优先主流后端语言；Dart 仅用于特定服务
         └─ 否 → 评估 Dart 生态是否覆盖你的关键依赖（DB/队列/可观测）
```

再叠加约束：部署形态（容器 / serverless / 边缘）、合规要求（数据驻留、审计）、性能目标、预算、上线时间。把结论写成 ADR（见 12.6），半年后回看时你会感谢自己。

### 1.6 “全覆盖”的边界与复杂度预算

本指南覆盖现代业务后端的所有主干能力，但**能力清单 ≠ 实施清单**。给出一个复杂度预算参考：

| 阶段 | 团队规模 | 建议具备 | 明确暂缓 |
|---|---|---|---|
| MVP | 1–3 人 | HTTP + 关系库 + 认证 + 日志 + 单机/托管部署 | 微服务、消息队列、CQRS、多区域 |
| 成长期 | 3–15 人 | 队列、缓存、CI/CD、指标与告警、备份演练 | 服务网格、事件溯源、自建 K8s |
| 规模化 | 15+ 人 | SLO 体系、容量规划、故障演练、数据分区 | 无明确收益的重写 |

**原则**：每一个新增组件都要能回答三个问题——它解决什么具体故障或瓶颈？它的运维成本是多少？没有它时业务会怎样？答不上来就先不加。

---

## 2. Dart 服务端运行环境与工具链

### 2.1 SDK 安装与版本管理

生产项目的第一条纪律是**固定 SDK 版本**。Dart 的次要版本会引入新语言特性和 lint 规则，团队与 CI 必须一致。

```yaml
# pubspec.yaml
name: order_service
description: Order and inventory API.
publish_to: none
version: 1.4.0

environment:
  sdk: '>=3.5.0 <4.0.0'   # 下界=实际用到的特性，上界=尚未验证的大版本
```

常见安装与管理方式：

- **官方安装包 / Homebrew**：适合本机快速开始。
- **版本管理器（fvm / asdf / mise）**：在仓库中放 `.fvmrc` 或 `.tool-versions`，让所有人 `fvm dart` 使用同一版本。
- **容器内固定镜像**：CI 与生产构建镜像锁定 Dart 版本标签，避免“昨天还能构建”。
- **CI 缓存**：缓存 pub 全局缓存目录可显著加快 `dart pub get`，但缓存键要包含 `pubspec.lock` 的哈希。

```bash
# 团队约定：统一通过 fvm 调用
fvm install
fvm dart --version
fvm dart pub get
fvm dart analyze
```

**SDK 升级流程**：新建分支 → 提升约束 → `dart pub upgrade` → 全量测试 → 修复 lint/弃用告警 → 预发布验证 → 合并。不要在生产热修分支里顺手升 SDK。

### 2.2 Pub 依赖解析机制

理解 pub 的行为能避免大量“本地能跑、CI 挂掉”的问题。

**版本求解**：`dart pub get` 读取 `pubspec.yaml` 的约束，在满足所有约束的组合中求解，结果写入 `pubspec.lock`。如果两个依赖对同一个包要求不兼容的版本区间，求解失败——这时要用 `dependency_overrides` 临时排查，但**不应长期保留**。

**锁文件策略**：

- 应用（application package）：**必须提交** `pubspec.lock`，保证开发/CI/构建一致。
- 库（被引用的 package）：**不要提交**锁文件，让下游自由求解。

**依赖覆盖（仅限临时）**：

```yaml
dependency_overrides:
  http: ^1.2.0   # 只为排查冲突；合并前删除，并在 PR 中说明原因
```

**路径依赖与 monorepo**：多包项目可用 `path:` 依赖本地包；Dart 3.6+ 支持 pub workspace，可在一个工作区内共享解析结果与锁文件。

```yaml
# 根 pubspec.yaml
name: _workspace
publish_to: none
environment:
  sdk: '>=3.6.0 <4.0.0'
workspace:
  - packages/shared_models
  - packages/order_service
  - packages/api_client
```

**依赖健康度检查清单**（引入任何包前）：

1. 最近 12 个月是否有发布？issue 是否有维护者响应？
2. 是否支持当前 SDK 约束？是否需要额外原生依赖？
3. 许可证是否与公司政策兼容？
4. 依赖树是否引入了大量传递依赖？
5. 是否有测试、CI 徽章、公开的安全公告渠道？
6. 是否可用更小的标准库能力替代？

### 2.3 命令与文件速查

| 命令 | 用途 | 生产注意 |
|---|---|---|
| `dart create -t console myapp` | 创建控制台/服务端项目 | 也可用 `-t package` 创建库 |
| `dart pub get` | 解析并下载依赖 | CI 中用 `--enforce-lockfile` 防止锁文件漂移 |
| `dart pub upgrade` | 在约束内升级 | 升级后在预发布验证 |
| `dart pub outdated` | 查看可升级项 | 定期执行，形成升级节奏 |
| `dart run bin/server.dart` | 本地运行 | 生产用 AOT 编译产物 |
| `dart analyze` | 静态分析 | `--fatal-infos` 可让 info 也失败 |
| `dart format .` | 格式化 | CI 用 `--output=none --set-exit-if-changed` |
| `dart test` | 运行测试 | 用 `-j` 控制并发，集成测试单独标签 |
| `dart test --coverage` | 覆盖率 | 覆盖率不是目标，关键路径覆盖才是 |
| `dart compile exe bin/server.dart -o build/server` | AOT 编译 | 产物与目标 OS/架构绑定 |
| `dart doc` | 生成 API 文档 | 公共库必备 |
| `dart fix --apply` | 自动修复 lint | 大批量修复要单独提交 |

**关键文件职责**：

| 文件 | 作用 | 是否入库 |
|---|---|---|
| `pubspec.yaml` | 依赖与元数据 | 是 |
| `pubspec.lock` | 精确版本锁定 | 应用：是；库：否 |
| `analysis_options.yaml` | lint 与分析配置 | 是 |
| `.dart_tool/package_config.json` | 生成的本机路径映射 | 否 |
| `build.yaml` | 代码生成配置 | 是 |

### 2.4 项目布局

**单服务（推荐起点）**

```text
order_service/
├── bin/
│   └── server.dart              # 仅装配 + 启动
├── lib/
│   ├── src/
│   │   ├── app.dart             # 组装 handler / 中间件
│   │   ├── config/
│   │   │   └── app_config.dart  # 配置解析 + 校验
│   │   ├── features/
│   │   │   ├── orders/
│   │   │   │   ├── order_routes.dart
│   │   │   │   ├── order_service.dart
│   │   │   │   ├── order_repository.dart
│   │   │   │   ├── order_dto.dart
│   │   │   │   └── order_domain.dart
│   │   │   └── inventory/
│   │   └── shared/
│   │       ├── db/
│   │       ├── auth/
│   │       ├── http/
│   │       ├── errors/
│   │       └── observability/
│   └── order_service.dart       # 对外导出（若作为库）
├── migrations/
│   ├── 0001_init.sql
│   └── 0002_add_order_idempotency.sql
├── test/
│   ├── unit/
│   └── integration/
├── tool/
│   └── seed.dart
├── analysis_options.yaml
├── pubspec.yaml
├── Dockerfile
└── README.md
```

**按功能（feature）而非按技术层**组织，是中型以上项目的默认选择：改一个业务时，相关文件都在一个目录里；跨业务共享的才上移到 `shared/`。

**多包工作区（monorepo）**

```text
backend/
├── packages/
│   ├── shared_models/    # DTO、错误码、枚举
│   ├── shared_auth/      # 令牌校验
│   ├── order_service/
│   └── inventory_service/
├── tool/                 # 脚本：迁移、代码生成、发布
└── pubspec.yaml          # workspace 声明
```

规则：`shared_*` 不允许反向依赖具体服务；服务之间不通过 `path:` 直接引用彼此的内部实现（否则等于没拆分）。

### 2.5 静态分析与 lint 基线

```yaml
# analysis_options.yaml
include: package:lints/recommended.yaml

analyzer:
  language:
    strict-casts: true
    strict-inference: true
    strict-raw-types: true
  errors:
    invalid_annotation_target: ignore   # 使用 freezed/json_serializable 时常见
    missing_required_param: error
    unused_import: error
  exclude:
    - '**/*.g.dart'
    - '**/*.freezed.dart'
    - build/**

linter:
  rules:
    - always_declare_return_types
    - avoid_dynamic_calls
    - avoid_print              # 服务端用结构化 logger
    - cancel_subscriptions
    - close_sinks
    - directives_ordering
    - prefer_final_locals
    - require_trailing_commas
    - unawaited_futures        # 关键：暴露忘记 await 的异步调用
    - use_build_context_synchronously  # Flutter 项目保留
```

`unawaited_futures` 与 `cancel_subscriptions` 对服务端尤其重要：前者防止未观察的异步失败，后者防止流订阅泄漏。

### 2.6 配置加载与校验（fail fast）

配置是启动期最常见的错误来源。推荐“显式读取 → 解析 → 校验 → 冻结”四步：

```dart
final class AppConfig {
  const AppConfig({
    required this.env,
    required this.port,
    required this.databaseUrl,
    required this.dbPoolSize,
    required this.requestTimeout,
    required this.logLevel,
  });

  final String env;
  final int port;
  final String databaseUrl;
  final int dbPoolSize;
  final Duration requestTimeout;
  final String logLevel;

  static AppConfig fromEnv(Map<String, String> env) {
    String require(String key) {
      final v = env[key]?.trim();
      if (v == null || v.isEmpty) {
        throw ConfigException('缺少必需配置 $key');
      }
      return v;
    }

    int requireInt(String key, {required int min, required int max}) {
      final raw = require(key);
      final value = int.tryParse(raw);
      if (value == null || value < min || value > max) {
        throw ConfigException('$key 必须是 $min..$max 的整数，实际为 "$raw"');
      }
      return value;
    }

    final envName = require('APP_ENV');
    if (!const {'dev', 'staging', 'prod'}.contains(envName)) {
      throw ConfigException('APP_ENV 不受支持：$envName');
    }

    return AppConfig(
      env: envName,
      port: requireInt('PORT', min: 1, max: 65535),
      databaseUrl: require('DATABASE_URL'),
      dbPoolSize: requireInt('DB_POOL_SIZE', min: 1, max: 100),
      requestTimeout: Duration(
        milliseconds: requireInt('REQUEST_TIMEOUT_MS', min: 100, max: 60000),
      ),
      logLevel: env['LOG_LEVEL'] ?? (envName == 'prod' ? 'info' : 'debug'),
    );
  }

  /// 日志中可安全打印的表示，永远不包含密码
  Map<String, Object> toSafeLog() => {
        'env': env,
        'port': port,
        'dbPoolSize': dbPoolSize,
        'requestTimeoutMs': requestTimeout.inMilliseconds,
        'logLevel': logLevel,
      };
}

final class ConfigException implements Exception {
  ConfigException(this.message);
  final String message;
  @override
  String toString() => 'ConfigException: $message';
}
```

要点：

- 配置对象**不可变**（`final` 字段 + `const` 构造），避免运行期被意外改写。
- 校验在启动时完成，失败直接退出并打印可操作的错误（哪个键、期望什么、实际什么）。
- **不要把 `databaseUrl` 原样打进日志**，其中的密码会泄露。写一个 `toSafeLog()`。
- 环境差异只体现在配置，不体现在代码分支。

### 2.7 本地开发环境

依赖用 `docker compose` 起，应用用 `dart run` 跑，保持“一键可复现”：

```yaml
# docker-compose.yml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: app
      POSTGRES_DB: app_dev
    ports: ['5432:5432']
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -U app -d app_dev']
      interval: 5s
      retries: 10
  redis:
    image: redis:7-alpine
    ports: ['6379:6379']
```

```bash
docker compose up -d
export DATABASE_URL=postgresql://app:app@localhost:5432/app_dev
export REDIS_URL=redis://localhost:6379
dart run bin/server.dart
```

开发期还需要：

- **迁移先行**：`dart run tool/migrate.dart up`，不要手工改本地库结构。
- **种子数据**：`tool/seed.dart` 提供可重复执行的测试数据。
- **请求样例**：`examples/*.http` 或 curl 脚本，方便新人和联调。
- **热重载**：服务端可用 `dart run --enable-vm-service` 配合文件监听工具实现自动重启；不要期望 Flutter 的热重载直接作用于服务端。

### 2.8 构建产物与运行时差异

Dart 有两种执行模式，后端必须清楚差别：

| 模式 | 命令 | 特点 | 适用 |
|---|---|---|---|
| JIT / VM | `dart run bin/server.dart` | 启动快、可动态加载、性能较低 | 本地开发、调试 |
| AOT 原生 | `dart compile exe bin/server.dart -o build/server` | 启动快、稳态性能好、单文件 | 生产部署 |

AOT 注意事项：

- 产物与编译机器的 **OS + CPU 架构**绑定；在 macOS 上编译的二进制不能直接跑在 Linux 容器里。
- 反射受限：运行期无法枚举类/字段，序列化必须显式或代码生成（这也是 `json_serializable` 存在的原因）。
- `dart compile exe` 支持交叉编译到部分目标（如 `--target-os=linux`），但需实际验证动态库依赖。
- 容器构建必须使用 `linux/amd64` 或 `linux/arm64` 一致的镜像，否则会得到 `exec format error`。

```bash
# 在容器内构建，天然得到目标平台产物
dart compile exe bin/server.dart -o build/server
./build/server   # 直接运行，无 VM 依赖
```

---

## 3. 后端必需的 Dart 语言能力

后端代码的生命周期通常远长于客户端：它会被反复修改、在凌晨三点被排查、被交接给陌生人。因此语言能力的重点不是“写得巧妙”，而是**表达力、可读性与可验证性**。

### 3.1 类型系统与空安全

Dart 的空安全是**编译期**保证：非空类型 `T` 永远不会持有 `null`（除非用 `late` 或 `!` 自欺）。

```dart
// 表达“可能不存在”，让调用方必须处理
Future<Account?> findById(String id);

// 表达“一定存在，不存在就是错误”，在边界处快速失败
Future<Account> getById(String id) async {
  final account = await _repo.findById(id);
  if (account == null) throw AccountNotFound(id);
  return account;
}
```

**空值处理的四个层次**

| 写法 | 语义 | 使用建议 |
|---|---|---|
| `T?` | 值可能缺失 | 仓储查询、可选字段 |
| `??` | 缺失时取默认 | 配置、分页默认值 |
| `?.` | 短路访问 | 可空链式调用 |
| `!` | 断言非空 | 仅在你比编译器知道更多且有注释说明时用 |

`!` 是“把编译期错误推迟成运行期崩溃”。后端代码里出现大量 `!`，通常意味着建模不准确。测试环境常用的 `mock!` 也应尽量替换为 `late final` 或构造注入。

**空安全提升（promotion）**：局部变量的 `null` 检查会让类型提升；但字段、`late`、被闭包捕获的可变变量可能无法提升，需要局部快照：

```dart
if (_client != null) {
  _client!.send();          // 字段不能直接提升
}
final client = _client;
if (client != null) {
  client.send();            // 局部快照可提升
}
```

**类型的使用纪律**

- 用 `Object?` 表示“任意值”，用 `dynamic` 时说明理由。
- 用 `Never` 表达一定抛错/不返回的函数（如 `fail()`）。
- 泛型约束用 `extends`（Dart 的 `extends` 在泛型边界上相当于“可赋值给”）：

```dart
T requireNonNull<T extends Object>(T? value, String name) {
  if (value == null) throw ArgumentError.notNull(name);
  return value;
}
```

### 3.2 sealed 类型与模式匹配：让非法状态不可表示

后端最危险的 bug 来源之一是“多个布尔/可空字段组合出非法状态”。`sealed` + `switch` 表达式能把状态空间收敛到编译器可检查的封闭集合。

```dart
sealed class PaymentState {
  const PaymentState();
}

final class Pending extends PaymentState {
  const Pending(this.createdAt);
  final DateTime createdAt;
}

final class Authorized extends PaymentState {
  const Authorized(this.authCode);
  final String authCode;
}

final class Captured extends PaymentState {
  const Captured(this.transactionId);
  final String transactionId;
}

final class Failed extends PaymentState {
  const Failed(this.reason);
  final String reason;
}
```

```dart
String describe(PaymentState s) => switch (s) {
      Pending(:final createdAt) => '等待支付，创建于 $createdAt',
      Authorized(:final authCode) => '已授权：$authCode',
      Captured(:final transactionId) => '已完成：$transactionId',
      Failed(:final reason) => '失败：$reason',
    };
```

好处：新增一个状态时，**所有未更新的 `switch` 会编译失败**（穷尽性检查），而不是在运行期悄悄返回 `null` 或落到错误的 `default`。

**常见误用**：`switch` 里写 `default => ...` 会把穷尽性检查关掉。除非确有“未来状态可忽略”的业务语义，否则不要写 `default`。

**记录（Records）** 适合返回多个值而不定义类：

```dart
({int total, List<Order> items}) pagedOrders = await repo.page(limit: 20);
```

但对外 API 的返回类型、需要序列化或需要行为的模型，仍应使用命名类。

### 3.3 值对象与不可变建模

后端大量概念是“带校验的值”，例如货币金额、邮箱、租户 ID。用值对象把校验放进构造，可以让非法值在系统中无法流通。

```dart
final class Money {
  Money._(this.minorUnits, this.currency);

  factory Money.ofMinor(int minorUnits, String currency) {
    if (!const {'CNY', 'USD', 'EUR'}.contains(currency)) {
      throw ArgumentError('不支持的币种：$currency');
    }
    return Money._(minorUnits, currency);
  }

  factory Money.ofDecimal(String decimal, String currency) {
    // 用字符串解析避免二进制浮点误差
    final match = RegExp(r'^(-?\d+)(?:\.(\d{1,2}))?$').firstMatch(decimal);
    if (match == null) throw FormatException('金额格式非法：$decimal');
    final whole = int.parse(match.group(1)!);
    final fraction = int.parse((match.group(2) ?? '0').padRight(2, '0'));
    final sign = whole < 0 ? -1 : 1;
    return Money.ofMinor(sign * (whole.abs() * 100 + fraction), currency);
  }

  final int minorUnits;   // 最小货币单位，例如“分”
  final String currency;

  Money operator +(Money other) {
    if (other.currency != currency) throw StateError('币种不一致');
    return Money._(minorUnits + other.minorUnits, currency);
  }

  Money operator -(Money other) {
    if (other.currency != currency) throw StateError('币种不一致');
    return Money._(minorUnits - other.minorUnits, currency);
  }

  bool isNegative() => minorUnits < 0;

  String toDecimalString() {
    final sign = minorUnits < 0 ? '-' : '';
    final abs = minorUnits.abs();
    return '$sign${abs ~/ 100}.${(abs % 100).toString().padLeft(2, '0')}';
  }

  @override
  bool operator ==(Object other) =>
      other is Money && other.minorUnits == minorUnits && other.currency == currency;

  @override
  int get hashCode => Object.hash(minorUnits, currency);

  @override
  String toString() => '${toDecimalString()} $currency';
}
```

**为什么金额不能用 `double`**：`0.1 + 0.2 != 0.3` 是二进制浮点的固有性质。金融计算必须用整数最小单位或十进制定点类型。

**不可变对象的实现要点**

- 所有字段 `final`，构造后不提供 setter。
- 集合字段防御性复制为 `List.unmodifiable`，否则外部仍能修改内部状态。
- `==` 与 `hashCode` 成对实现（或用 `equatable` / `freezed` 生成）。
- 需要“修改”时返回新实例：`copyWith`。

```dart
final class OrderLine {
  OrderLine({
    required this.sku,
    required this.quantity,
    required this.unitPrice,
  })  : assert(quantity > 0),
        _tags = List.unmodifiable(<String>[]);

  final String sku;
  final int quantity;
  final Money unitPrice;
  final List<String> _tags;

  List<String> get tags => _tags;
}
```

### 3.4 类、接口、mixin 与组合

Dart 没有 `interface` 关键字，任何类都可被 `implements`。后端项目推荐把“能力契约”写成抽象类：

```dart
abstract interface class OrderRepository {
  Future<Order?> findById(OrderId id);
  Future<void> save(Order order);
  Future<List<Order>> findExpiredBefore(DateTime cutoff, {int limit = 100});
}
```

- `abstract interface class`：只能被实现，不能被继承——适合端口（port）定义。
- `abstract class`：可带实现，适合需要共享默认行为的基类。
- `extends` 单继承；`implements` 可多实现；`with` 复用行为（mixin）。
- **优先组合**：`OrderService` 持有 `OrderRepository`，而不是 `extends RepositoryBase`。

**mixin 的合理用法**：横切能力，例如 `with TimestampedMixin`。不要用 mixin 做“万能工具类继承”。

**依赖倒置示例**：

```dart
// 领域层只认识抽象
final class CreateOrder {
  CreateOrder({
    required OrderRepository orders,
    required InventoryPort inventory,
    required PaymentPort payments,
    required Clock clock,
    required IdGenerator ids,
  })  : _orders = orders,
        _inventory = inventory,
        _payments = payments,
        _clock = clock,
        _ids = ids;

  final OrderRepository _orders;
  final InventoryPort _inventory;
  final PaymentPort _payments;
  final Clock _clock;
  final IdGenerator _ids;

  Future<OrderId> call(CreateOrderCommand cmd) async {
    final reservation = await _inventory.reserve(cmd.lines);
    try {
      final order = Order.create(
        id: _ids.next(),
        customerId: cmd.customerId,
        lines: cmd.lines,
        now: _clock.now(),
        reservationId: reservation.id,
      );
      await _orders.save(order);
      return order.id;
    } catch (_) {
      await _inventory.release(reservation.id);   // 补偿
      rethrow;
    }
  }
}
```

`Clock` 与 `IdGenerator` 抽象让时间相关逻辑可测试（不必真的等待或篡改系统时间）。

### 3.5 集合与迭代器

**核心 API**

```dart
final active = users.where((u) => u.isActive).toList();
final byId = {for (final u in users) u.id: u};
final names = users.map((u) => u.name).toSet();
final total = lines.fold<int>(0, (sum, l) => sum + l.quantity);
final firstAdmin = users.firstWhereOrNull((u) => u.isAdmin);   // package:collection
```

**惰性求值陷阱**

```dart
final query = users.where((u) => u.isActive);  // 尚未执行
users.add(newUser);                            // 修改了源集合
print(query.length);                           // 结果包含 newUser，且依赖源集合当前状态
```

`Iterable` 是惰性视图。需要稳定快照时立刻 `.toList()`；需要多次遍历时也先物化，避免重复计算或并发修改异常。

**排序与稳定性**：`List.sort` 不保证稳定排序。需要稳定结果时，比较器要包含唯一的兜底键（如主键），否则分页会出现重复/遗漏。

```dart
orders.sort((a, b) {
  final byDate = b.createdAt.compareTo(a.createdAt);
  return byDate != 0 ? byDate : b.id.compareTo(a.id);
});
```

**并发修改**：在遍历 `List` 时增删元素会抛 `ConcurrentModificationError`。先复制或收集待处理项。

**大集合**：避免在内存里一次性加载百万行。用游标分页、`Stream` 分批处理、SQL 侧聚合。

### 3.6 异常体系设计

后端异常应按“谁能处理、如何映射”分层：

```dart
/// 业务/输入层的可预期失败，映射为 4xx
sealed class AppException implements Exception {
  const AppException(this.code, this.message);
  final String code;
  final String message;
}

final class ValidationException extends AppException {
  const ValidationException(String message, {this.fieldErrors = const {}})
      : super('invalid_request', message);
  final Map<String, String> fieldErrors;

  Map<String, Object?> toProblemJson() => {
        'type': 'about:blank',
        'title': '请求参数不合法',
        'status': 422,
        'code': code,
        'detail': message,
        'errors': fieldErrors,
      };
}

final class NotFoundException extends AppException {
  const NotFoundException(String resource, Object id)
      : super('not_found', '$resource 不存在：$id');
}

final class ConflictException extends AppException {
  const ConflictException(super.code, super.message);
}

final class ForbiddenException extends AppException {
  const ForbiddenException([String message = '无权执行该操作'])
      : super('forbidden', message);
}

final class UnauthorizedException extends AppException {
  const UnauthorizedException([String message = '未认证'])
      : super('unauthenticated', message);
}
```

**技术性失败**（数据库断连、第三方超时）不要伪装成业务异常，它们应映射为 5xx 并触发告警：

```dart
final class DependencyUnavailable implements Exception {
  DependencyUnavailable(this.dependency, this.cause);
  final String dependency;
  final Object cause;
}
```

**异常使用纪律**

1. 不要用异常做正常控制流（例如用 try/catch 判断“记录是否存在”；用 `exists()` 或可空返回）。
2. 捕获要具体：`on TimeoutException catch (e)` 优于 `catch (e)`。
3. 重新抛出时保留原始错误：`throw DependencyUnavailable('payment', e)` 而非 `throw Exception('failed')`。
4. 不要在 `finally` 里 `return`，会吞掉异常。
5. `async` 函数中抛出的异常会包进 `Future`，调用方不 `await` 就永远看不到。

### 3.7 扩展方法、typedef 与工具设计

**扩展方法**适合给已有类型加“读起来像领域语言”的辅助能力，但不应用于隐藏副作用：

```dart
extension StringValidation on String {
  bool get isBlank => trim().isEmpty;
  bool get looksLikeEmail =>
      RegExp(r'^[^@\s]+@[^@\s]+\.[^@\s]+$').hasMatch(this);
}

extension DurationFormatting on Duration {
  String get inLogFormat =>
      '${inMilliseconds}ms';
}
```

**typedef** 让函数签名可读：

```dart
typedef Handler = Future<Response> Function(Request request);
typedef Middleware = Handler Function(Handler inner);
```

**扩展方法在 `dynamic` 上不可用**，且与静态解析绑定——同名扩展冲突时必须显式调用 `ExtensionName(x).method()`。

### 3.8 代码生成

Dart 的后端服务几乎一定需要代码生成，原因：无运行期反射 + 大量 DTO。

| 工具 | 用途 | 是否常用 |
|---|---|---|
| `json_serializable` | JSON 序列化/反序列化 | 常用 |
| `freezed` | 不可变数据类 + 联合类型 + copyWith | 常用 |
| `build_runner` | 运行生成器 | 必需 |
| `drift` | 类型安全 SQL（也可用于服务端 SQLite/Postgres 场景） | 视需求 |
| `serverpod_cli` | Serverpod 模型/协议生成 | 使用 Serverpod 时必需 |
| `dart_frog_cli` | 路由/开发服务器 | 使用 Dart Frog 时必需 |

```yaml
dev_dependencies:
  build_runner: ^2.4.0
  json_serializable: ^6.8.0
  freezed: ^2.5.0

dependencies:
  json_annotation: ^4.9.0
  freezed_annotation: ^2.4.0
```

```dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'order_dto.freezed.dart';
part 'order_dto.g.dart';

@freezed
class OrderDto with _$OrderDto {
  const factory OrderDto({
    required String id,
    required String status,
    required int totalMinor,
    required String currency,
    required DateTime createdAt,
  }) = _OrderDto;

  factory OrderDto.fromJson(Map<String, dynamic> json) =>
      _$OrderDtoFromJson(json);
}
```

**生成代码的工程纪律**

- 生成命令写进 README 与 `tool/`：`dart run build_runner build --delete-conflicting-outputs`。
- 生成文件是否入库：团队决定。入库便于 code review 与 IDE 跳转，但会产生大量 diff；不入库则 CI 必须先跑生成。
- CI 增加“生成结果无漂移”检查：生成后 `git diff --exit-code`。
- 不要手改 `.g.dart` / `.freezed.dart`。
- 生成步骤失败的报错要写进排障文档（版本不匹配是最常见原因）。

### 3.9 可见性、库组织与文档

- `_` 前缀是 **library 级**私有，不是文件级；同一 library 内（含 `part`）都可访问。
- `part` / `part of` 会把代码合并进同一 library，谨慎使用，否则私有边界被扩大到整个 part 组。
- 包的公共 API 通过 `lib/<package>.dart` 统一导出，`lib/src/` 下为内部实现。
- 使用 `@Deprecated('迁移到 xxx')` 标记弃用，并在 CHANGELOG 说明。
- 公共 API 写 `///` 文档注释，包含行为、异常、幂等性等调用方必须知道的信息。

```dart
/// 按下单请求创建订单。
///
/// 幂等：相同 [idempotencyKey] 重复调用返回首次结果。
/// 抛出 [ValidationException]（输入非法）、[ConflictException]（库存不足）。
Future<Order> execute(CreateOrderCommand cmd);
```

---

## 4. 异步、并发与运行时

这是 Dart 后端最容易“看起来会、其实不会”的一章。核心结论先给出：

> **Dart 的异步让 I/O 等待不阻塞线程，但不会让你的 CPU 计算变快；一个 isolate 只有一个线程和一个事件循环。**

### 4.1 事件循环（Event Loop）机制

一个 isolate 内有两个队列：

```text
        ┌──────────────────────────────┐
        │   Microtask Queue（微任务）    │  优先级高
        └──────────────┬───────────────┘
                       │ 每次事件循环迭代先清空微任务
        ┌──────────────▼───────────────┐
        │   Event Queue（事件）          │  I/O 回调、Timer、消息
        └──────────────────────────────┘
```

执行顺序：当前同步代码 → 清空所有微任务 → 取一个事件 → 执行 → 清空微任务 → 下一个事件……

```dart
void main() {
  print('A');                 // 同步
  Future(() => print('B'));   // 事件队列
  Future.microtask(() => print('C'));  // 微任务
  scheduleMicrotask(() => print('D'));
  print('E');
}
// 输出：A E C D B
```

**这对后端意味着什么**

1. **同步 CPU 工作会阻塞整个 isolate**：一个 500ms 的同步 JSON 解析会让该实例上所有并发请求的延迟都增加 500ms。
2. **微任务可以是饥饿源**：递归 `scheduleMicrotask` 会让事件队列永远得不到执行（I/O 回调无法运行）。
3. **`await` 不是“切换到别的线程”**：它只是保存续体并让出事件循环，续体在后续事件中继续执行。

### 4.2 Future 的完整用法

**并发等待与错误策略**

```dart
// 全部成功才成功；任何一个失败则整体失败（默认 eagerError: false 等全部完成）
final results = await Future.wait([loadUser(), loadOrders(), loadCart()]);

// 快速失败：第一个错误立即抛出，忽略其余结果
await Future.wait(tasks, eagerError: true);

// 只取最快成功的结果
final fastest = await Future.any([primary(), fallback()]);
```

**超时**：`Future.timeout` 只是“调用方不再等待”，**底层操作可能仍在继续**。

```dart
final payment = await gateway.charge(req).timeout(
  const Duration(seconds: 3),
  onTimeout: () => throw DependencyUnavailable('payment', 'timeout'),
);
```

因此支付、扣库存这类操作必须配合**幂等键**与**状态查询**，否则超时后的重试可能造成重复扣款。

**错误处理与链式传播**

```dart
Future<Order> load(String id) => _repo.findById(id)
    .then((o) => o ?? (throw NotFoundException('order', id)))
    .catchError((Object e, StackTrace st) {
      if (e is NotFoundException) rethrow;
      _log.error('订单加载失败', error: e, stackTrace: st);
      throw DependencyUnavailable('order_db', e);
    });
```

**`unawaited` 的正确用法**：明确表示“故意不等待”，并确保错误被处理：

```dart
unawaited(_audit.record(event).catchError((e, st) {
  _log.warn('审计写入失败', error: e, stackTrace: st);
}));
```

但“故意不等待”的长任务应该在受管生命周期中运行（队列、后台执行器），而不是裸 Future——进程退出时它们会消失。

**常见 Future 陷阱**

| 陷阱 | 后果 | 修复 |
|---|---|---|
| 忘记 `await` | 响应先返回，错误丢失，顺序错乱 | 启用 `unawaited_futures` lint |
| 在 `forEach` 中 `await` | 实际是串行且外层不等待 | 用 `for` 循环或 `Future.wait` |
| `catchError` 捕获后不回抛 | 调用方以为成功 | 只捕获能处理的错误 |
| 在 `finally` 中 `await` 长任务 | 拖慢响应 | 移出请求路径 |
| 无超时地等待外部服务 | 连接池耗尽 | 全链路超时 |

```dart
// 错误写法：forEach 不会等待异步回调
items.forEach((i) async {
  await process(i);   // 外层立即继续，异常无人观察
});

// 正确写法一：串行
for (final item in items) {
  await process(item);
}

// 正确写法二：受控并发
await Future.wait(items.map(process));
```

### 4.3 Stream 深入

**单订阅 vs 广播**

```dart
// 单订阅：只能有一个监听者，事件不缓存
Stream<int> single() async* { yield 1; }

// 广播：多个监听者，只收到订阅之后的事件
final controller = StreamController<Event>.broadcast();
```

**资源释放模式**

```dart
final class Ticker {
  Ticker(Duration interval) {
    _timer = Timer.periodic(interval, (_) => _controller.add(DateTime.now()));
  }

  final _controller = StreamController<DateTime>.broadcast();
  late final Timer _timer;

  Stream<DateTime> get ticks => _controller.stream;

  Future<void> dispose() async {
    _timer.cancel();
    await _controller.close();
  }
}
```

**背压（Backpressure）**：Dart 的 Stream 没有内建背压协议。如果生产者持续 `add` 而消费者处理慢，事件会在内存中堆积。

```dart
// 用 pause/resume 实现消费者驱动的节流
final subscription = source.listen((event) async {
  subscription.pause();
  try {
    await handler(event);
  } finally {
    subscription.resume();
  }
});
```

或者改用有界队列 + 拒绝/丢弃策略：

```dart
final queue = StreamController<Job>();
var inFlight = 0;
const maxInFlight = 32;

source.listen((job) {
  if (inFlight >= maxInFlight) {
    _log.warn('队列已满，拒绝任务', fields: {'jobId': job.id});
    return;                       // 或接入持久队列
  }
  inFlight++;
  queue.add(job);
});
```

**必须理解的行为**

- `listen` 的 `onError` 若未提供，错误会被当作未处理异常。
- 取消订阅后，`StreamController` 的 `onCancel` 是释放资源的钩子。
- `async*` 生成器在取消时会在 `yield` 处停止，`finally` 会执行。
- `Stream.transform`、`map`、`where` 等都是惰性组合，错误会沿链传播。

### 4.4 Isolate：真正的并行

**创建与通信**

```dart
import 'dart:isolate';

// 一次性任务：最常用
Future<int> sumAsync(List<int> data) => Isolate.run(() => data.reduce((a, b) => a + b));

// 长期 worker：通过端口通信
Future<Isolate> startWorker() async {
  final receivePort = ReceivePort();
  final isolate = await Isolate.spawn(_workerMain, receivePort.sendPort);
  return isolate;
}

Future<void> _workerMain(SendPort replyTo) async {
  final commands = ReceivePort();
  replyTo.send(commands.sendPort);
  await for (final message in commands) {
    // 处理命令
  }
}
```

**性能与数据传递**

- isolate 之间传对象会**复制**（除少数可转移类型），大对象传输有成本。
- `TransferableTypedData` / `TransferableRecord`（视 SDK 版本）可转移所有权，避免复制。
- 启动 isolate 有成本（毫秒级），不要为每个请求 spawn 一个；用 worker 池。
- 每个 isolate 有独立堆，内存占用是叠加的。

**适合拆 isolate 的工作**：图片/PDF 处理、压缩、密码学批量运算、大 JSON 转换、报表生成、CPU 密集算法。

**不适合**：只是等待 I/O 的普通请求处理。多开 isolate 只会增加内存与调度开销。

**worker 池的最小实现思路**

```dart
final class WorkerPool {
  WorkerPool({required this.size, required this.task}) {
    for (var i = 0; i < size; i++) {
      _idle.add(null);
    }
  }

  final int size;
  final Future<void> Function(Object? message) task;
  final _idle = <Isolate?>[];
  // 实际实现需处理：任务排队、错误回传、isolate 崩溃重建、关闭
}
```

生产建议：优先用成熟包或把重活交给独立工作进程/服务，自己实现池时要覆盖崩溃恢复与优雅关闭。

### 4.5 并发控制：信号量与并发池

外部依赖通常有容量上限（数据库连接、第三方 QPS）。必须有显式并发上限：

```dart
final class Semaphore {
  Semaphore(this._permits);

  int _permits;
  final _waiters = <Completer<void>>[];

  Future<T> run<T>(Future<T> Function() action) async {
    await _acquire();
    try {
      return await action();
    } finally {
      _release();
    }
  }

  Future<void> _acquire() {
    if (_permits > 0) {
      _permits--;
      return Future.value();
    }
    final completer = Completer<void>();
    _waiters.add(completer);
    return completer.future;
  }

  void _release() {
    if (_waiters.isNotEmpty) {
      _waiters.removeAt(0).complete();
    } else {
      _permits++;
    }
  }
}
```

使用：

```dart
final _dbGate = Semaphore(20);

Future<List<Row>> query(String sql) =>
    _dbGate.run(() => _pool.execute(sql));
```

**注意**：排队本身也需要时限。如果等待许可的时间超过请求预算，应快速失败（`503`/`429`），否则延迟会无界增长。

### 4.6 资源生命周期与优雅关闭

```dart
import 'dart:async';
import 'dart:io';

Future<void> main() async {
  final config = AppConfig.fromEnv(Platform.environment);
  final deps = await AppDeps.create(config);

  final server = await serveApp(deps, config.port);
  deps.log.info('服务已启动', fields: {'port': config.port});

  final shutdown = Completer<void>();

  Future<void> beginShutdown(String reason) async {
    if (shutdown.isCompleted) return;
    deps.log.info('开始优雅关闭', fields: {'reason': reason});
    shutdown.complete();

    await server.close(force: false);       // 停止接收新连接
    await deps.consumers.stop();             // 停止拉取新消息
    await deps.background.drain(
      timeout: const Duration(seconds: 20),  // 等待在途任务
    );
    await deps.db.close();
    await deps.http.close();
    deps.log.info('已关闭');
  }

  ProcessSignal.sigterm.watch().listen((_) => beginShutdown('SIGTERM'));
  ProcessSignal.sigint.watch().listen((_) => beginShutdown('SIGINT'));

  await shutdown.future;
}
```

**三个关键点**

1. `server.close(force: false)`：给在途请求完成的机会。但平台通常会在若干秒后强杀，所以要配置 `terminationGracePeriodSeconds` 与关闭超时。
2. **先停流量、再停消费者**：否则消费者停止期间新请求仍在写入，会造成处理积压。
3. **关闭顺序是依赖图的逆序**：先关上层（HTTP），再关闭下层（DB、队列）。

**不要在关闭时做重活**（如全量缓存刷新），那会吃掉宽限期。

### 4.7 后台任务与 Timer

- `Timer.periodic` 在 isolate 内运行，进程重启即丢失；只适合指标上报、缓存清理等可丢失任务。
- `Timer` 必须 `cancel()`，否则在测试中会阻止退出并造成泄漏。
- 长时间运行的 `await` 循环要检查终止标志：

```dart
while (!_stopping) {
  await _pollOnce();
  await Future<void>.delayed(_interval);
}
```

- 定时任务的时间漂移会累积；对“每小时整点”类需求，用计算下次触发时间而非固定 `Timer.periodic`。

---

## 5. HTTP 与网络基础

框架会帮你处理 90% 的细节，但剩下 10% 恰恰是事故来源：`Content-Length` 与实际 body 不一致、代理改写 `Host`、`X-Forwarded-For` 被伪造、预检请求 405、缓存把用户私有数据发给别人。

### 5.1 报文结构与语义

```text
POST /v1/orders?dryRun=true HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJ...
Content-Type: application/json
Content-Length: 132
X-Request-Id: 01J8Z...

{"customerId":"c_123","lines":[{"sku":"A-1","quantity":2}]}
```

```text
HTTP/1.1 201 Created
Content-Type: application/json
Location: /v1/orders/o_456
Cache-Control: no-store
X-Request-Id: 01J8Z...

{"id":"o_456","status":"pending"}
```

**方法的语义契约**

| 方法 | 安全（只读） | 幂等 | 典型语义 | 常见错误 |
|---|---|---|---|---|
| GET | 是 | 是 | 获取资源/集合 | 用 GET 做删除或修改 |
| HEAD | 是 | 是 | 只取头部 | 实现缺失导致监控误判 |
| POST | 否 | 否 | 创建、动作、批处理 | 无法安全重试 |
| PUT | 否 | 是 | 整体替换/幂等创建 | 语义不清导致字段丢失 |
| PATCH | 否 | 通常否 | 部分更新 | 用 JSON Merge Patch 却不支持 `null` |
| DELETE | 否 | 是 | 删除 | 删除不存在时返回 500 |

**幂等性的工程含义**：客户端/网关在网络超时后可能自动重试。只有幂等方法可以无条件重试；`POST` 必须靠幂等键（见 7.5）才安全。

### 5.2 状态码选择表

| 场景 | 状态码 | 说明 |
|---|---|---|
| 查询成功 | 200 | 带响应体 |
| 创建成功 | 201 + `Location` | 返回资源表示或仅 ID |
| 已受理的异步任务 | 202 | 返回任务状态查询地址 |
| 删除/无响应体成功 | 204 | 不要带 body |
| 客户端缓存仍有效 | 304 | 配合 `ETag` / `If-None-Match` |
| 参数格式错误 | 400 | JSON 无法解析、类型错误 |
| 未认证 | 401 + `WWW-Authenticate` | 令牌缺失/过期/无效 |
| 已认证但无权限 | 403 | 不要用 401 表示权限不足 |
| 资源不存在 | 404 | 也可用于隐藏资源存在性 |
| 方法不允许 | 405 + `Allow` 头 | 路由存在但方法不匹配 |
| 冲突 | 409 | 唯一键冲突、状态机不允许、版本冲突 |
| 前置条件失败 | 412 | `If-Match` 版本不匹配 |
| 请求体过大 | 413 | 上传/JSON 超限 |
| 语义校验失败 | 422 | 字段合法但业务规则不满足 |
| 限流 | 429 + `Retry-After` | 明确客户端何时重试 |
| 服务端错误 | 500 | 未预期异常 |
| 依赖不可用 | 502/503 | 上游无效 / 自身过载或维护 |
| 依赖超时 | 504 | 网关超时 |

**争议点**：422 与 400 的边界。建议团队统一约定：**结构/类型错误 → 400；结构正确但业务校验失败 → 422**，并写进 API 规范。

### 5.3 缓存语义

**响应头**

```text
Cache-Control: public, max-age=60, stale-while-revalidate=30
ETag: "v42"
Vary: Accept-Encoding, Authorization
```

```text
Cache-Control: no-store          # 用户私有、不可缓存（推荐用于含个人数据的 API）
Cache-Control: private, max-age=0, must-revalidate
```

**条件请求**

```http
GET /v1/products/42
If-None-Match: "v42"

→ 304 Not Modified（无 body）
```

实现要点：

- `ETag` 必须随表示内容变化，通常用内容哈希或聚合版本号。
- `Vary: Authorization` 防止共享缓存把 A 用户的响应给 B 用户（但更安全的做法是私有数据一律 `no-store`）。
- 写操作后要让相关缓存失效（或使用短 TTL）。
- CDN 缓存键要包含租户/版本等维度。

### 5.4 连接、超时与 TLS

**超时需要分层设置**

| 层 | 典型值 | 说明 |
|---|---|---|
| 客户端连接超时 | 1–3s | 建连阶段 |
| 网关读超时 | 5–30s | 面向用户 |
| 应用整体请求超时 | 小于网关 | 必须留出响应余量 |
| 数据库语句超时 | 1–5s | 按查询复杂度 |
| 外部 API 超时 | 1–5s | 按 SLA |

**原则**：下游超时之和必须小于上游超时，否则上层先断开，下层还在占资源。

**HTTP 客户端复用**

```dart
final client = HttpClient()
  ..connectionTimeout = const Duration(seconds: 2)
  ..idleTimeout = const Duration(seconds: 15)
  ..maxConnectionsPerHost = 20;
```

- 复用 `HttpClient`（或 `package:http` 的 `Client`）以复用 TCP/TLS 连接。
- 限制每主机连接数，防止单个下游拖垮进程。
- 未关闭的 `HttpClient` 会导致连接与 socket 泄漏。

**TLS**

- 生产必须 HTTPS；证书由网关或应用终止，二者选一并明确责任。
- 客户端校验服务端证书；内部服务间也建议 mTLS 或至少内网隔离。
- 证书到期是经典事故，纳入监控并提前 30 天告警。

### 5.5 代理、转发头与客户端 IP

在反向代理后，`request.connectionInfo.remoteAddress` 是代理地址。真实客户端 IP 通常在：

```text
X-Forwarded-For: 203.0.113.7, 10.0.0.12
X-Forwarded-Proto: https
X-Forwarded-Host: api.example.com
Forwarded: for=203.0.113.7;proto=https
```

**安全规则**

1. 只信任**你自己配置的**代理层写入的头。直接暴露在公网的应用必须丢弃客户端自带的 `X-Forwarded-*`。
2. `X-Forwarded-For` 是列表，取“最右侧的可信跳数”而不是最左侧（最左侧可被伪造）。
3. 用于限流、审计的 IP 必须来自可信解析结果。
4. 生成绝对 URL 时使用配置的 `PUBLIC_BASE_URL`，不要用 `Host` 头（Host 头注入可导致密码重置链接被劫持）。

### 5.6 CORS 细节

CORS 是浏览器策略，只影响浏览器发起的前端请求。

**预检请求**

```http
OPTIONS /v1/orders
Origin: https://app.example.com
Access-Control-Request-Method: POST
Access-Control-Request-Headers: authorization, content-type
```

**响应**

```http
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET, POST, PATCH, DELETE
Access-Control-Allow-Headers: authorization, content-type, idempotency-key
Access-Control-Max-Age: 600
Access-Control-Allow-Credentials: true
Vary: Origin
```

要点：

- `Access-Control-Allow-Origin: *` 与 `Allow-Credentials: true` 不能共存。
- 必须返回 `Vary: Origin`，否则中间缓存会串源。
- 白名单校验 Origin，不要把请求里的 Origin 原样回显（等于对所有站点开放）。
- 预检请求不携带凭据，认证中间件不能对它返回 401，否则浏览器直接失败。
- CORS 不是 CSRF 防护，也不是认证。

### 5.7 内容协商、压缩与分块

- `Accept: application/json` → `Content-Type: application/json; charset=utf-8`。
- 不支持的 `Accept` 返回 406（或按团队约定降级）。
- 压缩（gzip/br）对 JSON 效果好，但注意：压缩 + 加密的组合可能被 BREACH 类攻击利用（响应中含机密且反射用户输入时）。
- `Transfer-Encoding: chunked` 适合流式响应；需要设置正确的 `Trailer` 与错误处理。
- 流式响应一旦开始发送，就无法再改状态码；错误只能通过流内事件表达。

### 5.8 限流与背压

**常见算法**

| 算法 | 特点 | 适用 |
|---|---|---|
| 固定窗口 | 实现简单，边界突刺 | 粗粒度配额 |
| 滑动窗口 | 更平滑，需存储时间片 | API 配额 |
| 令牌桶 | 允许突发，长期限速 | 大多数 API |
| 漏桶 | 恒定输出速率 | 保护下游 |
| 并发限制 | 限制同时在途请求 | 保护连接池 |

**分层限流**

```text
边缘（按 IP） → 网关（按 API Key / 租户） → 应用（按用户 + 端点） → 依赖（按连接池）
```

返回 429 时给出 `Retry-After`，并在响应体说明配额维度（不要泄露他人配额）。

### 5.9 用 dart:io 直接写一个服务器

理解底层有助于排查框架问题：

```dart
import 'dart:convert';
import 'dart:io';

Future<void> main() async {
  final server = await HttpServer.bind(InternetAddress.anyIPv4, 8080);
  server.autoCompress = true;

  await for (final request in server) {
    try {
      if (request.uri.path == '/healthz') {
        request.response
          ..statusCode = HttpStatus.ok
          ..headers.contentType = ContentType.json
          ..write('{"status":"ok"}');
        await request.response.close();
        continue;
      }

      if (request.method != 'POST' || request.uri.path != '/echo') {
        request.response.statusCode = HttpStatus.notFound;
        await request.response.close();
        continue;
      }

      final body = await utf8.decoder.bind(request).join();
      final json = jsonDecode(body) as Map<String, dynamic>;

      request.response
        ..statusCode = HttpStatus.ok
        ..headers.contentType = ContentType.json
        ..write(jsonEncode({'received': json}));
      await request.response.close();
    } catch (e, st) {
      stderr.writeln('请求处理失败: $e\n$st');
      request.response.statusCode = HttpStatus.internalServerError;
      await request.response.close();
    }
  }
}
```

**这段代码暴露的问题**（也是为什么需要框架）：

- 无请求体大小限制（`join()` 会把超大 body 读进内存）。
- 无路由、无中间件、无统一错误映射。
- 无超时控制，慢客户端可长期占用连接。
- 无请求 ID、日志、指标。
- `await for` 串行处理？不会——`HttpServer` 的 `listen` 回调是并发调度的，但每个请求内的异步链要自己管理。

**必须补上的生产措施**

```dart
// 限制 body 大小
final chunks = <List<int>>[];
var size = 0;
await for (final chunk in request) {
  size += chunk.length;
  if (size > 1 << 20) {   // 1 MiB
    request.response.statusCode = HttpStatus.requestEntityTooLarge;
    await request.response.close();
    return;
  }
  chunks.add(chunk);
}
```

---

## 6. Web 框架与服务端分层

### 6.1 生态全景与选择矩阵

| 方案 | 抽象层级 | 路由 | 中间件 | DI | 数据库/ORM | 代码生成 | 学习曲线 | 适合 |
|---|---|---|---|---|---|---|---|---|
| `dart:io` | 最低 | 自建 | 自建 | 自建 | 自建 | 无 | 低 | 学习、极小工具、自定义协议 |
| **Shelf** | 低 | `shelf_router` | 一等公民 | 自建 | 自选 | 可选 | 低-中 | 想完全掌控架构的团队 |
| **Dart Frog** | 中 | 文件系统路由 | 目录约定 | 内置 | 自选 | CLI 生成 | 中 | 快速 REST API、轻量微服务 |
| **Serverpod** | 高 | Endpoint 类 | 内置 | 内置 | 内置 ORM + 迁移 | 大量生成 | 中-高 | Flutter 全栈、类型安全端到端 |
| 云函数 SDK | 中 | 平台定义 | 平台定义 | 自建 | 平台服务 | 可选 | 中 | 事件驱动、小服务 |

**选型时必做的验证**

1. 用真实业务写一个垂直切片（路由 + 认证 + 一个写接口 + 一个查询 + 一个测试）。
2. 验证目标部署形态：AOT 编译是否成功、冷启动、内存、健康检查。
3. 验证迁移工具：能否在 CI 中对空库和生产快照执行。
4. 验证可观测性：能否接入结构化日志与 OTel。
5. 检查项目活跃度与升级路径（大版本迁移成本）。

### 6.2 Shelf：最小可组合抽象

Shelf 的核心模型只有两个：

```dart
typedef Handler = FutureOr<Response> Function(Request request);
typedef Middleware = Handler Function(Handler inner);
```

`Pipeline` 把中间件串起来，最内层是业务 handler。

```dart
import 'dart:convert';
import 'package:shelf/shelf.dart';
import 'package:shelf/shelf_io.dart' as io;
import 'package:shelf_router/shelf_router.dart';

Handler buildApp(AppDeps deps) {
  final router = Router()
    ..get('/healthz', (Request req) => Response.ok('{"status":"ok"}'))
    ..get('/v1/orders/<id>', (Request req, String id) async {
      final order = await deps.orders.getById(OrderId(id));
      return Response.ok(
        jsonEncode(order.toJson()),
        headers: {'content-type': 'application/json; charset=utf-8'},
      );
    })
    ..post('/v1/orders', (Request req) async {
      final body = await req.readAsString();
      final input = CreateOrderInput.fromJson(jsonDecode(body));
      final order = await deps.createOrder.execute(input);
      return Response(
        201,
        body: jsonEncode(order.toJson()),
        headers: {
          'content-type': 'application/json; charset=utf-8',
          'location': '/v1/orders/${order.id}',
        },
      );
    });

  return const Pipeline()
      .addMiddleware(requestId())
      .addMiddleware(structuredLogging(deps.log))
      .addMiddleware(errorMapping(deps.log))
      .addMiddleware(securityHeaders())
      .addMiddleware(cors(deps.allowedOrigins))
      .addMiddleware(jsonErrors())
      .addHandler(router.call);
}
```

**中间件实现示例**

```dart
Middleware requestId() => (inner) => (request) async {
      final id = request.headers['x-request-id'] ?? _newId();
      final updated = request.change(headers: {'x-request-id': id});
      final response = await inner(updated);
      return response.change(headers: {'x-request-id': id});
    };

Middleware errorMapping(Logger log) => (inner) => (request) async {
      try {
        return await inner(request);
      } on AppException catch (e) {
        return _problem(e, request);
      } catch (e, st) {
        log.error('未处理异常', error: e, stackTrace: st);
        return Response.internalServerError(
          body: '{"code":"internal_error","message":"服务暂时不可用"}',
          headers: {'content-type': 'application/json'},
        );
      }
    };

Middleware securityHeaders() => (inner) => (request) async {
      final response = await inner(request);
      return response.change(headers: {
        'x-content-type-options': 'nosniff',
        'x-frame-options': 'DENY',
        'referrer-policy': 'no-referrer',
        'strict-transport-security': 'max-age=31536000; includeSubDomains',
      });
    };
```

**中间件顺序**：外到内依次是请求 ID → 日志 → 错误映射 → 安全头 → CORS → 认证 → 路由。错误映射要包住后面所有中间件，否则认证失败不会被统一格式化。

**`Request.change` 的重要性**：路由中间件切换 handler 时要更新 `handlerPath` 与 `url`，否则内层路由会重复匹配。

### 6.3 Dart Frog：约定式路由

Dart Frog 用文件系统定义路由，中间件用目录约定。

```text
routes/
├── _middleware.dart          # 全局中间件
├── index.dart                # GET /
├── healthz.dart              # GET /healthz
├── v1/
│   ├── _middleware.dart      # /v1 下所有路由的中间件
│   └── orders/
│       ├── index.dart        # GET /v1/orders, POST /v1/orders
│       └── [id].dart         # GET /v1/orders/<id>
```

```dart
// routes/v1/orders/index.dart
import 'package:dart_frog/dart_frog.dart';

Future<Response> onRequest(RequestContext context) async {
  return switch (context.request.method) {
    HttpMethod.get => _list(context),
    HttpMethod.post => _create(context),
    _ => Response(statusCode: 405),
  };
}

Future<Response> _create(RequestContext context) async {
  final body = await context.request.json() as Map<String, dynamic>;
  final service = context.read<CreateOrder>();
  final order = await service.execute(CreateOrderCommand.fromJson(body));
  return Response.json(
    statusCode: 201,
    body: {'id': order.id},
    headers: {'location': '/v1/orders/${order.id}'},
  );
}
```

```dart
// routes/_middleware.dart
import 'package:dart_frog/dart_frog.dart';

Handler middleware(Handler handler) {
  return handler
      .use(requestLogger())
      .use(provider<CreateOrder>((context) => CreateOrder(...)))
      .use(_errorMapper());
}
```

**Dart Frog 的特点与注意点**

- 路由即文件，目录结构一目了然；但路由发现依赖文件扫描，重构要同时移动文件。
- 依赖注入通过 `provider` + `context.read<T>()`，注意作用域（全局 vs 目录级）。
- 环境变量通过 `context.read<Env>()` 或平台注入。
- `dart_frog dev` 提供热重载；生产构建用 `dart_frog build` 生成产物。
- 中间件基于 Shelf，可复用 Shelf 生态。

### 6.4 Serverpod：全栈类型安全

Serverpod 的设计目标是“Dart 写全栈”，核心概念：

| 概念 | 作用 |
|---|---|
| Endpoint | 服务端可被客户端调用的方法集合 |
| Model（YAML） | 定义数据模型，生成服务端/客户端代码与迁移 |
| Migration | 版本化 schema 变更 |
| Authentication | 内置用户、会话、令牌 |
| Streaming | 内置实时流 |
| Future Call | 定时/延迟任务 |
| Cache | 内置缓存抽象 |

```yaml
# lib/src/protocol/order.yaml
class: Order
table: order
fields:
  customerId: String
  status: OrderStatus
  totalMinor: int
  currency: String
  createdAt: DateTime
```

```dart
// lib/src/endpoints/order_endpoint.dart
import 'package:serverpod/serverpod.dart';
import '../generated/protocol.dart';

class OrderEndpoint extends Endpoint {
  Future<Order> create(Session session, List<OrderLine> lines) async {
    final userId = session.authenticated?.userIdentifier;
    if (userId == null) throw NotAuthenticatedException();

    // 事务 + 库存校验 + 幂等检查（此处省略细节）
    final order = Order(
      customerId: userId,
      status: OrderStatus.pending,
      totalMinor: lines.fold(0, (sum, l) => sum + l.subtotalMinor),
      currency: 'CNY',
      createdAt: DateTime.now().toUtc(),
    );
    return await Order.db.insertRow(session, order);
  }
}
```

**Serverpod 的取舍**

- 收益：端到端类型安全、客户端代码生成、内置认证与实时，少写大量样板。
- 代价：强约定（项目结构、ORM、迁移流程）、升级需要跟框架节奏、排查生成代码问题需要理解其模型。
- 适合 Flutter 团队做 MVP 到中等规模产品；不适合必须完全自控数据层或有大量存量中间件的组织。

### 6.5 分层与依赖方向

**目标**：业务规则不依赖 HTTP、不依赖具体数据库。

```text
┌────────────────────────────────────────────┐
│ Interface（HTTP / gRPC / CLI / Consumer）   │  ← 可替换
├────────────────────────────────────────────┤
│ Application（用例、事务边界、编排）          │
├────────────────────────────────────────────┤
│ Domain（实体、值对象、领域服务、不变量）      │  ← 纯 Dart，无 I/O
├────────────────────────────────────────────┤
│ Infrastructure（SQL、Redis、MQ、HTTP 客户端）│  ← 实现 Domain 定义的端口
└────────────────────────────────────────────┘
```

依赖只能向内。Domain 层不 `import` `shelf`、`postgres`、`http`。

**反例**

```dart
// ❌ 领域层直接读 HTTP 请求
Future<void> cancelOrder(Request request, String id) async {
  final userId = request.headers['x-user-id'];   // 信任前端
  ...
}

// ✅ 用例接收显式命令
Future<void> cancelOrder(CancelOrderCommand cmd) async { ... }
```

**用例的职责边界**

| 应该做 | 不应该做 |
|---|---|
| 加载聚合、调用领域行为、保存 | 解析 JSON |
| 定义事务边界 | 决定 HTTP 状态码 |
| 编排多个端口（库存、支付、通知） | 直接写 SQL 字符串 |
| 发出领域事件 | 记录访问日志格式 |

### 6.6 请求上下文的传递

后端需要贯穿请求的 requestId、用户、租户、deadline。三种方式：

**1. 显式参数（最清晰，推荐）**

```dart
final class RequestContext {
  const RequestContext({
    required this.requestId,
    required this.principal,
    required this.deadline,
  });

  final String requestId;
  final Principal principal;
  final DateTime deadline;
}
```

**2. Zone（透明但易失控）**

```dart
runZoned(
  () => handler(request),
  zoneValues: {'requestId': id},
);
// 任意深度读取 Zone.current['requestId']
```

Zone 适合日志关联，但会让隐式依赖难以追踪；不要用它传递业务必需数据。

**3. 全局单例/静态变量（禁止）**

```dart
// ❌ 并发请求会互相覆盖
String? currentUserId;
```

这是服务端最经典的并发 bug：两个请求交错执行，A 的请求读到 B 的用户 ID，造成越权。

### 6.7 认证与授权在管线的位置

- **认证中间件**：只负责解析并验证凭据，构建 `Principal`，不查询业务资源。
- **授权**：在用例内针对具体资源执行，因为只有用例知道要操作哪个对象。
- 不要把“是否为管理员”的判断散落在每个 handler；用策略对象统一。

```dart
final class Authorizer {
  const Authorizer();

  void ensureCanCancel(Principal p, Order o) {
    if (p.isAdmin) return;
    if (o.customerId != p.userId) {
      throw ForbiddenException('无权取消该订单');
    }
  }
}
```

**存在性隐藏**：对于无权访问的资源，返回 404 而非 403，可避免泄露“该 ID 存在”。

### 6.8 统一响应与内容协商

```dart
final class ApiResponse {
  static Response json(Object? body, {int status = 200, Map<String, String>? headers}) =>
      Response(
        status,
        body: jsonEncode(body),
        headers: {
          'content-type': 'application/json; charset=utf-8',
          ...?headers,
        },
      );
}
```

- 所有成功响应统一 `application/json`。
- 204 响应不要带 `content-type` 与 body。
- 错误响应遵循统一错误模型（见 8.7）。
- 大响应考虑压缩与分页，不要一次返回全表。
- 时间统一 ISO-8601 UTC（`2026-11-21T08:15:00Z`）。

### 6.9 请求体读取的坑

```dart
// ❌ 无限制读取，可能 OOM
final body = await request.readAsString();

// ✅ 限制大小并处理编码
const maxBytes = 1 << 20;
final bytes = await request.read().expand((c) => c).toList();
if (bytes.length > maxBytes) {
  throw ValidationException('请求体过大');
}
final body = utf8.decode(bytes);
```

- `Request.read()` 只能读一次；读完后再读会得到空。
- 需要多次使用时先读成 String/Bytes 再构造新 Request。
- 非法 UTF-8 会抛 `FormatException`，要映射为 400。
- `Content-Type` 不是 JSON 时，按团队策略返回 415。

### 6.10 静态文件与上传

- 静态资源优先交给 CDN 或网关，不要用业务服务扛。
- 如需服务静态文件，注意路径穿越（`../`）、`Content-Type` 嗅探、缓存头与范围请求（`Range`）。
- 上传走对象存储预签名 URL（见 10.7），业务服务不中转大文件。

---

## 7. API 设计与契约

API 是团队之间、服务之间、公司与客户之间的**长期契约**。设计失误的代价不是改代码，而是兼容期、客户端发版、数据迁移和信任损失。

### 7.1 资源建模与命名

**基本规则**

- URL 用名词复数表示集合：`/v1/orders`、`/v1/customers`。
- 层级表达归属：`/v1/customers/{customerId}/orders`。
- 路径参数用 camelCase 或 snake_case（**团队统一**，不要混用）。
- 过滤/排序/分页放查询参数：`?status=pending&sort=-createdAt&limit=20`。
- 动作型端点用子资源或动词（团队约定）：`POST /v1/orders/{id}/cancel`。
- 不在 URL 里放大写、空格、动词混用（`/getOrder`）。
- ID 使用不可猜测的公开标识（如 ULID/UUID），避免自增 ID 泄露业务规模与遍历风险。

**一个订单域的完整资源设计**

```text
GET    /v1/orders?status=pending&limit=20&cursor=...
POST   /v1/orders
GET    /v1/orders/{orderId}
PATCH  /v1/orders/{orderId}            # 仅允许改地址等可改字段
POST   /v1/orders/{orderId}/cancel
POST   /v1/orders/{orderId}/refunds
GET    /v1/orders/{orderId}/events     # 审计/时间线
GET    /v1/orders/{orderId}/items
POST   /v1/orders/{orderId}/items
DELETE /v1/orders/{orderId}/items/{itemId}
```

**不要暴露内部结构**：数据库表名、内部状态枚举、自增主键不应直接映射为 API 契约，否则重构会被契约锁死。

### 7.2 请求与响应约定

**创建请求**

```json
POST /v1/orders
Idempotency-Key: 8f1d2c9e-...
Content-Type: application/json

{
  "customerId": "cus_01H...",
  "currency": "CNY",
  "lines": [
    { "sku": "SKU-1001", "quantity": 2 }
  ],
  "shippingAddress": {
    "recipient": "张三",
    "phone": "+8613800000000",
    "province": "浙江省",
    "city": "杭州市",
    "detail": "西湖区..."
  }
}
```

**响应（201）**

```json
{
  "id": "ord_01H...",
  "status": "pending",
  "currency": "CNY",
  "total": { "minor": 19900, "display": "199.00" },
  "createdAt": "2026-11-21T08:15:00Z",
  "links": {
    "self": "/v1/orders/ord_01H...",
    "cancel": "/v1/orders/ord_01H.../cancel"
  }
}
```

设计要点：

- **金额**同时给最小单位与展示值，明确 `currency`。
- **时间**统一 UTC ISO-8601，带 `Z`。
- **枚举**用字符串，不用数字（可读、可演进）。
- **不要**在响应里回显请求的全部内容（可能含敏感字段）。
- **可选字段**：`null` 与“字段缺失”要定义清楚，移动端解析要一致。

### 7.3 分页

**Offset 分页**

```sql
SELECT id, status, total_minor, created_at
FROM orders
WHERE customer_id = $1
ORDER BY created_at DESC, id DESC
LIMIT $2 OFFSET $3;
```

- 优点：可跳页、实现简单。
- 缺点：深分页慢（`OFFSET 100000` 要扫描并丢弃）；数据变化时重复/遗漏。

**Cursor（Keyset）分页**

```sql
SELECT id, status, total_minor, created_at
FROM orders
WHERE customer_id = $1
  AND (created_at, id) < ($2, $3)     -- 上一页最后一条的排序键
ORDER BY created_at DESC, id DESC
LIMIT $4;
```

- 优点：稳定、O(limit) 复杂度，适合大数据量。
- 缺点：不能跳页；游标要编码并校验（防止客户端伪造/注入）。
- 排序键必须**唯一且稳定**（`created_at, id` 组合）。

**游标编码**：Base64(JSON) 并签名，防止客户端篡改绕过过滤条件：

```dart
String encodeCursor(Map<String, Object?> payload) =>
    base64Url.encode(utf8.encode(jsonEncode(payload)));

Map<String, Object?> decodeCursor(String raw) {
  final json = jsonDecode(utf8.decode(base64Url.decode(raw)));
  if (json is! Map<String, Object?>) {
    throw ValidationException('游标格式非法');
  }
  return json;
}
```

**分页元数据**

```json
{
  "items": [...],
  "page": {
    "limit": 20,
    "nextCursor": "eyJjcmVhdGVkQXQiOi...",
    "hasMore": true
  }
}
```

### 7.4 过滤、排序与字段选择

**白名单是硬要求**

```dart
const _sortable = {'createdAt': 'created_at', 'total': 'total_minor'};
const _filterable = {'status': 'status', 'customerId': 'customer_id'};

String buildOrderBy(String? raw) {
  if (raw == null || raw.isEmpty) return 'created_at DESC, id DESC';
  final descending = raw.startsWith('-');
  final key = descending ? raw.substring(1) : raw;
  final column = _sortable[key];
  if (column == null) throw ValidationException('不支持的排序字段：$key');
  return '$column ${descending ? 'DESC' : 'ASC'}, id DESC';
}
```

**为什么不能直接拼接**：`ORDER BY (SELECT ...)` 可被用于时间盲注或拖库。

**IN 查询的批量限制**：`WHERE id = ANY($1)` 时限制数组长度（如 100），防止超大查询。

**字段选择（sparse fieldsets）**：`?fields=id,status` 要白名单化，且不能因为“没请求这个字段”就跳过授权检查。

### 7.5 幂等性设计

**问题**：客户端超时重试、网关重发、用户双击，都会导致重复下单/重复扣款。

**方案**：客户端提供 `Idempotency-Key`，服务端持久化“键 → 结果”。

```sql
CREATE TABLE idempotency_keys (
  key            text PRIMARY KEY,
  request_hash   text NOT NULL,
  status         text NOT NULL,          -- in_progress | completed | failed
  response_code  int,
  response_body  jsonb,
  created_at     timestamptz NOT NULL DEFAULT now(),
  expires_at     timestamptz NOT NULL
);

CREATE INDEX idx_idempotency_expires ON idempotency_keys (expires_at);
```

**执行流程**

```text
1. 读取 Idempotency-Key（缺失 → 400）
2. 计算请求体哈希
3. 尝试 INSERT (key, hash, 'in_progress')
   ├─ 成功 → 执行业务，写入结果，返回
   ├─ 唯一冲突 → 读取已存在记录
   │    ├─ hash 不一致 → 409（同键不同请求）
   │    ├─ status=completed → 原样返回结果（可加 Idempotent-Replayed 头）
   │    ├─ status=in_progress → 409/425（稍后重试）
   │    └─ status=failed → 允许重试或返回失败
   └─ 其他错误 → 500
4. 事务提交后再标记 completed（避免“标记成功但业务回滚”）
```

**关键细节**

- 幂等键要绑定**用户/租户**，防止跨用户猜键复用。
- 保留期（如 24h）要有清理任务。
- 并发同键请求必须由数据库唯一约束裁决，不能只靠内存锁（多实例）。
- 响应要包含 `Idempotency-Replayed: true`（可选）便于观测。

### 7.6 版本策略

**三种常见方案**

| 方案 | 示例 | 优点 | 缺点 |
|---|---|---|---|
| URL 版本 | `/v1/orders` | 直观、易路由、易缓存 | 大版本切换成本高 |
| Header 版本 | `Accept: application/vnd.api.v2+json` | URL 稳定 | 不直观、调试麻烦 |
| 字段演进（无版本） | 只增不删 | 客户端友好 | 破坏性变更仍需要窗口 |

**推荐**：以**向后兼容演进**为主，URL 大版本为兜底。

**兼容规则**

- 新增可选字段：兼容。
- 新增必填请求字段：**破坏**。
- 删除/重命名字段：**破坏**。
- 改变字段类型（string → number）：**破坏**。
- 改变枚举含义、单位、时区：**破坏**。
- 收紧校验（原本允许的值现在拒绝）：**破坏**。

**弃用流程**

```text
1. 文档标注 @deprecated + 替代方案
2. 响应头加 Sunset / Deprecation（RFC 8594 / RFC 9745）
3. 埋点统计旧接口调用量与调用方
4. 通知调用方，给出迁移窗口
5. 到期限流 → 下线
```

### 7.7 错误模型

**统一结构**

```json
{
  "error": {
    "code": "insufficient_inventory",
    "message": "库存不足",
    "status": 409,
    "requestId": "01J8Z...",
    "details": [
      { "field": "lines[0].quantity", "reason": "requested_gt_available", "available": 3 }
    ],
    "docs": "https://api.example.com/docs/errors/insufficient_inventory"
  }
}
```

或采用 **RFC 9457 Problem Details**：

```json
{
  "type": "https://api.example.com/problems/insufficient-inventory",
  "title": "库存不足",
  "status": 409,
  "detail": "SKU-1001 仅剩 3 件",
  "instance": "/v1/orders"
}
```

**错误码设计原则**

- 机器可读、稳定、`snake_case`：`invalid_credentials`、`rate_limited`。
- 不复用 HTTP 状态码作为唯一语义（409 可能是库存不足，也可能是版本冲突）。
- `message` 可改，`code` 不可随意改。
- 不返回堆栈、SQL、内部主机名。
- 字段级错误用于表单，顶层 `code` 用于流程分支。

### 7.8 OpenAPI 契约

```yaml
openapi: 3.1.0
info:
  title: Order API
  version: 1.4.0
servers:
  - url: https://api.example.com
paths:
  /v1/orders:
    post:
      summary: 创建订单
      parameters:
        - name: Idempotency-Key
          in: header
          required: true
          schema: { type: string, format: uuid }
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/CreateOrderRequest' }
      responses:
        '201':
          description: 创建成功
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Order' }
        '409':
          $ref: '#/components/responses/Conflict'
        '422':
          $ref: '#/components/responses/ValidationError'
components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
  schemas:
    CreateOrderRequest:
      type: object
      required: [customerId, currency, lines]
      properties:
        customerId: { type: string }
        currency: { type: string, enum: [CNY, USD] }
        lines:
          type: array
          minItems: 1
          maxItems: 50
          items: { $ref: '#/components/schemas/OrderLineInput' }
```

**契约治理**

- 契约文件与实现同仓库，PR 中一起评审。
- CI 用 lint（如 Spectral/Redocly）检查命名与规范。
- 生成客户端（Dart/TypeScript）并纳入构建，避免手写偏差。
- 用契约测试验证实现符合 schema（见 16.7）。

### 7.9 GraphQL / gRPC / 实时 / Webhook 的适用边界

**GraphQL**

- 适合：多客户端、字段需求差异大、BFF 层聚合。
- 必须解决：字段级授权、查询深度/复杂度限制、N+1（DataLoader）、缓存、错误规范。
- 不适合：简单的 CRUD（引入过多复杂度）、大文件上传。

**gRPC**

- 适合：内部服务高性能 RPC、强类型契约、流式传输。
- 必须解决：浏览器支持（gRPC-Web）、代理兼容、超时/重试、版本演进（字段编号不复用）。
- Dart 有 `grpc` 包；需评估与现有网关的兼容。

**WebSocket / SSE**

- 适合：实时通知、协同编辑、交易行情、长任务进度。
- 必须解决：连接鉴权与续期、心跳、断线重连、消息去重、跨实例广播、连接数上限。

**Webhook**

- 发送方：签名（HMAC）、重试（指数退避）、投递记录、幂等 ID、超时。
- 接收方：验签、防重放（时间戳 + nonce）、幂等处理、快速 2xx + 异步处理。
- 必须提供重放/调试能力，否则对接方无法排障。

### 7.10 API 设计评审清单

- [ ] 资源命名一致，方法语义正确
- [ ] 每个端点明确认证与授权规则
- [ ] 写操作有幂等策略
- [ ] 分页、排序、过滤有上限与白名单
- [ ] 错误码、状态码、字段级错误一致
- [ ] 请求/响应示例真实且可执行
- [ ] 时间/金额/枚举/可空语义明确
- [ ] 破坏性变更与弃用策略明确
- [ ] 契约已生成客户端或至少生成类型
- [ ] 性能预期（QPS、P99、响应大小）已说明

---

## 8. 输入校验、序列化与错误处理

### 8.1 信任边界的完整清单

所有跨越信任边界的数据都必须校验：

| 来源 | 需要校验 |
|---|---|
| 路径参数 | 格式、长度、存在性、归属 |
| 查询参数 | 类型、范围、枚举、数量上限 |
| 请求头 | 认证、内容类型、长度、幂等键格式 |
| JSON body | 结构、类型、必填、长度、嵌套深度、数组长度 |
| 上传文件 | 大小、MIME、扩展名、内容魔数 |
| Cookie | 签名、过期、作用域 |
| 第三方响应 | 状态码、schema、超时、字段范围 |
| 队列消息 | 版本、schema、幂等键、重试次数 |
| 数据库读出的值 | 历史脏数据、枚举扩展、可空性 |

**“已经校验过”不等于“永远可信”**：历史数据可能是旧版本写入的，外部系统可能变更 schema。

### 8.2 组合式校验器

不依赖重型框架，用小的可组合函数即可：

```dart
typedef Validator<T> = String? Function(T value);

String? requiredString(String? v, {int min = 1, int max = 255}) {
  if (v == null || v.trim().isEmpty) return '不能为空';
  final t = v.trim();
  if (t.length < min) return '至少 $min 个字符';
  if (t.length > max) return '最多 $max 个字符';
  return null;
}

String? validEmail(String? v) {
  final base = requiredString(v, max: 320);
  if (base != null) return base;
  if (!v!.looksLikeEmail) return '邮箱格式不正确';
  return null;
}

String? Function(T) compose<T>(List<Validator<T>> validators) {
  return (T value) {
    for (final v in validators) {
      final error = v(value);
      if (error != null) return error;
    }
    return null;
  };
}
```

**DTO 校验示例**

```dart
final class CreateOrderInput {
  const CreateOrderInput({
    required this.customerId,
    required this.currency,
    required this.lines,
  });

  final String customerId;
  final String currency;
  final List<OrderLineInput> lines;

  static CreateOrderInput fromJson(Map<String, dynamic> json) {
    final errors = <String, String>{};

    final customerId = json['customerId'];
    final customerIdError = requiredString(
      customerId is String ? customerId : null,
      max: 64,
    );
    if (customerIdError != null) errors['customerId'] = customerIdError;

    final currency = json['currency'];
    if (currency is! String || !const {'CNY', 'USD'}.contains(currency)) {
      errors['currency'] = '不支持的币种';
    }

    final rawLines = json['lines'];
    final lines = <OrderLineInput>[];
    if (rawLines is! List || rawLines.isEmpty) {
      errors['lines'] = '至少包含一个商品';
    } else if (rawLines.length > 50) {
      errors['lines'] = '最多 50 行';
    } else {
      for (var i = 0; i < rawLines.length; i++) {
        final item = rawLines[i];
        if (item is! Map<String, dynamic>) {
          errors['lines[$i]'] = '格式非法';
          continue;
        }
        try {
          lines.add(OrderLineInput.fromJson(item));
        } on ValidationException catch (e) {
          errors['lines[$i]'] = e.message;
        }
      }
    }

    if (errors.isNotEmpty) {
      throw ValidationException('请求参数不合法', fieldErrors: errors);
    }

    return CreateOrderInput(
      customerId: (customerId! as String).trim(),
      currency: currency as String,
      lines: List.unmodifiable(lines),
    );
  }
}
```

**要点**

- 一次收集**所有**错误再返回，而不是遇到第一个就抛（表单体验）。
- 不要 `as Map<String, dynamic>` 直接强转外部 JSON：元素类型可能是 `Map<String, Object?>`，强转会抛 `TypeError`。
- 校验后构造不可变对象，后续代码不必重复判空。

### 8.3 JSON 的隐藏陷阱

| 陷阱 | 说明 | 应对 |
|---|---|---|
| 数字精度 | JSON 数字在 JS 侧只有 double 精度，大整数会丢精度 | 大整数用字符串传输 |
| 整数溢出 | Dart int 在原生平台是 64 位，Web 是 double | 服务端限制范围 |
| `null` vs 缺失 | 语义不同：显式置空 vs 不修改 | PATCH 语义写清楚 |
| 未知字段 | 忽略可能掩盖客户端错误 | 严格模式或记录告警 |
| 重复键 | JSON 允许重复键，解析取最后一个 | 入口做检测或依赖严格解析 |
| 深度炸弹 | 深层嵌套导致栈溢出 | 限制最大深度 |
| 超大数组 | 内存耗尽 | 限制元素数量 |
| Unicode | 代理对、非法序列 | 校验 UTF-8 合法性 |
| 枚举演进 | 客户端收到未知枚举 | 提供 `unknown` 兜底 |
| 时区 | 无时区的时间字符串歧义 | 强制 ISO-8601 UTC |

**时间建模**

```dart
DateTime parseInstant(String raw) {
  final dt = DateTime.tryParse(raw);
  if (dt == null) throw ValidationException('时间格式非法');
  if (!dt.isUtc && !raw.endsWith('Z') && !RegExp(r'[+-]\d{2}:\d{2}$').hasMatch(raw)) {
    throw ValidationException('时间必须包含时区');
  }
  return dt.toUtc();
}
```

**金额**：见 3.3 的 `Money` 值对象；JSON 中建议 `{ "minor": 19900, "currency": "CNY" }` 或仅传字符串 `"199.00"`，避免浮点。

### 8.4 错误处理的分层策略

```text
┌─────────────────────────────────────────────┐
│ 接口层：把异常映射为 HTTP + 统一错误体         │
├─────────────────────────────────────────────┤
│ 应用层：业务失败用显式异常/结果类型            │
├─────────────────────────────────────────────┤
│ 领域层：违反不变量 → DomainException          │
├─────────────────────────────────────────────┤
│ 基础设施：包装驱动异常为端口异常               │
└─────────────────────────────────────────────┘
```

**基础设施层的包装示例**

```dart
Future<T> _withDbErrors<T>(Future<T> Function() action) async {
  try {
    return await action();
  } on PoolException catch (e) {
    throw DependencyUnavailable('postgres', e);
  } on SocketException catch (e) {
    throw DependencyUnavailable('postgres', e);
  } on QueryException catch (e) {
    // 唯一约束冲突转为业务冲突
    if (e.code == '23505') {
      throw ConflictException('duplicate_resource', '资源已存在');
    }
    rethrow;
  }
}
```

**全局错误映射**

```dart
Response problemResponse(Object error, String requestId) {
  return switch (error) {
    ValidationException(:final message, :final fieldErrors) => _json(422, {
        'error': {
          'code': 'invalid_request',
          'message': message,
          'fields': fieldErrors,
          'requestId': requestId,
        }
      }),
    UnauthorizedException(:final message) => _json(401, {
        'error': {'code': 'unauthenticated', 'message': message, 'requestId': requestId}
      }),
    ForbiddenException(:final message) => _json(403, {
        'error': {'code': 'forbidden', 'message': message, 'requestId': requestId}
      }),
    NotFoundException(:final message) => _json(404, {
        'error': {'code': 'not_found', 'message': message, 'requestId': requestId}
      }),
    ConflictException(:final code, :final message) => _json(409, {
        'error': {'code': code, 'message': message, 'requestId': requestId}
      }),
    DependencyUnavailable(:final dependency) => _json(503, {
        'error': {
          'code': 'dependency_unavailable',
          'message': '服务暂时不可用，请稍后重试',
          'requestId': requestId,
        }
      }),
    _ => _json(500, {
        'error': {
          'code': 'internal_error',
          'message': '服务暂时不可用',
          'requestId': requestId,
        }
      }),
  };
}
```

**纪律**

- 4xx 是“客户端可以修正”的问题，日志级别 `info/warn`；5xx 是“服务端的问题”，必须告警。
- 永远不要把原始异常消息返回给客户端（可能含 SQL、路径、内网 IP）。
- 每个 5xx 都要有 requestId，便于用户报障时定位。
- 预期业务失败不要打 `error` 级日志，否则会淹没真正的故障。

### 8.5 部分失败与批量操作

批量接口必须定义语义：

```json
POST /v1/orders/batch
{
  "items": [ {...}, {...} ]
}
```

两种策略：

1. **全有或全无**（事务）：简单、可预测，但单个失败会使整批失败。
2. **部分成功**：返回每条结果，客户端处理失败项。

```json
{
  "results": [
    { "index": 0, "status": "created", "id": "ord_1" },
    { "index": 1, "status": "failed", "code": "insufficient_inventory" }
  ],
  "summary": { "succeeded": 1, "failed": 1 }
}
```

HTTP 状态码：部分成功用 `207 Multi-Status` 或 `200` + 结果体（团队约定，必须在文档写清）。

**注意**：批量接口要限制条数与总大小；大批量用异步任务 + 任务查询接口。

---

## 9. 数据库、SQL 与数据建模

数据库是后端最难替换、最容易成为瓶颈、也最容易在凌晨三点出事的部分。ORM 能省掉样板代码，但省不掉对事务、索引、锁和执行计划的理解。

### 9.1 数据建模方法

**从问题出发，而不是从表出发**

1. 列出**名词**：客户、订单、订单行、商品、库存、支付、退款。
2. 区分**实体**（有独立生命周期和身份）与**值对象**（由属性定义，如地址、金额）。
3. 明确**关系与基数**：1:1、1:N、M:N。
4. 明确**不变量**：订单总额 = 行小计之和；库存不能为负；一个用户同一时刻只有一个未支付订单（如业务要求）。
5. 明确**生命周期**：创建、变更、取消、归档、删除。
6. 再落表：主键、外键、唯一约束、检查约束、索引。

**规范化 vs 反规范化**

| | 规范化（3NF） | 反规范化 |
|---|---|---|
| 优点 | 无冗余、更新一致 | 查询快、少 JOIN |
| 缺点 | 多表 JOIN、复杂查询 | 冗余不一致风险、写放大 |
| 建议 | 事务性主数据默认规范化 | 读多写少的报表/快照可反规范化，并明确同步机制 |

**订单快照**是典型反规范化：订单行保存“下单时的商品名与单价”，而不是运行时 JOIN 商品表——因为商品会改价、改名、下架。

### 9.2 SQL 核心能力

**JOIN 类型**

```sql
-- INNER：两边都匹配
SELECT o.id, c.name
FROM orders o
JOIN customers c ON c.id = o.customer_id;

-- LEFT：保留左表全部
SELECT c.id, c.name, COUNT(o.id) AS order_count
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
GROUP BY c.id, c.name;

-- 找出没有订单的客户
SELECT c.id, c.name
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
WHERE o.id IS NULL;
```

**聚合与 HAVING**

```sql
SELECT customer_id, COUNT(*) AS cnt, SUM(total_minor) AS total
FROM orders
WHERE created_at >= now() - interval '30 days'
GROUP BY customer_id
HAVING SUM(total_minor) > 100000
ORDER BY total DESC
LIMIT 20;
```

**窗口函数**

```sql
-- 每个客户最近三笔订单
SELECT *
FROM (
  SELECT o.*,
         row_number() OVER (
           PARTITION BY customer_id
           ORDER BY created_at DESC, id DESC
         ) AS rn
  FROM orders o
) t
WHERE rn <= 3;
```

**CTE 与递归**

```sql
-- 组织树
WITH RECURSIVE tree AS (
  SELECT id, parent_id, name, 1 AS depth
  FROM categories WHERE parent_id IS NULL
  UNION ALL
  SELECT c.id, c.parent_id, c.name, t.depth + 1
  FROM categories c
  JOIN tree t ON c.parent_id = t.id
  WHERE t.depth < 5
)
SELECT * FROM tree ORDER BY depth, name;
```

**UPSERT**

```sql
INSERT INTO inventory (sku, available)
VALUES ($1, $2)
ON CONFLICT (sku) DO UPDATE
SET available = inventory.available + EXCLUDED.available,
    updated_at = now()
RETURNING available;
```

**条件更新（避免竞态）**

```sql
-- 只有库存足够才扣减，返回受影响行数
UPDATE inventory
SET available = available - $2,
    updated_at = now()
WHERE sku = $1
  AND available >= $2
RETURNING available;
```

受影响行数为 0 → 库存不足。这是比“先 SELECT 再 UPDATE”更安全的做法。

### 9.3 索引原理与实践

**B-tree 索引**是默认选择，适合等值、范围、排序。

**复合索引的最左前缀原则**

```sql
CREATE INDEX idx_orders_customer_status_created
ON orders (customer_id, status, created_at DESC);
```

可支持：

- `WHERE customer_id = ?`
- `WHERE customer_id = ? AND status = ?`
- `WHERE customer_id = ? AND status = ? ORDER BY created_at DESC`

**不能**高效支持：

- `WHERE status = ?`（跳过最左列）
- `WHERE customer_id = ? ORDER BY created_at DESC`（中间列缺失，排序仍需额外处理）

**覆盖索引**：查询所需列都在索引中，避免回表。

```sql
CREATE INDEX idx_orders_covering
ON orders (customer_id, created_at DESC) INCLUDE (status, total_minor);
```

**部分索引**：只为常用子集建索引，减少体积。

```sql
CREATE INDEX idx_orders_pending
ON orders (created_at)
WHERE status = 'pending';
```

**唯一索引 = 业务约束**，不要只靠应用层检查：

```sql
CREATE UNIQUE INDEX uq_users_email_lower ON users (lower(email));
```

**JSONB 索引**

```sql
CREATE INDEX idx_orders_meta_gin ON orders USING gin (meta jsonb_path_ops);
-- 查询：WHERE meta @> '{"channel":"app"}'
```

**全文检索**

```sql
CREATE INDEX idx_products_fts ON products
USING gin (to_tsvector('simple', name || ' ' || coalesce(description, '')));
```

**索引的代价**

- 每次 INSERT/UPDATE/DELETE 都要维护索引 → 写放大。
- 占用磁盘与内存（缓存命中率）。
- 过多索引会让优化器选择变差、VACUUM 变慢。
- 定期用 `pg_stat_user_indexes` 找出从未使用的索引。

### 9.4 执行计划解读

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, status FROM orders
WHERE customer_id = 'c_123' AND status = 'pending'
ORDER BY created_at DESC
LIMIT 20;
```

需要关注的信号：

| 信号 | 含义 | 处理 |
|---|---|---|
| `Seq Scan`（大表） | 全表扫描 | 补索引或改写查询 |
| `rows=1000` 实际 `actual rows=100000` | 统计信息不准 | `ANALYZE`、调统计目标 |
| `Nested Loop` + 大表 | 可能 N+1 式放大 | 改 JOIN 或补索引 |
| `Sort` + 大量行 | 排序溢出磁盘 | 用索引满足排序 |
| `Buffers: read` 很高 | 缓存未命中 | 加内存/优化查询 |
| `Filter` 后大量行被丢弃 | 索引选择性差 | 调整索引列顺序 |

**慢查询定位**

```sql
SELECT query, calls, mean_exec_time, total_exec_time
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

### 9.5 事务、隔离级别与锁

**ACID**

- **Atomicity**：全成功或全失败。
- **Consistency**：约束始终成立。
- **Isolation**：并发事务互不干扰（按隔离级别）。
- **Durability**：提交后即使崩溃也不丢。

**隔离级别与异常现象**

| 隔离级别 | 脏读 | 不可重复读 | 幻读 | 写偏斜 |
|---|---|---|---|---|
| Read Uncommitted | 可能 | 可能 | 可能 | 可能 |
| Read Committed（PG 默认） | 否 | 可能 | 可能 | 可能 |
| Repeatable Read | 否 | 否 | PG 中否 | 可能 |
| Serializable | 否 | 否 | 否 | 否 |

**并发更新的两种策略**

```sql
-- 乐观锁：版本号
UPDATE orders
SET status = 'paid', version = version + 1
WHERE id = $1 AND version = $2
RETURNING version;
-- 0 行受影响 → 版本冲突 → 返回 409 或重试

-- 悲观锁：SELECT FOR UPDATE
BEGIN;
SELECT * FROM inventory WHERE sku = $1 FOR UPDATE;
UPDATE inventory SET available = available - $2 WHERE sku = $1;
COMMIT;
```

**死锁**：两个事务以相反顺序锁同一批行。处理方式：

1. 统一加锁顺序（如按主键升序）。
2. 缩短事务。
3. 捕获死锁错误（PostgreSQL `40P01`）并有限重试。

**长事务的危害**：阻塞 VACUUM、持有快照导致膨胀、占用连接。禁止在事务中调用外部 HTTP。

### 9.6 连接池

**为什么需要**：建立数据库连接涉及 TCP + TLS + 认证，成本高；连接数是数据库的硬上限。

**池大小估算**（PostgreSQL 经验法则）：

```text
单实例连接数 ≈ CPU 核数 × 2 ~ 4     （I/O 密集可略高，但别盲目调大）
总连接数 = 单实例池大小 × 实例数 + 迁移/运维预留
必须 < 数据库 max_connections × 0.8
```

**池参数**

| 参数 | 作用 | 建议 |
|---|---|---|
| `maxConnections` | 池上限 | 按上面公式 |
| `acquireTimeout` | 等待连接超时 | 小于请求预算 |
| `idleTimeout` | 空闲连接回收 | 与数据库/代理超时协调 |
| `maxLifetime` | 连接最大寿命 | 避免长期连接遇到网络设备超时 |
| 语句超时 | 单查询上限 | 必设 |

**池耗尽的典型症状**：请求延迟阶梯式上升、超时集中爆发、日志出现“等待连接超时”。根因通常是慢查询、长事务、连接泄漏或流量突增。

### 9.7 Dart 中的 PostgreSQL 访问

以 `postgres` 包（v3+）为例：

```dart
import 'package:postgres/postgres.dart';

final class Db {
  Db._(this._pool);

  static Future<Db> connect(String url, {int maxConnections = 20}) async {
    final endpoint = Endpoint.parse(url);
    final pool = Pool.withEndpoints(
      [endpoint],
      settings: PoolSettings(
        maxConnectionCount: maxConnections,
        connectTimeout: const Duration(seconds: 5),
        queryTimeout: const Duration(seconds: 10),
        applicationName: 'order_service',
      ),
    );
    return Db._(pool);
  }

  final Pool _pool;

  Future<List<ResultRow>> query(String sql, {Object? parameters}) =>
      _pool.execute(Sql.named(sql), parameters: parameters);

  Future<T> transaction<T>(Future<T> Function(TxSession tx) action) =>
      _pool.runTx((session) async => action(session));

  Future<void> close() => _pool.close();
}
```

**参数化查询（必须）**

```dart
// ✅ 命名参数
final rows = await db.query(
  'SELECT id, status FROM orders WHERE customer_id = @cid AND status = @status',
  parameters: {'cid': customerId, 'status': 'pending'},
);

// ✅ 位置参数（PostgreSQL $1/$2）
final rows2 = await db.query(
  r'SELECT id FROM orders WHERE id = $1',
  parameters: [orderId],
);

// ❌ 拼接：SQL 注入
final bad = await db.query("SELECT * FROM orders WHERE id = '$orderId'");
```

**事务示例**

```dart
final result = await db.transaction((tx) async {
  final updated = await tx.execute(
    Sql.named('''
      UPDATE inventory
      SET available = available - @qty
      WHERE sku = @sku AND available >= @qty
      RETURNING available
    '''),
    parameters: {'sku': sku, 'qty': qty},
  );

  if (updated.isEmpty) {
    throw ConflictException('insufficient_inventory', '库存不足');
  }

  return tx.execute(
    Sql.named('INSERT INTO order_lines (order_id, sku, quantity) VALUES (@oid, @sku, @qty)'),
    parameters: {'oid': orderId, 'sku': sku, 'qty': qty},
  );
});
```

**批量写入**：逐行 INSERT 很慢，用 `COPY` 或批量多值 INSERT。

```dart
final values = <String>[];
final params = <String, Object?>{};
for (var i = 0; i < rows.length; i++) {
  values.add('(@id$i, @sku$i, @qty$i)');
  params['id$i'] = rows[i].id;
  params['sku$i'] = rows[i].sku;
  params['qty$i'] = rows[i].qty;
}
await db.query(
  'INSERT INTO order_lines (order_id, sku, quantity) VALUES ${values.join(',')}',
  parameters: params,
);
```

注意：占位符数量要分批（如每批 500 行），避免 SQL 文本过长。

**LISTEN/NOTIFY**：轻量实时通知，适合缓存失效广播。

```dart
final channel = await db.listen('order_events');
await for (final notification in channel) {
  cache.invalidate('order:${notification.payload}');
}
```

**注意**：NOTIFY 不保证持久化，连接断开期间的通知会丢失；关键事件用消息队列。

### 9.8 ORM 与 Repository 模式

**对比**

| 方式 | 优点 | 缺点 | 适用 |
|---|---|---|---|
| 原生 SQL | 完全控制、性能可预测 | 样板代码、手写映射 | 复杂查询、性能敏感 |
| 查询构造器 | 类型安全、可组合 | 复杂查询表达受限 | 中等复杂度 |
| 全功能 ORM | 开发快、关系管理 | 隐式查询、N+1、学习成本 | CRUD 密集 |
| 代码生成 ORM | 类型安全 + 迁移 | 生成步骤、框架绑定 | 新项目 |

**Repository 接口**

```dart
abstract interface class OrderRepository {
  Future<Order?> findById(OrderId id);
  Future<void> save(Order order);
  Future<List<Order>> pageByCustomer(
    CustomerId customerId, {
    String? cursor,
    int limit = 20,
  });
  Future<int> countPending(CustomerId customerId);
}
```

**实现要点**

- 领域对象 ↔ 数据行映射集中在一处，不要散落在 service。
- `save` 语义要明确：insert-only、upsert、乐观锁更新。
- 查询返回分页对象，而不是无限 `List`。
- 不要暴露 `ResultRow` 给上层。

**避免 N+1**

```dart
// ❌ 每个订单查一次客户
for (final o in orders) {
  final c = await customerRepo.findById(o.customerId);
}

// ✅ 批量加载
final ids = orders.map((o) => o.customerId).toSet();
final customers = await customerRepo.findByIds(ids);
final byId = {for (final c in customers) c.id: c};
for (final o in orders) {
  final c = byId[o.customerId];
  // ...
}
```

或直接在 SQL 中 JOIN。

### 9.9 迁移策略

**工具形态**：SQL 文件按序执行（`0001_init.sql`）、代码内迁移（`drift`）、框架迁移（Serverpod）。

**版本表**

```sql
CREATE TABLE schema_migrations (
  version     text PRIMARY KEY,
  applied_at  timestamptz NOT NULL DEFAULT now(),
  checksum    text NOT NULL
);
```

**零停机迁移：Expand → Migrate → Contract**

```text
旧代码 A，新代码 B，目标：从 schema S1 → S2

1. Expand：加新列/新表（可空、有默认），A 与 B 都能运行
2. 部署 B（双写：同时写旧列与新列）
3. 回填历史数据（分批、限速）
4. 校验一致性
5. 切换读路径到新列
6. 停止写旧列
7. Contract：删除旧列/旧索引（在确认无回滚需求后）
```

**危险操作清单**

| 操作 | 风险 | 替代 |
|---|---|---|
| `ALTER TABLE ... ADD COLUMN NOT NULL`（无默认） | 锁表/失败 | 先加可空列 → 回填 → 加约束 |
| `ALTER COLUMN TYPE` | 全表重写 | 新增列 + 双写 + 切换 |
| 建索引（大表） | 长时间锁写 | `CREATE INDEX CONCURRENTLY` |
| 大量 `UPDATE` | 膨胀、锁、复制延迟 | 分批提交 |
| 删除列 | 旧代码崩溃 | 先停止引用，再删除 |

**迁移纪律**

- 迁移文件不可修改（已应用的要新加文件）。
- 每个迁移都要能在**生产数据快照**上跑通并计时。
- 迁移与应用发布解耦，先迁移后发布（向后兼容）。
- 有回滚方案或明确的前向修复方案。

### 9.10 分区、读写分离与分片

**分区**（PostgreSQL 声明式）

```sql
CREATE TABLE orders (
  id text NOT NULL,
  created_at timestamptz NOT NULL,
  ...
) PARTITION BY RANGE (created_at);

CREATE TABLE orders_2026_11 PARTITION OF orders
FOR VALUES FROM ('2026-11-01') TO ('2026-12-01');
```

适合时间序列、可按分区删除（归档）的场景。注意：跨分区查询、唯一约束（需含分区键）的限制。

**读写分离**：主库写、只读副本读。必须处理**复制延迟**：

- 写后立即读 → 强制走主库（读己之写）。
- 关键一致性查询走主库。
- 监控复制滞后，超过阈值自动降级为主库读。

**分片**：按租户/用户哈希拆分。代价：跨分片查询、分布式事务、运维复杂度剧增。除非单库确实到瓶颈，否则不要过早分片。

### 9.11 备份、恢复与数据生命周期

**备份类型**

- 全量 + WAL 归档（PITR）：可恢复到任意时间点。
- 逻辑备份（`pg_dump`）：跨版本迁移方便，恢复慢。
- 快照：云盘快照，恢复快，依赖存储一致性。

**必须回答的问题**

1. RPO（可接受丢多少数据）？RTO（多长时间恢复）？
2. 备份多久做一次？保留多久？存在哪（跨区/跨账号）？
3. 谁有恢复权限？如何防止误删备份？
4. 最近一次**成功恢复演练**是什么时候？

**数据生命周期**：热数据（主库）→ 温数据（归档表/低成本存储）→ 冷数据（对象存储）→ 删除。合规要求（如 GDPR 删除权）必须在设计中考虑，而不是事后清理。

### 9.12 NoSQL 的适用边界

| 类型 | 代表 | 适合 | 不适合 |
|---|---|---|---|
| 键值/缓存 | Redis | 会话、限流、热点缓存、排行榜 | 作为唯一持久数据源 |
| 文档 | MongoDB 类 | 结构多变的内容、聚合读取 | 强事务、复杂多表关联 |
| 宽列 | Cassandra 类 | 海量写入、时序、多地域 | 灵活查询、聚合分析 |
| 搜索 | Elasticsearch/OpenSearch | 全文检索、日志分析 | 事务性主数据 |
| 时序 | Timescale/InfluxDB | 指标、监控 | 事务业务数据 |

**选择依据**：访问模式 + 一致性要求 + 数据量 + 运维能力。不要因为“听说 NoSQL 扩展性好”就放弃关系模型。

---

## 10. 缓存、搜索与文件存储

### 10.1 缓存的价值与风险

缓存本质是**用一致性换延迟与成本**。引入前必须回答：

1. 数据能容忍多久的陈旧？(秒 / 分钟 / 可以一直旧)
2. 缓存未命中时，回源压力是否可控？
3. 缓存失效时，系统是否还能工作？
4. 缓存键是否包含所有影响结果的维度（用户、租户、语言、权限）？

### 10.2 常见缓存模式

**Cache-Aside（旁路缓存，最常用）**

```dart
Future<Product> getProduct(String id) async {
  final key = 'product:v3:$id';
  final cached = await redis.get(key);
  if (cached != null) {
    return Product.fromJson(jsonDecode(cached));
  }

  final product = await repo.findById(id) ?? (throw NotFoundException('product', id));
  await redis.set(key, jsonEncode(product.toJson()), ex: Duration(minutes: 5));
  return product;
}

Future<void> updateProduct(Product p) async {
  await repo.save(p);
  await redis.del('product:v3:${p.id}');   // 删除而非更新，避免并发写覆盖
}
```

**Write-Through**：写时同时写缓存与数据库。一致性好，但写延迟增加、可能缓存无用数据。

**Write-Behind**：先写缓存，异步刷库。性能最好，但宕机会丢数据，实现复杂。

**Refresh-Ahead**：在过期前异步刷新，避免冷启动抖动。

### 10.3 三大缓存故障

**缓存穿透**：查询不存在的数据，每次穿透到数据库。

```dart
// 方案一：缓存空值（短 TTL）
await redis.set(key, '__NULL__', ex: const Duration(seconds: 30));

// 方案二：布隆过滤器前置判断
if (!bloom.mightContain(id)) throw NotFoundException('product', id);
```

**缓存击穿**：热点键过期瞬间，大量请求同时回源。

```dart
// 单飞（single-flight）：同一键只允许一个回源
final _inFlight = <String, Future<Object?>>{};

Future<T> _loadOnce<T>(String key, Future<T> Function() loader) {
  final existing = _inFlight[key];
  if (existing != null) return existing.then((v) => v as T);

  final future = loader().whenComplete(() => _inFlight.remove(key));
  _inFlight[key] = future;
  return future;
}
```

多实例场景需要分布式锁（Redis `SET NX PX`）或逻辑过期（缓存不过期，后台刷新）。

**缓存雪崩**：大量键同时过期或缓存整体不可用。

- TTL 加随机抖动：`ttl = base + random(0, base*0.2)`。
- 多级缓存（本地 + Redis）。
- 缓存不可用时限流 + 降级，保护数据库。

### 10.4 缓存一致性策略

| 策略 | 一致性 | 复杂度 | 说明 |
|---|---|---|---|
| 先更新 DB 再删缓存 | 较高 | 低 | 推荐默认；仍有极短不一致窗口 |
| 延迟双删 | 较高 | 中 | 更新后延迟再删一次，覆盖并发读回填 |
| 订阅 binlog/CDC 失效 | 高 | 高 | 异步但有延迟 |
| 版本号/短 TTL | 最终一致 | 低 | 简单可靠，接受短暂陈旧 |

**为什么“删除”优于“更新”**：并发写时，两次更新可能以错误顺序落到缓存，导致缓存长期保存旧值。删除让下次读回源，更安全。

### 10.5 缓存键设计

```text
{namespace}:{version}:{entity}:{id}:{variant}
例如：
  prod:product:v3:p_123:zh-CN
  prod:order:v2:o_456:summary
  prod:user:v1:u_789:permissions
```

规则：

- 包含版本号，schema 变化时可整体切换。
- 包含租户/语言/权限变体，避免串数据。
- 键要短（内存与网络成本），避免放敏感信息（键可能出现在监控里）。
- 统一前缀便于按业务清理与统计。

### 10.6 Redis 能力与使用边界

| 结构 | 用途 | 注意 |
|---|---|---|
| String | 缓存、计数器 | `INCR` 原子；大 value 要限制 |
| Hash | 对象字段部分更新 | 适合小对象；大 hash 操作要小心 |
| List | 队列（简单） | 无 ACK，不适合关键任务 |
| Set | 去重、标签 | 大集合操作成本 |
| Sorted Set | 排行榜、延迟队列 | 延迟队列需轮询 |
| Stream | 消息流（有消费组） | 比 List 更适合可靠消费 |
| HyperLogLog | 基数估算 | 有误差 |
| Pub/Sub | 广播 | 不持久，断线丢失 |

**分布式锁**

```dart
Future<bool> tryLock(String key, Duration ttl) async {
  final ok = await redis.set(key, token, nx: true, px: ttl.inMilliseconds);
  return ok == 'OK';
}
```

注意：锁续期、误删他人锁（用 token 校验 + Lua 脚本）、锁失效后的正确性（Redlock 有争议，关键场景用数据库约束/Fencing Token）。

**限流（令牌桶，Redis + Lua 原子执行）**

```lua
-- KEYS[1]=key, ARGV[1]=capacity, ARGV[2]=refillPerSec, ARGV[3]=nowMs
local data = redis.call('HMGET', KEYS[1], 'tokens', 'ts')
local tokens = tonumber(data[1]) or tonumber(ARGV[1])
local ts = tonumber(data[2]) or tonumber(ARGV[3])
local delta = math.max(0, tonumber(ARGV[3]) - ts) / 1000.0
tokens = math.min(tonumber(ARGV[1]), tokens + delta * tonumber(ARGV[2]))
local allowed = tokens >= 1
if allowed then tokens = tokens - 1 end
redis.call('HMSET', KEYS[1], 'tokens', tokens, 'ts', ARGV[3])
redis.call('PEXPIRE', KEYS[1], 60000)
return allowed and 1 or 0
```

### 10.7 搜索

**索引同步**

```text
写路径：DB 提交 → outbox → 消费者 → 更新搜索索引
或：CDC（Debezium 类）→ 索引
```

**必须处理**

- 索引延迟：搜索结果是最终一致，UI 要提示“可能延迟”。
- 删除同步：软删除标记也要同步到索引。
- 重建：索引 schema 变化时，建新索引 → 双写 → 切别名 → 删旧索引。
- 权限过滤：搜索前/后都要按用户权限过滤，索引不是授权系统。
- 查询限制：结果数上限、复杂查询拒绝、超时。

### 10.8 对象存储与上传

**预签名上传流程**

```text
1. 客户端 POST /v1/uploads  { fileName, contentType, size }
2. 服务端校验（大小、类型、配额）→ 生成对象键 uploads/{tenant}/{uuid}
3. 返回预签名 PUT URL（5 分钟有效）
4. 客户端直传对象存储
5. 客户端通知 /v1/uploads/{id}/complete
6. 服务端 HEAD 校验对象存在、大小、类型（可选病毒扫描）
7. 写入业务记录，标记可用
8. 定时任务清理未完成的孤儿对象
```

**安全要点**

- 对象键由服务端生成，禁止客户端指定路径（防路径穿越与覆盖）。
- 限制 `Content-Length` 与 content-type 白名单。
- 下载使用短期签名 URL 或受控代理，不暴露存储桶。
- 存储桶默认私有，禁止公共读。
- 敏感文件加密（服务端加密 SSE）。
- 图片处理（缩放/转码）在独立服务中做，避免主服务被大图拖垮。

### 10.9 CDN 与边缘缓存

- 静态资源（JS/CSS/图片）用 CDN，设置长 `max-age` + 内容哈希文件名。
- API 响应经 CDN 缓存时必须使用 `Vary` 与正确的 `Cache-Control`，个人数据用 `private, no-store`。
- 缓存清除（purge）要有运维流程与权限控制。
- 边缘计算（如 Cloudflare Workers）可做鉴权前置、A/B、重写，但不要放核心业务逻辑。

---

## 11. 身份认证、授权与应用安全

安全不是一个模块，而是一条贯穿需求、设计、编码、部署、运维的约束线。本章按“先分清概念 → 再选方案 → 最后逐项防御”的顺序展开。

### 11.1 威胁建模入门（STRIDE）

在设计阶段花 30 分钟做一次威胁建模，胜过上线后补十个补丁。

| 威胁 | 含义 | 后端例子 | 对策 |
|---|---|---|---|
| **S**poofing | 冒充身份 | 伪造 JWT、盗用会话 | 强认证、签名校验、MFA |
| **T**ampering | 篡改数据 | 改请求体里的价格 | 服务端重算、完整性校验、签名 |
| **R**epudiation | 否认操作 | 用户否认下过单 | 审计日志、不可否认记录 |
| **I**nformation disclosure | 信息泄露 | 错误响应含 SQL、日志含令牌 | 脱敏、最小返回、访问控制 |
| **D**enial of service | 拒绝服务 | 超大请求、复杂查询 | 限流、大小限制、超时 |
| **E**levation of privilege | 提权 | 普通用户调用管理接口 | 服务端授权、最小权限 |

**做法**：画出数据流图（客户端 → 网关 → 服务 → DB/第三方），在每个“跨越信任边界”的箭头上标注上述六类威胁与缓解措施。

### 11.2 认证方案对比

| 方案 | 适用 | 优点 | 风险/成本 |
|---|---|---|---|
| 服务端会话（Session + Cookie） | 传统 Web、同域前端 | 可即时撤销、令牌不暴露给 JS（HttpOnly） | 需要会话存储；跨域/移动端支持较弱 |
| JWT（自签） | 服务间、短期令牌 | 无状态、易水平扩展 | 难撤销、密钥管理、过期与刷新复杂 |
| OAuth 2.0 + OIDC | 第三方登录、SSO | 标准化、用户不需交出密码 | 流程复杂、redirect URI 校验易错 |
| API Key | 服务集成、开发者 API | 简单、易轮换 | 不适合用户身份；泄露风险高 |
| mTLS | 服务间零信任 | 强身份、双向校验 | 证书生命周期管理复杂 |
| Passkey/WebAuthn | 面向用户的现代登录 | 抗钓鱼、无密码 | 客户端支持与迁移成本 |

### 11.3 服务端会话

```text
1. 登录成功 → 生成 256 位随机 sessionId
2. 存储：Redis `sess:{id}` → { userId, tenantId, createdAt, lastSeen, mfa }
3. Set-Cookie: sid=<id>; HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=...
4. 每次请求：查会话 → 校验过期/吊销 → 滑动续期（有上限）
5. 登出：删除会话；修改密码：吊销该用户全部会话
```

**必须做**

- `HttpOnly`（防 XSS 窃取）、`Secure`（仅 HTTPS）、`SameSite=Lax/Strict`（缓解 CSRF）。
- 会话 ID 高熵、不可预测，不用自增或时间戳。
- 登录后**轮换** sessionId（防会话固定攻击）。
- 绝对过期 + 空闲过期双重限制。
- 敏感操作（改密码、支付）要求二次验证或新鲜度检查。
- 会话存储不可用时的降级策略要明确（通常拒绝服务而非放行）。

### 11.4 JWT 的正确用法

**结构**：`header.payload.signature`，Base64URL 编码，**不是加密**，任何人都能读 payload。

```json
{
  "alg": "RS256",
  "typ": "JWT",
  "kid": "2026-11"
}
```

```json
{
  "iss": "https://auth.example.com",
  "sub": "usr_123",
  "aud": "order-api",
  "exp": 1763712000,
  "iat": 1763711100,
  "jti": "01J8Z...",
  "tenant": "t_42",
  "scope": "orders:write"
}
```

**校验清单（缺一不可）**

1. 签名算法白名单（拒绝 `none`，拒绝把 RS256 降级为 HS256）。
2. `iss` 必须等于预期签发者。
3. `aud` 必须包含本服务标识。
4. `exp` / `nbf` 校验，允许少量时钟偏移（如 60s）。
5. `kid` 对应到受信任的公钥（JWKS），支持轮换与缓存。
6. 令牌类型（access vs refresh）不能混用。
7. 不要把 JWT 放在 URL 查询参数里（会进日志、Referer）。

**Dart 中的实现建议**

```dart
final class TokenValidator {
  TokenValidator({required this.jwks, required this.issuer, required this.audience});

  final JwksProvider jwks;
  final String issuer;
  final String audience;

  Future<Principal> validate(String token) async {
    final parts = token.split('.');
    if (parts.length != 3) throw UnauthorizedException('令牌格式非法');

    final header = jsonDecode(utf8.decode(base64Url.decode(base64Url.normalize(parts[0]))));
    final alg = header['alg'];
    if (alg != 'RS256' && alg != 'ES256') {
      throw UnauthorizedException('不支持的签名算法');
    }

    final key = await jwks.keyFor(header['kid'] as String?);
    final valid = verifySignature(parts[0], parts[1], parts[2], key, alg as String);
    if (!valid) throw UnauthorizedException('签名无效');

    final claims = jsonDecode(utf8.decode(base64Url.decode(base64Url.normalize(parts[1]))))
        as Map<String, dynamic>;

    if (claims['iss'] != issuer) throw UnauthorizedException('签发者不匹配');
    final aud = claims['aud'];
    final audOk = aud is String ? aud == audience : (aud is List && aud.contains(audience));
    if (!audOk) throw UnauthorizedException('受众不匹配');

    final exp = claims['exp'];
    if (exp is! int || DateTime.now().millisecondsSinceEpoch ~/ 1000 > exp + 60) {
      throw UnauthorizedException('令牌已过期');
    }

    return Principal(
      userId: claims['sub'] as String,
      tenantId: claims['tenant'] as String?,
      scopes: (claims['scope'] as String? ?? '').split(' ').where((s) => s.isNotEmpty).toSet(),
    );
  }
}
```

**JWT 不能做的事**

- 不能即时撤销（除非维护黑名单，那就退化成有状态）。
- 不能存放敏感数据（payload 可读）。
- 不能替代授权（scope 只是粗粒度声明，资源级权限仍需查询）。

**短访问令牌 + 刷新令牌**是常见折中：access 15 分钟，refresh 7 天且可撤销、可轮换、可复用检测。

### 11.5 OIDC 授权码 + PKCE 流程

```text
用户        客户端(SPA/App)              授权服务器              资源服务
 |               |                          |                       |
 |--登录-------->|                          |                       |
 |               |--/authorize?response_type=code&code_challenge=...->|
 |<--------------|----------登录页-----------|                       |
 |--凭证-------->|                          |                       |
 |               |<-302 ?code=abc-----------|                       |
 |               |--POST /token (code+verifier)--------------------->|
 |               |<-------access_token + id_token-------------------|
 |               |--GET /api (Bearer access_token)----------------->|
 |               |                          |            校验 JWT/自省 |
```

**安全要点**

- `state` 参数防 CSRF（回调时校验）。
- `code_verifier` 只存在于客户端，`code_challenge = SHA256(verifier)`。
- `redirect_uri` 必须精确白名单匹配，禁止通配与开放重定向。
- `nonce` 绑定 ID Token，防重放。
- 授权码一次性、短有效期。
- 校验 ID Token 的签名、`iss`、`aud`、`nonce`、`exp`。

### 11.6 密码存储

**绝不允许**：明文、MD5、SHA1、SHA256 裸哈希、可逆加密。

**推荐**：Argon2id（首选，内存硬）、bcrypt（成熟，有 72 字节限制）、scrypt。

```dart
// 伪代码：使用经过审查的实现（如 package:cryptography 的 Argon2id）
final salt = secureRandomBytes(16);
final hash = argon2id(
  password: utf8.encode(plain),
  salt: salt,
  memory: 64 * 1024,   // 64 MiB
  iterations: 3,
  parallelism: 1,
  length: 32,
);
// 存储格式：$argon2id$v=19$m=65536,t=3,p=1$<salt>$<hash>
```

**要点**

- 每个用户独立盐（不要全局盐）。
- 参数随硬件演进，登录成功时可透明升级哈希参数。
- 用**恒定时间比较**校验哈希，避免时序侧信道。
- 密码策略：长度优先（≥12）、检查弱口令库、不做无意义的复杂度强制。
- 登录失败返回统一提示（“用户名或密码错误”），避免枚举账号。
- 限速按 IP + 账号双维度，并有渐进延迟与锁定策略。
- 密码重置令牌：高熵、一次性、短有效期、绑定用户、使用后立即失效。

### 11.7 授权模型

**RBAC**

```dart
enum Role { customer, support, admin }

const _permissions = {
  Role.customer: {'order:read:own', 'order:create', 'order:cancel:own'},
  Role.support: {'order:read:any', 'customer:read:any'},
  Role.admin: {'*'},
};

bool hasPermission(Role role, String permission) {
  final perms = _permissions[role] ?? const {};
  return perms.contains('*') || perms.contains(permission);
}
```

RBAC 的局限：无法表达“只能看自己部门的订单”。这时需要 ABAC/策略。

**ABAC / 策略授权**

```dart
final class OrderAccessPolicy {
  const OrderAccessPolicy();

  bool canRead(Principal p, Order o) {
    if (p.roles.contains(Role.admin)) return true;
    if (p.roles.contains(Role.support)) return p.tenantId == o.tenantId;
    return o.customerId == p.userId && p.tenantId == o.tenantId;
  }

  void ensureCanRead(Principal p, Order o) {
    if (!canRead(p, o)) throw NotFoundException('order', o.id);  // 隐藏存在性
  }
}
```

**ReBAC**（关系型授权，如 Zanzibar 模型）：适合“文档共享给团队”“组织层级继承”等复杂关系。实现成本高，通常用策略引擎或授权服务。

**通用原则**

- 授权在**服务端**、在**数据访问附近**执行。
- 默认拒绝（deny by default）。
- 每个新端点都要有明确的授权规则与测试。
- 列表接口要在 SQL 层过滤，不能查出全部再在内存里过滤（性能 + 遗漏风险）。
- 后台任务、导出、统计同样要带租户/权限上下文。

### 11.8 常见漏洞与防御（逐条）

**SQL 注入**

```dart
// ❌
final sql = "SELECT * FROM users WHERE email = '$email'";
// ✅ 参数化
await db.query('SELECT * FROM users WHERE email = @email', parameters: {'email': email});
```

`ORDER BY`、表名、列名无法参数化 → 白名单。

**XSS**（后端相关部分）

- API 返回 JSON 时设置 `Content-Type: application/json`，禁止浏览器嗅探（`X-Content-Type-Options: nosniff`）。
- 不把用户输入直接拼进 HTML 模板。
- 富文本用白名单净化（如允许的标签/属性）。
- Cookie 加 `HttpOnly`，减少被窃取面。

**CSRF**

- 使用 Cookie 认证时：`SameSite=Lax/Strict` + CSRF Token（双提交或同步令牌）。
- 纯 Bearer Token（不放 Cookie）天然免疫传统 CSRF，但要防 XSS 窃取令牌。
- 关键操作要求重新认证或二次确认。

**SSRF**

```dart
Future<bool> isAllowedUrl(Uri uri) async {
  if (uri.scheme != 'https' && uri.scheme != 'http') return false;
  final host = uri.host;
  if (host == 'localhost' || host.endsWith('.internal')) return false;

  final addresses = await InternetAddress.lookup(host);
  for (final addr in addresses) {
    if (addr.isLoopback || addr.isPrivate || addr.isLinkLocal) return false;
  }
  return true;
}
```

还要：禁止跟随重定向到内网、限制端口、DNS 重绑定防护（解析后固定 IP 连接）。

**IDOR / BOLA（越权访问对象）**

```dart
// ❌ 只按 ID 查询
final order = await repo.findById(id);
return order.toJson();

// ✅ 查询即带归属条件
final order = await repo.findByIdForCustomer(id, principal.userId);
if (order == null) throw NotFoundException('order', id);
```

**路径穿越**

```dart
String safeJoin(String baseDir, String userPath) {
  final normalized = p.normalize(p.join(baseDir, userPath));
  if (!p.isWithin(baseDir, normalized)) {
    throw ValidationException('非法路径');
  }
  return normalized;
}
```

**不安全反序列化**：不要根据外部输入加载任意类型；不要反序列化不可信的对象图。

**竞态条件**：优惠券重复领取、余额并发扣减 → 数据库约束/条件更新/分布式锁。

**日志注入**：用户输入含换行会伪造日志条目 → 结构化日志 + 转义。

**开放重定向**：`?next=https://evil.com` → 只允许站内相对路径或白名单域。

**文件上传漏洞**：见 10.8；额外注意 SVG（可含脚本）、压缩炸弹。

**依赖漏洞**：见 11.11。

**默认配置**：调试端点、Swagger UI、Actuator 类接口不得在生产公开。

### 11.9 加密与密钥管理

**传输**：TLS 1.2+，禁用弱套件，HSTS。

**静态**：数据库/对象存储开启服务端加密；高敏感字段（身份证、银行卡）应用层加密（信封加密：DEK 加密数据，KMS 加密 DEK）。

**哈希 vs 加密 vs 编码**

| 目的 | 手段 | 例子 |
|---|---|---|
| 完整性 | 哈希/HMAC | 文件校验、Webhook 签名 |
| 保密 | 对称/非对称加密 | 字段加密、TLS |
| 不可逆存储 | 密码哈希 | Argon2id |
| 传输编码 | Base64 | 不是加密！ |

**密钥管理**

- 密钥存于 KMS/Secrets Manager，不写配置文件、不入镜像。
- 支持轮换（双密钥期），并记录 `kid`。
- 最小权限：服务只能读自己的密钥。
- 密钥泄露的应急流程：轮换 → 吊销 → 审计访问 → 通知。

### 11.10 隐私与合规

- **数据最小化**：只收集业务必需字段；能不存就不存。
- **目的限制**：收集时说明用途，不挪作他用。
- **保留期限**：定义每类数据的保留期与自动清理。
- **删除权**：设计可执行的删除/匿名化流程（含备份、日志、派生数据）。
- **可携带**：导出用户数据的能力。
- **访问审计**：谁在何时访问了敏感数据。
- **日志脱敏**：邮箱、手机号、身份证、令牌、Cookie、请求体中的密码字段。
- **跨境传输**：明确数据驻留要求，选择区域化部署。
- **同意管理**：营销、分析类数据处理需有同意记录。

**脱敏实现示例**

```dart
String maskEmail(String email) {
  final at = email.indexOf('@');
  if (at <= 1) return '***';
  return '${email[0]}***${email.substring(at)}';
}

String maskPhone(String phone) {
  if (phone.length < 7) return '***';
  return '${phone.substring(0, 3)}****${phone.substring(phone.length - 4)}';
}
```

### 11.11 供应链与依赖安全

- 锁定依赖版本，提交锁文件。
- 定期运行漏洞扫描（`dart pub outdated` + 外部 SCA 工具）。
- 审查新增依赖的必要性与维护状态（见 2.2）。
- CI 中构建产物可复现，记录 SBOM（软件物料清单）。
- 镜像使用固定 digest，定期重建获取基础镜像安全更新。
- 限制 CI 密钥权限，防止供应链攻击窃取生产凭据。
- 关注 Dart SDK 与关键包的安全公告。

### 11.12 安全测试

| 类型 | 频率 | 内容 |
|---|---|---|
| SAST | 每次提交 | 静态规则、密钥扫描 |
| 依赖扫描 | 每日/每次构建 | CVE、许可证 |
| DAST | 每次预发布 | 动态扫描 API |
| 渗透测试 | 每年/大版本前 | 人工深度测试 |
| 越权专项 | 每个功能 | 横向/纵向越权用例 |
| 威胁建模 | 新功能设计时 | STRIDE |

**不要**把“扫描通过”当作“安全”。逻辑漏洞（越权、业务规则绕过）只能靠设计与人工审查发现。

---

## 12. 架构、领域建模与代码组织

架构的本质是**管理变化与依赖**。好的架构让常见改动只碰一两个文件，让不常见的改动不至于引发连锁故障。

### 12.1 三种常见架构风格

**分层架构（Layered）**

```text
Controller → Service → Repository → DB
```

优点：直观、上手快。风险：容易变成“Controller 很薄、Service 变上帝类”。

**六边形架构（Ports & Adapters）**

```text
        ┌───────────────┐
  HTTP →│  Adapter      │
        └──────┬────────┘
               │
        ┌──────▼────────┐        ┌──────────────┐
        │   Domain      │◄───────│  Port (接口)  │
        │  (业务核心)    │        └──────┬───────┘
        └──────▲────────┘               │
               │                 ┌──────▼───────┐
        ┌──────┴────────┐        │  Adapter     │→ DB / MQ
        │  CLI / Worker │        └──────────────┘
        └───────────────┘
```

核心：业务逻辑在中心，所有外部技术都是可替换的适配器。测试时可以只替换适配器。

**整洁架构（Clean）**

分层更细：Entities → Use Cases → Interface Adapters → Frameworks。与六边形本质相近，区别主要在术语与分层粒度。

**实用建议**：中小项目采用“功能目录 + 四层职责（接口/应用/领域/基础设施）”，不必强制多 package。关键是**依赖方向**，不是目录美观。

### 12.2 模块化与边界

**判断模块边界是否合理的标准**

1. 能否独立测试？
2. 能否独立理解（不需要读其他模块内部）？
3. 修改一个业务概念时，改动是否集中在一个目录？
4. 依赖是否单向？

**反模式**

```dart
// ❌ shared/utils.dart 变成万能垃圾桶
import 'shared/utils.dart' show everything;
```

```dart
// ✅ 明确的能力模块
import 'shared/money/money.dart';
import 'shared/id/id_generator.dart';
```

**跨模块通信**：通过公开接口或事件，不直接访问对方的表/内部类。

### 12.3 DDD 战术模式

**实体（Entity）**：有身份，生命周期内身份不变。

```dart
final class Order {
  Order._({
    required this.id,
    required this.customerId,
    required this.lines,
    required this.status,
    required this.version,
  });

  factory Order.create({
    required OrderId id,
    required CustomerId customerId,
    required List<OrderLine> lines,
    required DateTime now,
  }) {
    if (lines.isEmpty) throw DomainException('订单至少包含一个商品');
    if (lines.any((l) => l.quantity <= 0)) {
      throw DomainException('商品数量必须为正');
    }
    return Order._(
      id: id,
      customerId: customerId,
      lines: List.unmodifiable(lines),
      status: OrderStatus.pending,
      version: 0,
    );
  }

  final OrderId id;
  final CustomerId customerId;
  final List<OrderLine> lines;
  OrderStatus status;
  int version;

  Money get total => lines.fold(
        Money.ofMinor(0, 'CNY'),
        (sum, l) => sum + l.subtotal,
      );

  void cancel(String reason) {
    if (status == OrderStatus.shipped || status == OrderStatus.completed) {
      throw DomainException('已发货订单不能取消');
    }
    if (status == OrderStatus.cancelled) {
      throw DomainException('订单已取消');
    }
    status = OrderStatus.cancelled;
  }
}
```

**值对象**：见 3.3 的 `Money`；还有 `Address`、`Email`、`OrderId`。

**聚合（Aggregate）**：一致性边界。外部只能通过聚合根访问内部实体。

```text
Order（聚合根）
 ├── OrderLine（内部实体，只能通过 Order 修改）
 └── ShippingAddress（值对象）
```

规则：**一个事务只修改一个聚合**；跨聚合用领域事件与最终一致。

**领域事件**

```dart
final class OrderCancelled {
  const OrderCancelled({
    required this.orderId,
    required this.reason,
    required this.occurredAt,
  });
  final String orderId;
  final String reason;
  final DateTime occurredAt;
}
```

事件是“已发生的事实”，用过去式命名；命令用祈使句（`CancelOrder`）。

**仓储（Repository）**：为聚合提供集合式访问，不是通用 DAO。

```dart
abstract interface class OrderRepository {
  Future<Order?> findById(OrderId id);
  Future<void> save(Order order);   // insert or update（含乐观锁）
}
```

### 12.4 CQRS 与事件溯源

**CQRS（命令查询职责分离）**：写模型与读模型分离。

- 适用：读写负载差异大、读模型需要复杂聚合、多视图。
- 代价：同步延迟、双模型维护、最终一致的用户体验问题。

**事件溯源（Event Sourcing）**：状态由事件序列重建。

- 适用：需要完整审计、时间旅行、复杂领域状态。
- 代价：事件版本管理、快照、重放、查询需要投影、团队学习曲线陡峭。

**建议**：除非有强审计/回放需求，否则从“当前状态表 + 审计日志表”开始。事件溯源不是“更高级”，是“更贵”。

### 12.5 微服务拆分

**拆分信号**

- 团队规模导致协作瓶颈（康威定律）。
- 不同模块的伸缩需求差异巨大。
- 不同模块的发布节奏/合规要求不同。
- 明确的业务边界（订单 vs 支付 vs 库存）。

**拆分代价**

| 维度 | 单体 | 微服务 |
|---|---|---|
| 调用 | 进程内函数 | 网络（延迟、超时、重试、序列化） |
| 事务 | 本地事务 | 分布式一致性（Saga/Outbox） |
| 调试 | 单一日志 | 分布式追踪 |
| 部署 | 一次发布 | 多服务版本兼容 |
| 测试 | 端到端简单 | 契约测试 + 环境复杂度 |
| 运维 | 一个进程 | 多服务监控、告警、容量 |

**拆分前的替代方案**：模块化单体（清晰的模块边界 + 独立测试 + 可独立抽取）。多数团队应该先做好这个。

> 服务边界、服务间通信与认证、Saga 编排、契约治理、Dart 的 gRPC/HTTP 落地、多服务可观测与演进路线的完整展开，见第 13 章「微服务架构与落地」。

**服务间通信的必备机制**

```text
超时（每个调用） → 重试（仅幂等 + 退避 + 上限） → 熔断（连续失败打开）
→ 舱壁（隔离并发池）→ 降级（返回缓存/默认值）→ 追踪（traceId 传播）
```

**Saga 模式**：用一系列本地事务 + 补偿动作实现跨服务一致性。

```text
创建订单 → 预占库存 → 扣款 → 确认订单
失败时逆序补偿：退款 → 释放库存 → 取消订单
```

要求：每个步骤幂等、补偿幂等、有状态记录（Saga 日志）、可人工干预。

### 12.6 架构决策记录（ADR）

```markdown
# ADR-0007: 使用 PostgreSQL 而非文档数据库存储订单

## 状态
已接受（2026-11-20）

## 背景
订单需要强一致的多表事务（订单、行、库存扣减），并需要复杂报表查询。

## 决策
使用 PostgreSQL 16，Dart 侧通过 postgres 包访问，Repository 模式隔离。

## 备选方案
1. MongoDB：灵活但跨文档事务较弱，报表需额外管道。
2. MySQL：可行，但团队已有 PG 运维经验与分区/JSONB 需求。

## 后果
- 正面：事务保证、成熟运维、丰富索引能力。
- 负面：schema 变更需要迁移；水平扩展依赖分区/分片。
- 风险：单库写入成为瓶颈时的迁移成本。

## 复审条件
单库写入持续超过 70% 容量，或跨区域部署要求变化时。
```

ADR 的价值在于记录**为什么**，避免半年后有人“优化”掉关键决策。

### 12.7 技术债管理

- 建立债务清单：位置、影响、修复成本、触发条件。
- 每次迭代预留固定比例（如 15%）处理债务。
- 用“如果不管，会在什么情况下爆炸”来排序，而不是按“代码丑不丑”。
- 高风险债务（无测试的支付逻辑、手工部署）优先。
- 债务不是道德问题，是有意识的取舍——但必须被记录。

---

---

## 13. 微服务架构与落地

微服务不是“把单体拆小”，而是**把进程内调用换成网络调用**。这个替换会同时引入延迟、部分失败、版本共存、数据一致性和运维复杂度。因此本章的顺序是：先判断该不该拆，再谈边界，最后才是通信、认证、一致性与工具链。

### 13.1 先判断：你是否真的需要微服务

**真实驱动力**

| 驱动力 | 是否成立 | 说明 |
|---|---|---|
| 团队规模导致协作冲突 | 常见 | 多个团队改同一代码库，发布互相阻塞 |
| 模块伸缩需求差异巨大 | 常见 | 报表服务与下单服务资源模型完全不同 |
| 发布节奏/合规要求不同 | 常见 | 支付模块需要更严格的审计与变更审批 |
| 故障隔离要求 | 部分成立 | 需要配合舱壁、限流才有意义 |
| “技术栈要多样化” | 通常不成立 | 多数场景是组织问题，不是架构问题 |
| “微服务更先进” | 不成立 | 架构没有先进性，只有匹配度 |

**成本模型（必须量化）**

```text
每增加一个服务，你至少增加：
+ 1 套 CI/CD 流水线、1 个部署单元、1 组健康检查
+ 1 套日志/指标/告警/仪表盘
+ 1 份 API 契约及其兼容性管理
+ 若干网络调用（延迟 + 超时 + 重试 + 熔断 + 追踪）
+ 跨服务数据一致性方案（Saga / outbox / 补偿）
+ 本地开发与端到端测试环境的复杂度
+ 值班与故障定位的认知负担（跨服务排障）
```

**分布式单体（Distributed Monolith）的典型特征**

1. 服务之间共享数据库表，或直接读写对方表。
2. 一个业务变更需要同时发布 3 个以上服务。
3. 服务间同步调用成链，任一服务不可用导致整条链路失败。
4. 所有服务必须同版本部署才能工作。
5. 没有独立的数据所有权，事务跨库靠“先写 A 再写 B”。

如果出现其中任意两条，拆分带来的只有成本，没有收益。**此时应回到模块化单体**：清晰的模块边界、独立测试、可独立抽取的包结构（见 12.2）。

**决策清单**

- [ ] 能否说清每个服务的**唯一职责**与**数据所有权**？
- [ ] 团队是否有能力运维 N 个部署单元（监控、值班、发布）？
- [ ] 是否已具备分布式追踪与集中日志？
- [ ] 是否有自动化部署与回滚能力？
- [ ] 跨服务一致性方案是否已确定并有测试？
- [ ] 拆分后能否**独立发布**且不要求其他服务同步升级？

六项中若有超过两项答“否”，先补基础能力，再拆服务。

### 13.2 服务边界划分

**从限界上下文出发**

一个服务应对应一个**限界上下文（Bounded Context）**：一组内聚的业务能力，拥有自己的领域模型与术语。

```text
订单上下文：Order、OrderLine、订单状态机、下单/取消规则
库存上下文：SKU、可用量、预占、释放、盘点
支付上下文：支付单、渠道、退款、对账
```

**常见错误拆法**

| 错误方式 | 后果 |
|---|---|
| 按技术层拆（controller 服务、DAO 服务） | 一次业务变更要改所有服务，强耦合 |
| 按实体拆（User 服务、Address 服务、Phone 服务） | 过度细碎，调用链爆炸 |
| 按团队现有分工硬拆 | 边界与业务不一致，长期摩擦 |
| 共享数据库拆分 | 数据所有权不清，无法独立演进 |

**粒度判断**

- **事务边界**：需要强一致的写操作应尽量在同一服务内。
- **变更频率**：经常一起改的概念应在同一服务。
- **伸缩特征**：CPU 密集与 I/O 密集的服务可以分开。
- **团队边界**：一个服务最好由一个团队拥有（康威定律的正向利用）。
- **故障影响**：核心链路与非核心链路应能隔离。

**数据所有权规则**

```text
每条数据有且仅有一个写入方（system of record）。
其他服务只能通过 API 或事件读取，不得直接写表。
```

跨服务需要“对方的数据”时，选择：

1. **同步查询**：实时性高，但增加耦合与延迟。
2. **事件复制**：本地保存只读副本（物化视图），最终一致。
3. **共享只读视图**：仅在分析场景，且明确不可写。

**边界验收**

- 能独立部署而不要求其他服务同版本。
- 能独立扩容而不影响其他服务。
- 能独立故障（降级）而不拖垮全局。
- 数据库凭据只授予本服务。

### 13.3 通信方式选择

**同步 vs 异步**

| 维度 | 同步（HTTP/gRPC） | 异步（消息/事件） |
|---|---|---|
| 耦合 | 时间耦合（对方必须在线） | 时间解耦 |
| 延迟 | 低（一次往返） | 较高（排队 + 消费） |
| 一致性 | 请求内可立即得知结果 | 最终一致 |
| 失败处理 | 调用方立即感知 | 重试/死信/补偿 |
| 复杂度 | 超时/重试/熔断 | 幂等/顺序/重复投递 |
| 适用 | 查询、需要立即结果的操作 | 通知、派生数据、长流程 |

**经验法则**

- **查询** → 同步。
- **命令需要立即结果**（如“库存是否充足”）→ 同步。
- **副作用可延后**（发邮件、生成报表、同步索引）→ 异步。
- **跨多个服务的长流程** → 编排（Saga）+ 异步步骤。
- **广播型事实**（订单已创建）→ 事件。

**超时预算必须端到端传递**

```text
客户端 3s → 网关 2.8s → 订单服务 2.5s → 库存服务 1.5s → DB 1s
每个下游超时 < 上游剩余预算
```

Dart 中的实现方式：把 deadline 作为显式参数传递，而不是各层各设一个固定值。

```dart
final class CallContext {
  const CallContext({
    required this.requestId,
    required this.deadline,
    this.traceParent,
  });

  final String requestId;
  final DateTime deadline;
  final String? traceParent;

  Duration get remaining => deadline.difference(DateTime.now().toUtc());

  CallContext child(Duration maxForChild) {
    final childDeadline = DateTime.now().toUtc().add(
      remaining < maxForChild ? remaining : maxForChild,
    );
    return CallContext(
      requestId: requestId,
      deadline: childDeadline,
      traceParent: traceParent,
    );
  }
}
```

**重试规则（跨服务同样适用）**

- 只重试幂等操作，且必须在剩余预算内。
- 退避 + 抖动，次数上限（通常 2–3 次）。
- 下游已熔断时不再重试。
- 重试要记录尝试次数，便于观测重试风暴。

### 13.4 API 网关、BFF 与服务发现

**网关职责**

| 职责 | 说明 |
|---|---|
| TLS 终止 | 统一证书管理 |
| 认证前置 | 校验令牌、注入可信身份头 |
| 路由 | 按路径/租户/版本转发 |
| 限流与配额 | 边缘保护 |
| 请求/响应改写 | 头注入、CORS |
| 可观测 | 统一访问日志与追踪起点 |
| 灰度 | 按权重/标签分流 |

**网关不应该做的事**：承载业务逻辑、做数据库查询、实现复杂编排。否则会变成新的单体。

**BFF（Backend for Frontend）**

- 为不同客户端（Web/移动/第三方）提供定制聚合接口。
- 归属前端团队，减少客户端多次往返。
- 必须复用下游服务能力，不能复制业务规则（否则规则漂移）。

**服务发现**

| 方式 | 说明 | 适用 |
|---|---|---|
| DNS / K8s Service | 平台原生，简单 | 绝大多数容器部署 |
| 注册中心（Consul 等） | 动态注册、健康检查 | 混合云/VM |
| 客户端负载均衡 | 客户端拉取实例列表 | 需要精细路由 |

Dart 服务通常直接使用平台 DNS（如 `http://inventory-service:8080`），由 K8s Service 做 L4 负载均衡。只有在需要按请求做高级路由（金丝雀、同区域优先）时才引入更复杂的方案。

**内外网隔离**

```text
公网 → WAF → API 网关 → 内部服务（仅内网可达）
内部服务之间：网络策略白名单 + mTLS
管理端点：独立端口/路径 + 网络隔离
```

### 13.5 服务间认证与零信任

**为什么不能信任内网**：横向移动、配置错误、被攻陷的实例都可能从内网发起请求。

**三层机制**

1. **传输层**：mTLS，双方证书校验，服务身份由证书 SAN 表达。
2. **应用层**：服务令牌（OAuth2 client credentials）携带 scope，明确“我是谁、能做什么”。
3. **网络层**：网络策略只允许声明的调用关系（defense in depth）。

**mTLS 配置示意（Dart `HttpServer`/`HttpClient`）**

```dart
// 服务端
final context = SecurityContext()
  ..useCertificateChain('certs/server.crt')
  ..usePrivateKey('certs/server.key')
  ..setTrustedCertificates('certs/ca.crt')
  ..setClientAuthorities('certs/ca.crt');

final server = await HttpServer.bindSecure(
  InternetAddress.anyIPv4,
  8443,
  context,
  requestClientCertificate: true,
);

// 客户端
final clientContext = SecurityContext()
  ..useCertificateChain('certs/client.crt')
  ..usePrivateKey('certs/client.key')
  ..setTrustedCertificates('certs/ca.crt');

final client = HttpClient(context: clientContext);
```

生产要点：证书由平台（cert-manager、服务网格、云 CA）自动签发与轮换；应用只负责加载与热更新；证书过期必须有告警。

**用户上下文如何跨服务传播**

| 方案 | 说明 | 风险 |
|---|---|---|
| 透传用户令牌 | 下游直接校验原令牌 | 令牌权限过大、受众不符 |
| Token Exchange | 换取受众为目标服务的短期令牌 | 需要授权服务器支持 |
| 签名身份头 | 网关签名 `x-user-id`，内部服务验签 | 密钥管理、重放防护 |

**禁止**：直接信任客户端传来的 `x-user-id`；在内部服务里根据 header 自行“推断”身份。

**授权在哪个服务执行**

- 粗粒度（是否能调用该 API）：网关 / 服务级 scope。
- 细粒度（是否能操作该资源）：**拥有该资源的服务**，因为它掌握数据与规则。
- 不要在下游服务里重复实现上游的业务权限规则，规则应集中在资源所有者。

### 13.6 分布式一致性与 Saga

> 复制、Quorum、共识、一致性模型、分布式事务全景、分布式锁正确性与分布式 ID 的完整讨论，见第 14 章「分布式系统专题」。

**为什么少用两阶段提交（2PC）**：同步阻塞、协调者单点、跨异构系统支持差、失败恢复复杂。

**Saga**：把长事务拆成一系列本地事务，每个步骤都有补偿动作。

```text
T1 创建订单(draft) → T2 预占库存 → T3 扣款 → T4 确认订单
失败回滚：C3 退款 → C2 释放库存 → C1 取消订单
```

**编排（Orchestration）vs 协同（Choreography）**

| | 编排 | 协同 |
|---|---|---|
| 控制流 | 中心编排器显式驱动 | 各服务监听事件自行反应 |
| 可读性 | 流程集中，易理解 | 分散，需全局视角 |
| 耦合 | 编排器依赖各服务 | 服务只依赖事件契约 |
| 适用 | 步骤多、需要人工干预 | 简单链路、松耦合 |

**编排器实现（Dart 示意）**

```dart
enum SagaStep { createOrder, reserveStock, chargePayment, confirmOrder }

final class CreateOrderSaga {
  CreateOrderSaga({
    required this.orders,
    required this.inventory,
    required this.payments,
    required this.store,
    required this.clock,
  });

  final OrderPort orders;
  final InventoryPort inventory;
  final PaymentPort payments;
  final SagaStore store;
  final Clock clock;

  Future<void> run(String sagaId, CreateOrderCommand cmd) async {
    final state = await store.loadOrCreate(sagaId, cmd);
    try {
      if (state.step.index <= SagaStep.createOrder.index) {
        await orders.createDraft(cmd);
        await store.advance(sagaId, SagaStep.createOrder);
      }
      if (state.step.index <= SagaStep.reserveStock.index) {
        await inventory.reserve(cmd.lines);          // 幂等：以 sagaId 为键
        await store.advance(sagaId, SagaStep.reserveStock);
      }
      if (state.step.index <= SagaStep.chargePayment.index) {
        await payments.charge(sagaId, cmd.total);    // 幂等键 = sagaId
        await store.advance(sagaId, SagaStep.chargePayment);
      }
      await orders.confirm(cmd.orderId);
      await store.complete(sagaId);
    } catch (e) {
      await _compensate(sagaId, state, e);
      rethrow;
    }
  }

  Future<void> _compensate(String sagaId, SagaState state, Object cause) async {
    // 逆序补偿，每一步都幂等且可重试
    if (state.step.index >= SagaStep.chargePayment.index) {
      await payments.refund(sagaId, reason: cause.toString());
    }
    if (state.step.index >= SagaStep.reserveStock.index) {
      await inventory.release(sagaId);
    }
    if (state.step.index >= SagaStep.createOrder.index) {
      await orders.cancelDraft(sagaId);
    }
    await store.markCompensated(sagaId, cause.toString());
  }
}
```

**Saga 状态表**

```sql
CREATE TABLE sagas (
  id            text PRIMARY KEY,
  type          text NOT NULL,
  step          text NOT NULL,
  status        text NOT NULL,      -- running | completed | compensating | compensated | failed
  payload       jsonb NOT NULL,
  attempts      int NOT NULL DEFAULT 0,
  last_error    text,
  updated_at    timestamptz NOT NULL DEFAULT now()
);
```

**必须处理的边界**

1. **补偿失败**：补偿本身也可能失败，需要重试 + 人工介入队列 + 告警。
2. **超时**：每个步骤都有超时；编排器整体有最大时长。
3. **重复执行**：编排器崩溃重启后从 `step` 继续，每一步必须幂等。
4. **不可补偿操作**：发出去的邮件、外部已完成的转账，需要“正向修复”而非回滚。
5. **并发同一订单**：用订单 ID 或 Saga ID 做互斥（唯一约束/租约）。
6. **可观测**：Saga 状态必须可查询，支持人工推进。

**最终一致的用户体验**：告知用户“处理中”，提供状态查询，而不是假装同步完成。

### 13.7 契约与版本

**服务间契约**

| 协议 | 契约形式 | 兼容检查 |
|---|---|---|
| REST | OpenAPI | schema diff、字段必填性 |
| gRPC | `.proto` | 字段编号不复用、不删必填 |
| 事件 | JSON Schema/Avro | 消费方兼容性、版本字段 |

**兼容规则（提供方视角）**

- 只新增可选字段；不删除、不改类型、不改语义。
- gRPC 中删除字段要 `reserved`，禁止复用编号。
- 事件增加字段时，消费方必须容忍未知字段。
- 收紧校验（原本接受的值现在拒绝）也是破坏性变更。

**消费者驱动契约测试**

```text
1. 消费方声明期望（请求样例 + 期望响应字段）
2. 提供方 CI 用这些期望验证自己的实现
3. 任一方变更导致不兼容 → 构建失败
```

**发布顺序**

```text
提供方先发布“向后兼容”的新版本 → 观察 → 消费方切换 → 提供方再移除旧行为
```

**灰度与流量切换**：按服务维度独立灰度，避免“全链路必须同时上线”。数据库变更遵守 expand-contract（见 9.9）。

### 13.8 Dart 落地：gRPC 与 HTTP 客户端

**gRPC 服务端（示意）**

```protobuf
// protos/inventory.proto
syntax = "proto3";
package inventory.v1;

service InventoryService {
  rpc Reserve(ReserveRequest) returns (ReserveResponse);
  rpc Release(ReleaseRequest) returns (ReleaseResponse);
}

message ReserveRequest {
  string saga_id = 1;
  repeated Line lines = 2;
}
message Line { string sku = 1; int32 quantity = 2; }
message ReserveResponse { bool reserved = 1; string reservation_id = 2; }
```

```dart
import 'package:grpc/grpc.dart';
import 'generated/inventory.pbgrpc.dart';

final class InventoryGrpc extends InventoryServiceBase {
  InventoryGrpc(this._repo);

  final InventoryRepository _repo;

  @override
  Future<ReserveResponse> reserve(
    ServiceCall call,
    ReserveRequest request,
  ) async {
    final result = await _repo.reserve(
      sagaId: request.sagaId,
      lines: request.lines.map((l) => (sku: l.sku, qty: l.quantity)).toList(),
    );
    return ReserveResponse()
      ..reserved = result.ok
      ..reservationId = result.reservationId;
  }
}

Future<void> main() async {
  final server = Server.create(
    services: [InventoryGrpc(InventoryRepository(db))],
    // 拦截器：追踪、日志、认证
  );
  await server.serve(port: 9090);
}
```

**HTTP 服务间客户端（带 deadline、重试、追踪）**

```dart
final class InventoryClient {
  InventoryClient({
    required this.baseUrl,
    required HttpClient http,
    required this.tracer,
  }) : _http = http;

  final Uri baseUrl;
  final HttpClient _http;
  final Tracer tracer;

  Future<Reservation> reserve(CallContext ctx, List<Line> lines) async {
    final span = tracer.startSpan('inventory.reserve', parent: ctx.traceParent);
    try {
      final remaining = ctx.remaining;
      if (remaining <= Duration.zero) {
        throw DependencyUnavailable('inventory', 'deadline_exceeded');
      }

      final request = await _http
          .postUrl(baseUrl.resolve('/v1/reservations'))
          .timeout(remaining);
      request.headers
        ..contentType = ContentType.json
        ..set('x-request-id', ctx.requestId);
      if (ctx.traceParent != null) {
        request.headers.set('traceparent', ctx.traceParent!);
      }
      request.write(jsonEncode({'lines': lines.map((l) => l.toJson()).toList()}));

      final response = await request.close().timeout(remaining);
      final body = await utf8.decodeStream(response).timeout(remaining);

      if (response.statusCode == 409) {
        throw ConflictException('insufficient_inventory', '库存不足');
      }
      if (response.statusCode >= 500) {
        throw DependencyUnavailable('inventory', response.statusCode);
      }
      if (response.statusCode >= 400) {
        throw DependencyUnavailable('inventory', 'unexpected_${response.statusCode}');
      }
      return Reservation.fromJson(jsonDecode(body) as Map<String, dynamic>);
    } finally {
      span.end();
    }
  }
}
```

**要点**

- 复用 `HttpClient`（连接池），按目标设置 `maxConnectionsPerHost`。
- 超时用**剩余预算**，不是固定值。
- 传播 `traceparent` 与 `requestId`。
- 区分 4xx（调用方错误，不要重试）与 5xx（可能重试）。
- 服务端也要校验入站请求的身份与来源。

**错误码映射表（跨服务必须统一）**

| 语义 | HTTP | gRPC | 可重试 |
|---|---|---|---|
| 参数非法 | 400 | `INVALID_ARGUMENT` | 否 |
| 未认证 | 401 | `UNAUTHENTICATED` | 否 |
| 无权限 | 403 | `PERMISSION_DENIED` | 否 |
| 不存在 | 404 | `NOT_FOUND` | 否 |
| 冲突/状态不允许 | 409 | `FAILED_PRECONDITION` | 否 |
| 资源耗尽 | 429 | `RESOURCE_EXHAUSTED` | 是（退避） |
| 下游不可用 | 503 | `UNAVAILABLE` | 是（有上限） |
| 超时 | 504 | `DEADLINE_EXCEEDED` | 视幂等性 |

### 13.9 多服务可观测性

**追踪贯穿**

```text
客户端 → 网关（新建 trace）→ 订单服务（child span）
       → 库存服务（child span）→ DB（child span）
       → 支付服务（child span）→ 外部渠道（child span）
```

- 使用 W3C `traceparent`，所有出站请求自动注入。
- 服务端从入站头恢复上下文，缺省时新建。
- 采样：错误与慢请求 100%，正常流量按比例。
- span 上打业务属性（orderId、sagaId）便于检索。

**指标聚合**

```text
service=order-api  http_requests_total / duration
service=order-api  dependency_errors_total{dependency="inventory"}
service=inventory  grpc_server_handled_total{method="Reserve"}
```

按服务建立仪表盘，同时有一个“全局入口”仪表盘（用户可见错误率与延迟）。

**日志关联**

- 每条日志带 `traceId`、`spanId`、`requestId`、`tenantId`。
- 跨服务日志平台支持按 traceId 聚合查询。
- 不记录敏感字段（见 11.10）。

**依赖拓扑**

从追踪数据生成服务依赖图，识别：

- 循环依赖（架构味道）。
- 扇出过大（一个请求调用 20 个服务）。
- 强耦合（A 不可用时 B 完全不可用）。

**按服务设定 SLO**：入口 SLO 面向用户，内部服务 SLO 面向调用方；不要只监控入口而忽略内部依赖的劣化趋势。

### 13.10 多服务本地开发与测试

**本地编排**

```yaml
# docker-compose.override.yml（开发者本地）
services:
  order-api:
    build: ./packages/order_service
    environment:
      INVENTORY_URL: http://inventory-api:8080
      DATABASE_URL: postgresql://app:app@postgres:5432/orders
    depends_on: [postgres, inventory-api]
  inventory-api:
    build: ./packages/inventory_service
    environment:
      DATABASE_URL: postgresql://app:app@postgres:5432/inventory
```

**降低本地复杂度的策略**

1. **只起正在改的服务 + 依赖**，其他用共享开发环境。
2. **服务虚拟化**：用契约生成的 mock 替代不常改的服务。
3. **单仓库多包**，共享 DTO 与契约，减少版本错配。
4. **一人一命名空间**，避免共享环境互相污染。

**测试分层**

| 层级 | 范围 | 依赖 |
|---|---|---|
| 单元 | 单个服务内部 | 无 |
| 组件 | 单服务 + 真实 DB | 容器化 DB |
| 契约 | 服务间接口 | 契约文件 |
| 集成 | 2–3 个服务 | compose / 测试环境 |
| E2E | 关键业务链路 | 完整环境 |

**E2E 只覆盖关键路径**：下单、支付、退款。不要试图用 E2E 覆盖所有组合。

### 13.11 基础设施的增量引入

| 能力 | 何时需要 | 过早引入的代价 |
|---|---|---|
| 服务网格（Istio/Linkerd） | 多语言、需要统一 mTLS/流量治理 | 运维复杂、调试困难、资源开销 |
| 配置中心 | 多环境多服务配置治理 | 增加启动依赖与故障面 |
| 分布式限流 | 多服务共享配额 | 需要集中存储与一致性 |
| 消息中间件 | 异步解耦、削峰 | 运维与一致性复杂度 |
| 分布式追踪 | 服务数 ≥ 3 就开始需要 | 存储成本、采样策略 |
| 集中日志 | 服务数 ≥ 2 就需要 | 成本与合规 |

**原则**：每一层基础设施都要能回答“它解决了哪个已发生的具体问题”。先有痛点，再上工具。

### 13.12 演进路线：从单体到服务

**绞杀者模式（Strangler Fig）**

```text
1. 在单体前加路由层（网关）
2. 选一个边界清晰的模块（如通知）抽成独立服务
3. 路由层把该模块流量切到新服务（1% → 10% → 100%）
4. 观察指标，保留回切能力
5. 从单体删除该模块代码
6. 重复下一个模块
```

**并行运行与数据迁移**

- 双写期：新服务与单体同时写，比较结果（shadow read）。
- 数据迁移：批量回填 + 增量同步 + 校验。
- 切换读取：按用户/租户灰度。
- 停止旧写入：确认无回滚需求后。

**组织对齐**

- 一个服务一个 owner 团队。
- 团队具备“你构建、你运行”的责任（on-call）。
- 平台团队提供 CI/CD、可观测、密钥管理等公共能力。

**常见失败模式**

| 失败模式 | 表现 | 预防 |
|---|---|---|
| 拆得太细 | 调用链长、协作成本高 | 以业务能力为单位，先粗后细 |
| 共享数据库 | 无法独立演进 | 明确数据所有权 |
| 无追踪 | 排障靠猜 | 先上追踪再拆 |
| 同步链路过长 | 一个慢服务拖垮全局 | 异步化 + 超时预算 + 熔断 |
| 版本强耦合 | 必须同时发布 | 契约兼容 + 灰度 |
| 组织未变 | 微服务但仍是集中式审批 | 团队自治与平台化 |

### 13.13 微服务检查清单

**边界**

- [ ] 每个服务有唯一职责与明确的数据所有权
- [ ] 没有服务直接读写其他服务的表
- [ ] 服务可独立部署，不要求其他服务同版本
- [ ] 服务可独立扩容与降级

**通信**

- [ ] 同步/异步选择有明确理由
- [ ] 每个调用有超时，且遵守剩余预算
- [ ] 重试仅用于幂等操作，有退避与上限
- [ ] 有关键依赖的熔断与降级
- [ ] 错误码映射统一，4xx/5xx 语义正确

**安全**

- [ ] 服务间有身份认证（mTLS 或服务令牌）
- [ ] 网络策略只放行声明的调用关系
- [ ] 用户身份由可信来源传播，不信任客户端头
- [ ] 资源级授权在数据所有者服务内执行

**一致性**

- [ ] 跨服务写操作有 Saga/outbox 方案
- [ ] 每一步幂等，补偿可重试
- [ ] 有 Saga 状态查询与人工干预路径
- [ ] 用户体验反映“处理中”而非假装同步完成

**可观测与运维**

- [ ] trace 贯穿所有服务与依赖
- [ ] 日志可按 traceId 关联
- [ ] 每个服务有 SLO 与告警
- [ ] 依赖拓扑可查看
- [ ] 有跨服务排障 runbook

**交付**

- [ ] 每个服务独立 CI/CD 与回滚
- [ ] 契约测试覆盖服务间接口
- [ ] 数据库迁移遵守 expand-contract
- [ ] 本地开发可只启动必要服务

---

---

## 14. 分布式系统专题

分布式系统不是“多台机器”，而是**你无法再依赖以下三个假设**：网络可靠、延迟可忽略、时钟一致。本章先建立故障与一致性模型，再落到复制、分区、事务、锁、ID、多区域与验证。

### 14.1 故障模型：先接受“部分失败”

单机程序的失败通常是“全有或全无”：进程活着，或崩溃。分布式系统中，**部分失败**是常态：

| 现象 | 本质 | 工程含义 |
|---|---|---|
| 请求超时 | 你不知道对方是否执行了 | 必须幂等 + 可查询结果 |
| 连接被重置 | 可能是网络、对端崩溃、中间设备 | 不能据此判断业务失败 |
| 节点“假死” | GC 停顿、CPU 饥饿、网络分区 | 心跳超时可能误判 |
| 消息重复 | 重传、至少一次投递 | 消费端去重 |
| 消息乱序 | 多路径、多分区 | 需要顺序键或版本号 |
| 时钟回拨 | NTP 调整、虚拟机迁移 | 不能依赖墙上时钟排序 |

**异步网络模型**：消息延迟没有上界。因此**无法区分“节点已死”与“节点很慢”**——所有“超时即失败”的判断都是概率性的，必须配合幂等与状态查询。

常见故障类型：

- **崩溃故障（Crash）**：节点停止，不再发消息。
- **遗漏故障（Omission）**：消息丢失。
- **时序故障（Timing）**：延迟异常。
- **分区故障（Partition）**：网络被切成互不可达的子集。
- **拜占庭故障（Byzantine）**：节点作恶或返回错误结果。普通业务系统通常不防拜占庭，但区块链、跨信任域场景需要。

**设计含义**：任何“假设对方一定收到/一定没收到”的代码，都是 bug。

### 14.2 CAP 与 PACELC 的正确理解

**CAP**：在网络分区（P）发生时，只能在一致性（C）与可用性（A）之间取舍。注意：

- CAP 的 C 指**线性一致性**，不是“数据不丢”或“最终一致”。
- CAP 只在**分区期间**适用；无分区时系统可以同时具备 C 和 A。
- “三选二”是误导性简化：分区不是可选项，是客观事实。

**PACELC** 补齐了无分区时的情况：

```text
If Partition → tradeoff Availability vs Consistency
Else         → tradeoff Latency vs Consistency
```

即：即使没有分区，追求强一致也要付出跨节点协调的延迟成本。

**实践映射**

| 系统类型 | 分区时选择 | 无分区时倾向 |
|---|---|---|
| 银行账务、库存扣减 | C（拒绝写入/阻塞） | 一致性优先，接受延迟 |
| 社交 Feed、点赞数 | A（继续服务，可能不一致） | 延迟优先 |
| 会话/购物车 | 视业务，多数 A | 延迟优先 |
| 配置/选主元数据 | C | 一致性优先 |

**常见误读**：拿 CAP 证明“我们应该用 NoSQL”或“强一致没法扩展”。真正的问题是：**这个业务在分区时，宁可拒绝服务也不能给出错误答案，还是宁可给旧答案也要可用？**

### 14.3 一致性模型谱系

从强到弱：

| 模型 | 保证 | 典型实现 | 业务例子 |
|---|---|---|---|
| 线性一致（Linearizable） | 所有操作看起来在单一时间点原子生效，且顺序与真实时间一致 | 单主同步复制、共识 | 账户余额、库存 |
| 顺序一致（Sequential） | 所有节点看到相同顺序，但不要求与真实时间一致 | 某些共识系统 | 配置变更 |
| 因果一致（Causal） | 有因果关系的操作顺序一致 | 向量时钟、会话保证 | 评论与回复 |
| 读己之写（Read-your-writes） | 自己写后能读到自己的写 | 粘主、版本号 | 用户资料编辑 |
| 单调读（Monotonic reads） | 不会读到比之前更旧的数据 | 会话粘性 | 分页浏览 |
| 最终一致（Eventual） | 停止写入后最终收敛 | 异步复制、事件 | 点赞数、搜索索引 |

**如何选择**

1. 找出业务中“不能出错”的读：余额、库存、权限。
2. 找出“可以短暂陈旧”的读：列表、统计、推荐。
3. 对每一项标注可接受的最大陈旧时间。
4. 用这个表决定哪些走主库、哪些走副本/缓存。

**读己之写的实现**

```dart
final class UserProfileService {
  UserProfileService({required this.primary, required this.replica});

  final ProfileStore primary;   // 写 + 强一致读
  final ProfileStore replica;   // 只读副本

  Future<void> updateProfile(String userId, ProfileUpdate update) async {
    await primary.update(userId, update);
  }

  Future<Profile> getProfile(String userId, {required bool fresh}) async {
    // fresh=true：刚写过或用户明确要求最新 → 主库
    final store = fresh ? primary : replica;
    return store.find(userId);
  }
}
```

更系统的做法是“写后一段时间内粘主”（如 2 秒或复制位点追平），避免永久打主库。

**注意**：缓存也是一种“副本”，缓存一致性本质上是同一个问题（见 10.4）。

### 14.4 复制

**三种拓扑**

| 拓扑 | 写入 | 读 | 优点 | 缺点 |
|---|---|---|---|---|
| 单主（Primary-Replica） | 只写主 | 主或副本 | 简单、无写冲突 | 主故障需切换；副本滞后 |
| 多主（Multi-Primary） | 多节点可写 | 任意 | 写入本地、容灾强 | 冲突解决复杂 |
| 无主（Leaderless，Dynamo 风格） | 多节点 | Quorum | 高可用 | 需要读修复、冲突合并 |

**同步 vs 异步复制**

- **同步**：主等待副本确认 → 强一致风险低，但写入延迟高、副本故障影响可用性。
- **异步**：主提交即返回 → 延迟低，但主故障可能丢失未复制的数据（RPO > 0）。
- **半同步**：至少一个副本确认 → 折中。

**Quorum（无主/多副本）**

```text
N = 副本数
W = 写成功所需确认数
R = 读需要查询的副本数
若 R + W > N，则读写集合必有交集 → 读能看到最新写
例：N=3, W=2, R=2
```

代价：延迟取决于最慢的 W/R 个节点；节点故障时可能无法达成 quorum。

**复制滞后（Replication Lag）的处理**

1. 监控滞后秒数，超过阈值告警。
2. 关键读走主库。
3. 用户写后短期内粘主。
4. 读副本时校验最小版本/位点，不满足则回退主库。
5. 对外提示“数据可能有延迟”（如后台报表）。

**冲突解决（多主/无主）**

| 策略 | 说明 | 风险 |
|---|---|---|
| 最后写入胜出（LWW） | 用时间戳决定 | 时钟漂移导致丢更新 |
| 版本向量 | 检测并发冲突 | 需要业务合并逻辑 |
| CRDT | 数学上可合并的数据结构 | 适用范围有限 |
| 业务仲裁 | 人工/规则解决 | 流程成本 |

**结论**：能用单主就不要用多主。多主的复杂度会渗透到每一行业务代码。

### 14.5 分区与再平衡

**分区键选择**

- 高基数、分布均匀（用户 ID、租户 ID）。
- 避免热点：不要用“日期”“状态”作为唯一分区键（新数据全落一个分区）。
- 分区键一旦确定很难改，需提前评估增长。

**一致性哈希**

```text
节点 → 哈希环上的位置
键 → 哈希到环上，顺时针找第一个节点
增加节点只影响相邻区间（约 1/N 数据迁移）
虚拟节点：每个物理节点映射多个环位置，改善均衡
```

```dart
final class ConsistentHashRing {
  ConsistentHashRing(List<String> nodes, {int virtualNodes = 150}) {
    for (final node in nodes) {
      for (var i = 0; i < virtualNodes; i++) {
        _ring[_hash('$node#$i')] = node;
      }
    }
    _sortedKeys = _ring.keys.toList()..sort();
  }

  final _ring = <int, String>{};
  late final List<int> _sortedKeys;

  String nodeFor(String key) {
    if (_ring.isEmpty) throw StateError('哈希环为空');
    final h = _hash(key);
    var lo = 0, hi = _sortedKeys.length - 1;
    while (lo < hi) {
      final mid = (lo + hi) >> 1;
      if (_sortedKeys[mid] < h) {
        lo = mid + 1;
      } else {
        hi = mid;
      }
    }
    return _ring[_sortedKeys[lo]]!;
  }

  static int _hash(String s) {
    // 生产应使用稳定哈希（如 fnv1a / xxhash），不要用 String.hashCode（进程间不稳定）
    var hash = 0x811c9dc5;
    for (final unit in s.codeUnits) {
      hash ^= unit;
      hash = (hash * 0x01000193) & 0xFFFFFFFF;
    }
    return hash;
  }
}
```

**再平衡风险**

- 一次性迁移大量数据会打满带宽与磁盘 IO → 限速、分批、可暂停。
- 迁移期间双读/双写，确认后再删除旧副本。
- 避免“再平衡风暴”：节点抖动导致反复迁移，需要冷却期与最小稳定时间。

**热点**

- 单个超级租户/爆款商品 → 拆分热点键（加随机后缀）或本地缓存。
- 计数类热点 → 分片累加后汇总。
- 写热点 → 队列削峰或合并写。

**跨分片查询**：尽量在分区键内完成。跨分片聚合需要并行查询 + 归并，注意超时与部分失败。

### 14.6 时间与顺序

**墙上时钟不可靠**

- NTP 可能回拨（尤其虚拟机迁移、闰秒处理）。
- 不同机器的时钟有偏差（几十毫秒到秒级）。
- 因此**不能用 `DateTime.now()` 做跨节点排序或并发判断**。

**单调时钟**：用于测量时长（`Stopwatch`），不受时钟调整影响；但不能跨进程比较。

**逻辑时钟（Lamport）**：为事件建立偏序。

```text
每个节点维护 counter
发送事件：counter += 1，消息带上 counter
接收事件：counter = max(local, received) + 1
```

能表达“A happened-before B”，但无法判断并发。

**向量时钟**：每个节点一个维度，可检测并发写冲突。

```text
节点 A: {A:2, B:1}
节点 B: {A:1, B:3}
互不包含 → 并发冲突，需要业务合并
```

**混合逻辑时钟（HLC）**：物理时间 + 逻辑计数，兼顾可读性与因果性，常见于分布式数据库。

**工程建议**

- 业务时间用带时区的 UTC 时间戳（记录“何时发生”）。
- 排序用数据库序列/版本号，而不是时间戳。
- 需要“严格递增”的 ID 用专用生成器（见 14.10），不要用时间戳凑。
- 超时判断用单调时钟；记录事件时间用墙上时钟，并接受其偏差。

### 14.7 共识与协调

**共识解决的问题**：一组节点就某个值达成一致，且在主节点故障后仍能继续。典型用途：

- 选主（leader election）
- 配置/元数据的一致存储
- 分布式锁的底层（租约）
- 分片路由表

**Raft 的直觉**（不需要背论文，但要理解性质）：

```text
1. 任期（term）递增，每任期最多一个 leader
2. 写请求必须经 leader，追加到日志
3. 多数派复制成功后才提交
4. leader 故障 → 选举超时 → 获得多数票者成为新 leader
5. 新 leader 必须拥有最新日志（投票约束）
```

**关键性质**：只要多数派存活，系统就能继续工作；网络分区时少数派无法提交写入（这正是一致性优先的体现）。

**工程建议**

- **不要自研共识**。用 etcd、ZooKeeper、Consul、云托管服务，或直接用支持共识的数据库。
- 共识组不宜过大（3 或 5 个节点），跨地域部署会增加写延迟。
- 共识系统是“控制面”，不要用它存高吞吐业务数据。
- 读也要注意：默认读可能读到旧 leader 数据，需要线性一致读时使用 ReadIndex/租约读。

**Dart 的接入方式**：通过 etcd/Consul 的 HTTP API、云厂商 SDK 或 gRPC 使用协调服务；不要试图在 Dart 中实现 Paxos。

### 14.8 分布式事务全景

| 方案 | 一致性 | 复杂度 | 阻塞 | 适用 |
|---|---|---|---|---|
| 本地事务 | 强 | 低 | 短 | 单库单服务 |
| 2PC/XA | 强 | 高 | 同步阻塞、协调者单点 | 同构数据库、低频 |
| 3PC | 较强 | 很高 | 减少阻塞但仍有缺陷 | 少见 |
| TCC（Try-Confirm-Cancel） | 最终一致 | 高 | 非阻塞 | 资金、库存等需要预留资源 |
| Saga | 最终一致 | 中-高 | 非阻塞 | 跨服务长流程 |
| Outbox + 事件 | 最终一致 | 中 | 非阻塞 | 事件驱动、派生数据 |
| 本地消息表 | 最终一致 | 中 | 非阻塞 | 与 outbox 类似 |
| 对账补偿 | 兜底 | 中 | 非阻塞 | 与外部系统交互 |

**TCC 示例（库存）**

```text
Try：预占库存（available -= qty, frozen += qty）
Confirm：确认扣减（frozen -= qty）
Cancel：释放预占（available += qty, frozen -= qty）
```

要求：三个操作都幂等；Try 成功但 Confirm 迟迟不来时，需要超时回查与自动 Cancel。

**选型树**

```text
必须强一致？
├─ 是 → 能否放进同一个数据库/服务？
│        ├─ 是 → 本地事务（首选）
│        └─ 否 → 2PC（同构、低频）或重新划分服务边界
└─ 否 → 跨服务流程？
         ├─ 是 → Saga（有明确补偿）或 TCC（需要资源预留）
         └─ 否 → Outbox + 事件（派生数据、通知）
```

**最重要的原则**：**先尝试把需要强一致的数据放进同一个聚合/服务**。分布式事务是边界划分失败的补救，不是首选方案。

### 14.9 分布式锁的正确性

**常见实现**

```dart
// Redis：设置带过期时间的锁，值为唯一 token
final acquired = await redis.set('lock:order:$id', token, nx: true, px: 30000);

// 释放：必须校验 token，避免删掉别人的锁（用 Lua 保证原子）
const releaseScript = '''
if redis.call("get", KEYS[1]) == ARGV[1] then
  return redis.call("del", KEYS[1])
else
  return 0
end
''';
await redis.eval(releaseScript, keys: ['lock:order:$id'], args: [token]);
```

**必须理解的边界**

1. **锁过期 ≠ 业务完成**：GC 停顿或慢查询可能超过租约，导致两个持有者同时执行。
2. **续期有风险**：续期线程本身可能挂起。
3. **时钟漂移**：依赖过期时间的锁在时钟不准时不可靠。
4. **Redlock 争议**：多 Redis 实例的多数派方案在极端网络条件下无法提供严格安全性（业界有明确争论）。

**Fencing Token（栅栏令牌）**：让“过期后仍执行”的旧持有者无法造成破坏。

```text
锁服务每次授予递增 token：33 → 34 → 35
资源服务记录见过的最大 token
任何写请求携带 token，若小于已见最大值 → 拒绝
```

```dart
Future<void> writeWithFence(String resource, int token, Value value) async {
  final lastSeen = await store.getLastFence(resource);
  if (token <= lastSeen) {
    throw StaleLockException('令牌 $token 已过期（最新 $lastSeen）');
  }
  await store.write(resource, value, fence: token);
}
```

**更可靠的替代方案**

- 用**数据库唯一约束/条件更新**表达互斥（多数场景够用）。
- 用**租约表 + `WHERE locked_until < now()`** 获取任务所有权。
- 用**幂等**让重复执行无害——这往往比“保证只执行一次”更现实。

**结论**：分布式锁适合“减少重复工作”，不适合“保证正确性”。正确性应由幂等、约束和 fencing 保证。

### 14.10 分布式 ID

| 方案 | 有序性 | 长度 | 依赖 | 风险 |
|---|---|---|---|---|
| 数据库自增 | 严格递增 | 小 | 单库 | 扩展难、泄露规模 |
| 号段模式 | 递增（段内） | 中 | 数据库 | 需预分配、重启浪费 |
| UUIDv4 | 无序 | 36 字符 | 无 | 索引碎片、不可读 |
| UUIDv7 | 时间有序 | 36 字符 | 时钟 | 时钟回拨需处理 |
| ULID | 时间有序 | 26 字符 | 时钟 | 同上 |
| Snowflake | 趋势递增 | 64 位 | 机器 ID + 时钟 | 时钟回拨、机器 ID 管理 |

**Snowflake 结构**

```text
| 1 bit 符号 | 41 bit 毫秒时间戳 | 10 bit 机器 ID | 12 bit 序列号 |
每毫秒每机器最多 4096 个 ID
```

```dart
final class Snowflake {
  Snowflake({required this.machineId, this.epoch = const _Epoch()})
      : assert(machineId >= 0 && machineId < 1024);

  final int machineId;
  final _Epoch epoch;

  int _lastMs = -1;
  int _sequence = 0;

  int nextId() {
    var now = DateTime.now().millisecondsSinceEpoch;

    if (now < _lastMs) {
      // 时钟回拨：等待追平或使用备用逻辑（如借用未来序列/抛错）
      final drift = _lastMs - now;
      if (drift > 5) throw StateError('时钟回拨 ${drift}ms，拒绝生成 ID');
      while (now < _lastMs) {
        now = DateTime.now().millisecondsSinceEpoch;
      }
    }

    if (now == _lastMs) {
      _sequence = (_sequence + 1) & 0xFFF;
      if (_sequence == 0) {
        while (now <= _lastMs) {
          now = DateTime.now().millisecondsSinceEpoch;
        }
      }
    } else {
      _sequence = 0;
    }

    _lastMs = now;
    return ((now - epoch.value) << 22) | (machineId << 12) | _sequence;
  }
}

final class _Epoch {
  const _Epoch() : value = 1700000000000;
}
```

**注意**

- 机器 ID 必须由编排系统分配且唯一（K8s StatefulSet 序号、配置中心）。
- 公开 ID 不建议直接用自增（可枚举），可用“内部自增 + 外部 ULID”双 ID。
- ID 生成器本身要是单点无状态或具备高可用，不能成为下单瓶颈。

### 14.11 幂等、去重与对账

**幂等的三个层次**

1. **天然幂等**：`SET status='paid'`、`PUT` 整体替换。
2. **幂等键**：客户端或调用方提供唯一键，服务端记录结果（见 7.5）。
3. **业务去重**：唯一约束、状态机前置检查。

**去重窗口**：不能无限保留幂等记录。

```sql
-- 保留 24 小时，定期清理
DELETE FROM idempotency_keys WHERE expires_at < now() LIMIT 10000;
```

**对账（Reconciliation）**：分布式系统必须有“最终校验”机制。

```text
每日对账：
1. 拉取渠道流水（支付/退款）
2. 与本地交易表逐笔比对
3. 分类差异：本地有渠道无（掉单）、渠道有本地无（漏记）、金额不一致
4. 自动修复可判定的差异，其余进入人工工单
5. 记录对账结果与差异趋势
```

对账是“最终一致”的兜底，不是可选项。

### 14.12 分布式限流与配额

**集中式令牌桶**（Redis + Lua）见 10.6；关键点是原子性与时钟来源。

**配额分配**

```text
全局配额 10000 rpm
├─ 租户 A 3000
├─ 租户 B 5000
└─ 其他共享 2000
```

- 配额存储要有版本/审计，变更走审批。
- 热点租户的限流键要分片，避免单键 QPS 过高。
- 限流失败要“快速拒绝”，不要排队导致延迟雪崩。

**本地 + 全局两级**：本地令牌桶做第一道闸（防单实例过载），Redis 做全局配额（防总量超限）。本地桶按 `全局配额 / 实例数` 分配，并留余量。

### 14.13 多区域与容灾

**部署形态**

| 形态 | 写入 | 读 | RPO | RTO | 适用 |
|---|---|---|---|---|---|
| 单区域多可用区 | 单区域 | 本地 | ≈0（同步副本） | 分钟级 | 多数业务 |
| 主备（Active-Passive） | 主区域 | 主区域 | 秒-分钟 | 分钟-小时 | 灾备合规 |
| 双活（Active-Active） | 多区域 | 本地 | 取决于复制 | 秒级 | 全球低延迟 |
| 单元化（Cell-based） | 按用户路由 | 本地单元 | 单元内强一致 | 单元级隔离 | 超大规模 |

**关键问题**

1. **数据驻留**：某些地区要求数据不出境 → 区域化存储与路由。
2. **写冲突**：双活写入需要冲突解决或按用户归属单写区域。
3. **故障切换**：DNS/全局负载均衡切换、数据库提升、缓存预热。
4. **脑裂**：网络分区时两侧都认为自己是主 → 需要 quorum/fencing 防双写。
5. **切换演练**：不演练的容灾等于没有容灾。

**按用户归属单写**是双活的常见折中：用户数据只在其“归属区域”写，其他区域只读（或转发写请求）。这样避免了多主冲突，同时获得本地读延迟。

### 14.14 分布式系统的验证

**分层验证**

| 层级 | 验证什么 | 方法 |
|---|---|---|
| 单元 | 幂等函数、状态机、补偿逻辑 | 纯逻辑测试 |
| 集成 | 真实 DB/队列/Redis 的并发行为 | 容器化依赖 + 并发测试 |
| 故障注入 | 依赖超时、断连、重启 | toxiproxy、故障开关、杀进程 |
| 一致性检查 | 是否满足线性一致/最终一致 | Jepsen 类工具、历史检查 |
| 混沌工程 | 随机故障下的系统行为 | 随机杀实例、注入延迟 |
| 对账 | 与外部系统数据一致 | 每日对账任务 |

**并发正确性测试示例**

```dart
test('同一幂等键并发请求只创建一单', () async {
  final key = 'idem-${DateTime.now().microsecondsSinceEpoch}';
  final results = await Future.wait(
    List.generate(20, (_) => api.createOrder(key, payload)),
  );

  final ids = results.map((r) => r.body['id']).toSet();
  expect(ids, hasLength(1), reason: '并发请求应返回同一订单');

  final count = await db.query(
    'SELECT count(*) FROM orders WHERE idempotency_key = @k',
    parameters: {'k': key},
  );
  expect(count.single[0], 1);
});
```

**故障注入清单**

- [ ] 数据库连接中断 30 秒 → 服务返回 503，恢复后自动可用
- [ ] Redis 不可用 → 降级到数据库或限流，不雪崩
- [ ] 消息消费者重复投递 → 业务只生效一次
- [ ] 下游超时 → 上游按时返回，不无限挂起
- [ ] 实例被杀 → 在途请求被正确处理（重试或明确失败）
- [ ] 时钟回拨 → ID 生成器拒绝或等待，不产生重复
- [ ] 网络分区 → 少数派停止写入，不脑裂

### 14.15 常见误区与检查清单

**误区**

1. **认为 CAP 是“三选二”**：忽略了分区是常态、C 指线性一致。
2. **认为“最终一致”就是不管**：缺少对账与收敛监控的最终一致等于数据损坏。
3. **用时间戳做并发判断**：时钟漂移会导致丢更新。
4. **用分布式锁保证正确性**：锁会过期，需要幂等 + fencing。
5. **认为重试是安全的**：非幂等写会重复执行。
6. **用“多数派”解决脑裂而不做 fencing**：旧主仍可能写入。
7. **多主复制不做冲突解决**：冲突会被静默覆盖。
8. **把共识系统当数据存储**：etcd 不是业务数据库。
9. **忽略复制滞后**：用户写完立刻读不到，投诉“数据丢了”。
10. **没有对账**：跨系统金额差异永远无法发现。
11. **自研分布式 ID 不处理时钟回拨**：产生重复 ID 或负增长。
12. **分区扩容不限速**：再平衡打满 IO，引发连锁故障。

**检查清单**

- [ ] 每个跨节点调用都有超时与幂等保护
- [ ] 关键读的一致性要求已标注，并据此选择主/副本/缓存
- [ ] 复制滞后有监控与告警
- [ ] 热点键有拆分或本地缓存方案
- [ ] 分布锁仅用于“减少重复”，正确性由约束/幂等/fencing 保证
- [ ] 分布式 ID 有时钟回拨处理与唯一机器 ID 分配
- [ ] 跨服务流程有 Saga/outbox，且有补偿与人工干预
- [ ] 有每日对账与差异趋势监控
- [ ] 多区域部署的 RPO/RTO 明确，切换演练过
- [ ] 故障注入与混沌演练定期执行

---

## 15. 消息队列、实时通信与后台任务

同步请求应该短。任何“可以稍后做”的事情都应该移出请求路径：发邮件、生成报表、同步搜索索引、调用不稳定的第三方。

### 13.1 消息系统形态

| 形态 | 代表 | 语义 | 适用 |
|---|---|---|---|
| 队列（点对点） | SQS、RabbitMQ queue | 一条消息一个消费者 | 任务分发 |
| 发布订阅 | RabbitMQ exchange、SNS | 一条消息多个订阅者 | 事件广播 |
| 日志（分区） | Kafka、Redpanda | 持久、可重放、顺序（分区内） | 事件流、CDC、审计 |
| 轻量流 | Redis Streams | 消费组、ACK | 中小规模任务流 |
| 云托管 | Pub/Sub、EventBridge | 托管、集成云服务 | 云原生 |

**Dart 接入方式**：优先使用标准协议（AMQP、Kafka 协议、HTTP API）的客户端包，或通过云厂商 REST/gRPC API。选型时确认包的维护状态与重连/背压能力。

### 13.2 投递语义与幂等消费者

多数系统提供 **at-least-once**：消息可能重复投递。因此消费者必须幂等。

```dart
final class SendEmailHandler {
  SendEmailHandler({required this.inbox, required this.mailer});

  final InboxRepository inbox;   // 已处理消息表
  final Mailer mailer;

  Future<void> handle(Message msg) async {
    final claimed = await inbox.tryClaim(msg.id);   // INSERT ... ON CONFLICT DO NOTHING
    if (!claimed) {
      _log.info('消息已处理，跳过', fields: {'messageId': msg.id});
      return;                                       // 幂等
    }

    try {
      await mailer.send(msg.payload);
      await inbox.markDone(msg.id);
    } catch (e) {
      await inbox.markFailed(msg.id, e.toString());
      rethrow;                                      // 让队列重试
    }
  }
}
```

```sql
CREATE TABLE inbox_messages (
  message_id  text PRIMARY KEY,
  status      text NOT NULL,       -- processing | done | failed
  attempts    int  NOT NULL DEFAULT 0,
  last_error  text,
  received_at timestamptz NOT NULL DEFAULT now()
);
```

**关键点**：claim 与业务写入应在同一事务，或 claim 后业务失败要释放 claim（否则消息永久丢失）。

### 13.3 Transactional Outbox

**问题**：数据库提交成功但发消息失败 → 事件丢失；发消息成功但事务回滚 → 幽灵事件。

**方案**：把事件写入同库的 outbox 表（同一事务），再由独立进程投递。

```sql
CREATE TABLE outbox_events (
  id           bigserial PRIMARY KEY,
  aggregate_id text NOT NULL,
  type         text NOT NULL,
  payload      jsonb NOT NULL,
  created_at   timestamptz NOT NULL DEFAULT now(),
  published_at timestamptz,
  attempts     int NOT NULL DEFAULT 0
);

CREATE INDEX idx_outbox_unpublished
ON outbox_events (created_at)
WHERE published_at IS NULL;
```

```dart
Future<void> createOrder(CreateOrderCommand cmd) async {
  await db.transaction((tx) async {
    await tx.execute(Sql.named('INSERT INTO orders ...'), parameters: {...});
    await tx.execute(
      Sql.named('''
        INSERT INTO outbox_events (aggregate_id, type, payload)
        VALUES (@id, 'order.created', @payload)
      '''),
      parameters: {'id': orderId, 'payload': jsonEncode(event)},
    );
  });
}

// 独立投递循环
Future<void> publishLoop() async {
  while (!stopping) {
    final batch = await outbox.claimBatch(limit: 50);
    for (final e in batch) {
      await broker.publish(e.type, e.payload);
      await outbox.markPublished(e.id);
    }
    if (batch.isEmpty) await Future.delayed(const Duration(milliseconds: 500));
  }
}
```

**要点**：至少一次投递 → 消费端必须幂等；outbox 表要定期清理；投递失败要退避重试与告警。

### 13.4 重试、死信与毒消息

```text
消费失败 → 可重试？
   ├─ 是 → attempts < max ?
   │        ├─ 是 → 退避后重试（1s, 5s, 30s, 5m... + 抖动）
   │        └─ 否 → 进入死信队列 + 告警
   └─ 否 → 立即进入死信队列（记录原因）
```

**毒消息**：永远处理失败的消息（如 schema 不兼容）。必须能隔离，否则会阻塞整个分区/队列。

**重试的原则**

- 只对瞬时错误重试（超时、503、连接失败）。
- 不对确定性错误重试（400、schema 错误、业务规则拒绝）。
- 重试要有总时限与次数上限。
- 重试要加抖动，避免“重试风暴”同时打满下游。
- 记录每次尝试，便于排查。

### 13.5 定时任务

**单机 cron 的问题**：多副本同时触发。

**方案**

1. **分布式调度器**（K8s CronJob + 单实例、云调度服务）。
2. **数据库租约**：

```sql
UPDATE jobs
SET locked_by = $1, locked_until = now() + interval '5 minutes'
WHERE name = $2
  AND (locked_until IS NULL OR locked_until < now())
RETURNING id;
```

3. **幂等 + 去重键**：即使重复执行也安全。

**任务设计要点**

- 记录运行历史（开始、结束、处理条数、错误）。
- 支持断点续跑（游标/水位）。
- 分批处理并限速，避免打满数据库。
- 设置超时与最大执行时长。
- 错过调度的补跑策略明确（业务决定）。
- 时区与夏令时问题：用 UTC + 显式时区规则。

### 13.6 WebSocket 实战要点

```dart
// 服务端：Shelf + shelf_web_socket（示意）
Handler wsHandler(ConnectionRegistry registry) {
  return (Request request) async {
    final principal = await authenticateWs(request);   // 从 query/subprotocol/首帧令牌
    final socket = await WebSocketTransformer.upgrade(request);
    final conn = registry.add(principal, socket);

    socket.listen(
      (data) => conn.onMessage(data),
      onDone: () => registry.remove(conn),
      onError: (e) => registry.remove(conn),
      cancelOnError: true,
    );

    return Response.upgrade;   // 实际 API 依包而定
  };
}
```

**必须实现**

1. **连接鉴权**：握手时校验令牌；令牌过期后要求重新认证或断开。
2. **心跳**：`ping/pong` 或应用层心跳，检测半开连接。
3. **重连**：客户端指数退避；服务端支持断线后补发（用消息序号/游标）。
4. **消息去重**：客户端消息带 `messageId`，服务端幂等处理。
5. **背压**：发送队列有上限，超限断开慢客户端。
6. **跨实例广播**：Redis Pub/Sub、NATS、Kafka 等；记录“哪个实例持有哪个连接”。
7. **连接上限**：按用户/租户/IP 限制，防止资源耗尽。
8. **优雅关闭**：发送关闭帧、通知客户端重连其他实例。

**扩展性**：单实例连接数受文件描述符与内存限制；需要调整 `ulimit`、使用多实例 + 路由层。

### 13.7 SSE（Server-Sent Events）

适合单向推送（进度、通知、行情）。

```dart
Response sse(Stream<ServerEvent> events) {
  final controller = StreamController<List<int>>();
  events.listen(
    (e) => controller.add(utf8.encode('data: ${jsonEncode(e.toJson())}\n\n')),
    onDone: controller.close,
  );
  return Response.ok(
    controller.stream,
    headers: {
      'content-type': 'text/event-stream',
      'cache-control': 'no-cache',
      'connection': 'keep-alive',
    },
  );
}
```

注意：代理可能缓冲响应（需 `X-Accel-Buffering: no`）；必须处理客户端断开；需要心跳注释行防超时。

### 13.8 Webhook

**作为发送方**

```text
1. 生成事件 ID 与时间戳
2. 计算签名：HMAC-SHA256(secret, timestamp + "." + body)
3. 发送 POST，头：X-Event-Id, X-Timestamp, X-Signature
4. 2xx → 成功；否则按退避重试（如 1m, 5m, 30m, 2h, 6h）
5. 记录每次投递结果，提供重放接口
6. 连续失败 → 禁用端点 + 通知
```

**作为接收方**

```dart
bool verifyWebhook(String body, String timestamp, String signature, String secret) {
  // 1. 时间戳新鲜度（防重放）
  final ts = int.tryParse(timestamp);
  if (ts == null) return false;
  final now = DateTime.now().millisecondsSinceEpoch ~/ 1000;
  if ((now - ts).abs() > 300) return false;

  // 2. HMAC 恒定时间比较
  final expected = hmacSha256(secret, '$timestamp.$body');
  return constantTimeEquals(expected, signature);
}
```

再配合事件 ID 去重、快速返回 2xx、异步处理。

---

## 16. 测试策略与质量保障

测试的目标不是“覆盖率数字”，而是**让你敢于修改代码**。后端测试要回答三个问题：业务规则对不对？失败路径是否可控？部署后能否验证？

### 14.1 测试组合与投入比例

```text
        ╱╲          E2E（少量，关键流程）
       ╱  ╲
      ╱────╲        API / 集成测试（中量，真实依赖）
     ╱      ╲
    ╱────────╲      单元测试（大量，快、纯逻辑）
   ╱──────────╲
```

经验比例（不是教条）：

| 层级 | 占比 | 运行时间 | 稳定性 |
|---|---|---|---|
| 单元 | 60–70% | 秒级 | 高 |
| 集成（真实 DB） | 20–30% | 十秒–分钟 | 中 |
| API/契约 | 5–10% | 分钟 | 中 |
| E2E | <5% | 分钟–十分钟 | 较低 |

**反模式**：大量依赖真实网络的“单元测试”（慢且不稳定）、只有 E2E 没有单元（失败难定位）、只测 happy path。

### 14.2 单元测试基础

```dart
import 'package:test/test.dart';

void main() {
  group('Money', () {
    test('同币种相加', () {
      final a = Money.ofMinor(1000, 'CNY');
      final b = Money.ofMinor(250, 'CNY');
      expect((a + b).minorUnits, 1250);
    });

    test('币种不一致时拒绝', () {
      expect(
        () => Money.ofMinor(100, 'CNY') + Money.ofMinor(100, 'USD'),
        throwsStateError,
      );
    });

    test('小数解析避免浮点误差', () {
      expect(Money.ofDecimal('0.10', 'CNY').minorUnits, 10);
      expect(Money.ofDecimal('0.20', 'CNY').minorUnits, 20);
      expect(
        (Money.ofDecimal('0.10', 'CNY') + Money.ofDecimal('0.20', 'CNY'))
            .toDecimalString(),
        '0.30',
      );
    });

    test('非法金额抛出格式异常', () {
      expect(() => Money.ofDecimal('1.234', 'CNY'), throwsFormatException);
    });
  });
}
```

**命名**：`被测对象_场景_期望结果` 或中文描述句。测试名要能当文档读。

**Given-When-Then**

```dart
test('订单已发货时取消会被拒绝', () {
  // Given
  final order = createShippedOrder();

  // When / Then
  expect(() => order.cancel('用户请求'), throwsA(isA<DomainException>()));
});
```

**断言选择**

```dart
expect(value, equals(42));
expect(list, isEmpty);
expect(list, contains('a'));
expect(() => fn(), throwsA(isA<ValidationException>()));
expect(future, completion(isTrue));
await expectLater(stream, emitsInOrder([1, 2, 3]));
```

### 14.3 测试替身（Fake / Stub / Mock）

优先使用**手写 Fake**：可读、可调试、无框架魔法。

```dart
final class FakeOrderRepository implements OrderRepository {
  final _store = <String, Order>{};
  final _saved = <Order>[];

  @override
  Future<Order?> findById(OrderId id) async => _store[id.value];

  @override
  Future<void> save(Order order) async {
    _store[order.id.value] = order;
    _saved.add(order);
  }

  @override
  Future<List<Order>> pageByCustomer(
    CustomerId customerId, {
    String? cursor,
    int limit = 20,
  }) async =>
      _store.values.where((o) => o.customerId == customerId).take(limit).toList();

  List<Order> get saved => List.unmodifiable(_saved);
}
```

**Fake 时钟与 ID**

```dart
final class FixedClock implements Clock {
  FixedClock(this._now);
  final DateTime _now;
  @override
  DateTime now() => _now;
}

final class SequentialIds implements IdGenerator {
  var _n = 0;
  @override
  String next() => 'id_${++_n}';
}
```

**何时用 Mock（行为验证）**：只关心“是否调用了外部副作用”（发邮件、调支付）时。

```dart
test('支付失败时不发送确认邮件', () async {
  final mailer = RecordingMailer();
  final service = CreateOrder(..., payments: FailingPayment(), mailer: mailer);

  await expectLater(service.execute(cmd), throwsA(isA<PaymentDeclined>()));
  expect(mailer.sent, isEmpty);
});
```

**不要**对实现细节做过度断言（调用了几次某个私有方法），否则重构会大量误报。

### 14.4 HTTP 层测试

```dart
import 'package:shelf/shelf.dart';
import 'package:test/test.dart';

void main() {
  late Handler app;

  setUp(() {
    app = buildApp(testDeps());
  });

  test('GET /v1/orders/:id 返回 404 当订单不存在', () async {
    final response = await app(
      Request('GET', Uri.parse('http://localhost/v1/orders/o_missing')),
    );
    expect(response.statusCode, 404);
    final body = jsonDecode(await response.readAsString()) as Map<String, dynamic>;
    expect(body['error']['code'], 'not_found');
  });

  test('POST /v1/orders 缺少幂等键返回 400', () async {
    final response = await app(
      Request(
        'POST',
        Uri.parse('http://localhost/v1/orders'),
        body: jsonEncode({'customerId': 'c_1', 'currency': 'CNY', 'lines': []}),
        headers: {'content-type': 'application/json'},
      ),
    );
    expect(response.statusCode, 400);
  });
}
```

**必须覆盖的 HTTP 行为**

- 认证缺失/过期/无效 → 401。
- 无权限/跨租户 → 403 或 404（按约定）。
- 非法 JSON、错误类型、缺失字段 → 400/422 且字段级错误正确。
- 不支持的 `Content-Type` → 415。
- 方法不允许 → 405 + `Allow`。
- 分页边界（limit=0、limit=1000、非法 cursor）。
- 响应头（`content-type`、`location`、缓存头、requestId）。

### 14.5 数据库集成测试

**原则**：用真实数据库，不用内存替身（SQL 方言、约束、事务行为都不同）。

```dart
void main() {
  late Db db;

  setUpAll(() async {
    db = await Db.connect(Platform.environment['TEST_DATABASE_URL']!);
    await migrate(db, to: 'latest');
  });

  setUp(() async {
    await db.query('TRUNCATE orders, inventory RESTART IDENTITY CASCADE');
  });

  tearDownAll(() async => db.close());

  test('并发扣减库存不会超卖', () async {
    await db.query(
      "INSERT INTO inventory (sku, available) VALUES ('SKU-1', 10)",
    );

    final results = await Future.wait(
      List.generate(20, (_) => tryReserve(db, 'SKU-1', 1)),
    );

    expect(results.where((ok) => ok).length, 10);
    final rows = await db.query("SELECT available FROM inventory WHERE sku = 'SKU-1'");
    expect(rows.single[0], 0);
  });
}
```

**并发测试要真实并发**：`Future.wait` 在单 isolate 内对异步 I/O 是并发的（多个连接），但要注意数据库连接池大小要 ≥ 并发数，否则测的是排队而不是竞争。

**迁移测试**

```dart
test('从生产快照 schema 升级到最新版本', () async {
  await restoreSnapshot(db, 'test/fixtures/prod_schema_v12.dump');
  await migrate(db, to: 'latest');
  // 校验关键表结构、数据完整性、索引存在
});
```

### 14.6 契约测试

- 服务端：用 OpenAPI 校验实际响应符合 schema。
- 消费方：用 Pact 类工具或自建“录制-回放”验证提供方兼容。
- 内部服务：用 gRPC/JSON schema 的向后兼容检查（字段编号、必填性）。

```dart
test('创建订单响应符合契约', () async {
  final response = await client.postOrder(validPayload);
  final schema = loadJsonSchema('openapi.yaml#/components/schemas/Order');
  final errors = validateSchema(schema, response.body);
  expect(errors, isEmpty);
});
```

### 14.7 负载与压力测试

**目标要先定义**：

```text
场景：下单接口
- 目标吞吐：200 RPS
- P95 < 300ms，P99 < 800ms
- 错误率 < 0.1%
- 持续 30 分钟
```

**方法**

1. 准备接近生产规模的数据（不能空表压测）。
2. 逐步加压（ramp-up），观察拐点。
3. 记录 CPU、内存、连接池、DB 慢查询、GC（Dart VM）。
4. 找到极限后，把容量规划定在 50–70%。
5. 做**过载测试**：看系统在超载时是优雅降级还是雪崩。

**工具**：k6、wrk、Gatling 等；Dart 侧可写简单压测脚本但注意压测客户端自身不要成为瓶颈。

### 14.8 测试数据管理

- 每个测试自建数据，不依赖执行顺序。
- 用工厂函数生成合法对象，显式覆盖需要变化的字段。
- 集成测试用事务回滚或 `TRUNCATE`，避免污染。
- 敏感数据用合成数据，绝不复制生产个人信息到测试环境。
- 测试数据库与生产隔离（独立实例/独立账号/独立网络）。

```dart
Order sampleOrder({OrderStatus status = OrderStatus.pending}) => Order.create(
      id: OrderId('o_test'),
      customerId: CustomerId('c_test'),
      lines: [OrderLine(sku: 'SKU-1', quantity: 1, unitPrice: Money.ofMinor(100, 'CNY'))],
      now: DateTime.utc(2026, 1, 1),
    )..status = status;   // 仅测试便捷用法
```

### 14.9 CI 门禁

```yaml
# 概念示例
- dart format --output=none --set-exit-if-changed .
- dart analyze --fatal-infos
- dart run build_runner build --delete-conflicting-outputs
- git diff --exit-code    # 生成代码无漂移
- dart test --exclude-tags=integration
- docker compose up -d postgres && dart test --tags=integration
- dart compile exe bin/server.dart -o /tmp/server   # 构建验证
```

**门禁原则**

- 失败必须阻断合并。
- 测试要快（单元 < 60s），慢测试并行/分层。
- 禁止 flaky 测试长期存在：修复或删除，不要重跑掩盖。
- 覆盖率作为趋势指标，不作为唯一门禁；关键模块可设更高阈值。

### 14.10 对测试的常见误解

| 误解 | 事实 |
|---|---|
| 覆盖率高 = 质量高 | 覆盖率只说明执行过，不说明断言有效 |
| Mock 越多越好 | 过度 mock 测的是 mock 而非系统 |
| E2E 能替代单元测试 | E2E 慢、脆、定位差 |
| 测试是 QA 的事 | 开发者对自己代码的失败路径最清楚 |
| 手工点一遍就够了 | 手工无法回归，无法在每次提交执行 |

---

## 17. 性能、可靠性与可观测性

### 15.1 延迟预算与分解

一个 P95 = 400ms 的接口，通常长这样：

```text
网关        5ms
认证        3ms（含 JWKS 缓存）
参数校验    1ms
业务逻辑    10ms
DB 查询     80ms（2 次）
缓存        3ms
下游 API    250ms   ← 真正的瓶颈
序列化      5ms
网络与排队  43ms
```

**做法**：在代码里对每个阶段打点（Span），用真实数据找出瓶颈，而不是猜。

**优化顺序**

1. 消除 N+1 与重复查询。
2. 加合适的索引。
3. 减少下游调用次数（批量/缓存）。
4. 并行化独立调用。
5. 降低序列化/拷贝开销。
6. 最后才考虑加机器/换语言。

### 15.2 Dart 性能分析工具

| 工具 | 用途 |
|---|---|
| Dart DevTools（CPU Profiler） | 找热点函数 |
| Timeline | 事件时序、异步等待 |
| VM Service | 内存、堆快照、分配 |
| `--enable-vm-service` | 远程连接分析 |
| 基准测试（`benchmark_harness`） | 微基准，注意预热 |

**注意**：AOT 生产环境与 JIT 开发环境的性能特征不同；微基准要用 `benchmark_harness` 做预热与统计，不要用 `Stopwatch` 单次测量下结论。

### 15.3 常见后端性能问题

| 问题 | 症状 | 修复 |
|---|---|---|
| N+1 查询 | DB QPS 高、延迟随列表长度线性增长 | 批量加载/JOIN |
| 缺失索引 | 慢查询日志、`Seq Scan` | 建索引、验证执行计划 |
| 连接池过小 | 请求排队、acquire timeout | 调池 + 查泄漏 |
| 连接池过大 | DB CPU 饱和、上下文切换 | 降池、加只读副本 |
| 大响应 | 带宽高、序列化慢 | 分页、字段裁剪、压缩 |
| 同步 CPU 工作 | 事件循环阻塞、延迟尖刺 | isolate / 异步化 |
| 日志过多 | I/O 饱和、成本高 | 采样、结构化、降级 |
| 正则回溯 | 单请求 CPU 爆炸 | 限制输入长度、使用安全正则 |
| JSON 解析大文档 | 内存峰值 | 限制大小、流式解析 |
| 缓存 miss 风暴 | DB 突发 | 单飞、预热、抖动 TTL |

### 15.4 超时、重试、熔断与舱壁

```dart
final class ResilientClient {
  ResilientClient({
    required this.inner,
    this.timeout = const Duration(seconds: 2),
    this.maxAttempts = 3,
    this.breaker = const CircuitBreaker(failureThreshold: 5, resetAfter: Duration(seconds: 30)),
  });

  final HttpClientLike inner;
  final Duration timeout;
  final int maxAttempts;
  final CircuitBreaker breaker;

  Future<Response> get(Uri uri) async {
    return breaker.run(() async {
      var attempt = 0;
      while (true) {
        attempt++;
        try {
          return await inner.get(uri).timeout(timeout);
        } on TimeoutException catch (e) {
          if (attempt >= maxAttempts) rethrow;
          await Future<void>.delayed(_backoff(attempt));
        } on RetryableHttpException catch (e) {
          if (attempt >= maxAttempts) rethrow;
          await Future<void>.delayed(_backoff(attempt));
        }
      }
    });
  }

  Duration _backoff(int attempt) {
    final base = 200 * (1 << (attempt - 1));
    final jitter = Random().nextInt(100);
    return Duration(milliseconds: base + jitter);
  }
}
```

**熔断器状态机**

```text
Closed ──失败率超阈值──> Open ──等待 resetAfter──> Half-Open
  ▲                                                  │
  └────────── 试探成功 ──────────────────────────────┘
                     （试探失败 → 回到 Open）
```

**舱壁（Bulkhead）**：为不同下游分配独立并发池，防止一个慢依赖占满所有资源。

**重试的红线**

- 非幂等写操作不得自动重试，除非有幂等键。
- 重试必须在总 deadline 内，不能无限叠加。
- 下游已经过载时，重试会加剧故障 → 必须配合熔断。

### 15.5 限流算法对比

| 算法 | 突发容忍 | 精度 | 实现复杂度 |
|---|---|---|---|
| 固定窗口 | 边界可 2 倍突发 | 低 | 极低 |
| 滑动日志 | 精确 | 高 | 中（存储成本） |
| 滑动窗口计数 | 较好 | 中 | 中 |
| 令牌桶 | 可配置突发 | 高 | 低-中 |
| 漏桶 | 无突发 | 高 | 低 |

分布式限流必须用集中存储（Redis + Lua）或网关能力；本地限流只能防单实例过载。

### 15.6 日志规范

**结构化日志**

```dart
logger.info(
  'order.created',
  fields: {
    'orderId': order.id.value,
    'customerId': order.customerId.value,
    'totalMinor': order.total.minorUnits,
    'currency': order.total.currency,
    'requestId': ctx.requestId,
    'traceId': ctx.traceId,
    'durationMs': stopwatch.elapsedMilliseconds,
  },
);
```

**级别使用**

| 级别 | 用途 | 是否告警 |
|---|---|---|
| `debug` | 开发细节、SQL 参数（脱敏） | 否 |
| `info` | 业务关键事件（下单、支付成功） | 否 |
| `warn` | 可恢复异常、降级、重试 | 视情况 |
| `error` | 请求失败、依赖不可用 | 是（聚合后） |
| `fatal` | 进程无法继续 | 是（立即） |

**纪律**

- 一条日志一个事件，不要拼接多行文本。
- 必带 `requestId` / `traceId`。
- 不记录密码、令牌、完整卡号、身份证、Cookie。
- 避免在循环里打日志（用计数汇总）。
- 高流量端点考虑采样（如 1% debug、100% error）。
- 日志保留期与成本要规划（热 7 天、冷 90 天等）。

**脱敏白名单优于黑名单**：明确列出允许记录的字段，而不是试图穷举敏感字段名。

### 15.7 指标

**四个黄金信号**：延迟、流量、错误、饱和度。

**RED 方法（服务）**：Rate（请求率）、Errors（错误率）、Duration（延迟分布）。

**USE 方法（资源）**：Utilization、Saturation、Errors。

**关键指标清单**

```text
http_requests_total{route,method,status}
http_request_duration_seconds{route}      # histogram
http_in_flight_requests
db_pool_in_use / db_pool_size
db_query_duration_seconds{operation}
cache_hits_total / cache_misses_total
queue_depth / queue_oldest_message_age
consumer_lag
dependency_errors_total{dependency}
event_loop_lag_seconds          # Dart 特有，关键！
```

**标签基数陷阱**：不要把 `userId`、`orderId`、原始 URL 作为标签，会导致时序爆炸。用路由模板（`/v1/orders/:id`）。

**直方图分桶**：按业务 SLO 设置（如 5ms, 10ms, 25ms, 50ms, 100ms, 250ms, 500ms, 1s, 2s）。

### 15.8 分布式追踪

```dart
// 接收上游 traceparent，创建 span
final span = tracer.startSpan(
  'POST /v1/orders',
  parent: TraceContext.fromHeaders(request.headers),
  attributes: {'route': '/v1/orders', 'method': 'POST'},
);

try {
  final child = tracer.startSpan('db.insert_order', parent: span.context);
  await db.query(...);
  child.end();
} finally {
  span.end();
}
```

**要点**

- 使用 W3C `traceparent` 标准传播。
- 采样：头部采样（按比例）或尾部采样（保留错误/慢请求）。
- 在 span 上记录关键业务属性（订单 ID 可作为属性，但不是指标标签）。
- 追踪不能替代日志：追踪看链路，日志看细节。

### 15.9 SLO 与告警

**SLI**：可测量的指标，如“成功率 = 2xx+3xx / 总请求”。

**SLO**：目标，如“30 天内 99.9% 的请求成功”。

**Error Budget**：`1 - SLO`。预算耗尽 → 冻结发布、优先稳定性。

**告警设计**

| 类型 | 例子 | 处理 |
|---|---|---|
| 症状告警（首选） | 错误率 > 1% 持续 5 分钟 | 立即响应 |
| 饱和告警 | 连接池使用率 > 90% | 扩容/排查 |
| 预测告警 | 磁盘 4 小时内将满 | 计划处理 |
| 事件告警 | 证书 30 天后过期 | 工单 |

**避免**：对每个 5xx 单独告警（噪声）、对 CPU 瞬时峰值告警（无意义）、没有 runbook 的告警。

### 15.10 容量与韧性

- 定期容量评审：按增长率预测 3–6 个月。
- 关键路径做冗余（多副本、多可用区）。
- 依赖降级方案（缓存兜底、只读模式、关闭非核心功能）。
- 故障演练：随机杀实例、注入延迟、断开依赖、模拟磁盘满。
- 演练结果要改进 runbook 与告警，而不是只写报告。

### 15.11 Dart 特有的可靠性注意点

| 注意点 | 说明 |
|---|---|
| 事件循环阻塞 | 同步 CPU 工作会拖慢所有并发请求；用 isolate |
| 未处理的异步异常 | 建议设置全局错误处理并记录 |
| 内存增长 | 大缓存/无界队列/连接泄漏是常见原因；用堆快照排查 |
| `Isolate` 崩溃 | worker 池需检测并重建 |
| AOT 与 JIT 差异 | 生产用 AOT 验证性能与行为 |
| 文件描述符 | 每个连接/文件都占用；注意 `ulimit` |
| 平台信号 | SIGTERM 处理必须实现，否则滚动发布丢请求 |

---

## 18. 部署、云平台与交付流程

### 16.1 十二要素（12-Factor）与 Dart 服务

| 要素 | 在后端的落地 |
|---|---|
| 代码库 | 一个服务一个仓库或 monorepo，同一构建产物部署到所有环境 |
| 依赖 | `pubspec.lock` 锁定，构建时 `dart pub get --enforce-lockfile` |
| 配置 | 环境变量/密钥服务，不入库 |
| 后端服务 | 无状态进程；会话/文件放外部存储 |
| 构建/发布/运行 | 构建产物不可变，配置在运行时注入 |
| 进程 | 无守护化，由平台管理；支持优雅关闭 |
| 端口绑定 | 监听 `0.0.0.0:$PORT` |
| 并发 | 通过多进程/多实例扩展 |
| 可处置性 | 快速启动、可随时杀死 |
| 开发/生产一致 | 同一镜像、同一依赖、同一迁移 |
| 日志 | 输出到 stdout/stderr，由平台收集 |
| 管理进程 | 迁移、种子、维护脚本作为一次性任务运行 |

### 16.2 Dockerfile 实战

```dockerfile
# ---------- 构建阶段 ----------
FROM dart:stable AS build

WORKDIR /app

# 先复制依赖清单，利用层缓存
COPY pubspec.* ./
RUN dart pub get --enforce-lockfile

COPY . .

# 生成代码（如有）
RUN dart run build_runner build --delete-conflicting-outputs

# AOT 编译
RUN dart compile exe bin/server.dart -o /app/build/server

# ---------- 运行阶段 ----------
FROM debian:bookworm-slim AS runtime

RUN apt-get update \
 && apt-get install -y --no-install-recommends ca-certificates tzdata \
 && rm -rf /var/lib/apt/lists/*

RUN useradd --system --uid 10001 --create-home app
WORKDIR /app

COPY --from=build /app/build/server /app/server

ENV APP_ENV=prod \
    PORT=8080

USER app
EXPOSE 8080

STOPSIGNAL SIGTERM
ENTRYPOINT ["/app/server"]
```

**要点**

- 多阶段构建：镜像中不含 SDK 与源码。
- 非 root 用户运行（`USER app`）。
- 需要 `ca-certificates`（TLS）与 `tzdata`（时区）。
- 若使用 `dart:stable` 构建，生产镜像仍应是精简运行镜像；也可用 `scratch` + 静态编译，但需验证 DNS/证书需求。
- 镜像标签用不可变 digest（`@sha256:...`）而非 `latest`。
- 健康检查由平台配置（HTTP 探针），不要只在 Dockerfile 里写 `HEALTHCHECK`。

**.dockerignore**

```text
.git
.dart_tool
build
**/*.g.dart
test
docs
.env*
*.md
```

注意：如果 CI 依赖生成代码且不入库，不要忽略生成文件；团队需统一策略。

### 16.3 部署形态对比

| 形态 | 启动 | 伸缩 | 运维复杂度 | 适合 |
|---|---|---|---|---|
| 单 VM + systemd | 进程 | 手动/脚本 | 低 | 小规模、稳定流量 |
| 容器 + 托管编排（ECS/Cloud Run） | 秒级 | 自动 | 中 | 大多数业务服务 |
| Kubernetes | 秒级 | HPA/VPA | 高 | 多服务、需要统一平台 |
| Serverless 函数 | 毫秒-秒（冷启动） | 自动 | 低-中 | 事件驱动、低频任务 |
| 边缘运行时 | 毫秒 | 自动 | 中 | 鉴权/重写/低延迟入口 |

**Dart 在 Serverless 的现实**：平台是否支持 Dart 运行时、是否支持自定义容器、冷启动与包大小、执行时长上限、是否支持 WebSocket。**AOT 容器化**通常是 Dart 最稳妥的路径。

### 16.4 Kubernetes 关键配置

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-api
spec:
  replicas: 3
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels: { app: order-api }
  template:
    metadata:
      labels: { app: order-api }
    spec:
      terminationGracePeriodSeconds: 30
      containers:
        - name: app
          image: registry.example.com/order-api@sha256:...
          ports:
            - containerPort: 8080
          env:
            - name: APP_ENV
              value: prod
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef: { name: order-api-secrets, key: database-url }
          resources:
            requests: { cpu: '250m', memory: '256Mi' }
            limits:   { cpu: '1000m', memory: '512Mi' }
          readinessProbe:
            httpGet: { path: /readyz, port: 8080 }
            initialDelaySeconds: 3
            periodSeconds: 5
            failureThreshold: 3
          livenessProbe:
            httpGet: { path: /healthz, port: 8080 }
            initialDelaySeconds: 10
            periodSeconds: 15
            failureThreshold: 4
          lifecycle:
            preStop:
              exec:
                command: ['sh', '-c', 'sleep 5']   # 等待负载均衡摘除
```

**要点**

- `requests` 决定调度，`limits` 决定上限；内存 limit 过低会 OOMKill。
- `readinessProbe` 失败 → 摘流量但不重启；`livenessProbe` 失败 → 重启。
- `startupProbe` 用于启动慢的应用，避免 liveness 误杀。
- `preStop` + `terminationGracePeriodSeconds` 配合应用优雅关闭。
- `PodDisruptionBudget` 保证节点维护时仍有可用副本。
- HPA 基于 CPU/自定义指标；注意数据库连接数随副本线性增长。
- 不要在容器内写本地磁盘做持久化。

**探针实现**

```dart
// /healthz：进程存活，不检查外部依赖（避免依赖抖动导致全部重启）
Response healthz() => Response.ok('{"status":"ok"}');

// /readyz：是否可接流量（可检查迁移版本、连接池可用性）
Future<Response> readyz(Db db) async {
  try {
    await db.query('SELECT 1').timeout(const Duration(seconds: 1));
    return Response.ok('{"status":"ready"}');
  } catch (_) {
    return Response(503, body: '{"status":"not_ready"}');
  }
}
```

### 16.5 CI/CD 流水线

```yaml
name: ci
on:
  pull_request:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: app
          POSTGRES_PASSWORD: app
          POSTGRES_DB: app_test
        ports: ['5432:5432']
        options: >-
          --health-cmd "pg_isready -U app"
          --health-interval 5s
          --health-retries 10
    steps:
      - uses: actions/checkout@v4
      - uses: dart-lang/setup-dart@v1
        with:
          sdk: stable
      - run: dart pub get --enforce-lockfile
      - run: dart format --output=none --set-exit-if-changed .
      - run: dart analyze --fatal-infos
      - run: dart test --exclude-tags=integration
      - run: dart test --tags=integration
        env:
          TEST_DATABASE_URL: postgresql://app:app@localhost:5432/app_test
      - run: dart compile exe bin/server.dart -o build/server

  build-and-push:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: docker build -t $IMAGE:$GITHUB_SHA .
      - run: docker push $IMAGE:$GITHUB_SHA
```

**发布流程**

```text
1. 合并到 main → 构建不可变镜像（tag = commit sha）
2. 部署到 staging → 跑迁移 → smoke test
3. 手动/自动批准 → 部署到 prod（滚动或金丝雀）
4. 观察 15 分钟：错误率、P99、关键业务指标
5. 正常 → 完成；异常 → 回滚镜像（数据库需兼容）
```

### 16.6 发布策略

| 策略 | 原理 | 优点 | 缺点 |
|---|---|---|---|
| 滚动更新 | 逐个替换实例 | 简单、资源省 | 新旧版本共存期需兼容 |
| 蓝绿 | 两套环境切换 | 回滚快 | 资源翻倍 |
| 金丝雀 | 小比例流量先验证 | 风险可控 | 需要流量路由与指标对比 |
| 特性开关 | 代码内置开关 | 无需重新部署即可关闭 | 开关债务、测试组合爆炸 |

**特性开关纪律**

- 开关要有 owner、过期时间、清理计划。
- 关键业务路径不要嵌套过多开关。
- 开关状态要能被观测（日志/指标带上开关分组）。

### 16.7 数据库迁移的上线顺序

```text
错误顺序：先发布依赖新列的代码 → 再迁移 → 报错
正确顺序：
  1. Expand 迁移（加列/表，向后兼容）
  2. 发布代码 v2（兼容新旧 schema，必要时双写）
  3. 数据回填（后台任务，限速）
  4. 校验
  5. 切换读取
  6. Contract 迁移（删旧结构）
```

**回滚窗口**：回滚镜像时数据库必须仍兼容旧代码。因此删除列/收紧约束要等到确认不再回滚。

### 16.8 配置与密钥

| 内容 | 存放 |
|---|---|
| 非敏感配置 | ConfigMap / 环境变量 / 配置中心 |
| 密钥 | Secrets Manager / KMS / Vault |
| 证书 | 托管证书服务 + 自动续期 |
| 本地开发 | `.env`（不入库）+ 示例 `.env.example` |

**注入方式**

- 环境变量：简单，但可能出现在崩溃转储/子进程环境中。
- 文件挂载：更适合大块证书/密钥文件。
- 运行时拉取：启动时从 KMS 获取，短缓存。

**轮换**：数据库密码、API Key、JWT 签名密钥都要有轮换流程（双密钥期 + 灰度）。

### 16.9 成本优化

- 右调资源 requests/limits（避免请求 4 核实际用 0.2 核）。
- 使用 ARM 实例（通常更便宜）。
- 日志与指标采样、保留期分层。
- 对象存储生命周期策略（转冷归档）。
- 数据库：按需 vs 预留、只读副本按实际读流量。
- 出口流量：CDN、同区域调用、压缩。

---

## 19. 后端开发协作与日常工程实践

### 17.1 从需求到设计

**需求澄清清单**

1. 谁是调用方（前端/移动/第三方/内部服务）？
2. 核心实体与状态流转是什么？
3. 权限模型：谁能做什么？跨租户规则？
4. 数据量与增长率？峰值 QPS？
5. 一致性要求？能否接受延迟？
6. 失败时用户看到什么？能否重试？
7. 合规与数据保留要求？
8. 上线时间与可接受的复杂度？

**设计产出物**

- 一页设计文档（问题、方案、API、数据模型、风险）。
- 关键流程图/时序图。
- 迁移与回滚计划。
- 监控与告警项。
- 测试策略。

### 17.2 API 评审

评审会（30 分钟）检查：

- 资源命名与方法语义。
- 认证/授权规则是否完整。
- 幂等、分页、错误码。
- 兼容性与版本策略。
- 性能预期与上限。
- 文档与示例。

**产出**：评审记录 + 修改项 + 负责人。

### 17.3 代码评审清单

**正确性**

- [ ] 边界条件（空、0、负数、超长、并发）是否处理？
- [ ] 事务边界是否正确？异常时是否回滚？
- [ ] 幂等性是否满足？
- [ ] 授权是否在服务端执行？
- [ ] 是否有 N+1、无界查询、无超时调用？

**可维护性**

- [ ] 命名是否表达意图？
- [ ] 是否有重复逻辑可抽取？
- [ ] 错误处理是否区分业务/技术失败？
- [ ] 是否引入了不必要的依赖？
- [ ] 测试是否覆盖新增行为与失败路径？

**安全**

- [ ] 输入校验完整？
- [ ] 日志是否泄露敏感信息？
- [ ] 是否使用了参数化查询？
- [ ] 新端点是否有授权测试？

**运维**

- [ ] 新增配置是否有默认值与文档？
- [ ] 新增迁移是否可回滚/前向修复？
- [ ] 是否需要新指标/告警？
- [ ] 发布顺序是否说明？

### 17.4 Git 与提交规范

```text
<type>(<scope>): <subject>

feat(orders): 支持幂等创建订单
fix(auth): 修正 JWT audience 校验
refactor(db): 抽取连接池创建
docs(api): 补充错误码说明
test(orders): 增加并发扣减库存用例
chore(deps): 升级 postgres 到 3.5.0
```

- 一个提交做一件事，便于回滚与 review。
- PR 描述包含：背景、变更、测试证据、风险、回滚方式。
- 迁移文件与应用代码分开提交（便于按序发布）。
- 不提交 `.env`、密钥、生成物（按团队策略）。

### 17.5 文档体系

| 文档 | 内容 | 更新时机 |
|---|---|---|
| README | 本地启动、测试、部署、目录说明 | 每次流程变化 |
| OpenAPI | API 契约 | 每次 API 变更 |
| ADR | 架构决策 | 每次重大决策 |
| Runbook | 故障处理步骤、常见告警 | 每次事故后 |
| CHANGELOG | 对外变更 | 每次发布 |
| 数据字典 | 表/字段含义 | 每次迁移 |

**Runbook 示例结构**

```markdown
# 告警：order_api_error_rate_high

## 影响
下单接口错误率 > 1%，用户无法下单。

## 快速判断
1. 看 dashboard：错误是否集中在某个依赖（DB/支付/库存）。
2. 看最近发布：15 分钟内是否有部署。
3. 看日志：`error` 级别按 `code` 聚合。

## 处理
- 若是最近发布导致 → 回滚到上一个镜像。
- 若是依赖故障 → 开启降级开关 `orders.degrade_payment=false`。
- 若是数据库连接耗尽 → 扩容副本 + 检查慢查询。

## 升级
15 分钟未恢复 → 通知 on-call 负责人 + 数据库 DBA。

## 事后
24 小时内提交事故报告。
```

### 17.6 事故响应与复盘

**响应流程**

```text
发现 → 确认影响面 → 止血（回滚/降级/扩容）→ 通知 → 修复 → 验证 → 复盘
```

**原则**

- 止血优先于找根因。
- 指定一个 incident commander，避免多头指挥。
- 记录时间线（谁在何时做了什么）。
- 对外沟通有统一口径与频率。

**复盘模板（无指责）**

```markdown
# 事故：2026-11-20 下单接口 30 分钟不可用

## 影响
- 时长：14:02–14:32（30 分钟）
- 影响用户：约 1,200 次下单失败
- 数据影响：无重复扣款（幂等生效）

## 时间线
14:02 错误率上升
14:05 告警触发
14:08 值班确认
14:12 回滚到 v1.3.2
14:32 错误率恢复

## 根因
迁移新增 NOT NULL 列未设默认值，旧代码 INSERT 失败。

## 促成因素
- 迁移在发布后执行（顺序错误）
- 预发布数据量小，未触发
- 缺少“迁移兼容性”检查项

## 改进项（有 owner 与截止日期）
1. 迁移清单加入“向后兼容”检查（@alice，11-25）
2. CI 增加旧版本代码 + 新 schema 的集成测试（@bob，12-05）
3. 发布流程强制“先迁移后发布”（@carol，11-23）
```

### 17.7 依赖与升级策略

- 每月固定窗口检查 `dart pub outdated`。
- 安全补丁优先，功能升级按季度。
- 大版本升级单独分支，做完整回归。
- 维护“禁止使用”清单（已知不安全/无人维护的包）。
- 记录升级决策（为何跳过某版本）。

---

## 20. 学习路线、实战与面试

### 18.1 能力矩阵

| 能力 | 初级（能做事） | 中级（能负责） | 高级（能定方向） |
|---|---|---|---|
| Dart 语言 | 语法、空安全、集合 | sealed/模式匹配、泛型、代码生成 | 语言设计取舍、性能特征 |
| 异步并发 | async/await、Future | Stream、isolate、并发控制 | 事件循环调优、背压架构 |
| HTTP/API | 写 REST 端点 | 契约设计、版本、幂等 | API 治理、多协议架构 |
| 数据库 | CRUD、简单 SQL | 索引、事务、迁移、连接池 | 分区、分片、容量规划 |
| 安全 | 会用认证库 | 威胁建模、授权模型 | 安全体系、合规落地 |
| 架构 | 分层 | 模块边界、DDD 战术 | 服务拆分、一致性策略 |
| 运维 | 看日志 | 指标/追踪、告警 | SLO 体系、故障演练 |
| 协作 | 完成任务 | 评审、文档 | 技术规划、跨团队推动 |

### 18.2 12 周学习路线

**第 1–2 周：语言与工具**

- 重读 Dart 语言导览（类型、空安全、异步、isolate）。
- 用 `dart create` 建项目，配置 lint、测试、CI。
- 目标：写出带测试的 `Money`、`Order` 领域模型。

**第 3–4 周：HTTP 与框架**

- 用 `dart:io` 写一个 echo 服务器，理解请求生命周期。
- 用 Shelf 实现路由、中间件、错误映射、请求 ID。
- 目标：一个可运行、有日志的 CRUD API（内存存储）。

**第 5–6 周：数据库**

- PostgreSQL 安装与 SQL 练习（JOIN、聚合、窗口函数）。
- 连接池、事务、参数化查询、迁移。
- 目标：把 CRUD 换成真实数据库，写集成测试。

**第 7 周：认证与授权**

- 实现会话或 JWT 认证；资源级授权。
- 写越权测试（跨用户、跨租户）。
- 目标：所有端点都有授权测试。

**第 8 周：可靠性**

- 超时、重试、熔断、限流、优雅关闭。
- 结构化日志 + 指标 + 追踪。
- 目标：模拟依赖故障，服务不雪崩。

**第 9 周：异步与消息**

- outbox、幂等消费者、死信队列。
- 定时任务与分布式锁。
- 目标：下单后异步发通知，失败可重试。

**第 10 周：部署**

- Dockerfile、CI/CD、迁移流水线。
- 部署到云（容器或 VM），配置健康检查。
- 目标：一键发布 + 回滚演练。

**第 11 周：性能与容量**

- 压测、慢查询优化、连接池调优。
- 建立 SLO 与告警。
- 目标：给出容量报告与瓶颈分析。

**第 12 周：综合项目**

- 完成下方“订单与库存系统”，写设计文档与复盘。
- 目标：能向他人讲清每个技术决策的原因。

### 18.3 综合实战项目：订单与库存系统

**需求**

1. 用户注册/登录，支持角色（customer/support/admin）。
2. 商品目录：查询、分页、搜索。
3. 库存：查询、预占、释放。
4. 下单：幂等、库存校验、金额计算（服务端）。
5. 支付模拟：成功/失败/超时，回调幂等。
6. 订单状态机：pending → paid → shipped → completed / cancelled。
7. 通知：下单/支付成功异步发邮件（可重试）。
8. 审计：关键操作记录谁在何时做了什么。

**非功能要求**

- P95 < 200ms（下单除外，< 500ms）。
- 支持 100 RPS，水平扩展。
- 依赖故障时降级不雪崩。
- 关键路径有集成测试与并发测试。
- 有日志、指标、追踪、告警。

**验收清单**

- [ ] 并发 50 个请求抢 10 件库存，恰好 10 单成功。
- [ ] 相同幂等键重复提交只创建一单。
- [ ] 支付回调重复投递不重复扣款。
- [ ] 跨用户访问订单返回 404。
- [ ] 数据库断开时返回 503 且恢复后自动可用。
- [ ] 滚动发布期间无 5xx。
- [ ] 迁移可在生产快照上执行并计时。
- [ ] 能从 traceId 追出一次下单的完整链路。

### 18.4 面试题（后端通用 + Dart 特性）

**Dart 语言**

1. `final` 与 `const` 的区别？`const` 构造的要求是什么？
2. 空安全如何工作？什么情况下类型提升会失效？
3. `sealed class` 相比普通抽象类解决了什么问题？
4. `Future` 与 `Stream` 的区别？各自适用场景？
5. isolate 与线程的区别？为什么 Dart 不需要锁？
6. 为什么 `await` 不会创建新线程？CPU 密集任务怎么办？
7. `unawaited` 的作用？忘记 `await` 会发生什么？
8. AOT 与 JIT 的差异？为什么 AOT 下反射受限？
9. 为什么金额不能用 `double`？如何建模？
10. `List.unmodifiable` 能保证深度不可变吗？

**HTTP 与 API**

11. `PUT` 与 `PATCH` 的语义差异？
12. 401 与 403 的区别？何时用 404 隐藏资源？
13. 幂等键如何设计？并发同键如何处理？
14. offset 分页与 cursor 分页的取舍？
15. 如何设计 API 版本与弃用流程？
16. `Cache-Control: no-store` 与 `private` 的区别？
17. CORS 预检请求为什么不能要求认证？
18. 如何防止 `X-Forwarded-For` 伪造？

**数据库**

19. 复合索引的最左前缀原则？什么查询用不上索引？
20. 乐观锁与悲观锁的适用场景？
21. 什么是幻读？PostgreSQL 的 RR 级别如何处理？
22. 连接池设多大？为什么不是越大越好？
23. 如何做零停机迁移？
24. N+1 如何发现与解决？
25. 什么时候用 NoSQL？举一个不该用的例子。

**架构与可靠性**

26. 超时、重试、熔断如何配合？重试为什么危险？
27. at-least-once 下消费者如何幂等？
28. outbox 模式解决什么问题？
29. Saga 与两阶段提交的差异？
30. 单体何时该拆微服务？代价是什么？
31. 如何设计一个不丢任务的后台队列？
32. SLO、SLI、error budget 的关系？

**安全**

33. JWT 为什么不能即时撤销？如何缓解？
34. OIDC 授权码 + PKCE 的流程与防护点？
35. 如何防 IDOR？为什么不能只校验 ID 格式？
36. 密码哈希为什么用 Argon2id/bcrypt？
37. SSRF 如何防御？为什么只校验域名不够？
38. 如何安全地实现文件上传？

**场景设计题**

39. 设计一个短链服务（存储、缓存、冲突、统计）。
40. 设计一个秒杀系统（限流、库存一致性、防超卖、异步下单）。
41. 设计一个多租户 SaaS 的权限与数据隔离方案。
42. 线上订单接口 P99 从 200ms 涨到 2s，如何排查？
43. 数据库主从延迟导致用户下单后查不到订单，如何解决？
44. 如何设计可重放、可观测的 Webhook 系统？

---

## 21. 上线前检查清单

### 19.1 API 与业务

- [ ] 每个端点都有明确的认证与授权规则
- [ ] 资源级权限有测试（跨用户/跨租户）
- [ ] 写操作有幂等策略（尤其创建、支付、回调）
- [ ] 状态机转换有穷尽性检查与测试
- [ ] 分页、过滤、排序有上限与白名单
- [ ] 请求体、上传、数组长度、嵌套深度有上限
- [ ] 错误码稳定、文档化、不泄露内部信息
- [ ] 金额、时间、枚举、可空语义明确
- [ ] 契约（OpenAPI）与实现一致

### 19.2 数据

- [ ] 参数化查询覆盖所有用户输入
- [ ] 关键约束由数据库保证（唯一、外键、检查）
- [ ] 事务边界正确，无长事务、无事务内外部调用
- [ ] 并发写有正确性验证（条件更新/锁/版本）
- [ ] 索引覆盖主要查询，无未使用索引堆积
- [ ] 迁移可在生产快照执行，向后兼容，有回滚/前向修复
- [ ] 连接池大小经过计算，含所有副本总和
- [ ] 备份存在且**恢复演练**通过
- [ ] 数据保留与删除策略明确

### 19.3 安全与隐私

- [ ] 无密钥/令牌/密码进入仓库、镜像、日志
- [ ] HTTPS、HSTS、安全响应头
- [ ] 依赖漏洞扫描通过，无已知高危
- [ ] 日志脱敏（邮箱、手机、身份证、令牌、请求体敏感字段）
- [ ] 管理端点/调试端点不对外
- [ ] 限流、防暴力破解、账户锁定策略
- [ ] 文件上传校验与隔离
- [ ] 敏感字段加密与密钥轮换方案

### 19.4 可靠性与可观测性

- [ ] 所有外部调用有超时
- [ ] 重试有上限、退避、抖动，且仅用于幂等操作
- [ ] 关键依赖有熔断/降级
- [ ] 优雅关闭（SIGTERM → 停止接流量 → 排空 → 关闭依赖）
- [ ] 健康检查区分 liveness/readiness
- [ ] 结构化日志含 requestId/traceId
- [ ] 核心指标（RED/USE）与告警已配置
- [ ] 追踪可串起一次请求的完整链路
- [ ] 容量与瓶颈已评估，有扩容方案

### 19.5 交付

- [ ] CI 覆盖格式、分析、单元、集成、构建
- [ ] 生成代码无漂移
- [ ] 镜像非 root、固定 digest、无密钥
- [ ] 配置与密钥通过安全渠道注入
- [ ] 发布顺序（迁移 → 部署）明确
- [ ] 回滚步骤经过演练
- [ ] Runbook 已更新
- [ ] 发布后验证项（业务指标）明确

---

## 22. 常见误区

1. **把异步当并行**：`await` 不创建线程；CPU 密集任务会阻塞事件循环。
2. **以为 Dart 只能配 Flutter**：Dart 可以独立作为服务端语言，但需评估生态。
3. **在 handler 里写业务**：导致事务、授权、校验无法复用与测试。
4. **信任客户端**：价格、用户 ID、租户 ID、角色都必须服务端决定。
5. **登录成功即授权成功**：资源级权限必须单独校验。
6. **用 `!` 掩盖空值**：把编译期问题推迟成运行期 500。
7. **金额用 `double`**：浮点误差会造成对账灾难。
8. **时间不带时区**：跨时区、夏令时、排序都会出错。
9. **无超时调用**：一个慢依赖可以拖垮整个服务。
10. **无限重试**：放大故障，制造重试风暴。
11. **重试非幂等写**：重复下单、重复扣款。
12. **把内存队列当可靠队列**：重启即丢。
13. **多实例下用本地锁/本地缓存做全局一致**：假设不成立。
14. **先查再改**：并发下产生竞态；用条件更新或约束。
15. **ORM 代替 SQL 知识**：索引、执行计划、锁仍要懂。
16. **迁移与应用同时改且不兼容**：滚动发布期间必然报错。
17. **日志写敏感信息**：一次泄露就是合规事故。
18. **只有 happy path 测试**：生产故障多来自失败路径。
19. **覆盖率当质量**：执行过 ≠ 断言有效。
20. **过早微服务化**：分布式复杂度远超收益。
21. **把缓存当数据库**：缓存会失效、被淘汰、不可用。
22. **忽略优雅关闭**：每次发布都丢请求。
23. **只监控 CPU 和进程存活**：用户感知的是错误率与延迟。
24. **告警没有 runbook**：值班人员只能猜。
25. **依赖不锁版本**：构建不可复现，故障无法定位。
26. **把安全扫描当安全审计**：逻辑漏洞扫描不出来。
27. **不做容量规划**：增长到来时只能救火。
28. **文档缺失**：新人无法启动，故障无法交接。

---

## 23. 官方资料与延伸阅读

**Dart 官方**

- [Dart 服务端开发](https://dart.dev/server)：服务端生态与示例入口。
- [并发与 isolate](https://dart.dev/language/concurrency)：事件循环、Future、Stream、isolate。
- [异步编程](https://dart.dev/language/async)：`async`/`await`、Stream。
- [语言导览](https://dart.dev/language)：类型系统、空安全、模式匹配。
- [Packages 与 Pub](https://dart.dev/tools/pub/packages)：依赖解析、锁文件。
- [测试](https://dart.dev/tools/testing)：`package:test` 使用指南。
- [AOT 编译](https://dart.dev/tools/dart-compile)：`dart compile exe`。

**框架与库**

- [Shelf](https://pub.dev/packages/shelf)：Handler/Middleware 抽象。
- [shelf_router](https://pub.dev/packages/shelf_router)：路由。
- [Dart Frog](https://dart-frog.dev/)：约定式后端框架。
- [Serverpod](https://docs.serverpod.dev/)：Dart 全栈框架。
- [postgres](https://pub.dev/packages/postgres)：PostgreSQL 驱动。
- [redis](https://pub.dev/packages/redis)：Redis 客户端。

**规范与安全**

- [OpenAPI Specification](https://spec.openapis.org/oas/latest.html)
- [RFC 9457 Problem Details](https://www.rfc-editor.org/rfc/rfc9457)
- [OWASP API Security Top 10](https://owasp.org/www-project-api-security/)
- [OWASP ASVS](https://owasp.org/www-project-application-security-verification-standard/)
- [W3C Trace Context](https://www.w3.org/TR/trace-context/)

**数据库与运维**

- [PostgreSQL 文档](https://www.postgresql.org/docs/)
- [Use The Index, Luke](https://use-the-index-luke.com/)：SQL 索引与性能。
- [Google SRE Book](https://sre.google/sre-book/table-of-contents/)：SLO、告警、事故管理。
- [12-Factor App](https://12factor.net/zh_cn/)

> **选型提醒**：官方文档、包版本与平台支持会变化。新项目启动前请重新核对：Dart SDK 兼容性、依赖维护状态、许可证、安全公告、部署平台的运行时支持，以及团队的实际运维能力。本文中的数值（超时、池大小、比例）是起点而非标准答案，必须以压测与真实观测校准。
