# AI Agent 系统业界最佳实践

**基于 2024-2025 年生产环境实践总结**

## 1. 核心设计模式

### 1.1 反思模式 (Reflection Pattern)

**核心思想**：Agent 执行任务后自我评估并优化

**实现要点**：
```python
class ReflectionAgent:
    """带反思能力的 Agent"""
    
    def execute_with_reflection(self, task):
        # 1. 执行任务
        result = self.execute(task)
        
        # 2. 自我评估
        quality_score = self.evaluate_quality(result)
        
        # 3. 如果质量不达标，反思并重试
        if quality_score < 0.8:
            reflection = self.reflect(result, quality_score)
            result = self.execute_with_improvements(task, reflection)
        
        return result
    
    def evaluate_quality(self, result):
        """多维度质量评估"""
        criteria = {
            "accuracy": self._check_accuracy(result),
            "completeness": self._check_completeness(result),
            "relevance": self._check_relevance(result)
        }
        return sum(criteria.values()) / len(criteria)
    
    def reflect(self, result, score):
        """生成反思"""
        return f"""
        当前输出质量评分: {score}
        
        问题分析:
        - 准确性: {self._analyze_accuracy(result)}
        - 完整性: {self._analyze_completeness(result)}
        - 相关性: {self._analyze_relevance(result)}
        
        改进方向:
        {self._generate_improvements(result)}
        """
```

**应用场景**：
- ✅ 内容创作：自动优化文案质量
- ✅ 代码生成：自动执行测试并修复
- ✅ 数据分析：验证分析结果的准确性

**Crew AI 实现**：
```python
from crewai import Agent, Task, Crew

# 执行 Agent
executor_agent = Agent(
    role="执行专家",
    goal="完成任务",
    tools=[...],
    verbose=True
)

# 评审 Agent（反思角色）
reviewer_agent = Agent(
    role="质量评审员",
    goal="评估输出质量并提供改进建议",
    backstory="你是一位严格的质量评审专家...",
    verbose=True
)

# 任务1: 执行
execute_task = Task(
    description="生成运营策略",
    agent=executor_agent,
    expected_output="详细的运营策略"
)

# 任务2: 评审（反思）
review_task = Task(
    description="""
    评审以下策略的质量：
    {execute_task.output}
    
    评估维度：
    1. 数据准确性
    2. 逻辑完整性
    3. 可执行性
    
    如果质量不达标，提供具体改进建议。
    """,
    agent=reviewer_agent,
    context=[execute_task],
    expected_output="质量评估报告"
)

# 任务3: 改进（如果需要）
improve_task = Task(
    description="""
    根据评审意见改进策略：
    
    原策略: {execute_task.output}
    评审意见: {review_task.output}
    
    生成改进后的策略。
    """,
    agent=executor_agent,
    context=[execute_task, review_task],
    expected_output="改进后的策略"
)

reflection_crew = Crew(
    agents=[executor_agent, reviewer_agent],
    tasks=[execute_task, review_task, improve_task],
    process=Process.sequential
)
```

### 1.2 规划模式 (Planning Pattern)

**核心思想**：先制定完整计划，再按计划执行

**实现要点**：
```python
class PlanningAgent:
    """带规划能力的 Agent"""
    
    def execute_with_planning(self, goal):
        # 1. 制定计划
        plan = self.create_plan(goal)
        
        # 2. 验证计划
        if not self.validate_plan(plan):
            plan = self.revise_plan(plan)
        
        # 3. 按计划执行
        results = []
        for step in plan.steps:
            result = self.execute_step(step)
            results.append(result)
            
            # 4. 动态调整（如果需要）
            if self.should_replan(result):
                plan = self.replan(plan, results)
        
        # 5. 整合结果
        return self.synthesize_results(results)
    
    def create_plan(self, goal):
        """创建执行计划"""
        return {
            "goal": goal,
            "steps": [
                {"id": 1, "action": "收集数据", "dependencies": []},
                {"id": 2, "action": "分析数据", "dependencies": [1]},
                {"id": 3, "action": "生成策略", "dependencies": [2]},
                {"id": 4, "action": "验证策略", "dependencies": [3]}
            ],
            "estimated_time": "30 minutes"
        }
```

