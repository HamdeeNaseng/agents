# Episodic Learning Artifact

## Week 5 — Lab 2: AWS Strands and Pydantic AI

**หลักสูตร:** AI Engineer Agentic Track
**ผู้สอน:** Ed Donner
**Repository:** `ed-donner/agents`
**โฟลเดอร์:** `5_agent_frameworks/2_strands_pydantic`
**Notebooks:** `strands_lab.ipynb`, `pydantic_lab.ipynb`
**ไฟล์ประกอบ:** `strands_worker.py`, `pydantic_worker.py`, `board.py`, `workspace/`
**หัวข้อหลัก:** Model-driven Agent Loop, Typed Tools, MCP Lifecycle, Model Portability, Typed Outputs และ Framework Comparison
**สถานะ:** เรียนและสรุป Lab 2 แล้ว

---

# 1. Context

Week 5 ใช้โจทย์เดิมกับหลาย Framework เพื่อเปรียบเทียบวิธีสร้าง Agent โดยไม่เปลี่ยน Business Contract

Contract จาก Lab 1:

```text id="tlrz18"
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

Lab 1 ใช้ Google ADK

Lab 2 สร้าง Worker เดียวกันด้วย:

```text id="3168gn"
AWS Strands Agents
และ
Pydantic AI
```

สิ่งที่คงเดิม:

```text id="94bk5u"
Goal
Board schema
Board functions
Workspace
notes.txt
spanish.txt
Filesystem MCP server
Agent instructions
Definition of work
```

สิ่งที่เปลี่ยน:

```text id="p7euoa"
Agent constructor
Model adapter
Tool registration
Invocation API
MCP integration
Connection lifecycle
Result object
Observability
```

---

# 2. Learning Objectives

หลังจบ Lab 2 สามารถอธิบายได้ว่า:

1. Strands และ Pydantic AI สร้าง Agent อย่างไร
2. ทั้งสอง Framework ใช้ Model–Tool Loop แบบเดียวกันอย่างไร
3. Strands `Agent` และ `OpenAIModel` แยกหน้าที่กันอย่างไร
4. ทำไม Strands ต้องระบุ Model อย่างชัดเจน
5. `invoke_async()` ใช้รัน Strands Agent อย่างไร
6. `@tool` แปลง Python Function เป็น Strands Tool อย่างไร
7. Pydantic AI ใช้ Model String อย่างไร
8. `agent.run()` และ `result.output` ทำงานอย่างไร
9. Plain Python Function กลายเป็น Pydantic AI Tool อย่างไร
10. `tools` และ `toolsets` ต่างกันอย่างไร
11. Strands `MCPClient` เชื่อม Filesystem Server อย่างไร
12. Pydantic AI `MCPToolset` จัดการ MCP อย่างไร
13. ทำไม Pydantic AI Agent ที่มี MCP ต้องใช้ `async with`
14. Model Portability ของ Strands มีประโยชน์อย่างไร
15. Typed Output ของ Pydantic AI มีประโยชน์อย่างไร
16. Typed Validation แตกต่างจาก Real-world Validation อย่างไร
17. Worker Scripts รองรับ Standalone และ Shared-board Mode อย่างไร
18. Framework State, Board State และ File State ต่างกันอย่างไร
19. Business Risks ใดยังคงเหมือนเดิมแม้เปลี่ยน Framework
20. เมื่อใดควรเลือก Strands หรือ Pydantic AI

---

# 3. Prerequisites

ควรเข้าใจแนวคิดจาก Week 5 Lab 1:

```text id="9d6gwz"
Agent definition
Execution runtime
Model–tool loop
Function tool
MCP
SQLite todo board
Goal
Step
Application state
Filesystem artifact
```

Environment:

```text id="wayfwl"
Python >= 3.12
Node.js
npx
OpenAI API key
SQLite
```

Environment Variable:

```env id="25f2c5"
OPENAI_API_KEY=...
```

Dependencies หลัก:

```text id="u6v7bt"
strands-agents[openai]
pydantic-ai-slim[mcp,openai]
fastmcp
mcp
```

ติดตั้ง:

```powershell id="fydam6"
uv sync
```

ตรวจ Node:

```powershell id="56a3w4"
node --version
npx --version
```

Warm MCP Server:

```powershell id="k8yyjj"
npx -y @modelcontextprotocol/server-filesystem .
```

---

# 4. Week 5 Comparison Pattern

ทั้งสอง Framework ถูกเรียนผ่านห้าขั้น:

```text id="e1xc4a"
1. Create the agent
2. Run it
3. Add tools
4. Add MCP
5. Put it in a loop with a goal
```

ขั้นที่ 1 และ 2:

```text id="2fcr0f"
User
→ Model
→ Text response
```

ยังเป็น LLM Call เป็นหลัก

ขั้นที่ 3 และ 4 เพิ่ม:

```text id="0gxq56"
External capabilities
Tool calls
Environment observations
```

ขั้นที่ 5 เปลี่ยนเป็น Agent Loop:

```text id="nk13vo"
Goal
→ Plan
→ Tool
→ Observation
→ Next action
→ Completion
```

---

# Part A — AWS Strands

# 5. What Is Strands?

Strands เป็น Agent SDK ที่เน้น Model-driven Loop

```text id="qd2det"
Model เห็น Goal
→ เลือก Tool
→ อ่าน Tool Result
→ เลือก Tool ต่อ
→ หยุดเมื่อคิดว่างานเสร็จ
```

Mental Model:

```text id="wg8rhj"
Strands Agent
= Model
+ System Prompt
+ Tools
+ Agent Loop
```

Strands เน้น:

```text id="pwvq8r"
API ขนาดเล็ก
Model portability
Tools ที่ Reuse ได้
Model-driven execution
```

---

# 6. Strands Imports

```python id="58hkii"
import os

