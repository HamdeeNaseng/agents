# Episodic Learning Artifact

## Week 5 — Lab 3: Microsoft Agent Framework and Agno

**หลักสูตร:** AI Engineer Agentic Track
**ผู้สอน:** Ed Donner
**Repository:** `ed-donner/agents`
**โฟลเดอร์:** `5_agent_frameworks/3_maf_agno`
**Notebooks:** `maf_lab.ipynb`, `agno_lab.ipynb`
**ไฟล์ประกอบ:** `maf_worker.py`, `agno_worker.py`, `board.py`, `workspace/`
**หัวข้อหลัก:** Microsoft Agent Framework, Agno, Plain Function Tools, MCP Lifecycle, Durable Workflows, AgentOS และ Framework Benchmarking
**สถานะ:** เรียนและสรุป Lab 3 แล้ว

---

# 1. Context

Week 5 ใช้ Business Contract เดิมกับ Agent Framework หลายตัว เพื่อแยกให้เห็นว่า Framework เปลี่ยน แต่ Pattern ของ Agent และความรับผิดชอบของ Application ยังคงเดิม

Contract กลาง:

```text
SQLite Board มี Goal หนึ่งรายการ
        ↓
Worker อ่าน Goal
        ↓
Worker สร้าง Steps
        ↓
Worker อ่าน notes.txt
        ↓
Worker แปลข้อความเป็นภาษาสเปน
        ↓
Worker เขียน spanish.txt
        ↓
Workerปิด Steps
        ↓
Workerปิด Goal
```

Framework ที่เรียนก่อนหน้า:

```text
Lab 1
Google ADK

Lab 2
AWS Strands
Pydantic AI
```

Lab 3 เพิ่ม:

```text
Microsoft Agent Framework
Agno
```

สิ่งที่คงเดิม:

```text
Goal
SQLite schema
Board functions
Workspace
notes.txt
spanish.txt
Filesystem MCP server
Worker instructions
```

สิ่งที่เปลี่ยน:

```text
Agent constructor
Model adapter
Run method
Result object
Tool registration
MCP lifecycle
Production runtime direction
Observability
```

---

# 2. Learning Objectives

หลังจบ Lab 3 สามารถอธิบายได้ว่า:

1. Microsoft Agent Framework สร้าง Agent อย่างไร
2. `OpenAIChatClient` แยกจาก `Agent` อย่างไร
3. MAF Plain Agent ต่างจาก Workflow Engine อย่างไร
4. MAF Plain Python Functions กลายเป็น Tools อย่างไร
5. `MCPStdioTool` เชื่อม Filesystem MCP Server อย่างไร
6. ทำไม MAF MCP ต้องใช้ `async with`
7. MAF Workflow Engine รองรับงานระยะยาวอย่างไร
8. Agno สร้าง Agent จาก `OpenAIChat` อย่างไร
9. `arun()` และ `result.content` ใช้อย่างไร
10. Agno Plain Python Functions กลายเป็น Tools อย่างไร
11. Agno `MCPTools` จัด Connection Lifecycle อย่างไร
12. ทำไม Course ใช้ Async ตลอด Agno Notebook
13. Agent Instantiation Benchmark วัดอะไร
14. Agent Instantiation Benchmark ไม่ได้วัดอะไร
15. AgentOS เพิ่มอะไรจาก Agno SDK
16. MAF และ Agno รองรับ Shared-board Worker Mode อย่างไร
17. State ของ Framework, Board และ Filesystem ต่างกันอย่างไร
18. Business Risks ใดยังคงเหมือนเดิมเมื่อเปลี่ยน Framework
19. เมื่อใดควรเลือก MAF
20. เมื่อใดควรเลือก Agno

---

# 3. Prerequisites

ควรเข้าใจแนวคิดจาก Labs ก่อนหน้า:

```text
Agent definition
Model adapter
Model–tool loop
Typed function tool
MCP
Async context manager
SQLite board
Goal and steps
External artifacts
Shared worker contract
```

Environment:

```text
Python >= 3.12
Node.js
npx
SQLite
OpenAI API key
```

Environment Variable:

```env
OPENAI_API_KEY=...
```

Dependencies หลัก:

```text
agent-framework-core
agent-framework-openai
agno[mcp]
mcp
```

ติดตั้ง:

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

---

# 4. Shared Five-step Pattern

ทั้งสอง Framework ถูกเรียนผ่าน Pattern เดิม:

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
→ Text response
```

ยังเป็น LLM Call เป็นหลัก

ขั้นที่ 3–4:

```text
Model
→ Tool request
→ Environment
→ Tool result
→ Model
```

ขั้นที่ 5:

```text
Goal
→ Plan
→ Act
→ Observe
→ Update state
→ Repeat
→ Completion
```

---

# Part A — Microsoft Agent Framework

# 5. What Is Microsoft Agent Framework?

Microsoft Agent Framework หรือ MAF เป็น Agent SDK ของ Microsoft ที่รวมแนวคิดจากสายงานเดิม เช่น:

```text
AutoGen
Semantic Kernel
```

เข้าสู่ Framework และ API Direction เดียว

MAF มีสองระดับสำคัญ:

```text
Plain Agent
→ Model–Tool Loop แบบทั่วไป