**Crew AI 实现**：
```python
# 规划 Agent
planner_agent = Agent(
    role="任务规划师",
    goal="将复杂任务分解为可执行的步骤",
    backstory="你是一位经验丰富的项目经理...",
    verbose=True
)

# 执行 Agent
executor_agent = Agent(
    role="执行专家",
    goal="按计划执行任务",
    tools=[...],
    verbose=True
)

# 任务1: 制定计划
planning_task = Task(
    description="""
    为以下目标制定详细执行计划：
    
    目标: {goal}
    
    计划要求：
    1. 将目标分解为3-5个子任务
    2. 明确每个子任务的输入和输出
    3. 标注任务之间的依赖关系
    4. 估算每个任务的执行时间
    
    输出格式：
    ## 执行计划
    ### 子任务1: [名称]
    - 输入: ...
    - 输出: ...
    - 依赖: ...
    - 预计时间: ...
    """,
    agent=planner_agent,
    expected_output="详细的执行计划"
)

# 任务2-N: 按计划执行
execution_tasks = []
for i in range(1, 6):  # 假设最多5个子任务
    task = Task(
        description=f"执行计划中的子任务{i}",
        agent=executor_agent,
        context=[planning_task],
        expected_output=f"子任务{i}的执行结果"
    )
    execution_tasks.append(task)

planning_crew = Crew(
    agents=[planner_agent, executor_agent],
    tasks=[planning_task] + execution_tasks,
    process=Process.sequential
)
```

### 1.3 工具使用模式 (Tool Use Pattern)

**核心思想**：通过工具扩展 Agent 能力

**工具分类**：

```
┌─────────────────────────────────────────────────────────────┐
│                      Tool 分类体系                           │
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │  信息获取类                                         │    │
│  │  • 搜索引擎 (SerperDevTool)                        │    │
│  │  • 数据库查询 (DatabaseQueryTool)                  │    │
│  │  • API 调用 (APICallTool)                          │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │  计算分析类                                         │    │
│  │  • Python 解释器 (CodeInterpreterTool)             │    │
│  │  • 数据分析 (DataAnalysisTool)                     │    │
│  │  • 统计计算 (StatisticsTool)                       │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │  内容生成类                                         │    │
│  │  • 文本生成 (TextGenerationTool)                   │    │
│  │  • 图表生成 (ChartGenerationTool)                  │    │
│  │  • 报告生成 (ReportGenerationTool)                 │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │  操作执行类                                         │    │
│  │  • 批量更新 (BatchUpdateTool)                      │    │
│  │  • 文件操作 (FileOperationTool)                    │    │
│  │  • API 调用 (APIExecutionTool)                     │    │
│  └────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

**最佳实践**：

```python
from crewai_tools import BaseTool
from typing import Type
from pydantic import BaseModel, Field
import logging

class RobustTool(BaseTool):
    """生产级 Tool 模板"""
    
    name: str = "工具名称"
    description: str = "详细的工具描述（供 LLM 理解）"
    
    # 配置
    max_retries: int = 3
    timeout: int = 30
    cache_enabled: bool = True
    cache_ttl: int = 300
    
    def _run(self, **kwargs):
        """执行工具"""
        try:
            # 1. 参数验证
            self._validate_params(**kwargs)
            
            # 2. 检查缓存
            if self.cache_enabled:
                cached = self._get_cache(**kwargs)
                if cached:
                    return cached
            
            # 3. 执行（带重试）
            result = self._execute_with_retry(**kwargs)
            
            # 4. 验证结果
            self._validate_result(result)
            
            # 5. 缓存结果
            if self.cache_enabled:
                self._set_cache(result, **kwargs)
            
            # 6. 记录日志
            self._log_success(**kwargs)
            
            return result
            
        except Exception as e:
            # 7. 错误处理
            self._log_error(e, **kwargs)
            return self._handle_error(e)
    
    def _execute_with_retry(self, **kwargs):
        """带重试的执行"""
        from tenacity import retry, stop_after_attempt, wait_exponential
        
        @retry(
            stop=stop_after_attempt(self.max_retries),
            wait=wait_exponential(multiplier=1, min=4, max=10)
        )
        def execute():
            return self._execute(**kwargs)
        
        return execute()
    
    def _execute(self, **kwargs):
        """实际执行逻辑（子类实现）"""
        raise NotImplementedError
    
    def _validate_params(self, **kwargs):
        """参数验证"""
        # 实现参数验证逻辑
        pass
    
    def _validate_result(self, result):
        """结果验证"""
        # 实现结果验证逻辑
        pass
    
    def _log_success(self, **kwargs):
        """记录成功日志"""
        logging.info(f"{self.name} executed successfully")
    
    def _log_error(self, error, **kwargs):
        """记录错误日志"""
        logging.error(f"{self.name} failed: {str(error)}")
    
    def _handle_error(self, error):
        """错误处理"""
        return {
            "success": False,
            "error": str(error),
            "error_type": type(error).__name__
        }
