# Week 5 — Lab 3: Microsoft Agent Framework และ Agno

ตำแหน่ง Lab:

```text
5_agent_frameworks/
└── 3_maf_agno/
    ├── maf_lab.ipynb
    ├── agno_lab.ipynb
    ├── maf_worker.py
    ├── agno_worker.py
    ├── board.py
    └── workspace/
```

Lab 3 นำ Framework อีกสองตัวมาทำ Contract เดิม:

1. **Microsoft Agent Framework — MAF**
2. **Agno**

โจทย์ยังเหมือน Lab 1–2:

```text
อ่าน Goal จาก SQLite Board
→ สร้าง Steps
→ อ่าน notes.txt ผ่าน Filesystem MCP
→ แปลเป็นภาษาสเปน
→ เขียน spanish.txt
→ ปิด Steps
→ ปิด Goal
```

การคง `board.py`, Workspace และ Goal เดิม ทำให้เราเห็นความแตกต่างของ Framework โดยไม่สับสนกับความแตกต่างของ Business Logic.

---

## Learning Objectives

หลังจบ Lab นี้ควรอธิบายได้ว่า:

1. Microsoft Agent Framework สร้าง Agent จาก `client` และ `instructions` อย่างไร
2. MAF แตกต่างจาก Google ADK ในเรื่อง Runner อย่างไร
3. MAF Plain Python Functions กลายเป็น Tools อย่างไร
4. `MCPStdioTool` จัดการ Filesystem MCP อย่างไร
5. เหตุใดต้องเปิด MCP Tool ผ่าน `async with`
6. MAF Workflow Engine แตกต่างจาก Plain Agent อย่างไร
7. Agno สร้าง Agent จาก `OpenAIChat` และ `Agent` อย่างไร
8. `arun()` และ `result.content` ใช้อย่างไร
9. Agno Plain Functions กลายเป็น Tools อย่างไร
10. Agno `MCPTools` จัด Connection Lifecycle อย่างไร
11. เหตุใด Course ใช้ Async ตลอด Agno Notebook
12. Agent Instantiation Speed มีความหมายอย่างไร
13. AgentOS เพิ่มอะไรจาก Plain Agno SDK
14. ทั้งสอง Framework รองรับ Standalone และ Shared-board Mode อย่างไร
15. Business Risks ใดยังคงอยู่ไม่ว่าจะเปลี่ยน Framework
16. เมื่อใดควรเลือก MAF หรือ Agno

---

# 1. Setup

Repository ปัจจุบันกำหนด:

```text
Python >= 3.12

agent-framework-core >= 1.8.1
agent-framework-openai >= 1.8.1

agno[mcp] >= 2.6.14
```

และทั้งสอง Notebook ใช้:

```env
OPENAI_API_KEY=...
```

รวมถึง Node.js สำหรับ Filesystem MCP Server.

ติดตั้งจาก Repository Root:

```powershell
uv sync
```

ตรวจ Node:

```powershell
node --version
npx --version
```

Warm MCP Server:

```powershell
npx -y @modelcontextprotocol/server-filesystem .
```

เมื่อ Server เริ่มทำงานแล้วหยุดด้วย `Ctrl+C` จากนั้นเปิด Notebook ด้วย Python Environment ของ Repository

---

# 2. Shared Five-step Pattern

เหมือน Labs ก่อนหน้า:

```text
1. Create the agent
2. Run it
3. Add tools
4. Add MCP
5. Put it in a loop with a goal
```

ขั้นที่ 1–2:

```text
User
→ Model
→ Text
```

ขั้นที่ 3–4:

```text
Model
→ Tool
→ Environment
→ Observation
```

ขั้นที่ 5:

```text
Goal
→ Plan
→ Act
→ Observe
→ Update external state
→ Repeat
```

---

# Part A — Microsoft Agent Framework

# 3. Microsoft Agent Framework คืออะไร

Course อธิบาย MAF ว่าเป็น Agent SDK ของ Microsoft ที่รวมแนวทางจาก AutoGen และ Semantic Kernel เข้าสู่ API เดียว

Framework มีสองระดับสำคัญ:

```text
Plain Agent
→ Model–Tool Loop สำหรับงานทั่วไป

Workflow Engine
→ Typed graph สำหรับงานยาวและต้อง Resume ได้
```