from strands import Agent, tool
from strands.models.openai import OpenAIModel
```

หน้าที่:

```text id="13hb9x"
Agent
→ สร้างและรัน Agent Loop

tool
→ แปลง Python Function เป็น Tool

OpenAIModel
→ เชื่อม Strands กับ OpenAI-compatible Model
```

---

# 7. Step 1 — Create a Strands Agent

```python id="kfgdyc"
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

Strands แยกสอง Object:

```text id="faj4n4"
OpenAIModel
→ Provider configuration

Agent
→ Instructions, tools และ loop
```

---

# 8. Why Specify the Model Explicitly?

หากสร้าง:

```python id="rbwf65"
Agent()
```

โดยไม่ส่ง `model=`

Strands อาจใช้ Default Model Integration ที่เกี่ยวข้องกับ AWS Bedrock

ผลที่อาจเกิด:

```text id="gmwqvr"
ต้องมี AWS credentials
ใช้ Provider ที่ไม่ตั้งใจ
Configuration error
พฤติกรรมต่างจาก Course
```

Course จึงใช้:

```python id="bly90k"
Agent(
    model=model,
    ...
)
```

เสมอ

หลัก:

```text id="9zhy74"
Implicit model defaults
ลด Code

แต่
เพิ่ม Configuration ambiguity
```

---

# 9. Step 2 — Run Strands

```python id="zh6m8c"
result = await agent.invoke_async(
    "Say hello in Spanish."
)
```

Flow:

```text id="f57e4t"
Input
→ Agent
→ OpenAIModel
→ Text output
```

ตอนยังไม่มี Tools:

```text id="8gphtm"
ไม่มี Action Surface
ไม่มี Tool observation
ไม่มี Multi-step loop
```

ดังนั้นยังเป็น LLM Call ที่ถูกห่อด้วย Agent API

---

# 10. Strands Synchronous and Asynchronous APIs

Notebook ใช้:

```python id="us262s"
await agent.invoke_async(...)
```

เพื่อทำงานร่วมกับ Jupyter Event Loop

ใน Python Script ธรรมดาสามารถเรียก Agent ผ่าน Synchronous API ได้ตามรูปแบบของ Framework

ข้อสำคัญ:

```text id="k5g0ds"
Async
ช่วยจัดการ I/O

ไม่ได้ทำให้
Model reasoning ดีขึ้น
```

---

# 11. Step 3 — Strands Function Tools

ตัวอย่าง Read Tool:

```python id="0kcx8x"
@tool
def show_todos() -> list[dict]:
    """List every todo on the board."""

    return board.list_todos()
```

Planning Tool:

```python id="qgjl8m"
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

Completion Tool:

```python id="x6s9l5"
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

---

# 12. How Strands Builds Tool Schemas

Strands อ่าน:

```text id="tdt460"
Function name
Decorator metadata
Docstring
Args section
Type hints
Return value
```

ตัวอย่าง Schema Concept:

```text id="vnq6sa"
plan_steps

goal_id:
integer

steps:
array of strings
```

Model ใช้ Schema เพื่อตัดสินใจ:

```text id="ksbg9u"
ควรเรียก Tool หรือไม่
ควรส่ง Argument อะไร
Argument มีรูปแบบใด
```

---

# 13. Type Hints vs Docstrings

Type Hints บอก:

```text id="lid8o9"
goal_id เป็น integer
steps เป็น list of strings
```

Docstring บอก:

```text id="xzyw16"
goal_id หมายถึงอะไร
steps ควรมีลักษณะอย่างไร
Tool ควรถูกใช้เมื่อใด
```

ดังนั้น:

```text id="96338s"
Type
= Data structure

Description
= Semantic meaning
```

ทั้งสองอย่างจำเป็นต่อ Tool Selection ที่ดี

---

# 14. Add Tools to a Strands Agent

```python id="j3b17k"
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

เมื่อถาม:

```text id="2pmchz"
What is on the board?
```

Agent อาจทำ:

```text id="pb9hhd"
Model decides current data is needed
→ show_todos()
→ Board returns todos
→ Model summarizes
```

Agent Loop เริ่มจาก:

```text id="w45uyj"
Decide
→ Call
→ Observe
→ Answer
```

---

# 15. Step 4 — Strands MCP

Imports:

```python id="r5tvom"
from strands.tools.mcp import MCPClient
from mcp import (
    stdio_client,
    StdioServerParameters,
)
```

สร้าง Workspace:

```python id="vz3j55"
from pathlib import Path

workspace = Path(
    "workspace"
).resolve()
```

สร้าง MCP Client:

```python id="jk7q25"
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

---

# 16. Strands MCP Architecture

```text id="fbu7f3"
Strands Agent
    ↓
MCPClient
    ↓ stdio
Node.js Filesystem Server
    ↓
workspace/
```

Filesystem Server ให้ Tools เช่น:

```text id="3w46q3"
List files
Read file
Write file
Edit file
Create directories
```

ชื่อ Tool จริงอาจเปลี่ยนตาม MCP Server Version

---

