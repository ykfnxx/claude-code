# MCP 集成设计方案

## 1. 概述

MCP (Model Context Protocol) 是连接AI Agent与外部能力的标准协议。
本方案设计如何将Simu-Emperor的游戏交互能力通过MCP服务暴露。

## 2. 当前架构问题

### 2.1 V5的问题

```python
# 当前：工具与Agent代码紧耦合
class EmperorAgent(BaseAgent):
    async def _setup_tools(self):
        self.tools = [
            create_incident_tool,      # 定义在agent内部
            update_indicator_tool,     # 游戏逻辑与agent绑定
            create_task_session_tool,  # 难以独立更新
        ]
    
    async def handle_tool_call(self, name, args):
        # 工具实现与agent逻辑混合
        if name == "create_incident":
            return await self._call_server_api(...)
```

**问题：**
- 新增工具需要修改Agent代码
- 工具逻辑与agent生命周期绑定
- 不同agent难以共享工具实现
- 测试困难（需要完整agent环境）

### 2.2 V6的解决方案

```
┌─────────────────┐         ┌──────────────────┐
│   Agent (bub)   │◄───────►│  GameTools MCP   │
│                 │   SSE   │  Service         │
│  - 核心决策      │         │  - 游戏API封装   │
│  - 上下文管理    │         │  - 数据验证      │
│  - 记忆检索      │         │  - 错误处理      │
└─────────────────┘         └──────────────────┘
                                     │
                              ┌──────┴──────┐
                              │  Simu Server │
                              │  (FastAPI)   │
                              └─────────────┘
```

## 3. MCP服务设计

### 3.1 服务拆分策略

```
packages/mcp-services/
├── game-tools/           # 游戏交互工具
│   ├── src/
│   │   ├── tools/        # 工具实现
│   │   │   ├── incident.py
│   │   │   ├── indicator.py
│   │   │   ├── task_session.py
│   │   │   └── ...
│   │   ├── server.py     # MCP服务器
│   │   └── client.py     # 服务器API客户端
│   └── pyproject.toml
│
├── context-manager/      # 上下文管理服务 (未来)
│   ├── src/
│   │   ├── compression.py    # 上下文压缩
│   │   ├── summarization.py  # 自动摘要
│   │   └── server.py
│   └── pyproject.toml
│
└── memory-service/       # 记忆检索服务 (未来)
    ├── src/
    │   ├── retrieval.py      # 向量检索
    │   ├── injection.py      # 记忆注入
    │   └── server.py
    └── pyproject.toml
```

### 3.2 GameTools MCP服务

```python
# packages/mcp-services/game-tools/src/server.py
from mcp.server.fastmcp import FastMCP
from simu_client import SimuServerClient

mcp = FastMCP("simu-game-tools")
client = SimuServerClient(base_url="http://localhost:8000")

@mcp.tool()
async def create_incident(
    agent_id: str,
    title: str,
    description: str,
    effects: list[dict],
    data_scope: dict = None
) -> dict:
    """
    创建事件，影响游戏状态。
    
    Args:
        agent_id: 创建事件的Agent ID
        title: 事件标题
        description: 事件描述
        effects: 效果列表，每个效果包含target_path和add/factor
        data_scope: 数据范围权限，限制可见的指标
    
    Returns:
        创建的事件对象
    """
    # 1. 验证effects (本地验证，减少API调用)
    for effect in effects:
        if not validate_effect(effect):
            return {"error": f"Invalid effect: {effect}"}
    
    # 2. 调用服务器API
    result = await client.create_incident(
        agent_id=agent_id,
        title=title,
        description=description,
        effects=effects,
        data_scope=data_scope
    )
    
    # 3. 格式化返回 (便于LLM理解)
    return format_incident_for_llm(result)

@mcp.tool()
async def update_indicator(
    agent_id: str,
    path: str,
    value: float,
    reason: str = None
) -> dict:
    """更新指标值"""
    ...

@mcp.tool()
async def create_task_session(
    parent_agent_id: str,
    task_type: str,
    goal: str,
    assignee_id: str = None
) -> dict:
    """创建子任务会话"""
    ...

@mcp.tool()
async def query_indicators(
    agent_id: str,
    paths: list[str] = None,
    data_scope: dict = None
) -> dict:
    """查询指标当前值"""
    ...

@mcp.tool()  
async def get_available_agents(
    agent_id: str,
    agent_type: str = None
) -> list[dict]:
    """获取可用的其他Agent列表"""
    ...

if __name__ == "__main__":
    mcp.run(transport="stdio")  # 或 SSE
```

### 3.3 Agent端集成