Lab นี้ใช้ Plain Agent ก่อน แล้วปิดท้ายด้วยภาพรวม Workflow Engine.

Mental model:

```text
Agent
= Worker ที่ตัดสินใจใช้ Tools

Workflow
= Application กำหนด Nodes และ Edges
```

---

# 4. MAF Imports

```python
import warnings

warnings.filterwarnings(
    "ignore",
    message=r".*experimental.*",
)

from agent_framework import (
    Agent,
    MCPStdioTool,
)

from agent_framework.openai import (
    OpenAIChatClient,
)
```

Course ปิด Experimental Notices เพื่อให้ Notebook อ่านง่ายขึ้น แต่การซ่อน Warning ไม่ได้ทำให้ Feature เสถียรขึ้น

```text
Warning hidden
≠ Risk removed
```

---

# 5. Step 1 — Create a MAF Agent

สร้าง Model Client:

```python
MODEL = "gpt-5.4-mini"

client = OpenAIChatClient(
    model=MODEL
)
```

สร้าง Agent:

```python
agent = Agent(
    client=client,
    instructions=(
        "You are a concise, friendly "
        "assistant. Reply in a single "
        "short sentence."
    ),
)
```

MAF แยก:

```text
OpenAIChatClient
→ Provider และ Model communication

Agent
→ Instructions, tools และ agent loop
```

รูปแบบนี้ใกล้กับ Strands ที่ใช้ `OpenAIModel` แต่ MAF เรียก Model Adapter ว่า `client`.

---

# 6. Step 2 — Run a MAF Agent

```python
result = await agent.run(
    "Say hello in Spanish."
)

print(result.text)
```

Final Text อยู่ที่:

```python
result.text
```

MAF เป็น Async-first ใน Course:

```text
await agent.run(...)
```

ตอนยังไม่มี Tools:

```text
Input
→ Model
→ Result text
```

จึงยังเป็น LLM Call ที่ถูกห่อด้วย Agent API

---

# 7. MAF กับ Google ADK ต่างกันอย่างไร

Google ADK:

```text
LlmAgent
+
Runner
```

MAF:

```text
Agent
.run(...)
```

ใน ADK Developer เห็น Runtime Object แยกชัด:

```python
runner = InMemoryRunner(
    agent=agent
)
```

ใน MAF Plain Agent สามารถรันตัวเองได้:

```python
await agent.run(...)
```

ดังนั้น MAF Surface กระชับกว่าในงานง่าย แต่ Runtime Concerns บางส่วนถูกซ่อนอยู่ใน Agent API มากขึ้น

---

# 8. Step 3 — MAF Function Tools

MAF ใช้ Plain Typed Python Functions:

```python
def show_todos() -> list[dict]:
    """List every todo on the board."""

    return board.list_todos()
```

```python
def plan_steps(
    goal_id: int,
    steps: list[str],
) -> dict:
    """Break a goal into an ordered checklist."""

    step_ids = [
        board.add_step(
            goal_id,
            step,
        )
        for step in steps
    ]

    return {
        "goal_id": goal_id,
        "step_ids": step_ids,
    }
```

```python
def complete_task(
    task_id: int,
    result: str,
) -> dict:
    """Mark a todo as done."""

    board.complete_todo(
        task_id,
        result,
    )

    return {
        "task_id": task_id,
        "status": "done",
    }
```

Framework อ่าน:

```text
Function name
Type hints
Docstring
Parameters
```

แล้วสร้าง JSON Schema ให้โดยอัตโนมัติ

Decorator มีไว้เมื่ออยาก Customize เช่น Rename หรือเพิ่ม Constraints แต่ไม่จำเป็นสำหรับ Basic Tool.

---

# 9. Add MAF Tools

```python
board_agent = Agent(
    client=client,
    instructions=(
        "You manage a shared todo board."
    ),
    tools=[
        show_todos,
        complete_task,
    ],
)
```

เมื่อถาม:

```text
What is currently on the board?
```

Agent อาจทำ:

```text
Model decides current data is required
→ show_todos()
→ Receives todos
→ Produces answer
```

Agent Loop เริ่มหมุน:

```text
Decide
→ Call
→ Observe
→ Respond
```

---

# 10. Step 4 — MAF MCP