Workflow Engine
→ Graph-based process สำหรับงานระยะยาว
```

Mental Model:

```text
Plain Agent
= Worker ที่ Model เลือก Tools เอง

Workflow Engine
= Application กำหนด Nodes, Edges และ Gates
```

---

# 6. MAF Imports

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

Course ซ่อน Experimental Warnings เพื่อให้ Notebook อ่านง่าย

แต่:

```text
Warning hidden
≠ Feature stable
```

การปิด Warning ไม่ควรถูกใช้แทนการตรวจ Version และ Release Notes

---

# 7. Step 1 — Create a MAF Agent

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

Mental Model:

```text
Client
= ช่องทางเชื่อม Model Provider

Agent
= Worker ที่ใช้ Client นั้นตัดสินใจ
```

---

# 8. Agent and Client Separation

การแยก Client ออกจาก Agent ช่วยให้:

```text
Reuse Model Client
เปลี่ยน Agent Instructions
สร้าง Agents หลายตัว
รวม Provider Settings ไว้ที่เดียว
```

ตัวอย่าง:

```python
researcher = Agent(
    client=client,
    instructions="Research the topic."
)

writer = Agent(
    client=client,
    instructions="Write the report."
)
```

Agents สองตัวใช้ Model Client เดียวกันได้

---

# 9. Step 2 — Run a MAF Agent

```python
result = await agent.run(
    "Say hello in Spanish."
)

print(result.text)
```

Final Output อยู่ที่:

```python
result.text
```

Flow ตอนยังไม่มี Tools:

```text
User input
→ Agent
→ OpenAIChatClient
→ Model
→ Text result
```

ยังไม่มี Action–Observation Loop หลายรอบ

ดังนั้น:

```text
Agent without tools
≈ LLM call with agent configuration
```

---

# 10. MAF Is Async-first

Course ใช้:

```python
await agent.run(...)
```

ตลอด Notebook

ข้อดีของ Async:

```text
รองรับ Network I/O
รองรับ MCP process
รองรับ Concurrent executions
ไม่ Block Event Loop
```

แต่:

```text
Async
≠ Better reasoning
```

Async เป็นเรื่อง Runtime Efficiency ไม่ใช่ Model Intelligence

---

# 11. MAF vs Google ADK Runtime

Google ADK แยก:

```text
LlmAgent
+
Runner
```

ตัวอย่าง:

```python
runner = InMemoryRunner(
    agent=agent
)
```

MAF Plain Agent:

```python
await agent.run(...)
```

Agent Object จัดการการรันได้โดยตรง

เปรียบเทียบ:

```text
Google ADK
→ Runtime services ถูกเปิดเผยชัด

MAF Plain Agent
→ Runtime surface ถูกซ่อนใน Agent มากกว่า
```

---

# 12. Step 3 — MAF Function Tools

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

---

# 13. MAF Tool Schema

Framework อ่าน:

```text
Function name
Parameter names
Type hints
Docstring
Return value
```

เพื่อสร้าง JSON Schema

ตัวอย่าง Concept:

```text
plan_steps

goal_id:
integer

steps:
array[string]
```

Model ใช้ Schema เพื่อ:

```text
เลือก Tool
สร้าง Arguments
เข้าใจผลลัพธ์ที่คาดหวัง
```

---

# 14. Optional Tool Decorator

MAF มี Decorator สำหรับกรณีที่ต้องการ:

```text
เปลี่ยนชื่อ Tool
เพิ่ม Description
จำกัด Tool behavior
เพิ่ม Metadata
```

แต่ Basic Tools ใช้ Plain Function ได้

Course จึงแสดงว่า:

```text
Typed Python function
→ Agent tool
```

ได้โดยไม่ต้องมี Decorator ทุก Function

---

# 15. Add MAF Tools

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
Need live board data
→ call show_todos()
→ receive todos
→ answer user
```

นี่คือ Agent Loop เริ่มต้น:

```text
Decide
→ Act
→ Observe
→ Respond
```

---

# 16. Step 4 — MAF MCP

สร้าง Filesystem MCP Tool:

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

เปิด Lifecycle:

```python
async with filesystem:

    file_agent = Agent(
        client=client,
        instructions=(
            "Use your file tools."
        ),
        tools=[
            filesystem
        ],
    )

    result = await file_agent.run(
        "Read notes.txt and summarize it."
    )
```

MAF ใส่ MCP Tool Object ใน `tools=[...]` Surface เดียวกับ Python Functions

---

# 17. MAF MCP Architecture

```text
MAF Agent
    ↓
MCPStdioTool
    ↓ stdio
Node.js Filesystem MCP Server
    ↓
Workspace files
```

MCP Server อาจเปิด Tools เช่น:

```text
List files
Read file
Write file
Edit file
Create directory
```

Tool Names จริงขึ้นกับ MCP Server Version

---

# 18. MAF MCP Lifecycle

```text
async with filesystem
        ↓
Start child process
        ↓
Initialize MCP connection
        ↓
Discover tools
        ↓
Agent uses tools
        ↓
Close connection
        ↓
Stop process
```

หาก Lifecycle ไม่ถูกจัดการ อาจเกิด:

```text
MCP tools unavailable
Child process leak
Notebook hangs
Shutdown error
```

---

# 19. Notebook-specific MCP Wrapper