# 17. MCP in the `tools` List

Strands นำ Function Tools และ MCP Client ใส่ใน List เดียวกัน:

```python id="sb4vcf"
worker = Agent(
    model=model,
    tools=[
        show_todos,
        plan_steps,
        complete_task,
        filesystem,
    ],
)
```

จากมุมมอง Agent:

```text id="0hkkqr"
Python Tool
และ
MCP Tool
```

เป็นความสามารถที่เลือกใช้ผ่าน Agent Loop เหมือนกัน

---

# 18. Strands MCP Lifecycle

Strands จัดการ Connection Lifecycle ของ `MCPClient` ให้ระหว่าง Agent Invocation

Concept:

```text id="8l5ziw"
Agent starts
→ MCP process starts
→ Tools discovered
→ Agent calls tools
→ Invocation finishes
→ Connection closes
```

ข้อดี:

```text id="zg8rwj"
Code สั้น
ไม่ต้องเปิด Context Manager เอง
MCP อยู่ใน tools surface เดียวกัน
```

สิ่งที่ต้องระวัง:

```text id="0ehu8z"
Startup timeout
Child-process errors
Version mismatch
Stderr handling
Process cleanup
```

---

# 19. Step 5 — Strands Goal Loop

Worker:

```python id="x5hh44"
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

Run:

```python id="2iqj9m"
board.claim_todo(
    goal_id
)

await worker.invoke_async(
    "Please work the pending goal "
    "on the board."
)
```

Expected Loop:

```text id="mj64oy"
show_todos
→ Find Goal
→ plan_steps
→ Read notes.txt
→ Translate
→ Write spanish.txt
→ complete_task Step
→ Repeat
→ complete_task Goal
```

---

# 20. Strands Agent Loop

```text id="z9r8vg"
Model receives goal
        ↓
Model requests show_todos
        ↓
Board result
        ↓
Model requests plan_steps
        ↓
Step IDs
        ↓
Model calls filesystem tools
        ↓
File results
        ↓
Model updates board
        ↓
Model decides task is finished
```

Framework จัดการ Outer Tool Loop ให้

Application ไม่ต้องเขียน:

```python id="k2or8o"
while response.tool_calls:
    ...
```

---

# 21. Strands Model Portability

Strands สามารถชี้ `OpenAIModel` ไปยัง OpenAI-compatible Endpoint:

```python id="ajfif7"
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

สิ่งที่ไม่เปลี่ยน:

```text id="2y6r3w"
Tool functions
Board
MCP server
System prompt
Agent loop
Worker contract
```

หลัก:

```text id="yx7u3m"
Model
= Swappable dependency

Tools and contracts
= Stable application layer
```

---

# Part B — Pydantic AI

# 22. What Is Pydantic AI?

Pydantic AI เป็น Agent Framework จากทีม Pydantic

เน้น:

```text id="68acuo"
Python types
Validation
Structured outputs
Dependency injection
Observability
```

Mental Model:

```text id="9d2ni0"
Pydantic AI Agent
= Model
+ Instructions
+ Typed tools
+ Typed result contract
+ Agent loop
```

Agent Loop พื้นฐานยังเหมือน Strands:

```text id="iwzlmv"
Model
→ Tool
→ Observation
→ Model
```

---

# 23. Pydantic AI Imports

```python id="3334t4"
from pydantic_ai import Agent
from pydantic_ai.mcp import MCPToolset
from fastmcp.client.transports import (
    StdioTransport,
)
```

หน้าที่:

```text id="wp2mjz"
Agent
→ Agent definition และ runtime

MCPToolset
→ Tool collection จาก MCP

StdioTransport
→ เปิด MCP server ผ่าน subprocess
```

---

# 24. Step 1 — Create a Pydantic AI Agent

```python id="x6f45w"
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

Model String:

```text id="rfksqj"
openai-chat:gpt-5.4-mini
```

ประกอบด้วย:

```text id="j5cyr4"
openai-chat
→ Provider/API adapter

gpt-5.4-mini
→ Model ID
```

---

# 25. Why Use `openai-chat:`?

การระบุ:

```text id="lvnf0i"
openai-chat:
```

ทำให้ Framework ใช้ Chat Completions API อย่างชัดเจน

ข้อดี:

```text id="hgo8fr"
พฤติกรรมชัดเจน
ใช้กับ OpenAI-compatible endpoints ได้ง่าย
ลดผลกระทบจาก Default API ที่อาจเปลี่ยนใน Version ใหม่
```

หลัก:

```text id="yz7gke"
Explicit provider protocol
ลด Version ambiguity
```

---

# 26. Step 2 — Run Pydantic AI

```python id="ngl91q"
result = await agent.run(
    "Say hello in Spanish."
)

print(
    result.output
)
```

`result` เป็น Run Result Object

สามารถมี:

```text id="2b7zsd"
Final output
Messages
Usage
Model information
Run metadata
```

คำตอบสุดท้ายอยู่ใน:

```python id="o11hi8"
result.output
```

---

# 27. Sync and Async in Pydantic AI

Async:

```python id="smqx8a"
await agent.run(...)
```

Sync:

```python id="jkl3p2"
agent.run_sync(...)
```

Notebook ใช้ Async เพราะ Jupyter มี Event Loop อยู่แล้ว

```text id="nu1s35"
Notebook
→ await agent.run()