MAF ใช้:

```python
filesystem = MCPStdioTool(
    name="filesystem",
    command="npx",
    args=[
        "-y",
        "@modelcontextprotocol/"
        "server-filesystem",
        str(workspace),
    ],
    cwd=str(workspace),
)
```

จากนั้นต้องเปิด Connection:

```python
async with filesystem:
    file_agent = Agent(
        client=client,
        instructions=(
            "Use your file tools to "
            "read and write files."
        ),
        tools=[
            filesystem
        ],
    )

    result = await file_agent.run(
        "Read notes.txt and summarize it."
    )
```

MAF มอง MCP Server เป็น Tool Object และใส่ใน `tools=[...]` เหมือน Python Functions.

---

# 11. MAF MCP Lifecycle

```text
async with filesystem
        ↓
Start npx child process
        ↓
Initialize MCP session
        ↓
Discover filesystem tools
        ↓
Agent uses tools
        ↓
Close session
        ↓
Stop process
```

ถ้าไม่ปิด Lifecycle อย่างถูกต้อง อาจเกิด:

```text
MCP tools unavailable
Child process leak
Shutdown error
Hanging notebook
```

Course Worker สร้าง MCP Tool ภายใน `main()` แล้วเปิดด้วย `async with` ก่อนสร้าง Worker Agent.

---

# 12. Notebook-specific Filesystem Wrapper

Notebook ใช้ Wrapper รอบ `MCPStdioTool` เพื่อ:

```text
กำหนด Working Directory
ส่ง stderr ไป DEVNULL
ลด Startup Logging
แก้ปัญหา Jupyter บน Windows
```

ประโยชน์:

```text
Notebook output สะอาด
Child process เริ่มได้บน Windows
```

ข้อเสีย:

```text
MCP error details อาจหาย
Debugging ยากขึ้น
```

Production ควรส่ง Logs ไปยัง Structured Logger แทน Null Device

---

# 13. Step 5 — MAF Goal Loop

Worker:

```python
worker = Agent(
    client=client,
    instructions=INSTRUCTIONS,
    tools=[
        show_todos,
        plan_steps,
        complete_task,
        filesystem,
    ],
)
```

รัน:

```python
board.claim_todo(
    goal_id
)

async with filesystem:
    await worker.run(
        "Please work the pending "
        "goal on the board."
    )
```

Expected Loop:

```text
show_todos
→ Find active goal
→ plan_steps
→ Read notes.txt
→ Translate
→ Write spanish.txt
→ complete_task(step)
→ Repeat
→ complete_task(goal)
```

MAF จัด Model–Tool Loop ให้ภายใน `run()`.

---

# 14. MAF Worker Script

Standalone:

```powershell
uv run maf_worker.py
```

Script จะ:

```text
Reset board
→ Clear old spanish.txt
→ Add goal
→ Claim goal
→ Start MCP
→ Build Agent
→ Run loop
→ Print board and output
```

Shared-board Mode:

```powershell
uv run maf_worker.py `
  <taskId> `
  <boardPath>
```

Mode นี้จะ:

```text
ใช้ Shared Board
ใช้ Directory ของ Board เป็น WORK_DIR
Claim เฉพาะ Task ที่กำหนด
ไม่ Reset งานอื่น
ทำ Task แล้วปิดตัว
```

Worker ถูกเตรียมไว้สำหรับ Day 5 Orchestrator ที่เรียก Workers จากหลาย Framework เป็น Subprocesses.

---

# 15. MAF Workflow Engine

จุดเด่นที่ Course ยกขึ้นมาคือ Graph-based Workflow Engine

Concept:

```python
from agent_framework import (
    WorkflowBuilder,
)

workflow = (
    WorkflowBuilder()
    .add_edge(
        researcher,
        writer,
    )
    .add_edge(
        writer,
        reviewer,
    )
    .build()
)
```

Nodes อาจเป็น:

```text
Agent
Plain function
Validator
Human approval step
```

Workflow Engine เหมาะกับ:

```text
Long-running jobs
Checkpointing
Restartable workflows
Streaming
Human-in-the-loop
Typed routing
```

Mental model:

```text
Plain Agent
→ Model เลือกลำดับ Actions

Workflow Engine
→ Application กำหนดเส้นทางใหญ่
```

