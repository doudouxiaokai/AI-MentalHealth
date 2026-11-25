# AI-MentalHealth · 心理健康 AI 咨询平台

面向大学生的 AI 心理疏导应用，前后端分离：Spring Boot 提供 JWT 认证与 AI 流式对话，Vue 3 提供用户咨询端与管理后台。

> 「宁渡 AI 助手」——温暖、有同理心的对话伙伴，支持逐字流式回复、会话历史回看与情绪状态跟踪。

## 功能

**用户端**

- AI 心理疏导对话：SSE 流式逐字输出，可随时中止
- 会话管理：新建 / 切换 / 删除，历史消息回看
- 情绪花园：对话结束后展示情绪倾向、风险等级与疏导建议
- 情绪日记：选择情绪、记录触发事件
- 心理知识库：文章列表与详情

**管理端**

- 数据看板：情绪趋势、咨询量、用户活跃度（ECharts）
- 心理知识文章管理：富文本编辑（wangEditor）
- 咨询记录与情绪日志查阅

## 技术栈

Java · Spring Boot · Spring AI · Spring Security · JWT · MyBatis-Plus · MySQL · SSE 流式输出 · Vue · Pinia · Element Plus · ECharts · wangEditor

## 目录结构

```
AI-MentalHealth/
├── ai-spingboot/                  # 后端（Spring Boot）
│   ├── src/main/java/org/example/aispingboot/
│   │   ├── AiService/             # AI 对话核心：ChatClient 调用、系统提示词、流式输出结构
│   │   ├── config/                # ChatClient / Security / JWT 配置
│   │   ├── controller/            # 用户、心理对话接口
│   │   ├── service/               # 业务层（会话、消息、用户）+ DTO 转换
│   │   ├── mapper/                # MyBatis-Plus Mapper
│   │   ├── entity/                # User / ConsultationSession / ConsultationMessage
│   │   ├── DTO/                   # command（入参）/ response（出参）
│   │   ├── util/                  # JWT 工具与认证过滤器
│   │   └── common/                # 统一响应 Result、错误码、全局异常处理
│   └── src/main/resources/
│       └── application.yml        # 数据库、AI 模型、JWT 配置
└── ai-vue/                        # 前端（Vue 3 + Vite）
    ├── src/
    │   ├── api/                   # 接口封装（frontend.js 用户端 / admin.js 管理端）
    │   ├── components/            # 布局、富文本、Markdown 渲染、侧边栏等
    │   ├── views/                 # 页面：咨询、情绪日记、知识库、看板等
    │   ├── router/                # 路由与登录守卫（按 userType 分流）
    │   ├── stores/                # Pinia
    │   └── utils/request.js       # Axios 实例与拦截器
    └── vite.config.js             # 开发代理
```

## 快速开始

### 1. 环境要求

- JDK 17+
- Maven 3.8+（或直接用仓库内 `mvnw`）
- Node.js 18+
- MySQL 8+

### 2. 初始化数据库

```sql
CREATE DATABASE mental_health_assistant
  DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

USE mental_health_assistant;

CREATE TABLE `user` (
  id          BIGINT PRIMARY KEY AUTO_INCREMENT,
  username    VARCHAR(50)  NOT NULL UNIQUE,
  email       VARCHAR(100) UNIQUE,
  phone       VARCHAR(20),
  password    VARCHAR(100) NOT NULL,
  nickname    VARCHAR(50),
  avatar      VARCHAR(255),
  gender      TINYINT,
  birthday    DATE,
  user_type   TINYINT DEFAULT 1 COMMENT '1=用户端 2=管理端',
  status      TINYINT DEFAULT 1 COMMENT '1=正常 0=禁用',
  created_at  DATETIME DEFAULT CURRENT_TIMESTAMP,
  updated_at  DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

CREATE TABLE consultation_session (
  id                       BIGINT PRIMARY KEY AUTO_INCREMENT,
  user_id                  BIGINT NOT NULL,
  session_title            VARCHAR(100),
  started_at               DATETIME,
  last_emotion_analysis    TEXT,
  last_emotion_updated_at  DATETIME,
  KEY idx_user (user_id)
);

CREATE TABLE consultation_message (
  id           BIGINT PRIMARY KEY AUTO_INCREMENT,
  session_id   BIGINT NOT NULL,
  sender_type  TINYINT COMMENT '1=用户 2=AI',
  message_type TINYINT,
  content      TEXT,
  emotion_tag  VARCHAR(50),
  ai_model     VARCHAR(50),
  created_at   DATETIME DEFAULT CURRENT_TIMESTAMP,
  KEY idx_session (session_id)
);
```

