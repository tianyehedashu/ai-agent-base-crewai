# 交互模式与 Memory 设计

**Crew AI 1.3.0 人机交互与记忆系统**

## 1. 交互模式设计

### 1.1 Crew AI 的三种交互模式

Crew AI 1.3.0 支持三种主要的交互模式：

```
┌─────────────────────────────────────────────────────────────┐
│                    交互模式                                  │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  自动执行     │  │  人工审批     │  │  对话式交互   │      │
│  │  (Automated) │  │  (Approval)  │  │  (Chat)      │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│       │                  │                  │               │
│  全自动运行        关键节点审批      持续对话交互            │
│  无需人工          人工决策点        实时反馈                │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 模式一：自动执行（推荐用于批量任务）

**适用场景**：
- 数据收集和分析
- 定时任务执行
- 批量操作处理

**实现方式**：
```python
from crewai import Agent, Task, Crew

# 完全自动化的 Crew
automated_crew = Crew(
    agents=[data_agent, analysis_agent],
    tasks=[collect_task, analyze_task],
    verbose=True
)

# 直接执行，无需人工干预
result = automated_crew.kickoff()
```

### 1.3 模式二：人工审批（推荐用于关键决策）

**适用场景**：
- 批量价格调整
- 大额促销活动
- 合规审查决策

**实现方式**：

#### 方法 1: 使用 Human Input Task

```python
from crewai import Agent, Task, Crew

# 定义需要人工输入的 Task
approval_task = Task(
    description="""
    请审查以下批量操作计划：
    
    {operation_plan}
    
    请确认：
    1. 操作范围是否合理
    2. 价格调整幅度是否可接受
    3. 是否有潜在风险
    
    请输入 'approve' 批准，或 'reject' 拒绝，并说明原因。
    """,
    agent=approval_agent,
    expected_output="批准或拒绝的决定及原因",
    human_input=True  # 启用人工输入
)

# 在 Crew 中使用
approval_crew = Crew(
    agents=[planner_agent, approval_agent, executor_agent],
    tasks=[plan_task, approval_task, execute_task],
    verbose=True
)

# 执行时会暂停等待人工输入
result = approval_crew.kickoff()
```

#### 方法 2: 使用 Callback 实现审批流程

```python
from crewai import Agent, Task, Crew
from typing import Dict, Any

class ApprovalSystem:
    """审批系统"""
    
    def __init__(self):
        self.pending_approvals = []
    
    def request_approval(self, task_output: Dict[str, Any]) -> bool:
        """
        请求人工审批
        
        Args:
            task_output: 任务输出
            
        Returns:
            bool: 是否批准
        """
        print("\n" + "="*60)
        print("需要人工审批")
        print("="*60)
        print(f"任务: {task_output.get('task_name')}")
        print(f"内容: {task_output.get('content')}")
        print(f"影响范围: {task_output.get('impact')}")
        print("="*60)
        
        # 实际应用中，这里可以：
        # 1. 发送通知到 Slack/Email
        # 2. 在 Web 界面显示审批请求
        # 3. 调用审批系统 API
        
        decision = input("请输入决定 (approve/reject): ").strip().lower()
        
        if decision == "approve":
            print("✅ 已批准")
            return True
        else:
            reason = input("请输入拒绝原因: ")
            print(f"❌ 已拒绝: {reason}")
            return False

# 使用审批系统
approval_system = ApprovalSystem()

def approval_callback(task_output):
    """审批回调函数"""
    return approval_system.request_approval(task_output.dict())

# 定义带审批的 Task
execute_task = Task(
    description="执行批量操作",
    agent=executor_agent,
    expected_output="执行结果",
    callback=approval_callback  # 执行前触发审批
)
```

#### 方法 3: 在 Flow 中实现审批

```python
from crewai.flow.flow import Flow, listen, start
from pydantic import BaseModel

class OperationState(BaseModel):
    plan: dict = {}
    approval_status: str = "pending"
    execution_result: dict = {}