แนวคิดนี้ใกล้กับ LangGraph:

```text
State
Nodes
Edges
Checkpoints
Resume
```

---

# 16. หนึ่ง API ใน Python และ C#

MAF ถูกออกแบบให้มีแนวคิดใกล้กันระหว่าง Python และ C#

ประโยชน์เชิงองค์กร:

```text
ทีม Python และ .NET
ใช้ Mental Model เดียวกัน

Agent definitions
Workflow concepts
Tool contracts
มีรูปแบบใกล้เคียงกัน
```

สิ่งนี้สำคัญกับ Enterprise Environment ที่มีหลาย Technology Stacks มากกว่าการลด Code เพียงไม่กี่บรรทัด

---

# Part B — Agno

# 17. Agno คืออะไร

Agno เป็น Framework ที่เน้น:

```text
Agent instantiation ที่เบา
API ที่สั้น
Fast execution setup
Production runtime ผ่าน AgentOS
```

Course ใช้ Plain SDK แล้วทดสอบความเร็วในการสร้าง Agent จำนวนมากในตอนท้าย.

Mental model:

```text
Agno Agent
= Model
+ Instructions
+ Tools
+ Lightweight runtime
```

---

# 18. Agno Imports

```python
from agno.agent import Agent
from agno.models.openai import (
    OpenAIChat,
)
from agno.tools.mcp import MCPTools
from mcp import StdioServerParameters
```

หน้าที่:

```text
Agent
→ Model–Tool Loop

OpenAIChat
→ Model adapter

MCPTools
→ External MCP tool collection
```

---

# 19. Step 1 — Create an Agno Agent

```python
MODEL = "gpt-5.4-mini"

model = OpenAIChat(
    id=MODEL
)

agent = Agent(
    model=model,
    instructions=(
        "You are a concise, friendly "
        "assistant. Reply in a single "
        "short sentence."
    ),
)
```

Agno แยก:

```text
OpenAIChat
→ Model integration

Agent
→ Instructions, tools และ execution
```

---

# 20. Step 2 — Run Agno

```python
result = await agent.arun(
    input="Say hello in Spanish."
)

print(result.content)
```

Final Content อยู่ที่:

```python
result.content
```

Agno มี Synchronous Path แต่ Course ใช้ Async ตลอด เพราะเมื่อเพิ่ม MCP แล้ว Worker ต้องใช้ Async Lifecycle อยู่ดี

```text
Without MCP
→ Sync or async

With MCP
→ Async is the consistent path
```

---

# 21. Step 3 — Agno Function Tools

Agno รับ Plain Typed Python Functions:

```python
def show_todos() -> list[dict]:
    """List every todo on the board."""

    return board.list_todos()
```

```python
def plan_steps(
    goal_id: int,
    steps: list[str],
) -> dict:
    """Break a goal into steps."""

    return {
        "goal_id": goal_id,
        "step_ids": [
            board.add_step(
                goal_id,
                step,
            )
            for step in steps
        ],
    }
```

```python
def complete_task(
    task_id: int,
    result: str,
) -> dict:
    """Mark a todo done."""

    board.complete_todo(
        task_id,
        result,
    )

    return {
        "task_id": task_id,
        "status": "done",
    }
```

Agno อ่าน Type Hints และ Docstrings เพื่อสร้าง Schema

Optional `@tool` Decorator มีไว้สำหรับความสามารถเสริม เช่น:

```text
Caching
Custom metadata
Tool policies
```

Basic Tools ไม่จำเป็นต้องมี Decorator.

---

# 22. Add Agno Tools

```python
board_agent = Agent(
    model=model,
    instructions=(
        "You manage a shared todo board."
    ),
    tools=[
        show_todos,
        complete_task,
    ],
)
```

รัน:

```python
result = await board_agent.arun(
    input=(
        "What is on the board "
        "right now?"
    )
)
```

Agent จะเลือกเรียก `show_todos()` เมื่อเห็นว่าต้องใช้ข้อมูลล่าสุด

---

# 23. Step 4 — Agno MCP

สร้าง Server Parameters:

```python
server = StdioServerParameters(
    command="npx",
    args=[
        "-y",
        "@modelcontextprotocol/"
        "server-filesystem",
        str(workspace),
    ],
    cwd=str(workspace),
)
```

