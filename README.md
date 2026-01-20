### 1. 全局时序图 (Sequence Diagram)



```
sequenceDiagram
    autonumber
    actor User as 👤 用户 (Client)
    participant FastGPT as ⚡ FastGPT (业务中台)
    participant n8n as 🧠 n8n (AI 中间层)
    participant DeepSeek as 🤖 DeepSeek API

    Note over User, FastGPT: 阶段一：前端交互 & 上下文收集
    User->>FastGPT: 发送问题: "电脑蓝屏了"
    FastGPT->>FastGPT: 提取历史记录 (chat_history)
    FastGPT->>FastGPT: 注入上下文 (context: vip)

    Note over FastGPT, n8n: 阶段二：移交大脑 (HTTP POST)
    FastGPT->>n8n: POST /webhook/optimizer<br/>{message, history, context}
    
    rect rgb(240, 248, 255)
        Note right of n8n: 阶段三：元提示词优化 (Meta-Prompting)
        n8n->>n8n: JS: 格式化历史记录字符串
        n8n->>DeepSeek: 请求 1: [军师AI]<br/>"分析历史，生成最佳人设和温度"
        DeepSeek-->>n8n: 返回 JSON:<br/>{system_prompt: "资深技术专家...", temp: 0.1}
        n8n->>n8n: JS: 解析配置 (提取 Prompt & Temp)
    end

    rect rgb(255, 240, 245)
        Note right of n8n: 阶段四：执行回答 (Execution)
        n8n->>DeepSeek: 请求 2: [执行者AI]<br/>使用刚才生成的人设 + 低温(0.1)
        DeepSeek-->>n8n: 返回最终答案: "请尝试重启..."
    end

    Note over n8n, FastGPT: 阶段五：清洗与交付
    n8n->>n8n: JS: 清洗数据 (JSON.stringify)
    n8n-->>FastGPT: 返回 { "answer": "请尝试重启..." }
    
    FastGPT->>FastGPT: 提取字段 (answer)
    FastGPT-->>User: 展示最终回复
```

------

### 2. n8n 内部逻辑流向图 (Flowchart)



```
graph LR
    %% 样式定义
    classDef webhook fill:#ff9900,stroke:#333,stroke-width:2px,color:white;
    classDef code fill:#40e0d0,stroke:#333,stroke-width:1px;
    classDef ai fill:#ff6b6b,stroke:#333,stroke-width:2px,color:white;
    classDef response fill:#90ee90,stroke:#333,stroke-width:2px;

    %% 节点定义
    Start(("Webhook<br/>(接收请求)")):::webhook
    Process1["JS: 格式化上下文<br/>(将数组转为对话文本)"]:::code
    MetaAI{{"HTTP: 军师AI<br/>(生成配置)"}}:::ai
    Process2["JS: 提取配置<br/>(Parse JSON)"]:::code
    ExecAI{{"HTTP: 执行者AI<br/>(最终生成)"}}:::ai
    End(("Respond<br/>(返回 FastGPT)")):::response

    %% 连线逻辑
    Start -->|message + history| Process1
    Process1 -->|history_context| MetaAI
    
    subgraph 核心优化层 [Meta-Prompting Core]
        direction TB
        MetaAI -->|返回原始 JSON| Process2
        Process2 -->|输出: final_prompt + final_temp| ExecAI
    end
    
    ExecAI -->|返回 AI 回复| End
    
    %% 注释
    click MetaAI "根据历史判断意图"
    click ExecAI "根据军师指令执行"
```

------

### 3. FastGPT 编排逻辑图 (Flowchart)



```
graph TD
    %% 样式定义
    classDef start fill:#e1f5fe,stroke:#01579b,stroke-width:2px;
    classDef http fill:#fff9c4,stroke:#fbc02d,stroke-width:2px;
    classDef reply fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px;

    %% 节点
    UserStart["流程开始<br/>(用户输入问题)"]:::start
    VarInject["(隐式步骤)<br/>注入 context: VIP<br/>提取 chat_history"]
    
    subgraph 外部调用 [External Logic]
        Req["HTTP 请求<br/>(POST n8n Webhook)"]:::http
    end
    
    Extract["字段提取<br/>Key: answer"]
    UserEnd["指定回复<br/>(展示结果)"]:::reply

    %% 连线
    UserStart --> VarInject
    VarInject -->|Body: {msg, history, context}| Req
    Req -->|Response: {answer: '...'}| Extract
    Extract -->|{{answer}}| UserEnd
```