Notebook มีการปรับ MCP Tool เพื่อ:

```text
กำหนด cwd
ส่ง stderr ไป DEVNULL
ลด Startup Banner
แก้ Jupyter Windows issue
```

ข้อดี:

```text
Notebook output สะอาด
MCP process เริ่มได้
```

ข้อเสีย:

```text
Error details ถูกซ่อน
Root cause หาย
Debugging ยาก
```

Production ควรส่ง Error ไป Structured Logs มากกว่า Null Device

---

# 20. Step 5 — MAF Goal Loop

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
→ Find Goal
→ plan_steps
→ Read notes.txt
→ Translate
→ Write spanish.txt
→ Complete Step
→ Repeat
→ Complete Goal
```

---

# 21. MAF Internal Agent Loop

```text
Model receives assignment
        ↓
Model calls show_todos
        ↓
Board result
        ↓
Model calls plan_steps
        ↓
Step IDs
        ↓
Model uses filesystem
        ↓
File observations
        ↓
Model updates board
        ↓
Model decides to stop
```

Application ไม่ต้องเขียน Manual `while` Loop

Framework จัดการ:

```text
Model call
Tool execution
Tool result
Next model call
Termination
```

---

# 22. MAF Worker Script

Standalone:

```powershell
uv run maf_worker.py
```

Flow:

```text
Reset board
→ Clear spanish.txt
→ Add Goal
→ Claim Goal
→ Start MCP
→ Build Agent
→ Run
→ Show Board
→ Show File
```

Shared-board Mode:

```powershell
uv run maf_worker.py `
  <taskId> `
  <boardPath>
```

ใน Mode นี้ Worker:

```text
ใช้ Shared Board
ใช้ Shared Workspace
Claim Task ที่ได้รับ
ไม่ Reset Tasks อื่น
ทำ Task เดียว
ปิด Task แล้วหยุด
```

---

# 23. MAF Workflow Engine

Plain Agent เหมาะกับ Model-driven Loop

แต่ MAF ยังมี Workflow Engine สำหรับงานที่ Application ต้องควบคุมเส้นทาง

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
Agents
Functions
Validators
Human approval steps
```

---

# 24. Agent vs Workflow

## Plain Agent

```text
Model เลือกว่า:
ใช้ Tool ใด
ทำอะไรต่อ
หยุดเมื่อใด
```

## Workflow

```text
Application กำหนด:
Stage ใดมาก่อน
Stage ใดตามหลัง
Validation อยู่ตรงไหน
Approval อยู่ตรงไหน
```

Mental Model:

```text
Agent
= Flexible worker

Workflow
= Controlled production line
```

---

# 25. Durable Workflow Capabilities

Workflow Engine เหมาะกับ:

```text
Long-running tasks
Checkpointing
Restart after failure
Streaming
Human-in-the-loop
Typed node inputs
Typed node outputs
```

ตัวอย่าง:

```text
Research
→ Draft
→ Validate
→ Human approve
→ Publish
```

หาก Process ปิดกลางงาน Workflow ที่ Durable ควรสามารถกลับมาทำต่อจาก Checkpoint ได้

---

# 26. Relationship to LangGraph

MAF Workflow Engine มี Mental Model คล้าย LangGraph:

```text
Nodes
Edges
State
Checkpoint
Resume
Human approval
```

ความต่างอยู่ที่:

```text
API design
Microsoft ecosystem
Python/C# alignment
Runtime implementation
Integration surface
```

---

# 27. Python and C# Alignment

MAF เน้น API และ Mental Model ที่ใกล้กันระหว่าง:

```text
Python
C#
```

ประโยชน์ต่อองค์กร:

```text
หลายทีมใช้แนวคิดเดียวกัน
แชร์ Architecture Pattern
ลดความต่างด้าน Framework Vocabulary
เชื่อมระบบ .NET กับ Python ได้ง่ายขึ้น
```

คุณค่านี้อาจสำคัญกว่า Code ที่สั้นลงเล็กน้อยใน Enterprise Environment

---

# Part B — Agno

# 28. What Is Agno?

Agno เป็น Agent Framework ที่เน้น:

```text
Lightweight Agent objects
Fast instantiation
Simple tools
Production serving ผ่าน AgentOS
```

Mental Model:

```text
Agno Agent
= Model
+ Instructions
+ Tools
+ Lightweight runtime
```

Agno เคยเป็นที่รู้จักในชื่อ Phidata มาก่อน Version 2 มี API และ Product Direction ที่เปลี่ยนไปมาก

---

# 29. Agno Imports

```python
from agno.agent import Agent

from agno.models.openai import (
    OpenAIChat,
)

from agno.tools.mcp import (
    MCPTools,
)

from mcp import (
    StdioServerParameters,
)
```

หน้าที่:

```text
Agent
→ Agent loop

OpenAIChat
→ Model adapter

MCPTools
→ External tool collection
```

---

# 30. Step 1 — Create an Agno Agent

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
→ Instructions, tools และ loop
```

---

# 31. Step 2 — Run Agno

```python
result = await agent.arun(
    input="Say hello in Spanish."
)

