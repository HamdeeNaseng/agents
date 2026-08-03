# Week 5 — Lab 2: AWS Strands และ Pydantic AI

ตำแหน่ง Lab:

```text
5_agent_frameworks/
└── 2_strands_pydantic/
    ├── strands_lab.ipynb
    ├── pydantic_lab.ipynb
    ├── strands_worker.py
    ├── pydantic_worker.py
    ├── board.py
    └── workspace/
```

Lab 2 มี **สอง Framework ในวันเดียวกัน**:

1. **AWS Strands Agents**
2. **Pydantic AI**

ทั้งสองต้องทำงาน Contract เดียวกับ Google ADK ใน Lab 1:

```text
อ่าน Goal จาก SQLite Board
→ สร้าง Steps
→ อ่าน notes.txt ผ่าน MCP
→ แปลเป็นภาษาสเปน
→ เขียน spanish.txt
→ ปิด Steps
→ ปิด Goal
```

Board และโจทย์เหมือนเดิม เพื่อให้เห็นว่าความต่างมาจาก Framework ไม่ใช่ Business Logic.

---

## Learning Objectives

หลังจบ Lab นี้ควรอธิบายได้ว่า:

1. Strands `Agent` และ Pydantic AI `Agent` ต่างกันอย่างไร
2. ทั้งสอง Framework ซ่อน Model–Tool Loop ไว้อย่างไร
3. ทำไม Strands ต้องระบุ Model อย่างชัดเจน
4. `invoke_async()` และ `run()` ต่างกันอย่างไร
5. Strands `@tool` มีหน้าที่อะไร
6. Pydantic AI สร้าง Tool จาก Plain Python Function อย่างไร
7. `tools` กับ `toolsets` ใน Pydantic AI ต่างกันอย่างไร
8. Strands `MCPClient` จัดการ MCP อย่างไร
9. Pydantic AI `MCPToolset` ต้องเปิด Connection Lifecycle อย่างไร
10. ทั้งสอง Framework ใช้ SQLite Board เดียวกันได้อย่างไร
11. Framework State แตกต่างจาก Board State และ File State อย่างไร
12. Model Portability ของ Strands ทำงานอย่างไร
13. Typed Output ของ Pydantic AI มีประโยชน์อย่างไร
14. Logfire ช่วย Observability อย่างไร
15. ทำไม Typed หรือ Structured Output ยังไม่เท่ากับ Correctness
16. Worker Scripts รองรับทั้ง Standalone Mode และ Shared-board Mode อย่างไร
17. เมื่อใดควรเลือก Strands หรือ Pydantic AI

---

# 1. Setup

Repository ปัจจุบันกำหนด:

```text
Python >= 3.12
strands-agents[openai] >= 1.43.0
pydantic-ai-slim[mcp,openai] >= 1.107.0
```

ทั้งสอง Notebook ใช้:

```env
OPENAI_API_KEY=...
```

และใช้ Node.js สำหรับ Filesystem MCP Server.

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

เมื่อ Server เริ่มแล้วให้หยุดด้วย `Ctrl+C` จากนั้นเปิด Notebook โดยใช้ Python Environment ของ Repository

---

# 2. Contract ที่ใช้เปรียบเทียบ Framework

ทุก Framework ใน Week 5 ใช้ Pattern ห้าขั้น:

```text
1. Create the agent
2. Run it
3. Add tools
4. Add MCP
5. Put it in a loop with a goal
```

โจทย์ไม่ได้เปลี่ยน:

```text
Read notes.txt,
translate its contents into natural Spanish,
and write the Spanish to spanish.txt.
```

สิ่งที่เปลี่ยนคือ:

```text
Agent constructor
Model adapter
Tool declaration
MCP connection
Invocation method
Result object
Observability
```

---

# Part A — AWS Strands

# 3. Strands Mental Model

Strands ถูกออกแบบรอบ **Model-driven Loop**

```text
Task
→ Model decides
→ Calls tool
→ Reads result
→ Decides again
→ Stops when it thinks the task is complete
```

Strands มี API ที่ค่อนข้างสั้น:

```python
from strands import Agent, tool
```

Conceptually:

```text
Agent
= Model
+ System prompt
+ Tools
+ Built-in execution loop
```

Course ใช้ Strands `1.43.0` และเน้นเรื่องความเบาของ Loop กับ Model Portability.

---

# 4. Step 1 — Create a Strands Agent

```python
import os

from strands import Agent
from strands.models.openai import OpenAIModel

MODEL = "gpt-5.4-mini"

model = OpenAIModel(
    client_args={
        "api_key": os.environ[
            "OPENAI_API_KEY"
        ]
    },
    model_id=MODEL,
)

agent = Agent(
    model=model,
    system_prompt=(
        "You are a concise, friendly "
        "assistant. Reply in a single "
        "short sentence."
    ),
)
```

Strands แยก:

```text
OpenAIModel
→ Provider/model configuration

Agent
→ Instructions, tools และ loop
```

จุดที่ต้องระวังคือ หากสร้าง:

```python
Agent()
```

โดยไม่ระบุ Model ตัว Framework จะใช้ Default ที่เกี่ยวกับ AWS Bedrock และต้องการ AWS Credentials ดังนั้น Course จึงส่ง `model=` อย่างชัดเจนเสมอ.

---

# 5. Step 2 — Run a Strands Agent

ใน Notebook ใช้:

```python
result = await agent.invoke_async(
    "Say hello in Spanish."
)
```

สามารถใช้แบบ Synchronous ได้ด้วย แต่ใน Jupyter ใช้ Async เพื่อทำงานร่วมกับ Event Loop ของ Notebook

Flow ตอนยังไม่มี Tools:

```text
User message
→ OpenAI model
→ Text response
```

ยังไม่มี Agentic Loop หลายรอบ เพราะ Model ไม่มี Action ให้เลือก

```text
No tools
→ No action-observation cycle
```

---

# 6. Step 3 — Strands Tools

Strands ใช้ `@tool`:

```python
from strands import tool


@tool
def show_todos() -> list[dict]:
    """List every todo on the board."""

    return board.list_todos()
```

Tool ที่รับ Parameters:

```python
@tool
def plan_steps(
    goal_id: int,
    steps: list[str],
) -> dict:
    """Break a goal into an ordered checklist.

    Args:
        goal_id: The goal to break down.
        steps: Step descriptions in order.
    """

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

และ:

```python
@tool
def complete_task(
    task_id: int,
    result: str,
) -> dict:
    """Mark a todo as done.

    Args:
        task_id: The todo id.
        result: A short result summary.
    """

    board.complete_todo(
        task_id,
        result,
    )

    return {
        "task_id": task_id,
        "status": "done",
    }
```

Strands ใช้:

```text
Decorator
Function name
Docstring
Args section
Type hints
```

เพื่อสร้าง Tool Schema ที่ Model เห็น.

---

## ทำไม `Args:` Section สำคัญ

Type Hint อธิบายว่า:

```text
goal_id เป็น int
steps เป็น list[str]
```

แต่ไม่ได้อธิบายความหมายเชิงธุรกิจ

`Args:` ช่วยบอกว่า:

```text
goal_id
= ID ของ Goal ที่ต้องแตกงาน

steps
= ลำดับขั้นตอนที่ควรทำ
```

Schema ตรวจรูปแบบ แต่ Docstring ช่วย Model เลือก Argument ให้ถูกบริบท

---

# 7. Add Tools to Strands

```python
board_agent = Agent(
    model=model,
    system_prompt=(
        "You help manage a shared "
        "todo board."
    ),
    tools=[
        show_todos,
        complete_task,
    ],
)
```

จากนั้น:

```python
result = await board_agent.invoke_async(
    "What is on the board right now?"
)
```

Agent อาจตัดสินใจ:

```text
Need current board information
→ Call show_todos()
→ Receive list of todos
→ Summarize for user
```

นี่คือ Tool Loop รอบแรก:

```text
Decide
→ Call
→ Observe
→ Answer
```

---

# 8. Step 4 — Strands MCP

Imports:

```python
from strands.tools.mcp import MCPClient
from mcp import (
    stdio_client,
    StdioServerParameters,
)
```

สร้าง Workspace:

```python
from pathlib import Path