```

### 1.4 多 Agent 协作模式 (Multi-Agent Collaboration)

**核心思想**：多个专业 Agent 协同完成复杂任务

**协作模式**：

#### 模式 1: 顺序协作（Sequential）
```python
# 数据分析 -> 策略生成 -> 执行
crew = Crew(
    agents=[analyst, strategist, executor],
    tasks=[analyze_task, strategy_task, execute_task],
    process=Process.sequential  # 顺序执行
)
```

#### 模式 2: 层级协作（Hierarchical）
```python
# Manager 协调多个 Worker
manager = Agent(
    role="项目经理",
    goal="协调团队完成任务",
    allow_delegation=True  # 允许委托
)

workers = [
    Agent(role="数据分析师", ...),
    Agent(role="策略师", ...),
    Agent(role="执行专家", ...)
]

crew = Crew(
    agents=[manager] + workers,
    tasks=[...],
    process=Process.hierarchical,  # 层级流程
    manager_agent=manager
)
```

#### 模式 3: 并行协作（Parallel）
```python
# 多个 Agent 并行处理不同店铺
from crewai.flow.flow import Flow, start, listen

class ParallelFlow(Flow):
    @start()
    def split_stores(self):
        """分配店铺给不同 Agent"""
        self.state.store_groups = [
            ["store_1", "store_2"],
            ["store_3", "store_4"],
            ["store_5", "store_6"]
        ]
    
    @listen(split_stores)
    def process_group_1(self):
        """并行处理组1"""
        return crew1.kickoff(inputs={"stores": self.state.store_groups[0]})
    
    @listen(split_stores)
    def process_group_2(self):
        """并行处理组2"""
        return crew2.kickoff(inputs={"stores": self.state.store_groups[1]})
    
    @listen(split_stores)
    def process_group_3(self):
        """并行处理组3"""
        return crew3.kickoff(inputs={"stores": self.state.store_groups[2]})
```

## 2. 生产环境最佳实践

### 2.1 Prompt 工程

**原则**：
1. **清晰的角色定义**：明确 Agent 的身份和专长
2. **具体的目标**：使用可衡量的目标
3. **丰富的上下文**：提供必要的背景信息
4. **示例驱动**：包含输入输出示例
5. **格式约束**：明确期望的输出格式

**模板**：
```python
AGENT_PROMPT_TEMPLATE = """
# 角色
你是一位{role}，拥有{years}年的{domain}经验。

# 专长
你擅长：
{expertise_list}

# 目标
你的主要目标是：{goal}

# 工作方式
在处理任务时，你会：
1. {step_1}
2. {step_2}
3. {step_3}

# 输出要求
你的输出应该：
- {requirement_1}
- {requirement_2}
- {requirement_3}

# 示例
输入：{example_input}
输出：{example_output}
"""