### 3. 启动后端

先修改 `ai-spingboot/src/main/resources/application.yml`：

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/mental_health_assistant?useSSL=false&serverTimezone=UTC
    username: root
    password: 你的数据库密码
  ai:
    openai:
      api-key: 你的硅基流动 API Key   # https://cloud.siliconflow.cn 申请
      base-url: https://api.siliconflow.cn
```

然后启动：

```bash
cd ai-spingboot
./mvnw spring-boot:run        # Windows 用 mvnw.cmd spring-boot:run
```

服务默认运行在 **http://localhost:1236**。

### 4. 启动前端

```bash
cd ai-vue
npm install
npm run dev
```

访问终端提示的地址（默认 http://localhost:5173）。

> ⚠️ **注意**：`vite.config.js` 中的开发代理当前指向 `http://159.75.169.224:1235`，与本地后端端口 `1236` 不一致。本地联调前请改为：
>
> ```js
> proxy: { '/api': { target: 'http://localhost:1236', changeOrigin: true } }
> ```

## 核心接口

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| POST | `/api/user/add` | 注册（公开） |
| POST | `/api/user/login` | 登录，返回 JWT |
| GET | `/api/user/current` | 当前用户信息 |
| POST | `/api/psychological-chat/session/start` | 创建会话 |
| POST | `/api/psychological-chat/stream` | 流式对话，返回 SSE |

**SSE 事件格式**

```
event: message
data: {"code":"200","msg":"success","data":{"content":"你好","type":"normal"}}

event: done
data: {}
```

前端使用 `@microsoft/fetch-event-source` 而非原生 `EventSource`，因为需要 `POST` 方法并携带自定义 `token` 头。

## 实现要点

- **会话与记忆解耦**：数据库 `sessionId`（`session_1`）与模型 `conversationId`（`conversation_session_1`）分离映射，业务会话可持久化回看，模型记忆独立维护。
- **对话记忆**：`MessageWindowChatMemory` 保留最近 30 条消息，通过 `MessageChatMemoryAdvisor` 注入到每次请求。
- **流式落库**：`Flux` 逐块下发的同时累积完整回复，在 `doOnComplete` 中一次性写入 `consultation_message`，避免断流产生残缺记录。
- **首条消息幂等**：新建会话时初始消息已入库，流式接口通过「消息条数 + 发送方 + 内容比对」判定，避免重复保存。
- **认证链路**：`OncePerRequestFilter` 解析 JWT 的 `userId` / `roleType`，校验签名与过期后回查用户状态，再注入 `SecurityContext`；密码使用 BCrypt 加盐存储。

## 已知问题 / TODO

- 知识库、情绪日记、数据看板的前端页面与 API 封装已完成，对应后端接口（`/knowledge/*`、`/emotion-diary/*`、`/data-analytics/*`）尚未包含在本仓库。
- `application.yml` 中的密钥为明文配置，生产环境建议改为环境变量注入。
- 未接入 Redis，JWT 无主动失效机制（登出仅前端清除本地凭证）。

## 项目文档

| 文档 | 用途 |
| --- | --- |
| [技术文档 · 完整实现](docs/01-技术文档-完整实现.md) | 逐类逐方法的实现细节、调用链、接口契约、配置与已知缺陷 |
| [项目理解指南](docs/02-项目理解指南.md) | 用大白话讲清项目构成与设计取舍，面试前快速建立整体认知 |
| [面试问答手册](docs/03-面试问答手册.md) | 70 道常见提问的参考答案与追问方向，按难度分档 |

## License

MIT