print(result.content)
```

Final Output อยู่ที่:

```python
result.content
```

ตอนยังไม่มี Tools:

```text
Input
→ Model
→ Content
```

ยังเป็น LLM Call เป็นหลัก

---

# 32. Why Use Async Throughout Agno Lab?

Agno มี Synchronous Path

แต่ Course ใช้:

```python
await agent.arun(...)
```

ตั้งแต่ต้น เพราะเมื่อเพิ่ม MCP:

```text
MCPTools
→ Async lifecycle
```

การใช้ Async ตั้งแต่ต้นช่วยให้ Notebook มีรูปแบบเดียวกันทุก Step

```text
Without MCP
→ Sync or async

With MCP
→ Async
```

---

# 33. Step 3 — Agno Function Tools

Agno รับ Plain Typed Functions:

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

---

# 34. Agno Tool Schema

Agno อ่าน:

```text
Function name
Type hints
Parameters
Docstring
```

เพื่อสร้าง Tool Schema

Optional Decorator ใช้เมื่อต้องการความสามารถเพิ่ม เช่น:

```text
Caching
Custom name
Metadata
Hooks
```

Basic Tool ไม่ต้องมี Decorator

---

# 35. Add Agno Tools

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

เรียก:

```python
result = await board_agent.arun(
    input=(
        "What is on the board "
        "right now?"
    )
)
```

Agent อาจ:

```text
Recognize need for current data
→ Call show_todos
→ Read result
→ Respond
```

---

# 36. Step 4 — Agno MCP

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

เปิด MCP Tools:

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

Agno ใส่ MCP Tool Collection ลงใน:

```python
tools=[filesystem]
```

เหมือน Function Tools

---

# 37. Agno MCP Architecture

```text
Agno Agent
    ↓
MCPTools
    ↓ stdio
Node Filesystem MCP Server
    ↓
Workspace
```

MCP Tools เป็น Async Context Manager ซึ่งควบคุม:

```text
Process startup
Connection
Tool discovery
Execution
Cleanup
```

---

# 38. Agno MCP Lifecycle

```text
Create StdioServerParameters
        ↓
Enter MCPTools context
        ↓
Start process
        ↓
Initialize tools
        ↓
Create Agent
        ↓
Run Agent
        ↓
Exit context
        ↓
Stop process
```

หาก Lifecycle ไม่ถูกต้อง:

```text
MCP tools may be unavailable
Child process may remain
Notebook may hang
Cleanup may fail
```

---

# 39. Agno Windows/Jupyter MCP Patch

Course Patch:

```python
import functools
import subprocess

import agno.tools.mcp.mcp as agno_mcp

agno_mcp.stdio_client = functools.partial(
    agno_mcp.stdio_client,
    errlog=subprocess.DEVNULL,
)
```

จุดประสงค์:

```text
ซ่อน Startup Banner
แก้ปัญหา stderr บน Jupyter Windows
```

แต่ Patch นี้แก้ Internal Module-level Function

ความเสี่ยง:

```text
มีผลทั้ง Process
ผูกกับ Internal API
Library update อาจทำให้พัง
Error logs ถูกซ่อน
```

จึงเป็น Compatibility Workaround ไม่ใช่ Production Pattern

---

# 40. Step 5 — Agno Goal Loop

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
show_todos
→ Find Goal
→ plan_steps
→ Read notes.txt
→ Translate
→ Write spanish.txt
→ Complete Steps
→ Complete Goal
```

---

# 41. Agno Internal Agent Loop

```text
Model reads assignment
        ↓
Model calls board tool
        ↓
Board result
        ↓
Model plans steps
        ↓
Model calls MCP tools
        ↓
File observations
        ↓
Model completes todos
        ↓
Model stops
```

Framework จัด Tool Loop ภายใน `arun()`

---

# 42. Agno Worker Script

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

Worker รองรับ Environment:

```text
WORKER_MODEL
→ Model ID

BOARD_PATH
→ Shared Board

WORK_DIR
→ Shared Workspace
```

ใน Shared-board Mode Worker จะทำเฉพาะ Task ที่ได้รับแล้วหยุด

---

# 43. Agent Instantiation Benchmark

Notebook สร้าง Agent 1,000 ตัว:

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
```

แล้วคำนวณเวลาเฉลี่ยต่อ Agent

---

# 44. What the Benchmark Measures

Benchmark นี้วัด:

```text
Python object construction
Tool registration
Framework initialization overhead
```

ไม่ได้วัด:

```text
Model response latency
MCP startup time
Tool execution time
Token usage
Reasoning quality
Task correctness
End-to-end runtime
```

ดังนั้น:

```text
Fast Agent construction
≠ Fast task completion
```

---

# 45. When Agent Creation Speed Matters

สำคัญเมื่อ:

```text
สร้าง Agent ต่อ Request
สร้าง Ephemeral Workers
Spawn Agents จำนวนมาก
ใช้ Serverless
สร้าง Dynamic Teams
```

สำคัญน้อยเมื่อ:

```text
Agent ถูกสร้างครั้งเดียว
Model call ใช้เวลาหลายวินาที
External I/O เป็น Bottleneck
MCP startup ช้ากว่า object construction
```

---

# 46. Correct Benchmarking Layers

ควรวัดแยก:

```text
Agent construction time
First model call latency
MCP initialization time
Tool execution time
Full task duration
Total model calls
Total tokens
Artifact correctness
```

เพื่อไม่สรุปผิดจาก Metric เดียว

---

# 47. AgentOS

AgentOS เป็น Serving และ Operations Layer ของ Agno

เพิ่ม:

```text
FastAPI runtime
Sessions
Streaming
Tracing
Control-plane UI
```

Mental Model:

```text
Agno SDK
→ Agent logic