# 使用
agent = Agent(
    role="跨境电商数据分析师",
    goal="分析店铺数据并识别优化机会",
    backstory=AGENT_PROMPT_TEMPLATE.format(
        role="跨境电商数据分析师",
        years="10",
        domain="电商数据分析和运营优化",
        expertise_list="""
        - 多平台数据整合与清洗
        - 销售趋势分析和预测
        - 库存优化建议
        - 定价策略分析
        """,
        goal="通过数据分析发现业务机会，提供可执行的优化建议",
        step_1="收集并验证数据的完整性和准确性",
        step_2="使用统计方法分析趋势和异常",
        step_3="生成具体的、可衡量的优化建议",
        requirement_1="包含具体的数字和百分比",
        requirement_2="提供清晰的行动步骤",
        requirement_3="标注置信度和风险等级",
        example_input="店铺A最近30天的销售数据",
        example_output="""
        ## 数据分析报告
        
        ### 关键发现
        1. 销量增长15% (从1000件到1150件)
        2. 转化率下降2% (从3.5%到3.43%)
        3. 平均客单价上涨8% (从$45到$48.6)
        
        ### 优化建议
        1. 优化商品详情页，提升转化率
           - 预期效果：转化率回升至3.5%以上
           - 置信度：高 (85%)
        """
    )
)
```

### 2.2 错误处理与降级

**多层次降级策略**：

```python
class GracefulDegradation:
    """优雅降级系统"""
    
    def __init__(self):
        self.degradation_levels = [
            "full_service",      # 完整服务
            "cached_data",       # 使用缓存数据
            "simplified_logic",  # 简化逻辑
            "manual_fallback"    # 人工接管
        ]
        self.current_level = "full_service"
    
    def execute_with_degradation(self, task):
        """带降级的执行"""
        for level in self.degradation_levels:
            try:
                if level == "full_service":
                    return self.execute_full(task)
                elif level == "cached_data":
                    return self.execute_with_cache(task)
                elif level == "simplified_logic":
                    return self.execute_simplified(task)
                else:
                    return self.manual_fallback(task)
            except Exception as e:
                logging.warning(f"Level {level} failed, degrading...")
                continue
        
        raise Exception("All degradation levels failed")
    
    def execute_full(self, task):
        """完整服务"""
        # 调用所有 Agent 和工具
        result = crew.kickoff(inputs=task)
        return result
    
    def execute_with_cache(self, task):
        """使用缓存"""
        # 尝试从缓存获取
        cached = cache.get(task.cache_key)
        if cached:
            return cached
        
        # 缓存未命中，降级到下一级
        raise CacheMissError()
    
    def execute_simplified(self, task):
        """简化逻辑"""
        # 使用简化的规则引擎，不调用 LLM
        return rule_engine.process(task)
    
    def manual_fallback(self, task):
        """人工接管"""
        # 通知人工处理
        notification.send_to_ops_team(task)
        return {"status": "pending_manual_review"}
```

### 2.3 可观测性

**完整的监控体系**：

```python
from opentelemetry import trace, metrics
from opentelemetry.exporter.prometheus import PrometheusMetricReader
from prometheus_client import Counter, Histogram, Gauge

# 1. 指标定义
agent_execution_counter = Counter(
    'agent_executions_total',
    'Total number of agent executions',
    ['agent_role', 'status']
)

agent_execution_duration = Histogram(
    'agent_execution_duration_seconds',
    'Agent execution duration',
    ['agent_role']
)

tool_call_counter = Counter(
    'tool_calls_total',
    'Total number of tool calls',
    ['tool_name', 'status']
)

# 2. 追踪装饰器
def trace_agent_execution(agent_role):
    """Agent 执行追踪"""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            with tracer.start_as_current_span(f"agent_{agent_role}"):
                start_time = time.time()
                try:
                    result = func(*args, **kwargs)
                    agent_execution_counter.labels(
                        agent_role=agent_role,
                        status="success"
                    ).inc()
                    return result
                except Exception as e:
                    agent_execution_counter.labels(
                        agent_role=agent_role,
                        status="error"
                    ).inc()
                    raise
                finally:
                    duration = time.time() - start_time
                    agent_execution_duration.labels(
                        agent_role=agent_role
                    ).observe(duration)
        return wrapper
    return decorator

# 3. 使用
@trace_agent_execution("data_analyst")
def run_analysis_crew(inputs):
    return data_analysis_crew.kickoff(inputs=inputs)

# 4. 自定义事件
from crewai import Crew

crew = Crew(
    agents=[...],
    tasks=[...],
    verbose=True
)

@crew.on_event("task_started")
def log_task_start(event):
    logging.info(f"Task {event.task_id} started")
    metrics.increment("task_started")

@crew.on_event("task_completed")
def log_task_complete(event):
    logging.info(f"Task {event.task_id} completed in {event.duration}s")
    metrics.histogram("task_duration", event.duration)

@crew.on_event("agent_error")
def handle_agent_error(event):
    logging.error(f"Agent {event.agent_id} error: {event.error}")
    alert_ops_team(event)