Simple script
→ agent.run_sync()
หรือ asyncio.run(...)
```

---

# 28. Step 3 — Pydantic AI Function Tools

Course ใช้ Plain Python Functions:

```python id="jdm6jz"
def show_todos() -> list[dict]:
    """List every todo on the board."""

    return board.list_todos()
```

```python id="u78kmb"
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

```python id="m6cdt4"
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

# 29. Add Tools to Pydantic AI

```python id="obv5jy"
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

Pydantic AI สร้าง Tool Schema จาก:

```text id="j38sny"
Function signature
Parameter types
Docstring
Return type
```

ไม่จำเป็นต้องใช้ Decorator ใน Pattern นี้

---

# 30. `@agent.tool` vs `tools=[...]`

Pydantic AI รองรับ:

```python id="xy4kwq"
@agent.tool
def lookup(...):
    ...
```

และ:

```python id="1vvr0a"
Agent(
    ...,
    tools=[
        lookup
    ],
)
```

ความต่างเชิงออกแบบ:

```text id="j5vbab"
@agent.tool
→ Register กับ Agent Instance

tools=[function]
→ ส่ง Reusable Function ตอนสร้าง Agent
```

Course ใช้ `tools=[...]` เพราะ Worker Tools เดียวกันถูกนำไปใช้กับ Agent หลายตัวและ Script หลายรูปแบบ

---

# 31. Step 4 — Pydantic AI MCP

สร้าง Transport:

```python id="p0rd9h"
filesystem_transport = StdioTransport(
    command="npx",
    args=[
        "-y",
        "@modelcontextprotocol/"
        "server-filesystem",
        str(workspace),
    ],
    cwd=str(workspace),
    log_file=Path(
        os.devnull
    ),
)
```

สร้าง Toolset:

```python id="1yzlvp"
filesystem = MCPToolset(
    filesystem_transport,
    init_timeout=60,
)
```

---

# 32. `tools` vs `toolsets`

Pydantic AI แยก:

```text id="namy6s"
tools
→ Individual Python functions

toolsets
→ Collections of tools
   จาก MCP หรือ Dynamic provider
```

ตัวอย่าง:

```python id="011mjl"
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

การแยกนี้ทำให้เห็น Origin และ Lifecycle ของ Tool Sources ชัดเจนกว่า

---

# 33. MCP Lifecycle with `async with`

Pydantic AI ใช้:

```python id="fxczhs"
async with worker:
    result = await worker.run(
        "Please work the pending goal."
    )
```

Flow:

```text id="3q1xds"
Enter Agent context
→ Start MCP process
→ Initialize toolset
→ Discover tools
→ Run agent
→ Close transport
→ Stop child process
```

ถ้าไม่เปิด Context Manager อย่างถูกต้อง อาจเกิด:

```text id="d5xla9"
MCP tools unavailable
Transport not initialized
Child process leak
Shutdown errors
```

---

# 34. Why Redirect MCP Logs?

Course ใช้:

```python id="fapwgt"
log_file=Path(
    os.devnull
)
```

เพื่อ:

```text id="0eqddl"
ลด Startup logs
เลี่ยงปัญหา stderr ของ Jupyter บน Windows
ทำให้ Notebook output สะอาด
```

ข้อเสีย:

```text id="q66mn6"
MCP errors อาจถูกซ่อน
Debugging ยากขึ้น
```

Production ควร Redirect ไปยัง Structured Log File มากกว่า Null Device

---

# 35. Step 5 — Pydantic AI Goal Loop

```python id="lrvrxm"
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

Run:

```python id="0uwg6z"
board.claim_todo(
    goal_id
)

async with worker:
    result = await worker.run(
        "Please work the pending goal "
        "on the board."
    )

print(
    result.output
)
```

Expected Loop:

```text id="0a2fcv"
Read Board
→ Plan Steps
→ Read notes.txt
→ Translate
→ Write spanish.txt
→ Complete Steps
→ Close Goal
```

---

# 36. Typed Output

Pydantic AI สามารถกำหนด Output Contract ด้วย Pydantic Model

```python id="gf8gj3"
from pydantic import BaseModel


class WorkerResult(BaseModel):
    goal_id: int
    output_file: str
    language: str
    completed: bool
```

สร้าง Agent:

```python id="i936pl"
agent = Agent(
    MODEL,
    instructions=INSTRUCTIONS,
    output_type=WorkerResult,
)
```

Result:

```python id="gy5wnv"
result = await agent.run(task)

worker_result = result.output
```

`worker_result` จะเป็น Object ที่ผ่าน Validation

---

# 37. Benefits of Typed Output

```text id="v3hv2d"
Predictable fields
IDE autocomplete
Runtime validation
Clear downstream contract
Reduced text parsing
Simpler API integration
```

เหมาะกับ:

```text id="jb5uo5"
FastAPI responses
Workflow routing
Database records
Evaluation
Inter-agent communication
```

---

# 38. Typed Output Is Not Real-world Proof

Typed Result อาจบอกว่า:

```python id="9j8fef"
WorkerResult(
    goal_id=1,
    output_file="spanish.txt",
    language="Spanish",
    completed=True,
)
```

แต่ยังไม่ได้พิสูจน์ว่า:

```text id="hx3dmw"
spanish.txt มีอยู่จริง
ไฟล์ไม่ว่าง
เนื้อหาเป็นภาษาสเปน
คำแปลถูกต้อง
Board Steps ครบ
Goal ควรถูกปิด
```

ดังนั้น:

```text id="k16h2n"
Schema validation
≠ Environment validation
```

---

# 39. Pydantic AI and Logfire

Pydantic AI สามารถเชื่อม Logfire:

```python id="f1g6x2"
import logfire

logfire.configure()
logfire.instrument_pydantic_ai()
```

ช่วยดู:

```text id="tl5ksf"
Model requests
Tool calls
Validation
Errors
Latency
Token usage
```

Course ไม่รวม Dependency นี้ใน Shared Environment เพื่อให้ Environment เบา

แต่แนวคิดสำคัญคือ:

```text id="nhdi38"
Types
→ ตรวจ Data Contract

Observability
→ ตรวจ Execution Behavior
```

---

# 40. Pydantic AI Model Portability

```python id="c37xh9"
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
    instructions=INSTRUCTIONS,
)
```

สิ่งที่ยังคงเดิม:

```text id="h3259s"
Tools
Toolsets
Board
Instructions
Worker contract
```

---

# 41. Shared SQLite Board

ทั้ง Strands และ Pydantic AI ใช้ `board.py` เดียวกัน

```text id="ed02ni"
Same database
Same schema
Same functions
Same statuses
Same goal
```

เป้าหมายคือแยก:

```text id="zid8m2"
Framework behavior
ออกจาก
Application behavior
```

หาก Agent สองตัวมี Failure เหมือนกัน อาจเป็นเพราะ Board Contract ไม่ใช่ Framework

---

# 42. Worker Scripts

ไฟล์:

```text id="krtkzy"
strands_worker.py
pydantic_worker.py
```

Standalone:

```powershell id="jt10s5"
uv run strands_worker.py
```

```powershell id="wfvbad"
uv run pydantic_worker.py
```

Flow:

```text id="mwxkkk"
Reset Board
→ Clear spanish.txt
→ Add Goal
→ Claim Goal
→ Build Agent
→ Run Agent Loop
→ Show Board
→ Print spanish.txt
```

---

# 43. Shared-board Mode

Workers รองรับ:

```powershell id="tvl2gl"
uv run strands_worker.py `
  <taskId> `
  <boardPath>
```

และ:

```powershell id="ykffr0"
uv run pydantic_worker.py `
  <taskId> `
  <boardPath>
```

ใน Mode นี้ Worker:

```text id="vwpb56"
ไม่ Reset Shared Board
รับ Task ID จาก Orchestrator
ใช้ Board Path ที่ถูกส่งมา
ใช้ Directory เดียวกันเป็น Workspace
Claim Task ที่กำหนด
ทำเฉพาะ Task นั้น
ปิด Task แล้วหยุด
```

นี่เป็น Preparation สำหรับ Week 5 Agent Loop ที่ใช้ Workers จากหลาย Framework ร่วมกัน

---

# 44. Import-time Configuration

Worker ตั้ง:

```python id="ep2vqg"
os.environ.setdefault(
    "BOARD_PATH",
    sys.argv[2],
)
```

ก่อน:

```python id="wr1apk"
import board
```

เพราะ `board.py` อ่าน `BOARD_PATH` ตอน Module ถูก Import

ถ้าตั้ง Environment Variable หลัง Import:

```text id="d99y8l"
Module อาจยังใช้ Path เก่า
```

หลัก:

```text id="4kcqpp"
Import-time configuration
ต้องกำหนดก่อน import
```

---

# 45. State Surfaces

Lab นี้มี State อย่างน้อยสามส่วน:

```text id="49y4yf"
Framework run state
→ Messages และ Tool Loop

SQLite board state
→ Goals, Steps และ Status

Filesystem state
→ notes.txt และ spanish.txt
```

State เหล่านี้อาจไม่ตรงกัน

ตัวอย่าง:

```text id="s5xqfv"
Agent ตอบว่าเสร็จ
แต่ Goal ยัง in_progress

Goal เป็น done
แต่ spanish.txt ไม่มี

File มีอยู่
แต่คำแปลผิด

Steps เป็น done
แต่ Goal ยัง pending
```

---

# 46. Shared Business Weakness

ทั้ง Strands และ Pydantic AI ใช้:

```python id="c2qtvj"
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

Function ไม่ได้ตรวจ:

```text id="a4aj6n"
Task ID มีอยู่จริง
Task ยังไม่ Done
Goal มี Steps ค้างอยู่หรือไม่
spanish.txt ถูกสร้างหรือไม่
ไฟล์ไม่ว่างหรือไม่
คำแปลถูกต้องหรือไม่
```

Framework ทั้งสองจึงสามารถปิด Goal ผิดได้เหมือนกัน

---

# 47. Prompt vs Invariants

Prompt บอกว่า:

```text id="f6arlc"
สร้าง Steps
ทำ Steps ให้ครบ
ปิด Goal หลังงานเสร็จ
```

แต่ Model อาจไม่ทำตาม

Code ควรบังคับ:

```text id="tdbykz"
ห้ามปิด Goal หาก Steps ค้าง
ห้ามปิด Goal หาก Output ไม่มี
ห้าม Claim Task ซ้ำ
ห้ามเขียนไฟล์นอก Workspace
```

หลัก:

```text id="li0pan"
Model handles flexible decisions
Code enforces hard rules
```

---

# 48. Duplicate Claim Risk

Workers ใช้:

```python id="48r6ep"
board.claim_todo(
    TASK_ID
)
```

ถ้า Workers หลายตัวเรียกพร้อมกัน Task เดียวกัน:

```text id="e2ssvb"
Worker A เริ่มงาน
Worker B เริ่มงานเดียวกัน
```

SQLite WAL ช่วย Concurrency แต่ไม่ป้องกัน Duplicate Work

Safer SQL:

```sql id="jzo9y3"
UPDATE todos
SET status = 'in_progress'
WHERE id = ?
  AND status = 'pending'