เปิด Toolset:

```python
async with MCPTools(
    server_params=server,
    timeout_seconds=60,
) as filesystem:

    file_agent = Agent(
        model=model,
        instructions=(
            "Use filesystem tools."
        ),
        tools=[
            filesystem
        ],
    )

    result = await file_agent.arun(
        input=(
            "Read notes.txt and "
            "summarize it."
        )
    )
```

Agno ใส่ MCP Tool Collection ลง `tools=[...]` เหมือน MAF และ Strands แต่ Lifecycle เปิดผ่าน `async with MCPTools(...)`.

---

# 24. Agno MCP Lifecycle

```text
Build StdioServerParameters
        ↓
async with MCPTools
        ↓
Start MCP server
        ↓
Discover tools
        ↓
Pass filesystem to Agent
        ↓
Run agent
        ↓
Close process
```

Course Worker Patch `stdio_client` ให้ส่ง Error Log ไป `DEVNULL` เนื่องจาก Agno MCP Wrapper ไม่เปิด Parameter นี้โดยตรง และเพื่อให้ Jupyter บน Windows เปิด Child Process ได้.

จุดอ่อนของ Patch:

```text
แก้ Module-level function
มีผลทั้ง Process
ผูกกับ Internal Implementation
อาจพังเมื่อ Library update
```

นี่คือ Version-fragile workaround ไม่ควรถือเป็น Production Pattern

---

# 25. Step 5 — Agno Goal Loop

```python
async with MCPTools(
    server_params=server,
    timeout_seconds=60,
) as filesystem:

    worker = Agent(
        model=model,
        instructions=INSTRUCTIONS,
        tools=[
            show_todos,
            plan_steps,
            complete_task,
            filesystem,
        ],
    )

    await worker.arun(
        input=(
            "Please work the pending "
            "goal on the board."
        )
    )
```

Expected Loop:

```text
Read board
→ Plan steps
→ Read notes.txt
→ Translate
→ Write spanish.txt
→ Complete steps
→ Complete goal
```

Agno จัด Tool Loop ภายใน `arun()`.

---

# 26. Agno Worker Script

Standalone:

```powershell
uv run agno_worker.py
```

Shared-board Mode:

```powershell
uv run agno_worker.py `
  <taskId> `
  <boardPath>
```

Worker ใช้:

```text
WORKER_MODEL
→ เปลี่ยน Model ID

BOARD_PATH
→ ชี้ไป Shared SQLite Board

WORK_DIR
→ Workspace ของ Shared Task
```

Script จะ Claim Task ที่ระบุ ทำงานนั้น และหยุด โดยไม่แตะ Tasks อื่นบน Board.

---

# 27. Agno Instantiation Speed

Notebook วัดการสร้าง Agent 1,000 ตัว:

```python
import time

start = time.perf_counter()

agents = [
    Agent(
        model=model,
        tools=[show_todos],
    )
    for _ in range(1000)
]

elapsed = (
    time.perf_counter()
    - start
)

print(
    f"Built {len(agents)} agents "
    f"in {elapsed * 1000:.1f} ms"
)
```

และคำนวณ Microseconds ต่อ Agent.

สิ่งที่ Benchmark นี้วัด:

```text
Python object construction
Framework setup overhead
Tool registration overhead
```

สิ่งที่ไม่ได้วัด:

```text
Model latency
Tool execution
Network time
MCP startup
Reasoning quality
Token usage
End-to-end task completion
```

ดังนั้น:

```text
Fast agent creation
≠ Fast agent task
```

---

# 28. เมื่อ Agent Creation Speed สำคัญ

มีประโยชน์เมื่อ:

```text
สร้าง Agent ใหม่ต่อ Request
Spawn Workers จำนวนมาก
Ephemeral multi-agent teams
Serverless functions
High-concurrency runtime
```

ไม่มีผลมากนักเมื่อ:

```text
Agent ถูกสร้างครั้งเดียว
Model call ใช้เวลาหลายวินาที
Tool I/O เป็น Bottleneck หลัก
```

ในระบบ LLM ส่วนใหญ่ Model และ External Tools มักใช้เวลามากกว่า Agent Object Construction

---

# 29. AgentOS

