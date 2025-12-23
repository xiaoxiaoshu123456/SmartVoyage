# SmartVoyage 系统架构文档

## 架构概览

SmartVoyage 采用 **微服务架构** 和 **Agent2Agent (A2A) 模式**，实现了一个模块化、可扩展的智能旅行助手系统。

```
┌─────────────────────────────────────────────────────────────┐
│                   用户界面层                                 │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐        ┌─────────────┐                    │
│  │ Streamlit   │        │  命令行     │                    │
│  │  Web界面    │        │   CLI界面   │                    │
│  └──────┬──────┘        └──────┬──────┘                    │
└─────────┼───────────────────────┼───────────────────────────┘
          │                       │
          └───────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────┐
│                   业务逻辑层                                 │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────┐   │
│  │                意图识别引擎                          │   │
│  │  • 基于LLM的意图分类                                │   │
│  │  • 查询改写                                        │   │
│  │  • 多意图处理                                      │   │
│  └──────────────────────────┬──────────────────────────┘   │
│                              │                              │
│  ┌──────────────────────────▼──────────────────────────┐   │
│  │                代理路由器                            │   │
│  │  • 意图到代理的映射                                 │   │
│  │  • 并发代理调用                                     │   │
│  │  • 响应聚合                                        │   │
│  └──────────────────────────┬──────────────────────────┘   │
└─────────────────────────────┼───────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────┐
│                   代理服务层                                 │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │   天气查询   │  │   票务查询   │  │   票务预订   │        │
│  │   代理       │  │   代理       │  │   代理       │        │
│  │  (5005)     │  │  (5006)     │  │  (5007)     │        │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘        │
└─────────┼─────────────────┼─────────────────┼───────────────┘
          │                 │                 │
┌─────────▼─────────────────▼─────────────────▼───────────────┐
│                   数据访问层                                 │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │   MySQL     │  │   外部API    │  │   文件系统   │        │
│  │   数据库     │  │   (天气/票务) │  │   (日志)    │        │
│  └─────────────┘  └─────────────┘  └─────────────┘        │
└─────────────────────────────────────────────────────────────┘
```

## 核心组件详解

### 1. 用户界面层

#### 1.1 Streamlit Web 界面 (`app.py`)
- **技术栈**: Streamlit + 自定义 CSS
- **特性**:
  - 实时聊天界面
  - 响应式两栏布局
  - Agent Card 展示
  - 会话状态管理
- **核心状态**:
  ```python
  st.session_state.messages = []          # 对话历史
  st.session_state.agent_network = None   # 代理网络
  st.session_state.llm = None            # LLM 实例
  st.session_state.conversation_history = ""  # 完整对话历史
  ```

#### 1.2 命令行界面 (`main.py`)
- **技术栈**: 纯 Python + 标准输入输出
- **特性**:
  - 交互式命令行
  - 模拟 Streamlit 状态管理
  - 支持代理卡片查看
  - 简化部署和调试

### 2. 业务逻辑层

#### 2.1 意图识别引擎
**位置**: `app.py:85-106`, `main.py:56-85`
**工作流程**:
```
用户输入 → LLM意图识别 → JSON解析 → 意图分类
```

**关键算法**:
```python
def intent_agent(user_input):
    # 1. 构建提示词链
    chain = SmartVoyagePrompts.intent_prompt() | llm

    # 2. 调用LLM
    intent_response = chain.invoke({
        "conversation_history": history,
        "query": user_input,
        "current_date": current_date
    })

    # 3. 解析JSON响应
    intent_output = json.loads(intent_response)

    # 4. 提取结果
    return {
        "intents": intent_output.get("intents", []),
        "user_queries": intent_output.get("user_queries", {}),
        "follow_up_message": intent_output.get("follow_up_message", "")
    }
```

#### 2.2 代理路由器
**位置**: `app.py:152-213`, `main.py:115-175`
**路由逻辑**:
```python
意图映射表 = {
    "weather": "WeatherQueryAssistant",
    "flight": "TicketQueryAssistant",
    "train": "TicketQueryAssistant",
    "concert": "TicketQueryAssistant",
    "order": "TicketOrderAssistant",
    "attraction": "直接LLM生成"  # 特殊处理
}
```

**代理调用流程**:
```
1. 获取代理实例: agent_network.get_agent(agent_name)
2. 构建消息: Message + Task
3. 异步调用: asyncio.run(agent.send_task_async(task))
4. 处理响应: 解析状态和内容
5. 结果总结: 使用特定提示词模板
```

### 3. 代理服务层

#### 3.1 天气查询代理 (`a2a_server/weather_server.py`)
- **协议**: A2A (Agent2Agent)
- **端口**: 5005
- **功能**: 查询城市天气信息
- **数据源**: MySQL 数据库 + 外部天气 API

#### 3.2 票务查询代理 (`a2a_server/ticket_server.py`)
- **协议**: A2A
- **端口**: 5006
- **功能**: 查询航班、火车、演唱会票务
- **支持类型**: 单程、往返、多城市

#### 3.3 票务预订代理 (`a2a_server/order_server.py`)
- **协议**: A2A
- **端口**: 5007
- **功能**: 处理票务预订、订单管理
- **业务流程**: 查询→选择→预订→确认

### 4. 数据访问层