class OperationFlow(Flow[OperationState]):
    """带审批的操作流程"""
    
    @start()
    def generate_plan(self):
        """生成操作计划"""
        print("生成操作计划...")
        result = planning_crew.kickoff()
        self.state.plan = result
    
    @listen(generate_plan)
    def request_approval(self):
        """请求人工审批"""
        print("\n" + "="*60)
        print("请审批以下操作计划：")
        print("="*60)
        print(self.state.plan)
        print("="*60)
        
        # 方式1: 命令行输入
        decision = input("批准? (yes/no): ").strip().lower()
        
        # 方式2: Web API（实际应用）
        # decision = self.wait_for_web_approval(self.state.plan)
        
        # 方式3: 集成审批系统
        # decision = approval_system.request_approval(self.state.plan)
        
        if decision == "yes":
            self.state.approval_status = "approved"
        else:
            self.state.approval_status = "rejected"
            print("操作已取消")
    
    @listen(request_approval)
    def execute_operation(self):
        """执行操作"""
        if self.state.approval_status != "approved":
            print("未获批准，跳过执行")
            return
        
        print("执行操作...")
        result = execution_crew.kickoff(inputs={"plan": self.state.plan})
        self.state.execution_result = result

# 使用
flow = OperationFlow()
result = flow.kickoff()
```

### 1.4 模式三：对话式交互（推荐用于咨询场景）

**适用场景**：
- 运营咨询
- 数据查询
- 策略建议

**实现方式**：

#### 方法 1: 简单的对话循环

```python
from crewai import Agent, Task, Crew

class ChatInterface:
    """对话式交互界面"""
    
    def __init__(self, crew: Crew):
        self.crew = crew
        self.conversation_history = []
    
    def chat(self, user_message: str) -> str:
        """
        与 Agent 对话
        
        Args:
            user_message: 用户消息
            
        Returns:
            str: Agent 回复
        """
        # 添加到历史
        self.conversation_history.append({
            "role": "user",
            "content": user_message
        })
        
        # 创建对话任务
        chat_task = Task(
            description=f"""
            用户说: {user_message}
            
            对话历史:
            {self._format_history()}
            
            请根据上下文回复用户。
            """,
            agent=self.crew.agents[0],
            expected_output="对用户的回复"
        )
        
        # 执行
        result = self.crew.kickoff(inputs={"task": chat_task})
        
        # 添加到历史
        self.conversation_history.append({
            "role": "assistant",
            "content": result
        })
        
        return result
    
    def _format_history(self) -> str:
        """格式化对话历史"""
        formatted = []
        for msg in self.conversation_history[-5:]:  # 只保留最近5轮
            role = "用户" if msg["role"] == "user" else "助手"
            formatted.append(f"{role}: {msg['content']}")
        return "\n".join(formatted)
    
    def run(self):
        """运行对话循环"""
        print("对话开始（输入 'quit' 退出）")
        print("-" * 60)
        
        while True:
            user_input = input("\n你: ").strip()
            
            if user_input.lower() in ['quit', 'exit', '退出']:
                print("对话结束")
                break
            
            if not user_input:
                continue
            
            response = self.chat(user_input)
            print(f"\n助手: {response}")

# 创建对话 Agent
chat_agent = Agent(
    role="跨境电商运营顾问",
    goal="回答用户关于跨境电商运营的问题",
    backstory="""
    你是一位经验丰富的跨境电商运营专家，擅长回答关于
    店铺运营、数据分析、策略优化等方面的问题。
    """,
    tools=[
        QuerySalesDataTool(),
        AnalyzeDataTool(),
        GenerateReportTool()
    ],
    memory=True,  # 启用记忆
    verbose=True
)

chat_crew = Crew(
    agents=[chat_agent],
    tasks=[],
    memory=True,  # 启用 Crew 级别记忆
    verbose=True
)

# 启动对话
chat_interface = ChatInterface(chat_crew)
chat_interface.run()
```

#### 方法 2: 基于 FastAPI 的 Web 对话接口

```python
from fastapi import FastAPI, WebSocket
from fastapi.responses import HTMLResponse
from crewai import Agent, Crew
import json

app = FastAPI()

# 创建对话 Agent
chat_agent = Agent(
    role="运营助手",
    goal="帮助用户解决运营问题",
    backstory="...",
    memory=True,
    verbose=True
)

chat_crew = Crew(
    agents=[chat_agent],
    memory=True
)

# 存储会话
sessions = {}

@app.websocket("/ws/{session_id}")
async def websocket_endpoint(websocket: WebSocket, session_id: str):
    """WebSocket 对话端点"""
    await websocket.accept()
    
    # 初始化会话
    if session_id not in sessions:
        sessions[session_id] = {
            "history": [],
            "crew": chat_crew
        }
    
    try:
        while True:
            # 接收用户消息
            data = await websocket.receive_text()
            message = json.loads(data)
            
            # 添加到历史
            sessions[session_id]["history"].append({
                "role": "user",
                "content": message["content"]
            })
            
            # 调用 Crew
            response = chat_crew.kickoff(inputs={
                "message": message["content"],
                "history": sessions[session_id]["history"]
            })
            
            # 添加回复到历史
            sessions[session_id]["history"].append({
                "role": "assistant",
                "content": response
            })
            
            # 发送回复
            await websocket.send_json({
                "role": "assistant",
                "content": response
            })
            
    except Exception as e:
        print(f"WebSocket 错误: {e}")
    finally:
        await websocket.close()