AgentOS
→ Deployment and operations
```

Agent Logic เดิมสามารถถูกนำไป Serve โดยไม่ต้องเขียนใหม่ทั้งหมด

---

# 48. AgentOS Is Not Correctness

AgentOS ช่วย:

```text
Serve Agents
Manage Sessions
Observe Runs
Stream Results
Operate Deployments
```

แต่ไม่ได้พิสูจน์ว่า:

```text
Agent วางแผนถูก
Tool ถูกเลือกถูก
Artifact ถูกต้อง
Board State ถูกต้อง
```

Serving Layer และ Validation Layer เป็นคนละส่วน

---

# 49. Agno Model Portability

OpenAI-compatible Model:

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

สิ่งที่ไม่เปลี่ยน:

```text
Tools
Board
Workspace
MCP
Instructions
Worker contract
```

---

# 50. MAF vs Agno

| ประเด็น          | Microsoft Agent Framework               | Agno                                      |
| ---------------- | --------------------------------------- | ----------------------------------------- |
| Agent            | `Agent`                                 | `Agent`                                   |
| Model adapter    | `OpenAIChatClient`                      | `OpenAIChat`                              |
| Prompt           | `instructions`                          | `instructions`                            |
| Run              | `agent.run()`                           | `agent.arun()`                            |
| Final output     | `result.text`                           | `result.content`                          |
| Python tools     | Plain functions                         | Plain functions                           |
| MCP              | `MCPStdioTool`                          | `MCPTools`                                |
| MCP lifecycle    | `async with filesystem`                 | `async with MCPTools(...)`                |
| Main direction   | Durable enterprise workflow             | Lightweight agent and AgentOS             |
| Notable strength | Workflow engine, Python/C# alignment    | Fast construction, deployment path        |
| Main caveat      | API evolution and experimental surfaces | Internal MCP workaround and metric misuse |

---

# 51. MAF Workflow vs Agno AgentOS

สองสิ่งนี้แก้ปัญหาคนละประเภท

## MAF Workflow Engine

ตอบคำถามว่า:

```text
งานควรไหลผ่านขั้นตอนใด?
```

เน้น:

```text
Nodes
Edges
Checkpoints
Durability
Human approval
```

## AgentOS

ตอบคำถามว่า:

```text
จะ Serve และดูแล Agent อย่างไร?
```

เน้น:

```text
API serving
Sessions
Streaming
Tracing
Operations UI
```

สรุป:

```text
MAF Workflow
= Orchestration layer

AgentOS
= Serving and operations layer
```

---

# 52. Comparison with Previous Frameworks

| Framework   | Model configuration | Run method       | MCP            |
| ----------- | ------------------- | ---------------- | -------------- |
| Google ADK  | `LlmAgent` + Runner | Runner methods   | `McpToolset`   |
| Strands     | `OpenAIModel`       | `invoke_async()` | `MCPClient`    |
| Pydantic AI | Model string/object | `run()`          | `MCPToolset`   |
| MAF         | `OpenAIChatClient`  | `run()`          | `MCPStdioTool` |
| Agno        | `OpenAIChat`        | `arun()`         | `MCPTools`     |

Core Pattern:

```text
Model
→ Tool request
→ Tool result
→ Model
→ Completion
```

ยังเหมือนกันทั้งหมด

---

# 53. Shared Worker Contract

ทั้ง MAF และ Agno Worker รองรับ:

```text
Standalone Mode
Shared-board Mode
```

Standalone:

```text
Worker owns:
Board reset
Goal seed
Workspace cleanup
Execution
Output display
```

Shared-board:

```text
Orchestrator owns:
Board
Task assignment
Workspace
Console

Worker owns:
One task
Its steps
Its artifacts
Completion
```

---

# 54. Import-time Board Configuration

ก่อน Import `board` Worker ตั้ง:

```python
os.environ.setdefault(
    "BOARD_PATH",
    sys.argv[2],
)
```

เพราะ Board Module อ่าน Path ตอน Import

หลัก:

```text
Configuration needed by module initialization
ต้องถูกตั้งก่อน import
```

หากตั้งภายหลัง Worker อาจใช้ Board ผิดไฟล์

---

# 55. State Surfaces

Lab มี State อย่างน้อย:

```text
Framework execution state
SQLite board state
Filesystem state
Subprocess state
MCP process state
```

State เหล่านี้อาจไม่ตรงกัน

ตัวอย่าง:

```text
Agent says complete
แต่ Goal ยัง in_progress

Goal says done
แต่ spanish.txt ไม่มี

File exists
แต่ Translation ผิด