```python
# packages/sdk/simu_sdk/agent_v6.py
from bub import Bub, State, Event, Action
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

class SimuState(State):
    """Simu-Emperor状态机"""
    
    def __init__(self):
        self.history: list[Turn] = []
        self.context: ContextManager = ContextManager()
        self.mcp_tools: dict[str, callable] = {}
    
    async def process(self, event: Event) -> Action:
        if event.type == "user_message":
            return await self._handle_user_message(event)
        elif event.type == "tool_result":
            return await self._handle_tool_result(event)
        ...

class EmperorAgentV6:
    """V6 Agent - 基于bub + MCP"""
    
    def __init__(self, agent_id: str, config: dict):
        self.agent_id = agent_id
        self.bub = Bub(SimuState())
        self.mcp_session: ClientSession = None
        self.available_tools: list[Tool] = []
    
    async def initialize(self):
        """初始化MCP连接"""
        # 启动GameTools MCP服务
        server_params = StdioServerParameters(
            command="python",
            args=["-m", "simu_mcp.game_tools"],
            env={"SIMU_SERVER_URL": "http://localhost:8000"}
        )
        
        async with stdio_client(server_params) as (read, write):
            self.mcp_session = ClientSession(read, write)
            await self.mcp_session.initialize()
            
            # 获取可用工具列表
            self.available_tools = await self.mcp_session.list_tools()
            
            # 注册工具到bub状态机
            for tool in self.available_tools:
                self.bub.state.mcp_tools[tool.name] = \
                    lambda args, t=tool: self._call_mcp_tool(t, args)
    
    async def _call_mcp_tool(self, tool: Tool, args: dict) -> dict:
        """调用MCP工具"""
        result = await self.mcp_session.call_tool(tool.name, args)
        return result
    
    async def run_turn(self, user_input: str) -> str:
        """执行一个turn"""
        # 构建prompt (包含可用工具描述)
        prompt = self._build_prompt(user_input)
        
        # 调用LLM
        response = await self._call_llm(prompt)
        
        # 处理tool calls
        if response.tool_calls:
            for tool_call in response.tool_calls:
                # 通过MCP执行工具
                result = await self.bub.state.mcp_tools[tool_call.name](
                    tool_call.arguments
                )
                # 记录结果，继续对话
                ...
        
        return response.content
```

## 4. 工具设计规范

### 4.1 工具命名规范

```python
# 命名格式: <domain>_<action>_<target>
# 示例:
create_incident           # 创建事件
update_indicator          # 更新指标
query_indicators          # 查询指标
create_task_session       # 创建任务会话
finish_task_session       # 完成任务会话
get_agent_status          # 获取Agent状态
list_available_agents     # 列出可用Agent
```

### 4.2 工具描述规范

```python
@mcp.tool()
async def create_incident(...) -> dict:
    """
    创建事件，影响游戏世界状态。
    
    ## 使用场景
    - 需要改变游戏状态时 (如颁布政令、发生天灾)
    - 需要创建持久化效果时
    - 需要通知其他Agent时
    
    ## 参数说明
    - effects: 效果列表，支持add(加法)和factor(乘法)两种类型
      - add: 直接增加值，如 {"target_path": "国库.银两", "add": -10000}
      - factor: 百分比变化，如 {"target_path": "税收", "factor": 0.05} 表示+5%
    
    ## 注意事项
    - 效果必须是具体数值，避免模糊描述
    - 创建前建议先查询当前状态
    - 重大事件建议先创建草稿，确认后再提交
    
    ## 示例
    ```python
    # 减税5%
    create_incident(
        title="江南减税",
        effects=[{
            "target_path": "江南.税收",
            "factor": -0.05
        }]
    )
    ```
    """
```

### 4.3 错误处理规范

```python
@mcp.tool()
async def create_incident(...) -> dict:
    try:
        result = await client.create_incident(...)
        return {
            "success": True,
            "data": result,
            "summary": f"成功创建事件: {result['title']}"
        }
    except ValidationError as e:
        return {
            "success": False,
            "error_type": "validation_error",
            "error": str(e),
            "suggestion": "请检查effects参数格式"
        }
    except PermissionError as e:
        return {
            "success": False, 
            "error_type": "permission_denied",
            "error": str(e),
            "suggestion": "该操作超出您的权限范围"
        }
    except Exception as e:
        return {
            "success": False,
            "error_type": "internal_error",
            "error": "服务器内部错误",
            "suggestion": "请稍后重试或联系管理员"
        }
```

## 5. 迁移路径

### 5.1 阶段1: 并排运行 (Week 1-2)