workspace = Path(
    "workspace"
).resolve()
```

สร้าง MCP Client:

```python
filesystem = MCPClient(
    lambda: stdio_client(
        StdioServerParameters(
            command="npx",
            args=[
                "-y",
                "@modelcontextprotocol/"
                "server-filesystem",
                str(workspace),
            ],
            cwd=str(workspace),
        ),
    ),
    startup_timeout=60,
)
```

จากนั้นส่ง MCP Client เข้า `tools` List เดียวกับ Function Tools:

```python
file_agent = Agent(
    model=model,
    system_prompt=(
        "You can read and write files "
        "in your workspace."
    ),
    tools=[
        filesystem
    ],
)
```

ใน Strands:

```text
Python tools
และ
MCP client
```

ถูกนำเสนอให้ Agent ผ่าน Surface เดียวกันคือ:

```python
tools=[...]
```

Strands จัดการ Connection Lifecycle ของ MCP Client ระหว่าง Agent Invocation ให้.

---

# 9. Step 5 — Strands Goal Loop

Worker ตัวเต็ม:

```python
worker = Agent(
    model=model,
    system_prompt=INSTRUCTIONS,
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
board.claim_todo(goal_id)

await worker.invoke_async(
    "Please work the pending goal "
    "on the board."
)
```

Expected Flow:

```text
show_todos
    ↓
Find active goal
    ↓
plan_steps
    ↓
Read notes.txt through MCP
    ↓
Translate text
    ↓
Write spanish.txt through MCP
    ↓
complete_task(step)
    ↓
Repeat
    ↓
complete_task(goal)
```

Strands เป็นผู้จัด Internal Loop ให้:

```text
Model
→ Tool
→ Tool result
→ Model
→ Next tool
→ Final response
```

Application ไม่ต้องเขียน `while` เอง.

---

# 10. Strands Model Portability

Strands เน้นให้ Model เป็น Swappable Dependency

ตัวอย่าง OpenAI-compatible Endpoint:

```python
model = OpenAIModel(
    client_args={
        "api_key": OPENROUTER_KEY,
        "base_url": (
            "https://openrouter.ai/api/v1"
        ),
    },
    model_id="gpt-5.4-mini",
)
```

สิ่งที่ไม่ต้องเปลี่ยน:

```text
Tools
System prompt
SQLite board
MCP server
Agent loop
Worker contract
```

Strands ยังมี Model Classes สำหรับ Providers อื่น แต่ OpenAI-compatible `base_url` ครอบคลุมหลาย Endpoint ได้ด้วย Adapter เดียว.

หลัก:

```text
Model configuration changes
Application capabilities remain
```

---

# Part B — Pydantic AI

# 11. Pydantic AI Mental Model

Pydantic AI เน้น:

```text
Type safety
Validation
Structured outputs
Dependency injection
Observability
```

Agent Loop ยังคงเหมือนเดิม:

```text
Model
→ Tool call
→ Tool result
→ Model
```

แต่ Framework ให้ความสำคัญกับ Contract ของข้อมูลมากกว่า

```text
Inputs are typed
Tools are typed
Outputs can be typed
Validation is first-class
```

Course ใช้ Pydantic AI `1.107.0`.

---

# 12. Step 1 — Create a Pydantic AI Agent

```python
from pydantic_ai import Agent

MODEL = "openai-chat:gpt-5.4-mini"

agent = Agent(
    MODEL,
    instructions=(
        "You are a concise, friendly "
        "assistant. Reply in a single "
        "short sentence."
    ),
)
```

Model string:

```text
openai-chat:gpt-5.4-mini
```

แยกเป็น:

```text
openai-chat
→ Provider/API adapter

gpt-5.4-mini
→ Model identifier
```

`-chat` ระบุ OpenAI Chat Completions API อย่างชัดเจน ซึ่งเหมาะกับ OpenAI-compatible Endpoints ที่ใช้ Protocol เดียวกัน.

---

# 13. Step 2 — Run a Pydantic AI Agent

```python
result = await agent.run(
    "Say hello in Spanish."
)

print(result.output)
```

Result ไม่ใช่ String โดยตรง แต่เป็น Run Result Object

```text
result.output
→ Final output

result messages
→ Conversation/run information

usage
→ Model usage information
```

ใน Script ที่ไม่ได้ใช้ Async สามารถใช้:

```python
agent.run_sync(...)
```

แต่ใน Notebook ใช้ `await agent.run()` เพื่อไม่ Block Jupyter Event Loop

---

# 14. Step 3 — Pydantic AI Tools

Tool สามารถเป็น Plain Python Function:

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
    """Mark a todo done and record its result."""

    board.complete_todo(
        task_id,
        result,
    )

    return {
        "task_id": task_id,
        "status": "done",
    }
```

จากนั้นส่งเข้า Agent:

```python
board_agent = Agent(
    MODEL,
    instructions=(
        "You help manage a shared "
        "todo board."
    ),
    tools=[
        show_todos,
        complete_task,
    ],
)
```

Pydantic AI อ่าน:

```text
Function signature
Type hints
Docstring
```

และสร้าง JSON Schema ให้อัตโนมัติ.

---

## `@agent.tool` กับ `tools=[...]`

Pydantic AI รองรับ:

```python
@agent.tool
def some_tool(...):
    ...
```

แต่ Course ใช้:

```python
Agent(
    ...,
    tools=[some_tool],
)
```

เหตุผลคือ Function เดิมสามารถ Reuse กับ Agents หลายตัวได้ง่ายกว่า

Mental model:

```text
@agent.tool
→ Tool ผูกกับ Agent Instance

tools=[function]
→ Tool ถูกส่งเป็น Dependency ตอนสร้าง Agent
```

---

# 15. Step 4 — Pydantic AI MCP

Imports:

```python
from pydantic_ai.mcp import MCPToolset
from fastmcp.client.transports import (
    StdioTransport,
)
```

Transport:

```python
transport = StdioTransport(
    command="npx",
    args=[
        "-y",
        "@modelcontextprotocol/"
        "server-filesystem",
        str(workspace),
    ],
    cwd=str(workspace),
    log_file=Path(os.devnull),
)
```

Toolset:

```python
filesystem = MCPToolset(
    transport,
    init_timeout=60,
)
```

เพิ่มให้ Agent ผ่าน:

```python
file_agent = Agent(
    MODEL,
    instructions=(
        "Read and write files using "
        "your filesystem tools."
    ),
    toolsets=[
        filesystem
    ],
)
```

สังเกตว่า Pydantic AI แยก:

```text
tools
→ Python function tools

toolsets
→ Collections of external tools
   เช่น MCP server
```

ต่างจาก Strands ที่ใส่ทุกอย่างใน `tools=[...]`.

---

# 16. MCP Connection Lifecycle ใน Pydantic AI

ต้องเปิด Agent เป็น Async Context Manager:

```python
async with file_agent:
    result = await file_agent.run(
        "Read notes.txt and summarize it."
    )
```

เหตุผล:

```text
Enter context
→ Start MCP transport
→ Initialize tool discovery
→ Run agent
→ Close toolset and process
```

หากเรียก MCP-enabled Agent โดยไม่จัด Lifecycle ให้ถูกต้อง อาจเกิด:

```text
Connection not initialized
Child process leak
Transport shutdown error
MCP tools unavailable
```

ใน Worker ตัวเต็มก็ใช้ Pattern เดียวกัน:

```python
async with worker:
    result = await worker.run(message)
```

---

# 17. Step 5 — Pydantic AI Goal Loop

```python
worker = Agent(
    MODEL,
    instructions=INSTRUCTIONS,
    tools=[
        show_todos,
        plan_steps,
        complete_task,
    ],
    toolsets=[
        filesystem
    ],
)
```

รัน:

```python
board.claim_todo(goal_id)

async with worker:
    result = await worker.run(
        "Please work the pending goal "
        "on the board."
    )

print(result.output)
```

Agent ทำ Loop:

```text
Read Board
→ Plan Steps
→ Read File
→ Translate
→ Write File
→ Mark Steps
→ Close Goal
```

Pydantic AI จัด Model–Tool Loop ภายใน `run()` ให้เอง.

---

# 18. Typed and Validated Outputs

Pydantic AI เด่นเรื่อง Typed Output

ตัวอย่าง:

```python
from pydantic import BaseModel


class TranslationResult(BaseModel):
    source_file: str
    output_file: str
    language: str
    completed: bool
```

สร้าง Agent:

```python
agent = Agent(
    MODEL,
    instructions=INSTRUCTIONS,
    output_type=TranslationResult,
)
```

Result:

```python
result = await agent.run(task)

typed_result = result.output
```

`result.output` จะเป็น:

```python
TranslationResult(
    source_file="notes.txt",
    output_file="spanish.txt",
    language="Spanish",
    completed=True,
)
```

แทนข้อความที่ต้อง Parse เอง

Course เน้นว่า Pydantic AI สามารถรับ Pydantic Model ผ่าน `output_type=` แล้ว Validate Final Output ให้เป็น Instance ของ Type นั้น.

---

## Typed Output ไม่เท่ากับ Correctness

Schema สามารถตรวจว่า:

```text
output_file เป็น string
completed เป็น boolean
```

แต่ไม่ตรวจว่า:

```text
spanish.txt มีอยู่จริง
เนื้อหาเป็นภาษาสเปนจริง
คำแปลถูกต้อง
Goal ถูกปิดอย่างเหมาะสม
```

ดังนั้น:

```text
Valid Python object
≠ Valid real-world outcome
```

ต้องมี Artifact Validator เพิ่ม

---

# 19. Pydantic AI Observability

Pydantic AI เชื่อม Logfire ได้ด้วยแนวคิดประมาณ:

```python
import logfire

logfire.configure()
logfire.instrument_pydantic_ai()
```

จากนั้นสามารถดู:

```text
Model requests
Tool calls
Validation
Latency
Errors
Usage
```

Course ไม่ติดตั้ง Logfire Extra ไว้ใน Shared Environment เพื่อให้ Dependencies เบา แต่ยกให้เป็นจุดเด่นสำคัญของ Framework.

หลัก:

```text
Types
ช่วยตรวจ Data Contract

Logfire
ช่วยตรวจ Execution Behavior
```

แต่ทั้งสองไม่รับประกัน Factual Correctness

---

# 20. Pydantic AI Model Portability

สำหรับ OpenAI-compatible Endpoint:

```python
from pydantic_ai.models.openai import (
    OpenAIChatModel,
)
from pydantic_ai.providers.openai import (
    OpenAIProvider,
)

model = OpenAIChatModel(
    "gpt-5.4-mini",
    provider=OpenAIProvider(
        base_url=(
            "https://openrouter.ai/api/v1"
        ),
        api_key=OPENROUTER_KEY,
    ),
)

agent = Agent(
    model,
    instructions="...",
)
```

สิ่งที่ไม่เปลี่ยน:

```text
Functions
MCP Toolset
Board
Agent instructions
Goal contract
```

---

# 21. Worker Scripts

ทั้งสอง Framework มี Script:

```text
strands_worker.py
pydantic_worker.py
```

รัน Standalone:

```powershell
uv run strands_worker.py
```

หรือ:

```powershell
uv run pydantic_worker.py
```

ทั้งสองจะ:

```text
Reset board
→ Clear spanish.txt
→ Add goal
→ Claim goal
→ Create worker
→ Run loop
→ Print board
→ Print spanish.txt
```

---

# 22. Standalone Mode กับ Day 5 Mode

Worker Scripts รองรับสองรูปแบบ

## Standalone Day 2

```powershell
uv run strands_worker.py
```

Script สร้าง Board และ Goal เอง

## Shared-board Day 5

```powershell
uv run strands_worker.py `
  <taskId> `
  <boardPath>
```

หรือ:

```powershell
uv run pydantic_worker.py `
  <taskId> `
  <boardPath>
```

เมื่อรับ Arguments:

```text
taskId
boardPath
```

Worker จะ:

```text
ชี้ BOARD_PATH ไปยัง Shared Board
ใช้ Directory ของ Board เป็น Workspace
Claim เฉพาะ Task ที่ได้รับ
ทำเฉพาะ Task นั้น
ปิด Task แล้วหยุด
```

นี่เป็นการเตรียม Worker Contract สำหรับ Multi-framework Orchestration ใน Day 5.

---

# 23. เหตุใดต้องตั้ง `BOARD_PATH` ก่อน Import

Worker ทำประมาณ:

```python
TASK_ID = (
    int(sys.argv[1])
    if len(sys.argv) > 2
    else None
)

if TASK_ID is not None:
    os.environ.setdefault(
        "BOARD_PATH",
        sys.argv[2],
    )

import board
```

`board.py` อ่าน Environment Variable ตอน Module ถูก Import

ดังนั้นถ้า Import ก่อน:

```text
import board
→ BOARD_PATH ถูกกำหนดแล้ว
```

การเปลี่ยน Environment Variable ภายหลังอาจไม่เปลี่ยน Path ที่ Module เก็บไว้

หลัก:

```text
Configuration required at import time
ต้องถูกตั้งก่อน import
```

---

# 24. Strands vs Pydantic AI

| ประเด็น       | Strands                     | Pydantic AI                        |
| ------------- | --------------------------- | ---------------------------------- |
| Agent         | `strands.Agent`             | `pydantic_ai.Agent`                |
| Model         | `OpenAIModel` Object        | Model String หรือ Model Object     |
| Run           | `invoke_async()`            | `run()`                            |
| Final result  | Agent Result                | `result.output`                    |
| Python Tool   | ต้องใช้ `@tool`             | Plain Function หรือ `@agent.tool`  |
| MCP           | `MCPClient` ใน `tools`      | `MCPToolset` ใน `toolsets`         |
| MCP lifecycle | Agent/Client จัดการให้      | ใช้ `async with agent`             |
| จุดเด่น       | Minimal loop, portability   | Types, validation, Logfire         |
| Default risk  | ไม่ระบุ Model อาจไป Bedrock | Model string/API selection ต้องชัด |
| Typed output  | รองรับได้                   | เป็น Core Strength                 |

---

# 25. เปรียบเทียบกับ Google ADK

## Google ADK

```text
LlmAgent
+ Runner
+ Function tools
+ McpToolset
```

## Strands

```text
Agent
+ OpenAIModel
+ @tool
+ MCPClient
```

## Pydantic AI

```text
Agent
+ Model string
+ Typed functions
+ MCPToolset
```

ความแตกต่างสำคัญ:

```text
ADK
→ Agent definition แยกจาก Runner ชัดเจน

Strands
→ Agent object เรียก Loop ได้โดยตรง

Pydantic AI
→ Agent object เรียก Loop และคืน Typed Result ได้
```

แต่ Agent Pattern เหมือนกัน:

```text
Model
→ Tool call
→ Tool result
→ Model
→ Completion
```

---

# 26. State Surfaces

ทั้งสอง Worker มี State อย่างน้อยสามประเภท:

```text
1. Framework execution state
   → Messages และ tool-loop context

2. SQLite board state
   → Goal, Steps, status และ result

3. Filesystem state
   → notes.txt และ spanish.txt
```

Framework Run อาจจบ แต่ Board หรือ File อาจไม่ตรงกัน

ตัวอย่าง:

```text
Agent returns success
แต่ Goal ยัง in_progress

Goal เป็น done
แต่ spanish.txt ไม่มี

spanish.txt มี
แต่เนื้อหาไม่ใช่ภาษาสเปน
```

ดังนั้น Final Response ของ Agent ไม่ควรเป็น Authority เพียงอย่างเดียว

---

# 27. Shared Weakness: `complete_task()`

ทั้งสอง Worker ใช้ Function เดียวกัน:

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

ไม่ได้ตรวจว่า:

```text
Task ID มีอยู่จริงหรือไม่
Goal มี Steps ค้างหรือไม่
spanish.txt มีอยู่หรือไม่
Output ไม่ว่างหรือไม่
Translation ถูกต้องหรือไม่
```

ดังนั้นทั้งสอง Framework สามารถปิด Goal ผิดได้เหมือนกัน

นี่แสดงว่า:

> Framework ต่างกัน แต่ Business Invariant ที่ขาดยังทำให้เกิด Failure แบบเดียวกัน

---

# 28. Shared Weakness: Duplicate Claim

Worker ใช้:

```python
board.claim_todo(TASK_ID)
```

หากหลาย Workers เรียกพร้อมกัน Task เดียวกันอาจถูกทำซ้ำ

ควรเปลี่ยนเป็น Atomic Claim:

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

Agent Framework ไม่ได้แก้ Distributed Work Coordination ให้เอง

---

# 29. Shared Weakness: Filesystem Permissions

ทั้งสอง MCP Servers ถูก Scope ไปยัง `WORK_DIR`

ข้อดี:

```text
Agent เข้าถึงไฟล์นอก Workspace ไม่ได้ง่าย
```

แต่ภายใน Workspace Agent ยังสามารถ:

```text
อ่านทุกไฟล์
เขียนทับ
ลบ
สร้างข้อมูลผิด
ทำตาม Prompt Injection ในไฟล์
```

Root restriction คือ Permission Boundary ระดับหนึ่ง แต่ไม่ใช่ Content Validation หรือ Full Sandbox

---

# 30. เมื่อใดควรเลือก Strands

Strands เหมาะเมื่อ:

```text
ต้องการ API ขนาดเล็ก
ต้องการ Model-driven loop ตรงไปตรงมา
ต้องสลับ Model หรือ Provider บ่อย
ใช้ AWS ecosystem
ไม่ต้องการ Type Architecture ซับซ้อนมาก
```

จุดแข็ง:

```text
Agent constructor สั้น
Tools ชัด
MCP อยู่ใน tools list เดียวกัน
Model portability เด่น
```

จุดที่ต้องระวัง:

```text
ต้องระบุ Model ชัด
@tool และ Docstring ต้องดี
Application validation ยังต้องสร้างเอง
```

---

# 31. เมื่อใดควรเลือก Pydantic AI

Pydantic AI เหมาะเมื่อ:

```text
ระบบ Python ใช้ Pydantic อยู่แล้ว
ต้องการ Typed inputs/outputs
ต้องการ Validation ที่ชัดเจน
สร้าง FastAPI หรือ typed backend
ต้องการ Logfire observability
```

จุดแข็ง:

```text
Type contracts
Validated outputs
Dependency patterns
Clear result object
Integrated observability
```

จุดที่ต้องระวัง:

```text
MCP lifecycle ต้องจัดการ
Type-valid ไม่เท่ากับ fact-valid
Schema ที่ซับซ้อนอาจเพิ่ม retries
```

---

# 32. Misconceptions

### “Strands เป็น AWS-only”

ไม่ทั้งหมด Strands รองรับ Model Providers หลายรูปแบบ และ Course ใช้ OpenAI Model

### “Strands `Agent()` จะเลือก OpenAI ให้อัตโนมัติ”

ไม่จริง Course ระบุ Model เองเพื่อเลี่ยง Default Bedrock

### “Pydantic AI Tool ต้องมี Decorator เสมอ”

ไม่จริง สามารถส่ง Plain Functions ผ่าน `tools=[...]`

### “Pydantic Validation พิสูจน์ว่า Agent ทำงานสำเร็จ”

ไม่จริง ตรวจ Data Shape เป็นหลัก

### “MCP ทำงานเหมือนกันทุก Framework”

Protocol เหมือนกัน แต่ Lifecycle และ API Integration ต่างกัน

### “Strands กับ Pydantic AI มี Loop คนละแบบโดยสิ้นเชิง”

ไม่จริง ทั้งสองใช้ Model–Tool–Observation Loop เดียวกัน

### “เปลี่ยน Framework แล้ว Business Risks หาย”

ไม่จริง Board, Task Claiming และ Artifact Validation ยังคงเป็นความรับผิดชอบของ Application

---

# 33. Exercises

## Exercise 1 — Typed Pydantic Result

สร้าง:

```python
class WorkerResult(BaseModel):
    goal_id: int
    output_file: str
    steps_completed: int
    success: bool
```

แล้วตรวจว่า:

```text
success=True
ต้องสอดคล้องกับ Board และ File จริง
```

---

## Exercise 2 — Strands Model Swap

เปลี่ยน `base_url` ไปยัง OpenAI-compatible Endpoint แล้วเปรียบเทียบ:

```text
Tool selection
Step planning
Call count
Translation quality
Completion behavior
```

---

## Exercise 3 — Artifact Validator

สร้าง Function:

```python
def validate_spanish_output() -> dict:
    ...
```

ตรวจ:

```text
ไฟล์มีอยู่
ไม่ว่าง
Encoding ถูกต้อง
ไม่มีข้อความต้นฉบับอย่างเดียว
```

---

## Exercise 4 — MCP Failure

ปิดหรือทำให้ `npx` ไม่พร้อม แล้วเปรียบเทียบ Error Handling ของ:

```text
Strands MCPClient
Pydantic AI MCPToolset
```

---

## Exercise 5 — Framework Comparison Metrics

เก็บ:

```text
Model calls
Tool calls
Elapsed time
Token usage
Steps created
Board correctness
Artifact correctness
```

อย่าประเมินจาก Final Answer เพียงอย่างเดียว

---

# 34. Checklist

### Week 5 Lab 2 มี Framework ใดบ้าง

```text
AWS Strands
Pydantic AI
```

### Strands สร้าง Agent ด้วยอะไร

```python
Agent(
    model=model,
    system_prompt=...,
)
```

### Pydantic AI สร้าง Agent ด้วยอะไร

```python
Agent(
    MODEL,
    instructions=...,
)
```

### Strands เรียก Agent อย่างไร

```python
await agent.invoke_async(...)
```

### Pydantic AI เรียก Agent อย่างไร

```python
await agent.run(...)
```

### Strands Tool ใช้อะไร

```python
@tool
```

### Pydantic AI Tool ใช้อะไร

Plain typed function หรือ `@agent.tool`

### Strands MCP อยู่ที่ไหน

```python
tools=[filesystem]
```

### Pydantic MCP อยู่ที่ไหน

```python
toolsets=[filesystem]
```

### Pydantic MCP เปิด Lifecycle อย่างไร

```python
async with agent:
    await agent.run(...)
```

### Goal Loop ของทั้งสองคืออะไร

```text
Read
→ Plan
→ Act
→ Complete
→ Repeat
```

---

# แก่นของ Week 5 Lab 2

```text
Strands
= Minimal model-driven agent loop

Pydantic AI
= Typed and validated agent framework

SQLite Board
= Shared application contract

MCP
= Shared external tool protocol

Worker scripts
= Same task, different runtime
```

บทเรียนสำคัญที่สุดคือ:

> **เมื่อ Business Contract, Tools และ External State เหมือนกัน Agent Framework ส่วนใหญ่แตกต่างกันที่ Developer Experience, Type System, Model Adapter, MCP Lifecycle และ Observability มากกว่าความหมายพื้นฐานของ Agent Loop**

และ:

> **Strands ทำให้เห็นว่า Model และ Provider สามารถเป็นชิ้นส่วนที่สลับได้ ส่วน Pydantic AI ทำให้เห็นว่า Tool และ Output Contracts สามารถถูกทำให้ชัดและตรวจสอบได้ด้วย Type System**

แต่ข้อสรุปที่สำคัญกว่า Framework คือ:

> **ไม่ว่าใช้ Strands หรือ Pydantic AI ระบบยังต้องพึ่ง Application Code เพื่อบังคับ Task Ownership, ตรวจ Artifact และตัดสินว่างานสำเร็จจริงหรือไม่ เพราะ Agent ที่ปิด Goal ได้ ไม่ได้แปลว่าโลกภายนอกอยู่ในสถานะที่ถูกต้องเสมอ**