@app.get("/")
async def get():
    """简单的聊天界面"""
    return HTMLResponse("""
    <!DOCTYPE html>
    <html>
    <head>
        <title>运营助手</title>
    </head>
    <body>
        <h1>跨境电商运营助手</h1>
        <div id="chat"></div>
        <input type="text" id="message" placeholder="输入消息...">
        <button onclick="sendMessage()">发送</button>
        
        <script>
            const ws = new WebSocket('ws://localhost:8000/ws/session123');
            
            ws.onmessage = function(event) {
                const data = JSON.parse(event.data);
                document.getElementById('chat').innerHTML += 
                    '<p><strong>助手:</strong> ' + data.content + '</p>';
            };
            
            function sendMessage() {
                const input = document.getElementById('message');
                const message = input.value;
                
                document.getElementById('chat').innerHTML += 
                    '<p><strong>你:</strong> ' + message + '</p>';
                
                ws.send(JSON.stringify({content: message}));
                input.value = '';
            }
        </script>
    </body>
    </html>
    """)

# 运行: uvicorn main:app --reload
```

#### 方法 3: 集成 Streamlit 的对话界面

```python
import streamlit as st
from crewai import Agent, Crew

# 页面配置
st.set_page_config(
    page_title="运营助手",
    page_icon="🤖",
    layout="wide"
)

st.title("🤖 跨境电商运营助手")

# 初始化 Agent
@st.cache_resource
def init_crew():
    agent = Agent(
        role="运营顾问",
        goal="提供运营建议",
        backstory="...",
        memory=True
    )
    return Crew(agents=[agent], memory=True)

crew = init_crew()

# 初始化会话状态
if "messages" not in st.session_state:
    st.session_state.messages = []

# 显示历史消息
for message in st.session_state.messages:
    with st.chat_message(message["role"]):
        st.markdown(message["content"])

# 用户输入
if prompt := st.chat_input("有什么可以帮你的？"):
    # 显示用户消息
    st.session_state.messages.append({"role": "user", "content": prompt})
    with st.chat_message("user"):
        st.markdown(prompt)
    
    # 获取 Agent 回复
    with st.chat_message("assistant"):
        with st.spinner("思考中..."):
            response = crew.kickoff(inputs={
                "message": prompt,
                "history": st.session_state.messages
            })
            st.markdown(response)
    
    # 保存助手回复
    st.session_state.messages.append({"role": "assistant", "content": response})

