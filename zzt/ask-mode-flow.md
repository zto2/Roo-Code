# Roo-Code Ask 模式完整流程图

## 详细流程图

```mermaid
flowchart TD
    %% 用户输入阶段
    A[用户输入问题] --> B[VSCode 扩展接收输入]

    %% 任务初始化阶段
    B --> C[ClineProvider.handleNewTask]
    C --> D[创建 Task 实例]
    D --> E[Task.initializeTaskMode]
    E --> F[从提供者状态获取当前模式]

    %% 系统提示生成阶段
    F --> G[SYSTEM_PROMPT 函数]
    G --> H[getModeSelection 获取对应模式配置]
    H --> I[生成包含角色定义的系统提示]
    I --> J[添加工具描述和规则]
    J --> K[包含环境信息和自定义指令]

    %% API 请求处理阶段
    K --> L[Task.initiateTaskLoop]
    L --> M[构建用户消息内容]
    M --> N[添加环境详情 getEnvironmentDetails]
    N --> O[调用 attemptApiRequest]
    O --> P[发送到 AI 模型]

    %% 流式响应处理阶段
    P --> Q[流式接收助手响应]
    Q --> R[presentAssistantMessage 处理消息]

    %% 消息内容处理
    R --> S{消息类型?}
    S -->|文本内容| T[直接显示给用户]
    S -->|工具调用| U[validateToolUse 验证工具权限]

    %% 工具执行阶段
    U --> V{工具类型?}
    V -->|read_file| W[readFileTool 执行]
    V -->|search_files| X[searchFilesTool 执行]
    V -->|browser_action| Y[browserActionTool 执行]
    V -->|list_files| Z[listFilesTool 执行]
    V -->|codebase_search| AA[codebaseSearchTool 执行]
    V -->|其他工具| BB[相应工具执行函数]

    %% 工具结果处理
    W --> CC[pushToolResult 添加结果]
    X --> CC
    Y --> CC
    Z --> CC
    AA --> CC
    BB --> CC

    CC --> DD{是否还有更多内容块?}
    DD -->|是| R
    DD -->|否| EE[userMessageContentReady = true]

    %% 继续循环
    EE --> FF[准备下一轮 API 请求]
    FF --> M

    %% 任务完成阶段
    S -->|attempt_completion| GG[attemptCompletionTool 执行]
    GG --> HH[检查未完成 todos]
    HH --> II{有未完成 todos?}
    II -->|是| JJ[返回错误，不允许完成]
    II -->|否| KK[发送 completion_result 消息]
    KK --> LL[触发 TaskCompleted 事件]
    LL --> MM{是否为子任务?}
    MM -->|是| NN[askFinishSubTaskApproval 询问批准]
    MM -->|否| OO[询问用户是否满意结果]

    NN --> PP{用户批准?}
    PP -->|是| QQ[finishSubTask 完成子任务]
    PP -->|否| RR[继续当前任务]

    OO --> SS{用户满意?}
    SS -->|是| TT[任务完成]
    SS -->|否| UU[添加用户反馈到下一轮]
    UU --> M

    %% 错误处理
    U -->|验证失败| VV[返回工具错误]
    VV --> CC

    %% 样式定义
    classDef userInput fill:#e1f5fe
    classDef init fill:#f3e5f5
    classDef prompt fill:#e8f5e8
    classDef api fill:#fff3e0
    classDef process fill:#fce4ec
    classDef tool fill:#f1f8e9
    classDef complete fill:#e8eaf6
    classDef error fill:#ffebee

    class A,B userInput
    class C,D,E,F init
    class G,H,I,J,K prompt
    class L,M,N,O,P api
    class Q,R,S,T,U process
    class V,W,X,Y,Z,AA,BB tool
    class CC,DD,EE,FF process
    class GG,HH,II,KK,LL complete
    class MM,NN,OO,QQ,RR,SS,TT,UU complete
    class VV error
```

## 流程说明

### 1. 用户输入阶段

- 用户在 VSCode 中输入问题
- VSCode 扩展接收输入并创建新任务

### 2. 任务初始化阶段

- `ClineProvider.handleNewTask` 处理新任务创建
- 创建 `Task` 实例并初始化模式
- `Task.initializeTaskMode` 从提供者状态获取当前模式
- **所有模式都会经过这个阶段，只是根据不同模式生成不同的配置**

### 3. 系统提示生成阶段

- `SYSTEM_PROMPT` 函数生成完整的系统提示
- `getModeSelection` 获取对应模式的配置信息
- 包含角色定义、可用工具、规则和环境信息
- **所有模式都会生成系统提示，只是内容根据模式不同而不同**

### 4. API 请求处理阶段

- `Task.initiateTaskLoop` 开始任务循环
- 构建包含环境详情的用户消息
- 调用 `attemptApiRequest` 发送到 AI 模型

### 5. 流式响应处理阶段

- 流式接收 AI 模型的响应
- `presentAssistantMessage` 处理每个消息块

### 6. 消息内容处理

- **文本内容**: 直接显示给用户
- **工具调用**: 先验证工具权限，然后执行相应工具

### 7. 工具执行阶段

支持多种工具类型：

- `read_file`: 读取文件内容
- `search_files`: 搜索文件
- `browser_action`: 浏览器操作
- `list_files`: 列出目录内容
- `codebase_search`: 代码库搜索
- 其他工具...

### 8. 任务完成阶段

- `attemptCompletionTool` 处理任务完成
- 检查是否有未完成的 todos
- 发送完成结果消息
- 根据是否为子任务执行不同逻辑

## 关键设计特点

1. **统一处理**: 所有模式都经过相同的核心流程
2. **模式差异**: 通过系统提示和工具权限体现不同模式的差异
3. **工具验证**: 每个工具调用都经过权限验证
4. **流式处理**: 支持流式响应，提升用户体验
5. **错误处理**: 完善的错误处理和用户反馈
6. **状态管理**: 通过事件系统管理任务状态
7. **扩展性**: 支持自定义模式和 MCP 服务器

## 模拟执行流程

假设用户问："这个项目的 ask 模式是如何工作的？"

1. **输入处理**: 用户输入被接收，创建任务（可能是 ask 模式或其他模式）
2. **模式初始化**: 获取当前模式设置（可能是 ask 或其他模式）
3. **提示生成**: 根据当前模式生成相应的系统提示和工具配置
4. **API 调用**: 发送问题到 AI 模型
5. **响应处理**: AI 返回分析结果，可能调用工具读取相关代码
6. **工具执行**: 执行相应的工具获取信息（受模式权限限制）
7. **结果整合**: 将工具结果整合到最终回答中
8. **任务完成**: 用户满意后任务完成

这个流程确保了不同模式能够根据各自的权限和角色安全有效地处理任务，同时维护代码的安全性。