```python
# packages/sdk/simu_sdk/agent_v6.py
class EmperorAgentV6(EmperorAgent):
    """V6 Agent，支持MCP和传统模式切换"""
    
    def __init__(self, *args, use_mcp: bool = False, **kwargs):
        super().__init__(*args, **kwargs)
        self.use_mcp = use_mcp
        self.mcp_client = None
    
    async def _call_tool(self, tool_name: str, args: dict):
        """工具调用路由"""
        if self.use_mcp and tool_name in MCP_TOOLS:
            # 通过MCP调用
            return await self.mcp_client.call_tool(tool_name, args)
        else:
            # 传统方式调用
            return await super()._call_tool(tool_name, args)
```

### 5.2 阶段2: MCP默认 (Week 3-4)

```python
# 默认启用MCP
agent = EmperorAgentV6(agent_id="emperor-1", use_mcp=True)

# 传统模式作为fallback
if not mcp_available:
    agent.use_mcp = False
```

### 5.3 阶段3: 移除传统模式 (Week 5+)

```python
# 完全移除传统工具调用代码
class EmperorAgentV6:
    """纯MCP Agent"""
    
    async def _call_tool(self, tool_name: str, args: dict):
        # 只支持MCP
        return await self.mcp_client.call_tool(tool_name, args)
```

## 6. 部署方案

### 6.1 本地开发

```bash
# 启动MCP服务
python -m simu_mcp.game_tools \
    --transport stdio \
    --server-url http://localhost:8000

# 或SSE模式 (便于调试)
python -m simu_mcp.game_tools \
    --transport sse \
    --port 8080
```

### 6.2 生产部署

```yaml
# docker-compose.yml
version: '3'
services:
  simu-server:
    build: ./packages/server
    ports:
      - "8000:8000"
  
  game-tools-mcp:
    build: ./packages/mcp-services/game-tools
    environment:
      - SIMU_SERVER_URL=http://simu-server:8000
    # stdio模式，作为sidecar运行
  
  agent-1:
    build: ./packages/agents
    environment:
      - MCP_GAME_TOOLS_URL=http://game-tools-mcp:8080
    depends_on:
      - simu-server
      - game-tools-mcp
```

## 7. 测试策略

### 7.1 MCP服务测试

```python
# packages/mcp-services/game-tools/tests/test_tools.py
import pytest
from mcp.client.stdio import stdio_client

@pytest.fixture
async def mcp_client():
    """启动MCP服务并返回客户端"""
    server_params = StdioServerParameters(
        command="python",
        args=["-m", "simu_mcp.game_tools"],
    )
    async with stdio_client(server_params) as (read, write):
        session = ClientSession(read, write)
        await session.initialize()
        yield session

async def test_create_incident(mcp_client):
    """测试创建事件工具"""
    result = await mcp_client.call_tool("create_incident", {
        "agent_id": "test-agent",
        "title": "测试事件",
        "description": "这是一个测试",
        "effects": [{"target_path": "测试.指标", "add": 100}]
    })
    
    assert result["success"] is True
    assert result["data"]["title"] == "测试事件"
```

### 7.2 Agent集成测试

```python
# packages/sdk/tests/test_agent_v6.py
async def test_agent_with_mcp():
    """测试Agent通过MCP调用工具"""
    agent = EmperorAgentV6(agent_id="test", use_mcp=True)
    await agent.initialize()
    
    # 模拟用户请求创建事件
    response = await agent.run_turn(
        "创建一个减税事件，江南地区减税5%"
    )
    
    # 验证Agent正确调用了MCP工具
    assert "create_incident" in agent.bub.state.history
```

## 8. 性能考虑

### 8.1 连接复用

```python
# MCP连接池
class MCPConnectionPool:
    """复用MCP连接，避免频繁启动进程"""
    
    def __init__(self, max_connections: int = 5):
        self.pool: asyncio.Queue[ClientSession] = asyncio.Queue()
        self.max_connections = max_connections
    
    async def acquire(self) -> ClientSession:
        try:
            return self.pool.get_nowait()
        except asyncio.QueueEmpty:
            return await self._create_connection()
    
    async def release(self, session: ClientSession):
        await self.pool.put(session)
```

### 8.2 本地vs远程

| 模式 | 延迟 | 适用场景 |
|------|------|----------|
| stdio | <10ms | 本地开发，单进程 |
| SSE本地 | <20ms | 多agent共享服务 |
| SSE远程 | 50-200ms | 分布式部署 |

## 9. 总结

MCP集成的核心价值：

1. **解耦**: Agent核心与工具实现分离
2. **复用**: 不同Agent共享相同MCP服务
3. **测试**: 独立测试工具逻辑
4. **演进**: 工具可独立更新部署

实施优先级：
- P0: GameTools MCP服务 (2周)
- P1: Agent MCP客户端集成 (2周)
- P2: ContextManager MCP服务 (未来)
- P3: Memory MCP服务 (未来)