```

### 2.4 成本优化

**策略**：

```python
class CostOptimizer:
    """成本优化器"""
    
    def __init__(self):
        self.model_costs = {
            "gpt-4": 0.03,      # per 1K tokens
            "gpt-3.5-turbo": 0.002,
            "claude-3-opus": 0.015,
            "claude-3-sonnet": 0.003
        }
    
    def select_model(self, task_complexity):
        """根据任务复杂度选择模型"""
        if task_complexity == "high":
            return "gpt-4"
        elif task_complexity == "medium":
            return "claude-3-sonnet"
        else:
            return "gpt-3.5-turbo"
    
    def optimize_crew(self, crew):
        """优化 Crew 配置"""
        optimizations = []
        
        # 1. 简单任务使用便宜模型
        for agent in crew.agents:
            if agent.task_complexity == "low":
                agent.llm = ChatOpenAI(model="gpt-3.5-turbo")
                optimizations.append(f"Agent {agent.role}: gpt-4 -> gpt-3.5-turbo")
        
        # 2. 启用缓存
        crew.cache = True
        optimizations.append("Enabled caching")
        
        # 3. 批量处理
        crew.batch_size = 10
        optimizations.append("Enabled batch processing")
        
        return optimizations

# 使用
optimizer = CostOptimizer()

# 根据任务选择模型
agent = Agent(
    role="数据分析师",
    llm=ChatOpenAI(model=optimizer.select_model("medium")),
    ...
)

# 优化 Crew
optimizations = optimizer.optimize_crew(crew)
print(f"Applied optimizations: {optimizations}")
```

## 3. 安全与合规

### 3.1 输入验证

```python
class InputValidator:
    """输入验证器"""
    
    def validate_user_input(self, user_input):
        """验证用户输入"""
        # 1. 长度检查
        if len(user_input) > 10000:
            raise ValueError("Input too long")
        
        # 2. 注入攻击检查
        if self._contains_injection(user_input):
            raise SecurityError("Potential injection detected")
        
        # 3. 敏感信息检查
        if self._contains_sensitive_info(user_input):
            user_input = self._mask_sensitive_info(user_input)
        
        return user_input
    
    def _contains_injection(self, text):
        """检查注入攻击"""
        injection_patterns = [
            r"<script>",
            r"DROP TABLE",
            r"'; --",
            r"UNION SELECT"
        ]
        for pattern in injection_patterns:
            if re.search(pattern, text, re.IGNORECASE):
                return True
        return False
    
    def _contains_sensitive_info(self, text):
        """检查敏感信息"""
        patterns = {
            "credit_card": r"\d{4}[-\s]?\d{4}[-\s]?\d{4}[-\s]?\d{4}",
            "ssn": r"\d{3}-\d{2}-\d{4}",
            "api_key": r"sk-[a-zA-Z0-9]{48}"
        }
        for name, pattern in patterns.items():
            if re.search(pattern, text):
                return True
        return False
```

### 3.2 输出过滤

```python
class OutputFilter:
    """输出过滤器"""
    
    def filter_output(self, output):
        """过滤输出"""
        # 1. 移除敏感信息
        output = self._remove_sensitive_info(output)
        
        # 2. 验证内容合规性
        if not self._is_compliant(output):
            output = self._make_compliant(output)
        
        # 3. 添加免责声明
        output = self._add_disclaimer(output)
        
        return output
    
    def _remove_sensitive_info(self, text):
        """移除敏感信息"""
        # 移除 API keys
        text = re.sub(r"sk-[a-zA-Z0-9]{48}", "[API_KEY_REDACTED]", text)
        
        # 移除邮箱
        text = re.sub(r"\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b", 
                     "[EMAIL_REDACTED]", text)
        
        return text
    
    def _add_disclaimer(self, text):
        """添加免责声明"""
        disclaimer = "\n\n---\n*本内容由 AI 生成，仅供参考，请结合实际情况判断。*"
        return text + disclaimer
```

## 4. 测试策略

### 4.1 单元测试

```python
import pytest
from crewai import Agent, Task, Crew

