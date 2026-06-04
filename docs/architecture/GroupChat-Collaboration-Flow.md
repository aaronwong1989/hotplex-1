# GroupChat 多 Bot 协作 — 流程与架构

> 配套文档：[Platform-Messaging-Extension.md](./Platform-Messaging-Extension.md)（消息层架构）
> 源码位置：`internal/messaging/groupchat/`
> Issue：[#633](https://github.com/hrygo/hotplex/issues/633)

---

## 1. 整体流程

```mermaid
flowchart TD
    subgraph trigger["1. 触发阶段"]
        A["👤 用户发送命令<br/>/discuss @bot1 @bot2 话题"] --> B["pipeline.go<br/>ParseControlCommand()"]
        B --> C{"命令匹配?"}
        C -- No --> X["常规消息处理"]
        C -- Yes --> D["Manager.StartDiscussion()"]
    end

    subgraph validate["2. 校验阶段"]
        D --> E{"bots >= 2?"}
        E -- No --> E1["❌ 错误"]
        E -- Yes --> F{"topic <= MaxTopicLength?<br/>(rune-aware)"}
        F -- No --> F1["❌ 错误"]
        F -- Yes --> G{"全局/用户配额?"}
        G -- 超限 --> G1["❌ 错误"]
        G -- OK --> H["创建 GroupSession<br/>持久化到 DB"]
    end

    subgraph loop["3. Turn Loop (goroutine)"]
        H --> I["snapshot sender<br/>避免数据竞争"]
        I --> J["go runTurnLoop()"]

        J --> K{"ctx.Done()?"}
        K -- Yes --> END1["endDiscussion()<br/>EndUserStopped"]

        K -- No --> L{"ShouldTerminate()?<br/>轮次/成本/全跳过"}
        L -- Yes --> END2["endDiscussion()"]

        L -- No --> M["RoundRobinSelector<br/>选择下一个发言者"]

        M --> N["executeTurn()"]

        subgraph turn["executeTurn()"]
            N --> N1["DeriveGroupSessionKey()<br/>派生子 session ID"]
            N1 --> N2["bridge.StartSession()<br/>启动 Worker 子会话"]
            N2 --> N3["buildTurnPrompt()<br/>构建上下文 prompt"]
            N3 --> N4["worker.Input(prompt)"]
            N4 --> N5["waitForCompletion()<br/>轮询 session 状态"]
        end

        N5 -- "超时" --> O["记录超时<br/>连续超时 → 驱逐 bot"]
        N5 -- "错误" --> P["记录错误<br/>继续下一轮"]
        N5 -- "完成" --> Q["SanitizeContent()<br/>安全过滤 + rune-aware 截断"]
        Q --> R["sender.SendTurnResponse()<br/>回复到平台线程"]
        R --> S["store.AppendTurn()<br/>更新 transcript + cost"]
        S --> T["Cooldown 间隔"]
        T --> K

        O --> K
        P --> K
    end

    subgraph stop["4. 终止 & 清理"]
        END1 --> V["cleanup(): 移除 active map"]
        END2 --> V
        W["/gc-stop<br/>或网关关闭"] --> STOP["StopDiscussion()<br/>cancel() + ←done (10s 超时)"]
        STOP --> V
        V --> Z["DB: GroupStateCompleted<br/>或 GatewayRestart"]
    end

    style trigger fill:#16213e,stroke:#58a6ff,color:#e0e0e0
    style validate fill:#1a1a2e,stroke:#bc8cff,color:#e0e0e0
    style loop fill:#0f3460,stroke:#58a6ff,color:#e0e0e0
    style turn fill:#162447,stroke:#f85149,color:#e0e0e0
    style stop fill:#1a1a2e,stroke:#f85149,color:#e0e0e0
```

---

## 2. 组件依赖关系

```mermaid
graph LR
    subgraph gateway["Gateway"]
        GW_RUN["gateway_run.go<br/>DI 组装"]
        BRIDGE["Bridge<br/>session 创建"]
        SM["session.Manager<br/>状态机"]
    end

    subgraph messaging["Messaging"]
        PIPE["pipeline.go<br/>命令解析"]
        REG["BotRegistry<br/>Bot 注册表"]
    end

    subgraph groupchat["GroupChat"]
        MGR["Manager<br/>生命周期编排"]
        STORE["Store<br/>SQLite/PG 持久化"]
        SAN["sanitize.go<br/>安全过滤"]
        CFG["config.go<br/>配置 + Validate()"]
        GUARD["loop_guard.go<br/>终止检查"]
        SEL["RoundRobinSelector<br/>发言者轮换"]
    end

    GW_RUN --> MGR
    GW_RUN --> STORE
    GW_RUN --> CFG

    PIPE -->|"CMD_GROUP_DISCUSS"| MGR

    MGR --> BRIDGE
    MGR --> SM
    MGR --> REG
    MGR --> STORE
    MGR --> SAN
    MGR --> GUARD
    MGR --> SEL

    style groupchat fill:#0f3460,stroke:#58a6ff,color:#e0e0e0
    style gateway fill:#16213e,stroke:#3fb950,color:#e0e0e0
    style messaging fill:#1a1a2e,stroke:#d29922,color:#e0e0e0
```

---

## 3. 单次 Turn 详细时序

```mermaid
sequenceDiagram
    participant M as Manager
    participant BR as Bridge
    participant SM as SessionManager
    participant W as Worker (Claude Code)
    participant PL as Platform (飞书/Slack)
    participant DB as Store (SQLite/PG)

    M->>M: RoundRobinSelector.Next() 选择发言者
    M->>M: DeriveGroupSessionKey(groupID, botID, turnNum)

    rect rgb(22, 36, 71)
        Note over M,W: executeTurn() 子会话
        M->>BR: StartSession(subSessionID, ...)
        BR->>SM: GetOrCreate(session)
        SM-->>M: Worker 就绪
        M->>M: buildTurnPrompt(botName, topic, transcript)
        M->>W: Input(prompt)
        M->>W: CloseInput() (EOF)
        loop 轮询 (500ms 间隔)
            M->>SM: Get(sessionID)
            SM-->>M: state = Running
        end
        SM-->>M: state = Completed
        M->>M: extractResponse(sessionID)
        M->>SM: Transition(Terminated)
    end

    M->>M: SanitizeContent(response, maxLen)

    alt SKIP 响应
        M->>DB: AppendTurn(skipped=true)
    else 正常响应
        M->>PL: SendTurnResponse(botName, content)
        M->>DB: AppendTurn(content, cost)
        M->>DB: UpdateGroupCost(turnCount, costAccumulated)
    else 超时
        M->>PL: SendControlMessage("⏱️ 超时跳过")
        M->>DB: AppendTurn(timeout=true)
        Note over M,GUARD: 连续超时 → ShouldEvictBot()
    end

    M->>M: Cooldown (默认 5s)
```

---

## 4. 安全过滤流水线

```mermaid
flowchart LR
    subgraph input["Bot 原始输出"]
        RAW["content string"]
    end

    subgraph filter["SanitizeContent()"]
        direction TB
        STEP1["① 提取代码块<br/>→ 唯一 placeholder"]
        STEP2["② 安全模式匹配<br/>prompt injection / 命令注入"]
        STEP3["③ 替换命中内容<br/>→ [filtered]"]
        STEP4["④ 还原代码块<br/>代码内容不过滤"]
        STEP5["⑤ rune-aware 截断<br/>maxLen 限制"]

        STEP1 --> STEP2 --> STEP3 --> STEP4 --> STEP5
    end

    subgraph output["安全输出"]
        SAFE["filtered + reason"]
        WRAP["WrapForPeer()"]
        SAFE --> WRAP
    end

    RAW --> STEP1

    style filter fill:#162447,stroke:#f85149,color:#e0e0e0
    style input fill:#16213e,stroke:#58a6ff,color:#e0e0e0
    style output fill:#0f3460,stroke:#3fb950,color:#e0e0e0
```

安全模式列表：

| 模式 | 匹配目标 | 说明 |
|------|---------|------|
| `(?i)\bignore\s+(all\s+)?previous\s+instructions?\b` | Prompt 注入 | 忽略先前指令 |
| `(?i)\bforget\s+(all\s+)?previous\s+instructions?\b` | Prompt 注入 | 遗忘先前指令 |
| `(?i)\byou\s+are\s+now\b` | 身份篡改 | 切换角色 |
| `(?i)\bdeveloper\s+mode\b` | 模式切换 | 开发者模式 |
| `(?i)\bsystem\s+prompt\b` | 信息泄露 | 系统提示词探测 |
| `(?im)^/discuss\b` | 递归命令 | 嵌套群聊 |
| `(?im)^\$讨论\b` | 递归命令 | 中文嵌套群聊 |
| `(?im)^/(gc\|park\|reset\|new)\b` | 控制命令 | 干扰会话状态 |

---

## 5. 终止条件

```mermaid
flowchart TD
    TURNS["ListTurns()"] --> CHECK["ShouldTerminate()"]

    CHECK -->|"MaxTurns<br/>(默认 15)"| END1["🛑 EndMaxTurns"]
    CHECK -->|"CostLimitUSD<br/>(默认 $1.00)"| END2["💰 EndCostLimit"]
    CHECK -->|"AllSkip<br/>(所有 Bot 连续 SKIP)"| END3["💤 EndAllSkip"]
    CHECK -->|"无终止条件"| CONTINUE["继续下一轮"]

    CHECK --> EVICT{"ShouldEvictBot()?"}
    EVICT -->|"连续超时 >= 2"| REMOVE["🚫 驱逐 Bot<br/>从参与者列表移除"]
    EVICT -->|"未触发"| CONTINUE

    style CHECK fill:#0f3460,stroke:#d29922,color:#e0e0e0
```

---

## 6. DB Schema

```mermaid
erDiagram
    group_sessions {
        TEXT id PK
        TEXT topic
        TEXT platform
        TEXT channel_id
        TEXT thread_ts
        TEXT owner_id
        TEXT bot_ids "JSON array"
        TEXT state "active/completed/gateway_restart"
        INTEGER max_turns "默认 15"
        INTEGER turn_count
        REAL cost_accumulated
        INTEGER turn_timeout_sec "默认 120"
        INTEGER cooldown_ms "默认 5000"
        TEXT end_reason
        DATETIME created_at
        DATETIME ended_at
    }

    group_turns {
        TEXT id PK
        TEXT group_session_id FK
        TEXT bot_id
        TEXT bot_name
        INTEGER turn_num
        TEXT content
        INTEGER skipped "默认 0"
        INTEGER sanitized "默认 0"
        TEXT sanitize_reason
        INTEGER timeout_count
        REAL cost_usd
        DATETIME created_at
    }

    group_chat_audit {
        INTEGER id PK
        TEXT event_type "discussion_start/end/bot_evicted/gateway_restart"
        TEXT session_id
        TEXT bot_id
        TEXT initiator
        INTEGER turn_num
        TEXT detail
        DATETIME created_at
    }

    group_sessions ||--o{ group_turns : "has many"
    group_sessions ||--o{ group_chat_audit : "has many"
```

---

## 7. 配置参考

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `enabled` | `false` | 启用 GroupChat 功能 |
| `max_turns` | `15` | 单次讨论最大轮次 |
| `turn_timeout_sec` | `120` | 单轮超时（秒） |
| `cooldown_ms` | `5000` | 轮次间冷却（毫秒） |
| `max_group_sessions` | `20` | 全局最大并行讨论数 |
| `max_sessions_per_user` | `2` | 每用户最大并行讨论数 |
| `max_turn_content_length` | `50000` | 单轮回复最大长度（字符） |
| `max_total_context_length` | `80000` | transcript 最大长度（字符） |
| `max_topic_length` | `500` | 话题最大长度（字符，rune-aware） |
| `cost_limit_usd` | `1.00` | 单次讨论成本上限（USD） |
| `pool_reservation` | `10` | session pool 预留数 |