```

แล้วตรวจ:

```text id="heaybw"
affected rows == 1
```

---

# 49. Workspace Risk

MCP Server จำกัด Root ไปยัง Workspace

ข้อดี:

```text id="ed3pmk"
ลดพื้นที่ที่ Agent เข้าถึง
แยกไฟล์ของงาน
```

แต่ภายใน Root Agent อาจ:

```text id="go8me7"
เขียนทับ notes.txt
ลบไฟล์
สร้างไฟล์จำนวนมาก
อ่านคำสั่งอันตรายในไฟล์
```

ดังนั้น:

```text id="lr41qd"
Filesystem root
≠ Complete security sandbox
```

---

# 50. Strands vs Pydantic AI

| ประเด็น          | Strands                   | Pydantic AI                           |
| ---------------- | ------------------------- | ------------------------------------- |
| Agent            | `strands.Agent`           | `pydantic_ai.Agent`                   |
| Model            | Model Object              | Model String หรือ Object              |
| Instruction      | `system_prompt`           | `instructions`                        |
| Run              | `invoke_async()`          | `run()`                               |
| Result           | Agent result              | `result.output`                       |
| Function tools   | `@tool`                   | Plain functions หรือ Decorator        |
| MCP              | `MCPClient`               | `MCPToolset`                          |
| MCP registration | `tools=[...]`             | `toolsets=[...]`                      |
| MCP lifecycle    | Framework-managed         | `async with agent`                    |
| จุดเด่น          | Minimal loop, portability | Types, validation, observability      |
| Main caveat      | Model default ambiguity   | Valid schema ไม่เท่ากับ valid outcome |

---

# 51. Comparison with Google ADK

## Google ADK

```text id="n2nsuo"
LlmAgent
Runner
Function Tools
McpToolset
Session services
```

## Strands

```text id="lxx0gq"
Agent
OpenAIModel
@tool
MCPClient
```

## Pydantic AI

```text id="e37stf"
Agent
Model string
Plain typed tools
MCPToolset
Typed result
```

Agent Pattern เหมือนกัน:

```text id="1bmi6d"
Model
→ Tool
→ Observation
→ Model
→ Completion
```

---

# 52. Major Design Difference

Google ADK แยก:

```text id="rldm9i"
Agent definition
ออกจาก
Runner
```

Strands และ Pydantic AI ให้ Agent Object รัน Loop ได้โดยตรง

```text id="9jx62f"
Strands:
agent.invoke_async()

Pydantic AI:
agent.run()
```

ผล:

```text id="xz46ka"
ADK
→ Runtime services เห็นชัด