#### 4.1 数据库设计
**核心表**:
```sql
-- 天气表
CREATE TABLE weather_data (
    city VARCHAR(50),
    date DATE,
    temperature_range VARCHAR(20),
    weather_desc VARCHAR(100),
    humidity INT,
    wind_direction VARCHAR(20),
    precipitation VARCHAR(20)
);

-- 票务表
CREATE TABLE ticket_data (
    type ENUM('flight', 'train', 'concert'),
    departure_city VARCHAR(50),
    arrival_city VARCHAR(50),
    departure_time DATETIME,
    price DECIMAL(10,2),
    seats_available INT
);

-- 订单表
CREATE TABLE orders (
    order_id VARCHAR(50) PRIMARY KEY,
    user_id VARCHAR(50),
    ticket_type VARCHAR(20),
    ticket_details JSON,
    order_time DATETIME,
    status ENUM('pending', 'confirmed', 'cancelled')
);
```

#### 4.2 外部 API 集成
- **天气 API**: 阿里云天气服务
- **票务 API**: 第三方票务平台接口
- **支付 API**: 支付宝/微信支付集成

## 通信协议

### A2A 协议消息格式
```json
{
  "task": {
    "id": "task-uuid",
    "message": {
      "role": "user",
      "content": {
        "type": "text",
        "text": "查询今天北京天气"
      }
    }
  }
}
```

### 代理响应格式
```json
{
  "status": {
    "state": "completed",
    "message": {
      "content": {
        "text": "查询成功"
      }
    }
  },
  "artifacts": [
    {
      "parts": [
        {
          "text": "北京今天晴，温度15-25°C..."
        }
      ]
    }
  ]
}
```

## 并发处理

### 多意图并发
```python
# 支持同时处理多个意图
for intent in intents:
    if intent == "weather":
        # 启动天气查询
    elif intent == "flight":
        # 启动机票查询
    # 所有查询并发执行
```

### 异步调用模式
```python
# 使用 asyncio 进行异步代理调用
async def call_agent_async(agent, task):
    return await agent.send_task_async(task)

# 在主线程中运行
raw_response = asyncio.run(agent.send_task_async(task))
```

## 错误处理机制

### 分层错误处理
```
用户界面层 → 业务逻辑层 → 代理服务层 → 数据访问层
     │           │           │           │
     ▼           ▼           ▼           ▼
友好错误提示 → 重试机制 → 代理降级 → 数据回退
```

### 具体实现
```python
try:
    # 意图识别
    intents, user_queries, follow_up_message = intent_agent(prompt)

    # 代理调用
    raw_response = asyncio.run(agent.send_task_async(task))

    # 结果处理
    if raw_response.status.state == 'completed':
        # 正常处理
    else:
        # 异常处理
        agent_result = raw_response.status.message['content']['text']

except json.JSONDecodeError:
    # JSON 解析错误
    logger.error("意图识别JSON解析失败")

except Exception as e:
    # 通用异常
    logger.error(f"处理异常: {str(e)}")
```

## 扩展性设计

### 添加新代理的步骤
1. **创建代理服务器**: 在 `a2a_server/` 下新建文件
2. **注册代理**: 在配置中添加代理 URL
3. **更新意图映射**: 在意图识别中添加新意图类型
4. **添加提示词**: 在 `main_prompts.py` 中添加总结提示词
5. **更新界面**: 在 Web 和 CLI 界面中显示新代理

### 插件化架构
```
新代理 → 注册到AgentNetwork → 自动出现在界面中
         │
         ▼
   意图识别自动适配
         │
         ▼
   用户无需修改代码
```

## 性能优化

### 缓存策略
```python
# 查询结果缓存
cache = {}
def get_weather_with_cache(city, date):
    key = f"{city}_{date}"
    if key in cache:
        return cache[key]
    else:
        result = query_weather(city, date)
        cache[key] = result
        return result
```

### 连接池管理
```python
# 数据库连接池
from mysql.connector import pooling

connection_pool = pooling.MySQLConnectionPool(
    pool_name="travel_pool",
    pool_size=5,
    host=conf.host,
    user=conf.user,
    password=conf.password,
    database=conf.database
)
```

## 监控与日志

### 日志级别策略
- **DEBUG**: 开发调试，记录详细过程
- **INFO**: 正常运行，记录关键操作
- **WARNING**: 潜在问题，记录异常情况
- **ERROR**: 错误情况，记录失败操作

### 监控指标
1. **代理健康度**: 响应时间、成功率
2. **意图识别准确率**: 分类准确度
3. **用户满意度**: 对话完成率
4. **系统负载**: 并发请求数、资源使用

## 部署架构

### 单机部署
```
同一台机器运行所有组件
+--------------------------------+
|  SmartVoyage 完整系统          |
|  +--------------------------+  |
|  | 代理服务器 (5005-5007)   |  |
|  |  Web服务器 (8501)        |  |
|  |  MySQL数据库             |  |
|  +--------------------------+  |
+--------------------------------+
```

### 分布式部署
```
微服务架构，组件分离
+------------+    +------------+    +------------+
|   Web前端   |    |  代理集群   |    |  数据库集群  |
|  (负载均衡) |    | (5005-5007)|    |  (主从复制) |
+------------+    +------------+    +------------+
```

---

**架构优势**:
1. **模块化**: 各组件独立，易于维护和扩展
2. **可扩展**: 支持水平扩展代理服务
3. **容错性**: 分层错误处理，系统健壮
4. **灵活性**: 支持多种界面和部署方式
5. **标准化**: 使用 A2A 协议，接口统一