MCP process failed
แต่ Worker ยังรายงานคำตอบ
```

---

# 56. Shared Weakness — `complete_task()`

ทั้งสองใช้:

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

ไม่ได้ตรวจ:

```text
Task ID ถูกต้อง
Task เป็นของ Worker นี้
Steps ทั้งหมดเสร็จ
spanish.txt มีอยู่
File ไม่ว่าง
Translation ถูกต้อง
```

ดังนั้น Agent สามารถปิด Goal เร็วเกินไปได้

---

# 57. Prompt vs Code Invariants

Prompt บอกว่า:

```text
สร้าง Steps
ทำ Steps ให้ครบ
ปิด Goal เมื่อเสร็จ
```

แต่ Prompt เป็น Guidance

Code ควรบังคับ:

```text
ห้ามปิด Goal หาก Step ค้าง
ห้ามปิด Goal หาก Artifact ไม่มี
ห้าม Claim Task ซ้ำ
ห้ามแก้ไฟล์นอก Workspace
```

หลัก:

```text
LLM handles ambiguity

Code enforces invariants
```

---

# 58. Duplicate Task Claim

Worker ใช้:

```python
board.claim_todo(
    TASK_ID
)
```

ถ้ามี Workers สองตัว:

```text
MAF Worker
Agno Worker
```

Claim Task เดียวกันพร้อมกัน ทั้งสองอาจทำงานซ้ำ

Safer SQL:

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

---

# 59. Database Concurrency vs Work Ownership

SQLite WAL ช่วย:

```text
Read/write concurrency
ลด database locked errors
```

แต่ไม่ช่วย:

```text
Worker ownership
Duplicate task prevention
Lease timeout
Retry ownership
```

Application ต้องออกแบบ Work Coordination เอง

---

# 60. Workspace Security

Filesystem MCP ถูก Scope ไปยัง `WORK_DIR`

ข้อดี:

```text
จำกัดพื้นที่เข้าถึง
แยกไฟล์ของงาน
ลด accidental access
```

แต่ภายใน Workspace Agent ยังอาจ:

```text
เขียนทับ notes.txt
ลบ Artifact
สร้างไฟล์ผิด
ทำตาม Prompt Injection ในไฟล์
```

ดังนั้น:

```text
Workspace scope
≠ Full sandbox
```

---

# 61. MCP Error Visibility

ทั้งสอง Notebook มี Workaround ที่ซ่อน `stderr`

ผล:

```text
Notebook ใช้งานง่าย
Output สะอาด
```

แต่:

```text
Server startup error ถูกซ่อน
Transport error หาย
Root cause diagnosis ยาก
```

Production ควรมี:

```text
Structured MCP logs
Process exit codes
Timeouts
Health checks
Error categories
```

---

# 62. When to Choose MAF

เหมาะเมื่อ:

```text
องค์กรใช้ Microsoft stack
มี Python และ .NET teams
ต้องการ Durable workflows
งานต้องรอดจาก Process restart
มี Human approval
มี Long-running business process
```

จุดแข็ง:

```text
Workflow engine
Typed graph direction
Checkpoint and resume
Enterprise ecosystem
Cross-language consistency
```

จุดที่ต้องระวัง:

```text
API เปลี่ยนเร็ว
Old AutoGen examples อาจไม่ตรง
Experimental features
Production setup ซับซ้อนกว่า Plain Agent
```

---

# 63. When to Choose Agno

เหมาะเมื่อ:

```text
ต้องการ API กระชับ
สร้าง Agent จำนวนมาก
ต้องการ FastAPI serving path
ต้องการ Sessions และ Tracing ผ่าน AgentOS
ต้องการเริ่มต้นเร็ว
```

จุดแข็ง:

```text
Lightweight objects
Plain function tools
Simple Agent API
AgentOS deployment path
Model portability
```

จุดที่ต้องระวัง:

```text
Fast construction อาจไม่ใช่ Bottleneck
Version 2 ต่างจาก Phidata examples
Internal MCP patch เปราะบาง
Business validation ยังต้องเขียนเอง
```

---

# 64. Misconceptions Corrected

## ความเข้าใจคลาดเคลื่อนที่ 1

> Microsoft Agent Framework คือ AutoGen เปลี่ยนชื่อ

**ข้อเท็จจริง:**
เป็น Framework ใหม่ที่รวมแนวคิดและทิศทางจากหลายโครงการของ Microsoft

---

## ความเข้าใจคลาดเคลื่อนที่ 2

> MAF Plain Agent เป็น Durable Workflow อัตโนมัติ

**ข้อเท็จจริง:**
ต้องใช้ Workflow Engine และ Persistence ที่เหมาะสม

---

## ความเข้าใจคลาดเคลื่อนที่ 3

> MAF ใช้ได้เฉพาะ Azure OpenAI

**ข้อเท็จจริง:**
Course ใช้ OpenAI ผ่าน `OpenAIChatClient`

---

## ความเข้าใจคลาดเคลื่อนที่ 4

> Agno เร็วกว่าเพราะ Model ตอบเร็วกว่า

**ข้อเท็จจริง:**
Benchmark วัด Agent Object Construction

---

## ความเข้าใจคลาดเคลื่อนที่ 5

> AgentOS ทำให้ Agent ถูกต้อง

**ข้อเท็จจริง:**
AgentOS เป็น Serving และ Operations Layer

---

## ความเข้าใจคลาดเคลื่อนที่ 6

> Plain Typed Function ปลอดภัยโดยอัตโนมัติ

**ข้อเท็จจริง:**
Type Hints ตรวจรูปแบบ ไม่ได้ตรวจ Authority และ Business Rules

---

## ความเข้าใจคลาดเคลื่อนที่ 7

> MCP เหมือนกันทุก Framework

**ข้อเท็จจริง:**
Protocol เหมือนกัน แต่ Lifecycle และ Error Handling ต่างกัน

---

## ความเข้าใจคลาดเคลื่อนที่ 8

> Goal เป็น Done หมายถึงงานสำเร็จ

**ข้อเท็จจริง:**
ต้องตรวจ Board Invariants และ Artifact จริง

---

# 65. Risks Identified

## 65.1 Premature Completion

Agent ปิด Goal ก่อน Steps ครบ

## 65.2 Missing Artifact

Board Done แต่ไม่มี `spanish.txt`

## 65.3 Incorrect Translation

File มีอยู่แต่เนื้อหาผิด

## 65.4 Duplicate Claim

MAF และ Agno ทำ Task เดียวกัน

## 65.5 Tool Argument Error

Model ส่ง Task ID ผิด

## 65.6 MCP Startup Failure

Node Process เริ่มไม่ได้

## 65.7 Hidden MCP Error

`stderr` ถูกส่งไป DEVNULL

## 65.8 Child Process Leak

Context Manager หรือ Cleanup ล้มเหลว

## 65.9 Internal API Patch Risk

Agno Patch พังหลัง Library Update

## 65.10 Benchmark Misinterpretation

วัด Object Construction แล้วสรุปว่า Framework เร็วกว่าโดยรวม

## 65.11 Workspace Mutation

Agent เขียนทับ Input

## 65.12 State Divergence

Framework, Board, Files และ MCP State ไม่ตรงกัน

---

# 66. Production Improvements

```text
Atomic task claiming
Goal-completion validator
Required artifact validation
Workspace per task
Filesystem permissions
MCP health checks
MCP structured logging
Tool timeouts
Retry limits
Model-call budgets
Tool-call budgets
Persistent workflow checkpoints
Human approval
Artifact hashes
Audit logs
Acceptance tests
```

---

# 67. Suggested Exercise — MAF Workflow Gate

สร้าง Workflow:

```text
Read Board
→ Worker Agent
→ Artifact Validator
→ Complete Goal
```

Validator ต้องตรวจ:

```text
ทุก Step เป็น done
spanish.txt มีอยู่
File ไม่ว่าง
```

---

# 68. Suggested Exercise — Agno Performance Breakdown

วัด:

```text
Agent construction
MCP startup
First model response
Full task completion
Tool calls
Token usage
```

เพื่อแยก Bottleneck จริง

---

# 69. Suggested Exercise — Shared Goal Validator

สร้าง Function เดียวให้ MAF และ Agno ใช้:

```python
def validate_goal(
    goal_id: int,
) -> dict:
    ...