Strands/Pydantic AI
→ Developer API กระชับกว่า
```

---

# 53. When to Choose Strands

เหมาะเมื่อ:

```text id="jmigcy"
ต้องการ Agent API ขนาดเล็ก
ต้องการ Model-driven loop
ต้องเปลี่ยน Model Provider บ่อย
ใช้ AWS ecosystem
ต้องการรวม MCP กับ Tool Surface เดียว
```

จุดแข็ง:

```text id="pw6hkr"
สร้าง Agent สั้น
Loop ชัด
Model portability สูง
MCP integration กระชับ
```

จุดที่ต้องระวัง:

```text id="kovguq"
ต้องระบุ Model ชัด
Tool docstrings ต้องดี
ไม่มี Business validation ให้อัตโนมัติ
```

---

# 54. When to Choose Pydantic AI

เหมาะเมื่อ:

```text id="sles0c"
ใช้ Pydantic หรือ FastAPI อยู่แล้ว
ต้องการ Typed outputs
ต้องการ Validation
ต้องการ Dependency Injection
ต้องการ Logfire observability
```

จุดแข็ง:

```text id="76ke6z"
Types throughout the flow
Validated result objects
Reusable plain functions
Clear MCP toolsets
Strong observability story
```

จุดที่ต้องระวัง:

```text id="5il6pg"
MCP lifecycle ต้องจัดการให้ถูก
Schema complexity อาจเพิ่ม retries
Type-valid output อาจยังผิดในโลกจริง
```

---

# 55. Misconceptions Corrected

## ความเข้าใจคลาดเคลื่อนที่ 1

> Strands ใช้ได้เฉพาะ AWS Models

**ข้อเท็จจริง:**
รองรับ OpenAI และ Providers อื่นผ่าน Model Adapters

---

## ความเข้าใจคลาดเคลื่อนที่ 2

> `Agent()` ของ Strands จะเลือก OpenAI ให้เอง

**ข้อเท็จจริง:**
Course ระบุ Model ชัดเพื่อหลีกเลี่ยง Default Bedrock

---

## ความเข้าใจคลาดเคลื่อนที่ 3

> Pydantic AI Tools ต้องมี Decorator

**ข้อเท็จจริง:**
สามารถส่ง Plain Python Functions ผ่าน `tools=[...]`

---

## ความเข้าใจคลาดเคลื่อนที่ 4

> Typed Output พิสูจน์ว่างานสำเร็จ

**ข้อเท็จจริง:**
พิสูจน์ Data Shape ไม่ใช่ External Outcome

---

## ความเข้าใจคลาดเคลื่อนที่ 5

> MCP API เหมือนกันทุก Framework

**ข้อเท็จจริง:**
Protocol เหมือนกัน แต่ Registration และ Lifecycle ต่างกัน

---

## ความเข้าใจคลาดเคลื่อนที่ 6

> Strands และ Pydantic AI มี Agent Loop คนละชนิด

**ข้อเท็จจริง:**
แก่นยังเป็น Model–Tool–Observation Loop เหมือนกัน

---

## ความเข้าใจคลาดเคลื่อนที่ 7

> เปลี่ยน Framework แล้ว Board Risks หาย

**ข้อเท็จจริง:**
Task ownership และ Artifact validation ยังเป็นหน้าที่ Application

---

## ความเข้าใจคลาดเคลื่อนที่ 8

> Goal เป็น Done แปลว่างานสำเร็จ

**ข้อเท็จจริง:**
ต้องตรวจ Board Invariants และ Output File

---

# 56. Risks Identified

## 56.1 Premature Completion

Agent ปิด Goal ก่อน Steps ครบ

## 56.2 Missing Artifact

Goal Done แต่ไม่มี `spanish.txt`

## 56.3 Invalid Translation

File ถูกสร้างแต่เนื้อหาผิด

## 56.4 Duplicate Task Claim

Workers ทำงานเดียวกันพร้อมกัน

## 56.5 Tool Argument Error

Model ส่ง Goal ID หรือ Task ID ผิด

## 56.6 MCP Startup Failure

Node Process เริ่มไม่ได้

## 56.7 MCP Process Leak

Connection Lifecycle ถูกจัดการไม่ถูกต้อง

## 56.8 Hidden MCP Errors

Logs ถูกส่งไป Null Device

## 56.9 File Mutation

Agent เขียนทับ Input หรือ Artifact

## 56.10 Model-provider Drift

เปลี่ยน Model แล้ว Tool Selection เปลี่ยน

## 56.11 Schema-only Confidence

Typed Result ดูน่าเชื่อถือแต่ Environment ไม่ตรง

## 56.12 State Divergence

Framework, Board และ Filesystem State ไม่ตรงกัน

---

# 57. Production Improvements

```text id="wyuxdm"
Atomic task claiming
Goal completion validator
Required file checks
Translation validation
Workspace per task
Filesystem permissions
MCP timeouts
MCP structured logging
Tool-call limits
Model-call limits
Structured tool results
Artifact hashes
Durable audit logs
Acceptance tests
```

---

# 58. Suggested Exercise — Goal Completion Gate

สร้าง Tool แยก:

```python id="ghnu0i"
def complete_goal(
    goal_id: int,
    result: str,
) -> dict:
    ...
```

ตรวจ:

```text id="0m928n"
ทุก Step เป็น done
spanish.txt มีอยู่
ไฟล์ไม่ว่าง
Output เป็น UTF-8
```

ก่อนปิด Goal

---

# 59. Suggested Exercise — Typed Pydantic Result

```python id="3spm4t"
class WorkerResult(BaseModel):
    goal_id: int
    output_file: str
    completed_steps: int
    success: bool
```

จากนั้นเปรียบเทียบ:

```text id="2h92yh"
Declared success
กับ
Actual Board and File State
```

---

# 60. Suggested Exercise — Strands Model Swap

เปลี่ยน Model Endpoint แล้วบันทึก:

```text id="t63n8c"
Steps created
Tool-call order
Number of model calls
Completion behavior
Translation quality
```

---

# 61. Suggested Exercise — MCP Failure Comparison

ทำให้ `npx` หรือ Filesystem Server เริ่มไม่ได้

เปรียบเทียบ:

```text id="4df7oq"
Strands error
Pydantic AI error
Cleanup behavior
User-facing message
```

---

# 62. Suggested Exercise — Framework Benchmark

เก็บ Metrics:

```text id="87ih7k"
Execution time
Model calls
Tool calls
Tokens
Steps created
Goal status
Artifact existence
Artifact quality
Error recovery
```

อย่าประเมินจาก Final Text อย่างเดียว

---

# 63. Patterns Learned

## Model-driven Loop

```text id="f2n31q"
Model
→ Action
→ Observation
→ Next action
```

## Reusable Tool Contract

```text id="22htgu"
Same Python behavior
→ Different framework adapters
```

## MCP Toolset Pattern

```text id="6msl4g"
External server
→ Protocol
→ Framework tool surface
```

## Model Portability Pattern

```text id="0wgi32"
Swap model adapter
→ Keep application contract
```

## Typed Result Pattern

```text id="xbebwo"
Model output
→ Schema validation
→ Typed Python object
```

## Shared-board Worker Pattern

```text id="cfxs5v"
Task ID
+ Board Path
→ Framework worker subprocess
```

---

# 64. Connection to Week 5 Lab 1

Lab 1 ใช้ Google ADK:

```text id="ysko4j"
LlmAgent
Runner
McpToolset
SQLite Board
```

Lab 2 ใช้ Framework API ที่กระชับขึ้น:

```text id="un345k"
Strands Agent
หรือ
Pydantic AI Agent
```

แต่ Contract เดิมพิสูจน์ว่า:

```text id="j1bd3b"
Framework API ต่างกัน

