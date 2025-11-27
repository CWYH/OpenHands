# OpenHands 项目架构总结

> **⚠️ Mermaid 图表渲染说明**:
>
> 如果您的预览器显示的是 Mermaid 代码而不是图表，请按以下方式解决：
>
> **VS Code 用户**:
> 1. 安装扩展：`Markdown Preview Mermaid Support` (作者: Matt Bierner)
> 2. 或安装：`Markdown Preview Enhanced` (功能更强大)
> 3. 按 `Ctrl+Shift+V` (Windows/Linux) 或 `Cmd+Shift+V` (Mac) 打开预览
>
> **其他编辑器**:
> - **Typora**: 原生支持，直接预览即可
> - **Obsidian**: 需要安装 "Mermaid" 插件
> - **GitHub/GitLab**: 在网页上查看，原生支持 Mermaid
> - **在线查看**: 复制 Mermaid 代码到 [Mermaid Live Editor](https://mermaid.live/) 查看
>
> **验证方法**: 如果看到的是渲染后的图表（而不是代码块），说明配置成功 ✅

## 1. 整体架构概述

OpenHands 项目采用现代化的前后端分离架构。它主要由一个基于 React 的单页应用前端和一个作为 AI 代理编排核心的模块化 Python 后端组成。

两者之间的通信主要通过 REST API 和 WebSocket 进行，形成一个标准的客户端-服务器（Client-Server）模型。该架构具有良好的解耦性，使得前端和后端可以独立开发和部署。

## 2. 主要组件

### a. 前端 (`frontend`)

- **职责**: 提供用户界面（UI），允许用户与 AI 代理进行交互、发起任务和可视化结果。
- **技术栈**:
  - **框架**: React
  - **构建工具**: Vite
  - **状态管理**: Zustand, Tanstack Query
  - **UI**: Tailwind CSS
- **通信**: 前端通过 HTTP 请求与后端 API (`/api/v1`) 进行通信。`frontend/src/api/open-hands-axios.ts` 文件是配置 API 请求客户端的核心。

### b. 后端 (`openhands`)

- **职责**: 作为项目的核心业务逻辑引擎。它负责处理来自前端的请求，通过 AI 模型（LLM）进行决策和内容生成，管理数据库，并在沙箱环境中执行代码。
- **技术栈**:
  - **框架**: FastAPI
  - **ORM**: SQLAlchemy
  - **LLM 集成**: `litellm` (用于连接不同的语言模型)
  - **沙箱环境**: Docker
- **结构**: 后端代码高度模块化，通过 `openhands/app_server/v1_router.py` 文件聚合了用户、会话、沙箱等不同功能的 API 路由，结构清晰。

### c. 企业版 (`enterprise`)

- **职责**: 根据目录结构和命名推断，此部分可能包含为 SaaS 或企业级部署设计的扩展功能，例如更高级的认证、多租户支持或特定的集成，它构建于 `openhands` 核心功能之上。

## 3. 技术栈总结

| 层次 | 技术 | 用途 |
| --- | --- | --- |
| **前端** | React, Vite | 构建用户界面 |
| | Zustand | 状态管理 |
| | Tailwind CSS | 样式与布局 |
| **后端** | Python, FastAPI | API 服务开发 |
| | SQLAlchemy | 数据库交互与 ORM |
| | `litellm` | 与多种 LLM 服务集成 |
| | Docker | 提供安全的代码执行沙箱 |

## 4. 通信模式

- **主要模式**: 客户端-服务器模式。
- **协议**:
  - **REST API**: 前端通过 `axios` 客户端向后端 FastAPI 服务发起 HTTP 请求，用于大多数数据交互。
  - **WebSocket**: 可能用于需要实时通信的功能，例如实时日志流或状态更新。

## 5. 系统架构图

### 5.1 整体架构图

```mermaid
graph TB
    subgraph "前端层 Frontend Layer"
        UI[React UI<br/>React + Vite + Tailwind]
        State[状态管理<br/>Zustand + TanStack Query]
    end

    subgraph "服务器层 Server Layer"
        API[FastAPI Server<br/>REST API + WebSocket]
        CM[ConversationManager<br/>会话管理]
        Session[Session<br/>会话实例]
    end

    subgraph "核心控制层 Core Control Layer"
        AC[AgentController<br/>代理控制器]
        ES[EventStream<br/>事件流总线]
        ST[State<br/>状态管理]
    end

    subgraph "代理层 Agent Layer"
        Agent[Agent<br/>代理基类]
        CodeAct[CodeActAgent<br/>代码执行代理]
        Browse[BrowsingAgent<br/>浏览代理]
        ReadOnly[ReadonlyAgent<br/>只读代理]
    end

    subgraph "运行时层 Runtime Layer"
        Runtime[Runtime<br/>运行时接口]
        Docker[Docker Runtime<br/>Docker 容器]
        Local[Local Runtime<br/>本地执行]
        K8s[Kubernetes Runtime<br/>K8s 集群]
        Sandbox[Sandbox<br/>沙箱环境]
    end

    subgraph "LLM 层 LLM Layer"
        LLM[LLM Interface<br/>LLM 接口]
        LiteLLM[LiteLLM<br/>统一 LLM 接口]
        OpenAI[OpenAI]
        Claude[Anthropic Claude]
        Azure[Azure AI]
    end

    subgraph "存储层 Storage Layer"
        FileStore[FileStore<br/>文件存储]
        LocalFS[本地文件系统]
        S3[Amazon S3]
        GCS[Google Cloud Storage]
        Memory[Memory<br/>对话记忆]
    end

    UI -->|HTTP/WebSocket| API
    State --> UI
    API --> CM
    CM --> Session
    Session --> AC
    Session --> ES
    Session --> Runtime

    AC --> ES
    AC --> ST
    AC --> Agent
    ES -->|发布/订阅| AC
    ES -->|发布/订阅| Runtime
    ES -->|发布/订阅| Memory

    Agent --> CodeAct
    Agent --> Browse
    Agent --> ReadOnly
    Agent --> LLM
    Agent --> ST

    AC -->|执行 Action| Runtime
    Runtime -->|返回 Observation| ES
    Runtime --> Docker
    Runtime --> Local
    Runtime --> K8s
    Runtime --> Sandbox

    LLM --> LiteLLM
    LiteLLM --> OpenAI
    LiteLLM --> Claude
    LiteLLM --> Azure

    FileStore --> LocalFS
    FileStore --> S3
    FileStore --> GCS
    ES -->|持久化| FileStore
    Memory --> FileStore
```

### 5.2 数据流架构图

```mermaid
sequenceDiagram
    participant User as 用户
    participant Frontend as 前端 UI
    participant Server as FastAPI Server
    participant CM as ConversationManager
    participant AC as AgentController
    participant ES as EventStream
    participant Agent as Agent
    participant LLM as LLM Service
    participant Runtime as Runtime
    participant Sandbox as Sandbox

    User->>Frontend: 输入任务
    Frontend->>Server: POST /api/v1/conversations
    Server->>CM: 创建会话
    CM->>AC: 初始化 AgentController
    AC->>ES: 订阅事件流

    Frontend->>Server: WebSocket 连接
    Server->>ES: 添加用户消息事件
    ES->>AC: 通知新事件
    AC->>Agent: step(State)
    Agent->>LLM: 生成提示词并调用
    LLM-->>Agent: 返回响应
    Agent->>Agent: 解析响应为 Action
    Agent-->>AC: 返回 Action
    AC->>ES: 发布 Action 事件
    ES->>Runtime: 通知执行 Action
    Runtime->>Sandbox: 执行命令/操作
    Sandbox-->>Runtime: 返回结果
    Runtime->>ES: 发布 Observation 事件
    ES->>AC: 通知新观察结果
    AC->>AC: 更新 State
    ES->>Server: 事件更新
    Server->>Frontend: WebSocket 推送
    Frontend->>User: 显示结果
```

### 5.3 组件交互图

```mermaid
graph LR
    subgraph "事件驱动架构 Event-Driven Architecture"
        ES[EventStream<br/>事件流总线]
    end

    subgraph "发布者 Publishers"
        User[用户输入]
        Agent[Agent 生成 Action]
        Runtime[Runtime 执行结果]
        Memory[Memory 检索]
    end

    subgraph "订阅者 Subscribers"
        AC[AgentController]
        RT[Runtime]
        MEM[Memory]
        SRV[Server]
        Main[Main Loop]
    end

    User -->|发布 MessageAction| ES
    Agent -->|发布各种 Action| ES
    Runtime -->|发布 Observation| ES
    Memory -->|发布 RecallAction| ES

    ES -->|通知事件| AC
    ES -->|通知事件| RT
    ES -->|通知事件| MEM
    ES -->|通知事件| SRV
    ES -->|通知事件| Main

    AC -->|订阅事件| ES
    RT -->|订阅事件| ES
    MEM -->|订阅事件| ES
    SRV -->|订阅事件| ES
    Main -->|订阅事件| ES
```

### 5.4 运行时架构图

```mermaid
graph TB
    subgraph "Runtime 抽象层"
        Base[Runtime Base<br/>运行时基类]
    end

    subgraph "Runtime 实现"
        DockerRT[Docker Runtime<br/>Docker 容器执行]
        LocalRT[Local Runtime<br/>本地进程执行]
        CLIRT[CLI Runtime<br/>命令行执行]
        K8sRT[Kubernetes Runtime<br/>K8s Pod 执行]
        RemoteRT[Remote Runtime<br/>远程 API 执行]
    end

    subgraph "执行环境"
        Container[Docker Container<br/>隔离的容器环境]
        Process[Local Process<br/>本地进程]
        Pod[K8s Pod<br/>Kubernetes Pod]
        Server[Action Execution Server<br/>动作执行服务器]
    end

    subgraph "支持功能"
        Browser[Browser Environment<br/>浏览器环境]
        FileOps[File Operations<br/>文件操作]
        CmdExec[Command Execution<br/>命令执行]
        IPython[IPython Kernel<br/>Python 交互]
    end

    Base --> DockerRT
    Base --> LocalRT
    Base --> CLIRT
    Base --> K8sRT
    Base --> RemoteRT

    DockerRT --> Container
    LocalRT --> Process
    CLIRT --> Process
    K8sRT --> Pod
    RemoteRT --> Server

    Container --> Browser
    Container --> FileOps
    Container --> CmdExec
    Container --> IPython

    Process --> FileOps
    Process --> CmdExec

    Pod --> Browser
    Pod --> FileOps
    Pod --> CmdExec
    Pod --> IPython
```

### 5.5 Agent 类型架构图

```mermaid
graph TB
    subgraph "Agent 基类"
        BaseAgent[Agent<br/>抽象基类]
    end

    subgraph "具体 Agent 实现"
        CodeAct[CodeActAgent<br/>代码执行代理<br/>- 执行代码<br/>- 文件操作<br/>- 命令执行]
        Browse[BrowsingAgent<br/>网页浏览代理<br/>- 网页导航<br/>- 交互操作]
        ReadOnly[ReadonlyAgent<br/>只读代理<br/>- 仅读取文件<br/>- 分析代码]
        LOC[LOCAgent<br/>基于位置的代理]
        Visual[VisualBrowsingAgent<br/>可视化浏览代理]
    end

    subgraph "Agent 组件"
        Prompt[Prompt Manager<br/>提示词管理]
        Tools[Tools<br/>工具集合]
        Parser[Response Parser<br/>响应解析器]
    end

    subgraph "LLM 集成"
        LLM[LLM Interface]
        LiteLLM[LiteLLM]
    end

    BaseAgent --> CodeAct
    BaseAgent --> Browse
    BaseAgent --> ReadOnly
    BaseAgent --> LOC
    BaseAgent --> Visual

    CodeAct --> Prompt
    CodeAct --> Tools
    CodeAct --> Parser
    CodeAct --> LLM

    Browse --> Prompt
    Browse --> Tools
    Browse --> Parser
    Browse --> LLM

    ReadOnly --> Prompt
    ReadOnly --> Tools
    ReadOnly --> Parser
    ReadOnly --> LLM

    LLM --> LiteLLM
```

### 5.6 存储架构图

```mermaid
graph TB
    subgraph "存储抽象层"
        FileStore[FileStore<br/>文件存储接口]
    end

    subgraph "存储实现"
        Local[Local FileStore<br/>本地文件系统]
        S3[S3 FileStore<br/>Amazon S3]
        GCS[Google Cloud FileStore<br/>Google Cloud Storage]
    end

    subgraph "存储内容"
        Events[Event History<br/>事件历史]
        State[Session State<br/>会话状态]
        Files[Workspace Files<br/>工作空间文件]
        Memory[Conversation Memory<br/>对话记忆]
    end

    subgraph "记忆管理"
        MemoryMgr[Memory Manager<br/>记忆管理器]
        Condenser[Memory Condenser<br/>记忆压缩器]
        View[Memory View<br/>记忆视图]
    end

    FileStore --> Local
    FileStore --> S3
    FileStore --> GCS

    Events --> FileStore
    State --> FileStore
    Files --> FileStore
    Memory --> FileStore

    MemoryMgr --> Condenser
    MemoryMgr --> View
    MemoryMgr --> FileStore
```

### 5.7 部署架构图

```mermaid
graph TB
    subgraph "用户访问层"
        Browser[Web Browser<br/>网页浏览器]
        CLI[CLI Client<br/>命令行客户端]
    end

    subgraph "部署选项"
        Local[本地部署<br/>Local Deployment]
        Docker[Docker 部署<br/>Docker Deployment]
        K8s[Kubernetes 部署<br/>K8s Deployment]
        Cloud[云部署<br/>Cloud Deployment]
    end

    subgraph "Azure 部署方案"
        ACI[Azure Container Instances<br/>容器实例]
        AKS[Azure Kubernetes Service<br/>K8s 服务]
        AppService[Azure App Service<br/>应用服务]
        VM[Azure VM<br/>虚拟机]
    end

    subgraph "基础设施"
        ACR[Azure Container Registry<br/>容器注册表]
        Storage[Azure Storage<br/>存储服务]
        KeyVault[Azure Key Vault<br/>密钥保管库]
        Monitor[Azure Monitor<br/>监控服务]
    end

    Browser --> Local
    Browser --> Docker
    Browser --> K8s
    Browser --> Cloud
    CLI --> Local
    CLI --> Docker

    Cloud --> ACI
    Cloud --> AKS
    Cloud --> AppService
    Cloud --> VM

    ACI --> ACR
    AKS --> ACR
    AppService --> ACR
    VM --> ACR

    ACI --> Storage
    AKS --> Storage
    AppService --> Storage
    VM --> Storage

    ACI --> KeyVault
    AKS --> KeyVault
    AppService --> KeyVault

    ACI --> Monitor
    AKS --> Monitor
    AppService --> Monitor
    VM --> Monitor
```