class TestDataAnalysisCrew:
    """数据分析 Crew 测试"""
    
    @pytest.fixture
    def mock_data(self):
        """测试数据"""
        return {
            "store_id": "test_store",
            "sales": [100, 120, 150],
            "dates": ["2024-01-01", "2024-01-02", "2024-01-03"]
        }
    
    @pytest.fixture
    def analysis_crew(self):
        """创建测试 Crew"""
        agent = Agent(
            role="测试分析师",
            goal="分析测试数据",
            llm=MockLLM()  # 使用 Mock LLM
        )
        
        task = Task(
            description="分析数据",
            agent=agent,
            expected_output="分析报告"
        )
        
        return Crew(agents=[agent], tasks=[task])
    
    def test_crew_execution(self, analysis_crew, mock_data):
        """测试 Crew 执行"""
        result = analysis_crew.kickoff(inputs=mock_data)
        
        assert result is not None
        assert "分析" in result
    
    def test_error_handling(self, analysis_crew):
        """测试错误处理"""
        with pytest.raises(ValueError):
            analysis_crew.kickoff(inputs={"invalid": "data"})
```

### 4.2 集成测试

```python
class TestEndToEndFlow:
    """端到端流程测试"""
    
    def test_complete_operation_flow(self):
        """测试完整运营流程"""
        # 1. 数据收集
        data = data_collection_crew.kickoff(
            inputs={"store_ids": ["test_store_1"]}
        )
        assert data["success"] == True
        
        # 2. 数据分析
        analysis = analysis_crew.kickoff(inputs={"data": data})
        assert "insights" in analysis
        
        # 3. 策略生成
        strategy = strategy_crew.kickoff(inputs={"analysis": analysis})
        assert "recommendations" in strategy
        
        # 4. 执行（模拟）
        result = execution_crew.kickoff(
            inputs={"strategy": strategy},
            dry_run=True  # 不实际执行
        )
        assert result["status"] == "success"
```

### 4.3 性能测试

```python
import time
from concurrent.futures import ThreadPoolExecutor

class PerformanceTest:
    """性能测试"""
    
    def test_throughput(self, crew, n_requests=100):
        """测试吞吐量"""
        start_time = time.time()
        
        with ThreadPoolExecutor(max_workers=10) as executor:
            futures = [
                executor.submit(crew.kickoff, inputs={"test": i})
                for i in range(n_requests)
            ]
            results = [f.result() for f in futures]
        
        duration = time.time() - start_time
        throughput = n_requests / duration
        
        print(f"Throughput: {throughput:.2f} requests/second")
        assert throughput > 1.0  # 至少 1 req/s
    
    def test_latency(self, crew, n_samples=50):
        """测试延迟"""
        latencies = []
        
        for i in range(n_samples):
            start = time.time()
            crew.kickoff(inputs={"test": i})
            latency = time.time() - start
            latencies.append(latency)
        
        p50 = np.percentile(latencies, 50)
        p95 = np.percentile(latencies, 95)
        p99 = np.percentile(latencies, 99)
        
        print(f"Latency P50: {p50:.2f}s, P95: {p95:.2f}s, P99: {p99:.2f}s")
        assert p95 < 10.0  # P95 < 10秒
```

## 5. 总结：生产级 Checklist

### ✅ 设计阶段
- [ ] 明确每个 Agent 的单一职责
- [ ] 设计清晰的 Task 描述（80/20 法则）
- [ ] 选择合适的协作模式（Sequential/Hierarchical/Parallel）
- [ ] 规划 Tool 的复用和扩展
- [ ] 设计降级策略

### ✅ 开发阶段
- [ ] 使用生产级 Tool 模板（错误处理、重试、缓存）
- [ ] 实现完善的输入验证
- [ ] 添加输出过滤和合规检查
- [ ] 编写单元测试和集成测试
- [ ] 添加详细的日志和追踪

### ✅ 部署阶段
- [ ] 配置监控和告警
- [ ] 设置性能基线
- [ ] 实施成本优化策略
- [ ] 配置安全策略（认证、授权、加密）
- [ ] 准备回滚方案

### ✅ 运维阶段
- [ ] 监控关键指标（成功率、延迟、成本）
- [ ] 定期审查日志和告警
- [ ] 收集用户反馈
- [ ] 持续优化 Prompt
- [ ] 定期更新依赖和安全补丁

---

**文档版本**: v1.0  
**创建日期**: 2025-11-06  
**参考来源**: 
- AWS AI Agent 最佳实践
- Crew AI 官方文档
- 业界生产环境案例
- 学术研究论文