Agent pattern
ยังเหมือนเดิม
```

---

# 65. Connection to Week 4

## LangChain `create_agent`

คล้ายกับ:

```text id="h54i6u"
Strands Agent
Pydantic AI Agent
```

เพราะ Framework จัด Model–Tool Loop ให้

## LangGraph

ยังช่วยอธิบายสิ่งที่เกิดภายใน:

```text id="5fkilx"
State
Model node
Tool node
Cycle
Termination
```

## Deep Agents

Planning Tool ใน Lab 2 ถูกสร้างผ่าน SQLite Board:

```text id="rp6kc6"
plan_steps
→ External planning state
```

แทน Built-in Todo Middleware

---

# 66. Lab 2 Mental Model

## Strands

```text id="vdqzfv"
Goal
→ Strands Agent
→ @tool board functions
→ MCPClient filesystem
→ Board and file updates
→ Completion
```

## Pydantic AI

```text id="2chqkt"
Goal
→ Pydantic AI Agent
→ Plain typed functions
→ MCPToolset filesystem
→ Validated result
→ Board and file updates
```

Shared Reality:

```text id="tsomv7"
SQLite Board
+
Workspace Files
=
Actual external state
```

---

# 67. Final Lessons

## Lesson 1

Agent Frameworks ส่วนใหญ่ใช้ Model–Tool Loop แบบเดียวกัน

## Lesson 2

ความต่างหลักอยู่ที่ API, Type System, Lifecycle และ Observability

## Lesson 3

Strands เน้น Minimal Loop และ Model Portability

## Lesson 4

Pydantic AI เน้น Typed Contracts และ Validation

## Lesson 5

Strands ต้องระบุ Model ชัดเพื่อหลีกเลี่ยง Default Provider ที่ไม่ตั้งใจ

## Lesson 6

Strands ใช้ `@tool` สำหรับ Python Functions

## Lesson 7

Pydantic AI สามารถใช้ Plain Functions เป็น Tools ได้

## Lesson 8

Strands รวม MCP Client ไว้ใน `tools`

## Lesson 9

Pydantic AI แยก MCP เป็น `toolsets`

## Lesson 10

Pydantic AI MCP ต้องจัด Connection Lifecycle ผ่าน Context Manager

## Lesson 11

Typed Output ช่วยให้ Downstream Code ปลอดภัยขึ้น แต่ไม่พิสูจน์ External Outcome

## Lesson 12

Model สามารถเปลี่ยนได้โดยไม่ต้องเปลี่ยน Board และ Tool Contract

## Lesson 13

Board และ Files ทำหน้าที่เป็น Shared External State ข้าม Framework

## Lesson 14

Business Invariants ยังต้องถูกบังคับด้วย Application Code

## Lesson 15

Framework Comparison ควรวัดจาก Trace, State และ Artifact ไม่ใช่ Final Answer เท่านั้น

---

# 68. Memory Summary

```text id="aqx65e"
Week 5 Lab 2 มี:
AWS Strands
Pydantic AI

Folder:
5_agent_frameworks/2_strands_pydantic

Notebooks:
strands_lab.ipynb
pydantic_lab.ipynb

Worker scripts:
strands_worker.py
pydantic_worker.py

Shared contract:
Read Goal
Plan Steps
Read notes.txt
Translate
Write spanish.txt
Complete Steps
Complete Goal

Strands:
Agent
OpenAIModel
@tool
MCPClient
invoke_async()

Strands model:
ต้องระบุ explicit

ไม่ระบุ:
อาจใช้ Bedrock default

Strands MCP:
ใส่ใน tools list

Strands strength:
Minimal loop
Model portability

Pydantic AI:
Agent
Model string
Plain typed functions
MCPToolset
run()
result.output

Model string:
openai-chat:gpt-5.4-mini

Pydantic tools:
Plain function
หรือ @agent.tool

Pydantic MCP:
toolsets=[filesystem]

MCP lifecycle:
async with agent

Pydantic strength:
Types
Validation
Structured output
Logfire observability

Typed output:
ตรวจ Schema

ไม่ตรวจ:
File existence
Translation quality
Board correctness

Shared external state:
SQLite board
Filesystem workspace

Framework run state:
แยกจาก Board และ Files

Shared weakness:
complete_task ไม่มี validation

Shared concurrency risk:
claim_todo ไม่ atomic

Worker modes:
Standalone
Shared-board Day 5 mode

Shared-board arguments:
taskId
boardPath

ต้องตั้ง BOARD_PATH:
ก่อน import board

Strands เหมาะกับ:
Lightweight loop
Model portability

Pydantic AI เหมาะกับ:
Typed Python systems
FastAPI
Validated outputs
Observability

Application ยังต้องดูแล:
Task ownership
Artifact validation
Permissions
Timeouts
Budgets
Acceptance tests
```

---

# 69. Next Episode

Lab ถัดไปจะใช้ Framework ชุดใหม่กับ Contract เดิม

สิ่งที่ควรจับตา:

```text id="n7ofx9"
Agent constructor
Model abstraction
Tool declaration
MCP lifecycle
Team or multi-agent support
Observability
Model portability
```

คำถามสำคัญสำหรับ Lab ถัดไปคือ:

> เมื่อ Framework ทุกตัวสามารถสร้าง Model–Tool Loop ได้เหมือนกัน ความแตกต่างใดมีผลต่อระบบ Production จริงมากที่สุด ระหว่างความง่ายของ API, Type Safety, State Management, Multi-agent Support และ Observability?