Course อธิบายว่า Agent เดิมสามารถนำไปวางใน AgentOS ซึ่งเพิ่ม:

```text
FastAPI runtime
Sessions
Streaming
Tracing
Control-plane UI
```

โดยไม่ต้องเขียน Agent Logic ใหม่ทั้งหมด.

Mental model:

```text
Agno SDK
→ Agent logic

AgentOS
→ Serving and operations layer
```

สิ่งนี้คล้ายกับ:

```text
Google ADK
→ adk web / runtime services

LangChain
→ LangGraph runtime / LangSmith

MAF
→ Workflow engine
```

แต่ Product Boundary และ Deployment Model ต่างกัน

---

# 30. Agno Model Portability

สำหรับ OpenAI-compatible Endpoint Agno ใช้แนวคิด:

```python
from agno.models.openai.like import (
    OpenAILike,
)

model = OpenAILike(
    id="model-name",
    base_url="https://...",
    api_key="...",
)
```

Tools, Board, Workspace และ MCP Contract สามารถคงเดิม

```text
Change model adapter
→ Keep worker behavior contract
```

---

# 31. MAF vs Agno

| ประเด็น            | Microsoft Agent Framework               | Agno                               |
| ------------------ | --------------------------------------- | ---------------------------------- |
| Agent              | `Agent`                                 | `Agent`                            |
| Model adapter      | `OpenAIChatClient`                      | `OpenAIChat`                       |
| Instructions       | `instructions`                          | `instructions`                     |
| Run                | `await agent.run()`                     | `await agent.arun()`               |
| Final text         | `result.text`                           | `result.content`                   |
| Python tools       | Plain functions                         | Plain functions                    |
| Optional decorator | Customize tool                          | Extras เช่น caching                |
| MCP                | `MCPStdioTool`                          | `MCPTools`                         |
| MCP lifecycle      | `async with filesystem`                 | `async with MCPTools(...)`         |
| Enterprise feature | Durable workflow engine                 | AgentOS                            |
| จุดเด่น            | Workflows, Python/C# alignment          | Lean instantiation, serving layer  |
| Main caveat        | APIs เปลี่ยนเร็วและบางส่วน experimental | Internal MCP patch มี version risk |

---

# 32. เปรียบเทียบกับ Framework ก่อนหน้า

| Framework   | Agent creation   | Run            | MCP lifecycle             |
| ----------- | ---------------- | -------------- | ------------------------- |
| Google ADK  | `LlmAgent`       | Runner         | `McpToolset`              |
| Strands     | `Agent`          | `invoke_async` | Framework-managed client  |
| Pydantic AI | `Agent`          | `run`          | `async with agent`        |
| MAF         | `Agent` + client | `run`          | `async with MCPStdioTool` |
| Agno        | `Agent` + model  | `arun`         | `async with MCPTools`     |

ทั้งหมดใช้ Core Loop เดียวกัน:

```text
Model
→ Tool call
→ Tool result
→ Model
→ Stop
```

---

# 33. MAF Workflow Engine vs Agno AgentOS

สองสิ่งนี้ไม่ใช่คู่แข่งตรงกันทั้งหมด

## MAF Workflow Engine

เน้น:

```text
Control flow
Typed nodes
Edges
Durability
Checkpoints
Human approval
```

คำถามหลัก:

> งานควรเดินผ่านขั้นตอนใดบ้าง?

## AgentOS

เน้น:

```text
Serving
Sessions
Streaming
Tracing
Operations UI
```

คำถามหลัก:

> จะนำ Agent นี้ไปให้บริการและดูแลอย่างไร?

สรุป:

```text
MAF Workflow
= Orchestration layer

AgentOS
= Serving and operations layer
```

Agno เองก็มี Workflow และ Team Abstractions แต่ “จุดเด่นของ Lab” คือ Lightweight Agent กับ AgentOS

---

# 34. Shared State Surfaces

ทั้งสอง Worker มี State อย่างน้อยสามส่วน:

```text
Framework execution state
→ Messages และ Tool Loop

SQLite Board
→ Goal, Steps, Status, Result

Filesystem
→ notes.txt และ spanish.txt
```

ใน Shared-board Mode ยังมี:

```text
Orchestrator process
Worker subprocess
Shared board file
Shared working directory
```