```

ตรวจ:

```text
Task ownership
Unfinished steps
Required output
Output size
Goal state
```

---

# 70. Suggested Exercise — MCP Failure Comparison

ทำให้ `npx` ไม่พร้อม แล้วเปรียบเทียบ:

```text
Error type
Startup timeout
Cleanup behavior
Log visibility
User-facing message
```

ของ MAF และ Agno

---

# 71. Suggested Exercise — Duplicate Worker Test

รัน:

```text
maf_worker.py
agno_worker.py
```

พร้อม Task ID เดียวกัน

ตรวจ:

```text
มี Worker กี่ตัว Claim สำเร็จ
File ถูกเขียนกี่ครั้ง
Board Result ถูกแก้กี่ครั้ง
```

จากนั้นทำ Atomic Claim

---

# 72. Patterns Learned

## Client–Agent Separation

```text
Model client
→ Reused across agents
```

## Plain Function Tool Pattern

```text
Typed Python function
→ Tool schema
→ Model invocation
```

## Async MCP Lifecycle Pattern

```text
Context manager
→ Start
→ Use
→ Cleanup
```

## Agent vs Workflow Pattern

```text
Agent
→ Flexible decisions

Workflow
→ Controlled stages
```

## Lightweight Agent Pattern

```text
Low object setup overhead
→ Dynamic agent creation
```

## Serving Layer Pattern

```text
Agent logic
→ AgentOS
→ API, sessions and tracing
```

## Shared Worker Contract

```text
Task ID
+ Board path
→ Framework-specific worker
```

---

# 73. Connection to Week 5 Lab 1

Google ADK:

```text
LlmAgent
Runner
McpToolset
```

MAF:

```text
Agent
OpenAIChatClient
MCPStdioTool
```

Agno:

```text
Agent
OpenAIChat
MCPTools
```

Framework API ต่างกัน แต่:

```text
Model–Tool Loop
ยังเหมือนเดิม
```

---

# 74. Connection to Week 5 Lab 2

Strands เน้น:

```text
Minimal loop
Model portability
```

Pydantic AI เน้น:

```text
Types
Validation
Observability
```

MAF เน้น:

```text
Durable workflow
Enterprise integration
```

Agno เน้น:

```text
Lightweight agents
Serving through AgentOS
```

นี่แสดงว่า Framework Differentiation มักอยู่รอบ Agent Loop มากกว่าใน Loop เอง

---

# 75. Connection to Week 4

## LangChain `create_agent`

ใกล้กับ MAF และ Agno Plain Agents:

```text
Model
+ Tools
→ Prebuilt tool loop
```

## LangGraph

ใกล้กับ MAF Workflow Engine:

```text
Nodes
Edges
Persistence
Resume
```

## Deep Agents

ใกล้กับ Shared Board Pattern:

```text
External planning state
Filesystem
Goal-driven loop
```

---

# 76. Lab 3 Mental Model

## Microsoft Agent Framework

```text
Goal
→ MAF Agent
→ Plain Python tools
→ MCPStdioTool
→ Board and files
→ Completion
```

Optional Production Direction:

```text
Agents and functions
→ Workflow graph
→ Checkpoint and resume
```

## Agno

```text
Goal
→ Agno Agent
→ Plain Python tools
→ MCPTools
→ Board and files
→ Completion
```

Optional Production Direction:

```text
Agno Agent
→ AgentOS
→ API, sessions and tracing
```

---

# 77. Final Lessons

## Lesson 1

MAF และ Agno ใช้ Model–Tool Loop พื้นฐานเหมือน Framework ก่อนหน้า

## Lesson 2

MAF แยก Model Client ออกจาก Agent อย่างชัดเจน

## Lesson 3

MAF Plain Agent เหมาะกับ Flexible Tool Loop

## Lesson 4

MAF Workflow Engine เหมาะกับ Durable Controlled Processes

## Lesson 5

MAF API Alignment ระหว่าง Python และ C# มีคุณค่าต่อ Enterprise Teams

## Lesson 6

Agno ใช้ Plain Typed Functions เป็น Tools ได้โดยตรง

## Lesson 7

Agno ใช้ Async ตลอด Course เพราะ MCP ต้องมี Async Lifecycle

## Lesson 8

Agno MCP Workaround ผูกกับ Internal API และอาจเปราะบาง

## Lesson 9

Agent Instantiation Benchmark ไม่ได้วัด End-to-end Performance

## Lesson 10

AgentOS เป็น Serving และ Operations Layer ไม่ใช่ Validation Layer

## Lesson 11

MCP Lifecycle ต้อง Start และ Cleanup อย่างถูกต้อง

## Lesson 12

Framework State, Board State และ Filesystem State เป็นคนละ Surface

## Lesson 13

Goal Completion ต้องถูกตรวจด้วย Code ไม่ใช่เชื่อ Agent Self-report

## Lesson 14

Atomic Task Claiming เป็นความรับผิดชอบของ Application

## Lesson 15

Framework ควรถูกเลือกจาก Production Requirements ไม่ใช่เพียงจำนวนบรรทัดของ Demo

---

# 78. Memory Summary

```text
Week 5 Lab 3 มี:
Microsoft Agent Framework
Agno