# 侧边栏
with st.sidebar:
    st.header("会话控制")
    if st.button("清空对话"):
        st.session_state.messages = []
        st.rerun()
    
    st.header("统计")
    st.metric("对话轮数", len(st.session_state.messages) // 2)

# 运行: streamlit run app.py
```

## 2. Memory 设计

### 2.1 Crew AI 的三层记忆系统

```
┌─────────────────────────────────────────────────────────────┐
│                    Memory 架构                               │
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │  Short-term Memory (短期记忆)                       │    │
│  │  • 当前任务执行期间的上下文                          │    │
│  │  • Agent 间的信息传递                               │    │
│  │  • 存储: Redis (TTL: 1小时)                        │    │
│  └────────────────────────────────────────────────────┘    │
│                           ↓                                 │
│  ┌────────────────────────────────────────────────────┐    │
│  │  Long-term Memory (长期记忆)                        │    │
│  │  • 跨任务的知识积累                                  │    │
│  │  • 历史决策和经验                                    │    │
│  │  • 存储: Vector DB (Pinecone/Chroma)               │    │
│  └────────────────────────────────────────────────────┘    │
│                           ↓                                 │
│  ┌────────────────────────────────────────────────────┐    │
│  │  Entity Memory (实体记忆)                           │    │
│  │  • 特定实体的信息（店铺、商品、用户）                 │    │
│  │  • 实体关系图谱                                      │    │
│  │  • 存储: Graph DB (Neo4j) 或 PostgreSQL            │    │
│  └────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 启用 Memory

#### 基础配置

```python
from crewai import Agent, Crew

# Agent 级别启用记忆
agent = Agent(
    role="数据分析师",
    goal="分析店铺数据",
    backstory="...",
    memory=True,  # 启用 Agent 记忆
    verbose=True
)

# Crew 级别启用记忆
crew = Crew(
    agents=[agent],
    tasks=[...],
    memory=True,  # 启用 Crew 记忆
    verbose=True
)
```

#### 高级配置

```python
from crewai import Agent, Crew
from crewai.memory.short_term.short_term_memory import ShortTermMemory
from crewai.memory.long_term.long_term_memory import LongTermMemory
from crewai.memory.entity.entity_memory import EntityMemory

# 配置记忆系统
memory_config = {
    "short_term": ShortTermMemory(
        storage="redis",
        config={
            "host": "localhost",
            "port": 6379,
            "ttl": 3600  # 1小时
        }
    ),
    "long_term": LongTermMemory(
        storage="chroma",
        config={
            "collection_name": "crew_memory",
            "embedding_model": "text-embedding-ada-002"
        }
    ),
    "entity": EntityMemory(
        storage="postgres",
        config={
            "connection_string": "postgresql://..."
        }
    )
}

# 使用自定义记忆配置
crew = Crew(
    agents=[...],
    tasks=[...],
    memory=True,
    memory_config=memory_config,
    verbose=True
)
```

### 2.3 集成外部记忆系统 (Mem0)

```python
import os
from crewai import Crew
from mem0 import MemoryClient

# 设置 Mem0 API Key
os.environ["MEM0_API_KEY"] = "your-api-key"

# 创建 Mem0 客户端
memory_client = MemoryClient(user_id="store_manager_001")

# 配置 Crew 使用 Mem0
crew = Crew(
    agents=[...],
    tasks=[...],
    memory=True,
    memory_config={
        "provider": "mem0",
        "config": {
            "user_id": "store_manager_001"
        }
    },
    verbose=True
)

# 手动添加记忆
memory_client.add(
    "用户偏好使用激进的定价策略",
    user_id="store_manager_001"
)

# 查询记忆
memories = memory_client.search(
    "定价策略",
    user_id="store_manager_001"
)
```

### 2.4 实战示例：带记忆的运营助手

```python
from crewai import Agent, Task, Crew
from datetime import datetime

class OperationAssistant:
    """带记忆的运营助手"""
    
    def __init__(self):
        # 创建 Agent
        self.agent = Agent(
            role="运营顾问",
            goal="提供个性化的运营建议",
            backstory="""
            你是一位经验丰富的运营顾问，能够记住用户的偏好、
            历史决策和店铺特点，提供个性化的建议。
            """,
            memory=True,  # 启用记忆
            verbose=True
        )
        
        # 创建 Crew
        self.crew = Crew(
            agents=[self.agent],
            memory=True,  # 启用 Crew 记忆
            verbose=True
        )
        
        # 本地会话记忆
        self.session_memory = {
            "user_preferences": {},
            "store_context": {},
            "conversation_history": []
        }
    
    def chat(self, user_message: str, store_id: str = None) -> str:
        """
        与助手对话
        
        Args:
            user_message: 用户消息
            store_id: 店铺ID（可选）
            
        Returns:
            str: 助手回复
        """
        # 构建上下文
        context = self._build_context(user_message, store_id)
        
        # 创建任务
        task = Task(
            description=f"""
            用户消息: {user_message}
            
            上下文信息:
            {context}
            
            请根据用户的历史偏好和店铺情况，提供个性化的建议。
            如果用户提到了新的偏好，请记住它。
            """,
            agent=self.agent,
            expected_output="对用户的回复"
        )
        
        # 执行
        response = self.crew.kickoff(inputs={"task": task})
        
        # 更新会话记忆
        self._update_session_memory(user_message, response, store_id)
        
        return response
    
    def _build_context(self, message: str, store_id: str = None) -> str:
        """构建上下文"""
        context_parts = []
        
        # 用户偏好
        if self.session_memory["user_preferences"]:
            context_parts.append("用户偏好:")
            for key, value in self.session_memory["user_preferences"].items():
                context_parts.append(f"- {key}: {value}")
        
        # 店铺上下文
        if store_id and store_id in self.session_memory["store_context"]:
            context_parts.append(f"\n店铺 {store_id} 信息:")
            for key, value in self.session_memory["store_context"][store_id].items():
                context_parts.append(f"- {key}: {value}")
        
        # 最近对话
        if self.session_memory["conversation_history"]:
            context_parts.append("\n最近对话:")
            for conv in self.session_memory["conversation_history"][-3:]:
                context_parts.append(f"- {conv['timestamp']}: {conv['summary']}")
        
        return "\n".join(context_parts) if context_parts else "无历史上下文"
    
    def _update_session_memory(
        self,
        user_message: str,
        response: str,
        store_id: str = None
    ):
        """更新会话记忆"""
        # 添加到对话历史
        self.session_memory["conversation_history"].append({
            "timestamp": datetime.now().isoformat(),
            "user_message": user_message,
            "response": response,
            "summary": user_message[:50] + "..."
        })
        
        # 提取用户偏好（简单的关键词匹配）
        if "喜欢" in user_message or "偏好" in user_message:
            # 这里可以用 NLP 提取偏好
            pass
        
        # 保持历史记录在合理范围
        if len(self.session_memory["conversation_history"]) > 50:
            self.session_memory["conversation_history"] = \
                self.session_memory["conversation_history"][-50:]

# 使用示例
assistant = OperationAssistant()

# 第一轮对话
response1 = assistant.chat(
    "我想了解店铺 A 的销售情况",
    store_id="store_A"
)
print(response1)

# 第二轮对话（带上下文）
response2 = assistant.chat(
    "根据这个数据，你有什么建议？",
    store_id="store_A"
)
print(response2)

# 第三轮对话（记住偏好）
response3 = assistant.chat(
    "我偏好保守的策略，不要太激进",
    store_id="store_A"
)
print(response3)

# 第四轮对话（应用偏好）
response4 = assistant.chat(
    "帮我制定一个促销方案",
    store_id="store_A"
)
print(response4)  # 应该会考虑用户的保守偏好
```

### 2.5 Memory 最佳实践

#### 1. 合理设置 TTL

```python
# 短期记忆：当前会话
short_term_ttl = 3600  # 1小时

# 中期记忆：当天任务
medium_term_ttl = 86400  # 24小时

# 长期记忆：永久存储
# 使用向量数据库，无需 TTL
```

#### 2. 定期清理

```python
from crewai import Crew
import schedule
import time

def cleanup_old_memories():
    """清理过期记忆"""
    # 清理超过30天的短期记忆
    # 清理低相关性的长期记忆
    pass

# 每天凌晨2点清理
schedule.every().day.at("02:00").do(cleanup_old_memories)

while True:
    schedule.run_pending()
    time.sleep(3600)
```

#### 3. 记忆优先级

```python
class PrioritizedMemory:
    """带优先级的记忆"""
    
    def add_memory(self, content: str, priority: str = "normal"):
        """
        添加记忆
        
        Args:
            content: 记忆内容
            priority: 优先级 (critical/high/normal/low)
        """
        memory_entry = {
            "content": content,
            "priority": priority,
            "timestamp": datetime.now(),
            "access_count": 0
        }
        
        # 高优先级记忆永不过期
        if priority == "critical":
            memory_entry["ttl"] = None
        elif priority == "high":
            memory_entry["ttl"] = 86400 * 30  # 30天
        else:
            memory_entry["ttl"] = 86400 * 7  # 7天
        
        # 存储记忆
        self._store(memory_entry)
```

## 3. 综合示例：带交互和记忆的完整系统

```python
from crewai import Agent, Task, Crew, Flow
from typing import Dict, List
import streamlit as st

class InteractiveOperationSystem:
    """交互式运营系统"""
    
    def __init__(self):
        # 初始化 Agents
        self.agents = self._init_agents()
        
        # 初始化 Crews
        self.crews = self._init_crews()
        
        # 记忆系统
        self.memory = {
            "user_preferences": {},
            "store_contexts": {},
            "decision_history": []
        }
    
    def _init_agents(self) -> Dict[str, Agent]:
        """初始化 Agents"""
        return {
            "analyst": Agent(
                role="数据分析师",
                goal="分析店铺数据",
                memory=True,
                verbose=True
            ),
            "strategist": Agent(
                role="策略规划师",
                goal="制定运营策略",
                memory=True,
                verbose=True
            ),
            "executor": Agent(
                role="执行专家",
                goal="执行运营操作",
                memory=True,
                verbose=True
            )
        }
    
    def _init_crews(self) -> Dict[str, Crew]:
        """初始化 Crews"""
        return {
            "analysis": Crew(
                agents=[self.agents["analyst"]],
                memory=True
            ),
            "strategy": Crew(
                agents=[self.agents["strategist"]],
                memory=True
            ),
            "execution": Crew(
                agents=[self.agents["executor"]],
                memory=True
            )
        }
    
    def analyze_stores(self, store_ids: List[str]) -> Dict:
        """分析店铺（自动执行）"""
        return self.crews["analysis"].kickoff(
            inputs={"store_ids": store_ids}
        )
    
    def generate_strategy(
        self,
        analysis: Dict,
        interactive: bool = True
    ) -> Dict:
        """生成策略（可选交互）"""
        strategy = self.crews["strategy"].kickoff(
            inputs={"analysis": analysis}
        )
        
        if interactive:
            # 请求用户反馈
            feedback = self._request_feedback(strategy)
            if feedback["action"] == "modify":
                # 根据反馈调整策略
                strategy = self._adjust_strategy(strategy, feedback)
        
        return strategy
    
    def execute_strategy(
        self,
        strategy: Dict,
        require_approval: bool = True
    ) -> Dict:
        """执行策略（需要审批）"""
        if require_approval:
            approved = self._request_approval(strategy)
            if not approved:
                return {"status": "rejected"}
        
        return self.crews["execution"].kickoff(
            inputs={"strategy": strategy}
        )
    
    def _request_feedback(self, strategy: Dict) -> Dict:
        """请求用户反馈"""
        print("\n生成的策略:")
        print(strategy)
        print("\n请提供反馈:")
        print("1. 接受")
        print("2. 修改")
        print("3. 拒绝")
        
        choice = input("选择 (1/2/3): ")
        
        if choice == "1":
            return {"action": "accept"}
        elif choice == "2":
            modifications = input("请描述需要修改的地方: ")
            return {"action": "modify", "modifications": modifications}
        else:
            return {"action": "reject"}
    
    def _request_approval(self, strategy: Dict) -> bool:
        """请求审批"""
        print("\n需要审批:")
        print(strategy)
        
        decision = input("批准? (yes/no): ")
        
        # 记录决策
        self.memory["decision_history"].append({
            "strategy": strategy,
            "decision": decision,
            "timestamp": datetime.now()
        })
        
        return decision.lower() == "yes"
    
    def _adjust_strategy(
        self,
        strategy: Dict,
        feedback: Dict
    ) -> Dict:
        """根据反馈调整策略"""
        # 使用 Agent 根据反馈调整策略
        adjusted = self.crews["strategy"].kickoff(inputs={
            "original_strategy": strategy,
            "feedback": feedback["modifications"]
        })
        return adjusted

# 使用示例
system = InteractiveOperationSystem()

# 1. 自动分析
analysis = system.analyze_stores(["store_A", "store_B"])

# 2. 交互式生成策略
strategy = system.generate_strategy(analysis, interactive=True)

# 3. 需要审批的执行
result = system.execute_strategy(strategy, require_approval=True)
```

## 4. 总结

### 4.1 交互模式选择指南

| 场景 | 推荐模式 | 原因 |
|------|---------|------|
| 数据收集分析 | 自动执行 | 无需人工干预，提高效率 |
| 批量操作 | 人工审批 | 关键决策点需要确认 |
| 运营咨询 | 对话式交互 | 需要持续沟通和反馈 |
| 定时任务 | 自动执行 | 按计划自动运行 |
| 策略制定 | 混合模式 | 自动生成 + 人工审批 |

### 4.2 Memory 使用建议

1. **默认启用**：建议所有 Agent 和 Crew 都启用 `memory=True`
2. **合理分层**：短期记忆用于当前任务，长期记忆用于知识积累
3. **定期清理**：避免记忆系统膨胀影响性能
4. **优先级管理**：重要记忆设置更长的 TTL
5. **隐私保护**：敏感信息不要存入长期记忆

### 4.3 最佳实践

```python
# ✅ 推荐：清晰的交互设计
crew = Crew(
    agents=[...],
    memory=True,  # 启用记忆
    verbose=True  # 便于调试
)

# ✅ 推荐：在关键节点请求审批
@listen(generate_strategy)
def request_approval(self):
    if self.state.risk_level == "high":
        approved = self.wait_for_approval()
        if not approved:
            return

# ✅ 推荐：保存重要决策到记忆
memory_client.add(
    f"用户在{datetime.now()}批准了策略X",
    user_id="user_001"
)

# ❌ 避免：过度交互
# 不要在每个小步骤都请求用户输入

# ❌ 避免：记忆过载
# 不要存储所有细节，只存储关键信息
```

---

**文档版本**: v1.0  
**创建日期**: 2025-11-06  
**适用版本**: Crew AI 1.3.0+