State เหล่านี้อาจไม่ตรงกัน เช่น:

```text
Agent says complete
แต่ Board ยัง in_progress

Board says done
แต่ spanish.txt ไม่มี

File exists
แต่เนื้อหาไม่ถูกต้อง
```

---

# 35. Shared Weakness: `complete_task()`

ทั้งสอง Framework ใช้ Tool:

```python
def complete_task(
    task_id: int,
    result: str,
) -> dict:

    board.complete_todo(
        task_id,
        result,
    )

    return {
        "task_id": task_id,
        "status": "done",
    }
```

มันไม่ได้ตรวจว่า:

```text
Task ID มีอยู่จริง
Task เป็นของ Worker นี้
Goal มี Steps ค้างหรือไม่
spanish.txt มีอยู่หรือไม่
File ไม่ว่างหรือไม่
Translation ถูกต้องหรือไม่
```

ดังนั้น MAF และ Agno สามารถปิด Goal ผิดได้เหมือน Framework ก่อนหน้า

---

# 36. Shared Weakness: Task Claim

```python
board.claim_todo(
    TASK_ID
)
```

ยังไม่เป็น Atomic Business Claim

Workers สองตัวอาจทำ Task เดียวกันพร้อมกัน

Safer Query:

```sql
UPDATE todos
SET status = 'in_progress'
WHERE id = ?
  AND status = 'pending'
```

แล้วตรวจ:

```text
affected rows == 1
```

Framework ไม่สามารถแก้ Business-level Concurrency ที่ Board ไม่ได้บังคับ

---

# 37. Shared Weakness: MCP Workspace

Filesystem Server จำกัด Root ไปยัง `WORK_DIR`

ช่วย:

```text
ลดพื้นที่เข้าถึง
แยก Workspace
```

แต่ Agent ยังอาจ:

```text
เขียนทับ notes.txt
ลบ Artifact
สร้างไฟล์ผิด
ทำตาม Prompt Injection ที่อยู่ในไฟล์
```

Root Restriction ไม่ใช่ Full Security Sandbox

---

# 38. เมื่อใดควรเลือก MAF

เหมาะเมื่อ:

```text
องค์กรใช้ Microsoft/.NET
ต้องการ Python และ C# concepts ที่ใกล้กัน
ต้องการ Durable Workflows
มี Long-running processes
ต้องมี Checkpoint และ Resume
ต้องการ Human approval ใน Workflow
```

จุดแข็ง:

```text
Enterprise-oriented
Graph workflow engine
Typed workflow structure
OpenAI client integration
Cross-language consistency
```

จุดที่ต้องระวัง:

```text
API evolution เร็ว
Old AutoGen/Semantic Kernel tutorials อาจไม่ตรง
Experimental surfaces
Infrastructure complexity สูงกว่า Plain Agent
```

---

# 39. เมื่อใดควรเลือก Agno

เหมาะเมื่อ:

```text
ต้องการ Agent API ที่เบา
สร้าง Agents จำนวนมาก
ต้องการ FastAPI serving path
ต้องการ AgentOS sessions และ tracing
ต้องการเริ่มต้นเร็ว
```

จุดแข็ง:

```text
Agent construction เบา
Plain function tools
AgentOS integration
Model portability
API กระชับ
```

จุดที่ต้องระวัง:

```text
Fast instantiation อาจไม่ใช่ Bottleneck จริง
Version 2 ต่างจาก Phidata-era APIs
MCP workaround ผูกกับ Internal Module
Business validation ยังต้องสร้างเอง
```

---

# 40. Misconceptions

### “Microsoft Agent Framework คือ AutoGen เปลี่ยนชื่อ”

ไม่ตรงทั้งหมด มันเป็น Framework ใหม่ที่รวมแนวคิดและทิศทางจากหลายโครงการของ Microsoft

### “MAF Plain Agent เป็น Durable Workflow โดยอัตโนมัติ”

ไม่ใช่ ต้องใช้ Workflow Engine และ Persistent Services ที่เหมาะสม

### “MAF ต้องใช้ Azure OpenAI เท่านั้น”

ไม่จริง Course ใช้ `OpenAIChatClient` กับ OpenAI Model

### “Agno เร็วกว่าเพราะ Model ตอบเร็วกว่า”