Folder:
5_agent_frameworks/3_maf_agno

Notebooks:
maf_lab.ipynb
agno_lab.ipynb

Worker scripts:
maf_worker.py
agno_worker.py

Shared contract:
Read Goal
Plan Steps
Read notes.txt
Translate
Write spanish.txt
Complete Steps
Complete Goal

MAF:
Agent
OpenAIChatClient
MCPStdioTool
agent.run()
result.text

MAF tools:
Plain typed Python functions

MAF MCP:
async with filesystem

MAF key feature:
Graph-based durable workflows
Python/C# alignment

MAF plain agent:
Model-driven tool loop

MAF workflow:
Application-controlled nodes and edges

Agno:
Agent
OpenAIChat
MCPTools
agent.arun()
result.content

Agno tools:
Plain typed Python functions

Agno MCP:
async with MCPTools(...)

Agno key feature:
Lightweight construction
AgentOS path

AgentOS:
FastAPI runtime
Sessions
Streaming
TracingFastAPI runtime
Sessions
Streaming
Tracing
Control-plane UI

Instantiation benchmark:
วัด Python object setup

ไม่วัด:
Model latency
MCP startup
Tool execution
Task correctness

Both frameworks:
Use same SQLite board
Use same filesystem MCP
Use same goal contract

Shared state:
Framework state
Board state
Filesystem state
MCP process state

Shared weakness:
complete_task ไม่มี validation

Shared concurrency risk:
claim_todo ไม่ atomic

Shared security risk:
Workspace ไม่ใช่ full sandbox

MAF เหมาะกับ:
Enterprise workflows
Long-running jobs
Durability
Microsoft stack

Agno เหมาะกับ:
Lightweight agents
Dynamic workers
FastAPI serving
AgentOS

Application ยังต้องดูแล:
Task ownership
Artifact validation
Permissions
Timeouts
Budgets
Audit logs
```

---

# 79. Next Episode

Lab ถัดไปจะนำ Framework ใหม่มาทำ Contract เดิมอีกครั้ง

สิ่งที่ควรจับตา:

```text
Agent abstraction
Tool declaration
MCP lifecycle
Multi-agent support
Observability
Production deployment
Error recovery
```

คำถามสำคัญสำหรับ Lab ถัดไปคือ:

> เมื่อ Agent Loop แทบเหมือนกันทุก Framework การเลือก Framework ควรให้น้ำหนักกับอะไรระหว่าง Developer Experience, Durable State, Multi-agent Orchestration, Deployment Ecosystem และความสามารถในการตรวจสอบระบบ?