Benchmark ใน Lab วัด Agent Object Construction ไม่ได้วัด Model Response

### “Agno AgentOS ทำให้ Agent ถูกต้อง”

ไม่จริง AgentOS ช่วย Serving และ Operations ไม่ใช่ Factual Validation

### “Plain Typed Functions ปลอดภัยโดยอัตโนมัติ”

ไม่จริง Type Schema ตรวจรูปแบบ แต่ไม่ตรวจ Business Authority

### “MCP ใช้เหมือนกันทุก Framework”

Protocol เหมือนกัน แต่ Connection Lifecycle และ Error Handling ต่างกัน

---

# 41. Exercises

## Exercise 1 — MAF Workflow

สร้าง Workflow:

```text
Read board
→ Worker agent
→ Artifact validator
→ Close goal
```

ให้ Validator ปฏิเสธการปิด Goal หาก `spanish.txt` ไม่มี

---

## Exercise 2 — Agno Instantiation vs Execution

วัดแยก:

```text
Agent construction
First model call
MCP startup
Full task runtime
```

เพื่อหา Bottleneck จริง

---

## Exercise 3 — Shared Goal Validator

สร้าง Function เดียวที่ทั้ง MAF และ Agno ใช้:

```python
def validate_goal(
    goal_id: int,
) -> dict:
    ...
```

ตรวจ:

```text
Steps ครบ
File มี
File ไม่ว่าง
Goal ownership ถูกต้อง
```

---

## Exercise 4 — MCP Failure

ทำให้ `npx` ใช้งานไม่ได้ แล้วเปรียบเทียบ:

```text
MAF error
Agno error
Cleanup behavior
Error visibility
```

---

## Exercise 5 — Multi-worker Claim

รัน:

```text
maf_worker
และ
agno_worker
```

พร้อมกันกับ Task ID เดียวกัน แล้วตรวจ Duplicate Execution

จากนั้นแก้ `claim_todo()` ให้ Atomic

---

# 42. Checklist

### Lab 3 มี Framework ใด

```text
Microsoft Agent Framework
Agno
```

### MAF Model Adapter คืออะไร

```python
OpenAIChatClient(
    model=MODEL
)
```

### MAF รันอย่างไร

```python
await agent.run(...)
```

### MAF Final Text อยู่ที่ไหน

```python
result.text
```

### MAF MCP คืออะไร

```python
MCPStdioTool(...)
```

### Agno Model Adapter คืออะไร

```python
OpenAIChat(
    id=MODEL
)
```

### Agno รันอย่างไร

```python
await agent.arun(
    input=...
)
```

### Agno Final Content อยู่ที่ไหน

```python
result.content
```

### Agno MCP คืออะไร

```python
MCPTools(...)
```

### Both Frameworks ใช้ Tool แบบใด

```text
Plain typed Python functions
```

### Goal Loop คืออะไร

```text
Read
→ Plan
→ Act
→ Complete
→ Repeat
```

---

# แก่นของ Week 5 Lab 3

```text
Microsoft Agent Framework
= Agent SDK + durable workflow direction

Agno
= Lightweight agent SDK + AgentOS path

SQLite Board
= Shared application contract

MCP
= Shared external tool protocol

Worker Scripts
= Same task, different framework
```

บทเรียนสำคัญที่สุดคือ:

> **MAF และ Agno ต่างกันที่ Architecture Emphasis มากกว่าพื้นฐานของ Agent Loop—MAF ให้ความสำคัญกับ Durable Enterprise Workflows ส่วน Agno ให้ความสำคัญกับ Lightweight Agents และเส้นทางนำไป Serve ผ่าน AgentOS**

อีกบทเรียนคือ:

> **Benchmark ของ Framework ต้องวัดให้ตรงสิ่งที่สนใจ การสร้าง Agent ได้เร็วอาจมีประโยชน์ แต่ไม่ได้บอกว่า Model ตอบเร็ว งานถูกต้อง หรือ Agent ใช้ Tools ได้มีประสิทธิภาพ**

และข้อสรุปที่ยังเหมือนทุก Lab:

> **Framework ทำให้ Model–Tool Loop สะดวกขึ้น แต่ Application ยังคงต้องบังคับ Task Ownership, Artifact Validation, Permissions และ Definition of Done ด้วย Code**
