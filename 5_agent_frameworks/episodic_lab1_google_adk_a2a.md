# Episodic Learning Artifact

## Week 5 — Lab 1: Google ADK, MCP และ A2A

**หลักสูตร:** AI Engineer Agentic Track
**ผู้สอน:** Ed Donner
**Repository:** `ed-donner/agents`
**โฟลเดอร์:** `5_agent_frameworks/1_google_adk_a2a`
**Notebook:** `lab.ipynb`
**ไฟล์ประกอบ:** `board.py`, `worker.py`, `task_worker/agent.py`, `a2a_demo/server.py`, `a2a_demo/spanish_concierge/agent.py`, `SWAP_AI.md`
**หัวข้อหลัก:** Google ADK, Agent Loop, SQLite Todo Board, Function Tools, MCP และ Agent2Agent Protocol
**สถานะ:** เรียนและสรุป Lab 1 แล้ว

---

# 1. Context

Week 4 ศึกษา Agentic System ผ่าน LangChain และ LangGraph ตั้งแต่ Building Blocks ไปจนถึง Product-level Application:

```text
Week 4 Lab 1
Model
Messages
Tools
Manual tool loop

Week 4 Lab 2
State
Nodes
Edges
Checkpointing

Week 4 Lab 3
create_agent
Middleware
MCP

Week 4 Lab 4
Planning
Filesystem
Sub-agents
Skills

Week 4 Lab 5
Worker
Evaluator
Human approval
Persistent environment
```

Week 5 เปลี่ยนเป้าหมายจากการเรียน Framework เดียวอย่างลึก มาเป็นการเปรียบเทียบ Agent Framework หลายตัวผ่าน Contract เดียวกัน

Pattern ที่ใช้ตลอด Week 5:

```text
1. Create the agent
2. Run it
3. Add tools
4. Add MCP
5. Put it in a loop with a goal
```

Lab 1 ใช้ Google Agent Development Kit หรือ Google ADK และปิดท้ายด้วย Agent2Agent Protocol หรือ A2A

---

# 2. Learning Objectives

หลังจบ Lab 1 สามารถอธิบายได้ว่า:

1. `LlmAgent` คืออะไร
2. Agent ที่ไม่มี Tools ต่างจาก Tool-using Agent อย่างไร
3. `Runner` แยกจาก Agent อย่างไร
4. `InMemoryRunner` เหมาะกับงานแบบใด
5. `run_debug()` และ `run_async()` ต่างกันอย่างไร
6. Python Function กลายเป็น ADK Tool อย่างไร
7. Function Name, Docstring และ Type Hints มีผลต่อ Tool Schema อย่างไร
8. SQLite Todo Board ทำหน้าที่เป็น Shared Work State อย่างไร
9. Goal และ Step มีความสัมพันธ์กันอย่างไร
10. Board แตกต่างจาก Agent Session และ Memory อย่างไร
11. MCP เชื่อม Agent กับ Filesystem Tool Server อย่างไร
12. Function Tool และ MCP Tool ต่างกันอย่างไร
13. Agent Loop แบบ Read–Plan–Act–Complete ทำงานอย่างไร
14. `worker.py` แบ่งหน้าที่กับ Agent อย่างไร
15. `adk web` ใช้สังเกต Agent Execution อย่างไร
16. A2A แตกต่างจาก MCP อย่างไร
17. `to_a2a()` เปิด Agent เป็น Remote Service อย่างไร
18. `RemoteA2aAgent` ใช้ Remote Agent เหมือน Local Sub-agent อย่างไร
19. Agent Card ใช้สำหรับ Discovery อย่างไร
20. จุดใดต้องใช้ Deterministic Validation แทน Prompt Guidance

---

# 3. Prerequisites

ควรเข้าใจแนวคิดต่อไปนี้:

```text
LLM call
System instruction
Tool schema
Tool request
Tool result
Agent loop
Session state
External state
Filesystem
MCP
Sub-agent
```

Environment ที่ใช้:

```text
Python >= 3.12
Google ADK 2.x
Node.js
npx
SQLite
```

Environment Variables:

```env
GOOGLE_API_KEY=...
```

Lab ใช้ Gemini API โดยตั้งค่า:

```python
os.environ.setdefault(
    "GOOGLE_GENAI_USE_VERTEXAI",
    "FALSE",
)
```

ติดตั้ง Dependencies:

```powershell
uv sync
```

ตรวจ Node:

```powershell
node --version
npx --version
```

Warm Filesystem MCP Server:

```powershell
npx -y @modelcontextprotocol/server-filesystem .
```

---

# 4. Week 5 Core Pattern

ทุก Framework ใน Week 5 จะถูกมองผ่านห้าขั้นเดียวกัน:

```text
Create
→ Run
→ Add tools
→ Add MCP
→ Give a goal and loop
```

ความแตกต่างระหว่าง Framework มักอยู่ที่:

```text
ชื่อ Class
Runner API
Tool registration
Session handling
Tracing
MCP integration
Multi-agent integration
```

แต่ Agent Pattern พื้นฐานยังเหมือนเดิม:

```text
Goal
→ Model decision
→ Tool action
→ Observation
→ Next decision
→ Completion
```

---

# 5. Step 1 — Create the Agent

ตัวอย่าง:

```python
from google.adk.agents import LlmAgent

MODEL = "gemini-3.1-flash-lite"

agent = LlmAgent(
    model=MODEL,
    name="assistant",
    instruction=(
        "You are a concise, friendly assistant. "
        "Reply in a single short sentence."
    ),
)
```

องค์ประกอบ:

```text
model
→ Model ที่ Agent ใช้

name
→ Identity ของ Agent

instruction
→ System-level guidance
```

Mental Model:

```text
LlmAgent
= Model
+ Identity
+ Instruction
+ Optional tools
+ Optional sub-agents
```

---

# 6. `LlmAgent` Is Not the Runner

ต้องแยก:

```text
LlmAgent
= นิยามว่า Agent คือใคร
และทำงานอย่างไร

Runner
= ระบบที่นำ Agent ไปรัน
จัดการ Session และ Events
```

เปรียบเทียบ:

```text
Agent
= นักแสดง

Runner
= เวที ผู้กำกับ และระบบจัดการการแสดง
```

การสร้าง `LlmAgent` อย่างเดียวไม่ได้ทำให้ Execution เริ่มขึ้น

---

# 7. Agent Without Tools

Agent ตัวแรกมีเพียง Model และ Instruction

Flow:

```text
User message
→ Model
→ Text response
```

ยังไม่มี:

```text
Tool calls
External actions
Multi-step execution
Environment feedback
```

ดังนั้น Agent ตัวนี้ยังใกล้เคียงกับ LLM Call มากกว่า Autonomous Agent

---

# 8. Step 2 — Run the Agent

Notebook ใช้:

```python
from google.adk.runners import InMemoryRunner

result = await InMemoryRunner(
    agent=agent
).run_debug(
    "Say hello in Spanish.",
    verbose=True,
)
```

`InMemoryRunner` จัดเตรียม Runtime Services ใน Memory

เช่น:

```text
Session service
Artifact service
Memory service
Event handling
```

เหมาะกับ:

```text
Notebook
Learning
Prompt experiments
Local debugging
```

ข้อจำกัด:

```text
Kernel restart
→ State หาย

Process restart
→ Session หาย
```

---

# 9. `run_debug()`

`run_debug()` เป็น Convenience Method

ทำงานประมาณ:

```text
Create session
→ Send message
→ Run agent
→ Print trace
→ Return events
```

ข้อดี:

```text
เริ่มทดลองได้เร็ว
เห็น Tool Calls
เห็น Model Responses
เหมาะกับ Notebook
```

ข้อจำกัด:

```text
ไม่ได้ให้ Application ควบคุม Lifecycle เต็มรูปแบบ
ไม่เหมาะเป็น Production Runtime หลัก
```

---

# 10. `run_async()`

สำหรับ Application ที่ต้องควบคุม Session เอง:

```python
from google.genai import types

runner = InMemoryRunner(
    agent=agent
)

session = await runner.session_service.create_session(
    app_name=runner.app_name,
    user_id="user-1",
)

message = types.UserContent(
    "Say hi in French."
)

async for event in runner.run_async(
    user_id="user-1",
    session_id=session.id,
    new_message=message,
):
    print(
        event.author,
        event.content,
    )
```

Developer เป็นผู้ควบคุม:

```text
User ID
Session ID
Input message
Event stream
UI updates
Logging
Artifacts
```

---

# 11. `run_debug()` vs `run_async()`

## `run_debug()`

```text
Convenient
Automatic session
Automatic trace output
เหมาะกับการทดลอง
```

## `run_async()`

```text
Application-controlled
Explicit session
Event streaming
เหมาะกับระบบจริงมากกว่า
```

สรุป:

```text
run_debug
= ทดลอง Agent

run_async
= สร้าง Application รอบ Agent
```

---

# 12. SQLite Todo Board

Week 5 ใช้ SQLite Todo Board เป็น Shared Work Substrate

Schema:

```sql
CREATE TABLE todos (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    parent_id INTEGER,
    task TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending',
    result TEXT NOT NULL DEFAULT ''
)
```

ชนิดของ Todo:

```text
Goal
→ parent_id = NULL

Step
→ parent_id = goal.id
```

สถานะ:

```text
pending
→ in_progress
→ done
```

---

# 13. Board Functions

```text
reset_board()
→ สร้าง Board ใหม่

add_goal()
→ เพิ่ม Goal หลัก

add_step()
→ เพิ่ม Step ใต้ Goal

list_todos()
→ อ่าน Goals และ Steps

claim_todo()
→ เปลี่ยนเป็น in_progress

complete_todo()
→ เปลี่ยนเป็น done และเก็บผล

show_board()
→ แสดง Board สำหรับมนุษย์
```

---

# 14. Goal and Steps

ตัวอย่าง:

```text
Goal #1
Read notes.txt, translate it,
and write spanish.txt

Step #2
Read notes.txt

Step #3
Translate the content

Step #4
Write spanish.txt

Step #5
Verify the output
```

ความสัมพันธ์:

```text
Goal
= Outcome ที่ต้องการ

Steps
= แผนการลงมือทำ
```

Agent เป็นผู้สร้าง Steps เองผ่าน Tool

---

# 15. Board Is External Application State

ต้องแยก State หลายชนิด:

```text
ADK Session State
→ Conversation และ Events

SQLite Board
→ Goals, Steps และ Status

Workspace Files
→ Input และ Output Artifacts

Model Context
→ ข้อมูลที่ Model เห็นในแต่ละ Call
```

Board ไม่ใช่:

```text
Model memory
Semantic memory
Conversation state
```

มันคือ Application Database ที่ Agent เข้าถึงผ่าน Tools

---

# 16. Board Persistence

แม้ใช้ `InMemoryRunner`:

```text
Session อาจหายหลัง Process ปิด
```

แต่:

```text
board.sqlite
notes.txt
spanish.txt
```

ยังอยู่บน Disk

ดังนั้น:

```text
Session persistence
≠ Board persistence
≠ File persistence
```

แต่ละ Surface มี Lifecycle ของตัวเอง

---

# 17. SQLite Concurrency Settings

Board เปิด:

```text
WAL mode
busy_timeout = 5000 ms
```

ประโยชน์:

```text
รองรับหลาย Readers
ลด Database locked errors
ช่วยให้หลาย Processes ใช้งานได้ดีขึ้น
```

แต่:

```text
Database concurrency
≠ Work coordination
```

WAL ไม่ได้รับประกันว่า Workers สองตัวจะไม่ Claim Goal เดียวกัน

---

# 18. Step 3 — Add Function Tools

Tools หลัก:

```python
def show_todos() -> list[dict]:
    """List every todo on the board."""
    return board.list_todos()


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


def complete_task(
    task_id: int,
    result: str,
) -> dict:
    """Mark a todo as done and record its result."""

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

# 19. Tool Schema

ADK อ่าน:

```text
Function name
Docstring
Parameters
Type hints
Return structure
```

แล้วสร้าง Tool Definition ที่ Model เข้าใจได้

ตัวอย่าง:

```text
show_todos
→ ไม่มี Argument

plan_steps
→ goal_id: integer
→ steps: list[string]

complete_task
→ task_id: integer
→ result: string
```

---

# 20. Tool Description Is Guidance

Model ใช้ Docstring ตัดสินว่า:

```text
ควรเรียก Tool ใด
ส่ง Argument อะไร
Tool จะคืนอะไร
```

แต่:

```text
Docstring
≠ Security policy
≠ Business validation
```

Tool Implementation ต้องบังคับ Rules ที่สำคัญเอง

---

# 21. Model Proposes, Application Executes

Flow:

```text
Model:
เรียก plan_steps(goal_id=1, ...)

ADK Runtime:
ตรวจ Tool
Execute Python Function

Board:
บันทึก Steps

Tool Result:
ส่งกลับ Model
```

หลักยังเหมือน Week 4:

```text
Model proposes
Application authorizes and executes
Environment changes
Observation returns
```

---

# 22. Tool That Reaches the Real World

Notebook แสดง Push Notification Tool

Concept:

```python
def send_push_notification(
    message: str,
) -> str:
    """Send a notification to the user's phone."""
    ...
```

นี่คือ Side-effect Tool

แตกต่างจาก `show_todos()`:

```text
show_todos
→ Read-only

send_push_notification
→ External side effect
```

Side-effect Tool ควรมี:

```text
Timeout
Status validation
Retry policy
Idempotency
Human approval
Audit log
```

---

# 23. Step 4 — Add MCP

MCP ถูกอธิบายว่า:

```text
Tools ที่ Agent ไม่ได้เขียนเอง
แต่เชื่อมผ่าน Protocol มาตรฐาน
```

Lab ใช้ Filesystem MCP Server:

```python
from google.adk.tools.mcp_tool import (
    McpToolset,
    StdioConnectionParams,
)

from mcp import StdioServerParameters

filesystem = McpToolset(
    connection_params=StdioConnectionParams(
        server_params=StdioServerParameters(
            command="npx",
            args=[
                "-y",
                "@modelcontextprotocol/server-filesystem",
                str(WORKSPACE),
            ],
            cwd=str(WORKSPACE),
        ),
        timeout=60,
    ),
)
```

---

# 24. MCP Architecture

```text
Google ADK Agent
        ↓
McpToolset
        ↓ stdio
Node.js MCP Server
        ↓
Local workspace
```

Tool Server ให้ความสามารถ เช่น:

```text
List files
Read file
Write file
Edit file
Create directory
```

Tool Names จริงขึ้นกับ Version ของ Server

---

# 25. Function Tool vs MCP Tool

## Function Tool

```text
Python Agent Process
→ Python Function
→ Result
```

Developer เขียน Function เอง

## MCP Tool

```text
Agent Process
→ MCP Client
→ Protocol
→ External Tool Server
→ Result
```

Implementation อาจ:

```text
อยู่ต่าง Process
ใช้คนละภาษา
พัฒนาโดยบุคคลอื่น
Reuse ได้หลาย Framework
```

---

# 26. MCP Is Not Multi-agent

MCP เชื่อม:

```text
Agent
→ Tools and resources
```

ไม่ใช่:

```text
Agent
→ Autonomous agent team
```

MCP Server อาจไม่มี Model หรือ Agent Loop เลย

ตัวอย่างใน Lab:

```text
Filesystem MCP Server
= File operations

ไม่ใช่
= File-research Agent
```

---

# 27. Workspace Scope

MCP Server ถูกจำกัดไปยัง:

```text
task_worker/workspace/
```

Agent จึงสามารถเข้าถึง:

```text
notes.txt
spanish.txt
```

ภายใน Workspace เท่านั้น

ข้อดี:

```text
ลดพื้นที่เข้าถึง
ลดความเสี่ยงเขียนไฟล์นอกงาน
แยก Artifact ของ Worker
```

แต่:

```text
Workspace root
≠ Container sandbox
```

Agent ยังอาจแก้หรือลบไฟล์ทั้งหมดภายใน Root ได้

---

# 28. Step 5 — Put It in a Loop with a Goal

Worker ตัวเต็ม:

```python
root_agent = LlmAgent(
    model=MODEL,
    name="task_worker",
    description=(
        "Works one goal from the SQLite "
        "board using its files."
    ),
    instruction=INSTRUCTIONS,
    tools=[
        show_todos,
        plan_steps,
        complete_task,
        filesystem,
    ],
)
```

Instruction:

```text
อ่าน Goal ที่รออยู่
สร้าง Steps ที่จำเป็น
ทำงานผ่าน File Tools
ปิด Step เมื่อเสร็จ
ปิด Goal หลังทุก Step เสร็จ
```

---

# 29. Agent Loop

Expected Loop:

```text
show_todos
    ↓
Read pending goal
    ↓
plan_steps
    ↓
Read notes.txt
    ↓
Translate text
    ↓
Write spanish.txt
    ↓
complete_task(step)
    ↓
Read board again
    ↓
Complete remaining steps
    ↓
complete_task(goal)
```

สรุป:

```text
Read
→ Plan
→ Act
→ Check off
→ Repeat
```

---

# 30. Where Autonomy Begins

ก่อน Step 5:

```text
User asks question
→ Agent uses one Tool
→ Agent answers
```

Step 5:

```text
Application gives Goal
→ Agent creates its own plan
→ Agent performs multiple actions
→ Agent updates external state
→ Agent decides when finished
```

นี่คือ Agentic Loop ที่มี Autonomy มากขึ้น

---

# 31. `worker.py`

หน้าที่ของ Application Code:

```text
Reset Board
Create Workspace
Remove old output
Seed Goal
Claim Goal
Start Runner
Display final Board
Display spanish.txt
```

Code:

```python
def seed() -> int:
    board.reset_board()
    WORKSPACE.mkdir(exist_ok=True)
    spanish_file.unlink(
        missing_ok=True
    )

    return board.add_goal(TASK)
```

---

# 32. Application vs Agent Responsibilities

## Application

```text
จัดการ Lifecycle
สร้างงาน
เลือก Workspace
Claim งาน
เริ่ม Worker
ตรวจ Artifact หลัง Run
```

## Agent

```text
อ่าน Goal
วางแผน Steps
เลือก File Tools
แปลเนื้อหา
สร้าง Output
ปิด Steps
ปิด Goal
```

หลัก:

```text
Application
= Authority and lifecycle

Agent
= Flexible execution
```

---

# 33. Run the Worker

```powershell
uv run worker.py
```

Flow:

```text
Reset
→ Seed
→ Claim
→ Agent runs
→ Board displayed
→ spanish.txt displayed
```

Seed อย่างเดียว:

```powershell
uv run worker.py --seed-only
```

ใช้เมื่อต้องการรันผ่าน ADK Web

---

# 34. ADK Web

เริ่ม:

```powershell
uv run adk web
```

จากนั้นเลือก:

```text
task_worker
```

Prompt:

```text
Please work the pending task on the board.
```

ADK Web ช่วยดู:

```text
Conversation
Agent Events
Tool calls
Tool arguments
Tool results
Execution timing
```

เหมาะกับ Development และ Debugging

ไม่ใช่ Production UI โดยอัตโนมัติ

---

# 35. Event Stream

เมื่อใช้ `run_async()` สามารถตรวจ Events:

```python
async for event in runner.run_async(...):
    for call in event.get_function_calls():
        print(
            call.name,
            call.args,
        )

    if event.is_final_response():
        print(event.content)
```

Event สามารถแสดง:

```text
Agent author
Tool request
Tool response
Final response
State transition
```

ช่วยให้เข้าใจ Agent Loop มากกว่าดูเพียง Final Answer

---

# 36. Trace Is Not Correctness

แม้ Trace แสดง:

```text
read_file
write_file
complete_task
```

ก็ยังไม่พิสูจน์ว่า:

```text
Translation ถูกต้อง
Output ไม่ว่าง
Encoding ถูกต้อง
Steps ครบ
Goal ควรถูกปิด
```

ต้องตรวจ Artifact และ Board State จริง

---

# 37. Weakness of `complete_task()`

Function ปัจจุบัน Mark Todo เป็น Done โดยตรง

ไม่ได้ตรวจ:

```text
Task ID มีอยู่จริง
Task ยังไม่ Done
Goal มี Steps ค้างอยู่หรือไม่
Output File ถูกสร้างหรือไม่
Result ไม่ว่างหรือไม่
```

Agent จึงอาจปิด Goal เร็วเกินไป

---

# 38. Safer Goal Completion

Concept:

```python
def complete_goal(
    goal_id: int,
    result: str,
) -> dict:

    unfinished = get_unfinished_steps(
        goal_id
    )

    if unfinished:
        raise ValueError(
            "Unfinished steps remain."
        )

    output = (
        WORKSPACE
        / "spanish.txt"
    )

    if not output.exists():
        raise ValueError(
            "Required output is missing."
        )

    if not output.read_text(
        encoding="utf-8"
    ).strip():
        raise ValueError(
            "Output file is empty."
        )

    board.complete_todo(
        goal_id,
        result,
    )

    return {
        "goal_id": goal_id,
        "status": "done",
    }
```

---

# 39. Prompt vs Code Enforcement

Prompt บอกว่า:

```text
สร้าง Steps
ทำ Steps ให้ครบ
ปิด Goal หลังงานเสร็จ
```

แต่ Prompt เป็นเพียง Guidance

Code ควรบังคับ:

```text
ห้ามปิด Goal หาก Steps ค้าง
ห้ามปิด Goal หาก Output ไม่มี
ห้าม Claim งานซ้ำ
ห้ามเขียนไฟล์นอก Workspace
```

หลัก:

```text
LLM handles flexibility
Code enforces invariants
```

---

# 40. Atomic Claim Risk

`claim_todo()` ปัจจุบันเปลี่ยน Status โดย ID

ถ้า Workers สองตัวเรียกพร้อมกัน:

```text
Worker A claims Goal #1
Worker B claims Goal #1
```

ทั้งสองอาจทำงานเดียวกันซ้ำ

Safer SQL:

```sql
UPDATE todos
SET status = 'in_progress'
WHERE id = ?
  AND status = 'pending'
```

แล้วตรวจจำนวน Rows ที่ถูก Update

```text
1 row
→ Claim สำเร็จ

0 rows
→ งานถูก Claim แล้ว
```

---

# 41. Database Locking vs Business Locking

```text
SQLite WAL
→ ลดปัญหาการเข้าถึง Database พร้อมกัน

Atomic claim condition
→ ป้องกัน Workers รับงานเดียวกัน
```

ดังนั้น:

```text
Database concurrency
≠ Task ownership
```

ต้องมี Business Rule แยกต่างหาก

---

# 42. A Quick Look at A2A

A2A ย่อมาจาก:

```text
Agent2Agent Protocol
```

ใช้สำหรับ:

```text
Agent discovery
Remote task delegation
Cross-service communication
Agent capability sharing
```

Architecture:

```text
Local Agent
    ↓ HTTP/A2A
Remote Agent Service
```

---

# 43. MCP vs A2A

## MCP

```text
Agent
→ Tool Server
```

ปลายทางให้:

```text
Tools
Resources
Prompts
```

## A2A

```text
Agent
→ Remote Agent
```

ปลายทางอาจ:

```text
วางแผน
ใช้ Tools
ทำงานหลายขั้น
สร้างผลลัพธ์
```

Mental Model:

```text
MCP
= ยืมเครื่องมือ

A2A
= มอบหมายงานให้ผู้เชี่ยวชาญ
```

---

# 44. Expose an Agent with `to_a2a()`

Translator Agent:

```python
from google.adk.agents import LlmAgent

root_agent = LlmAgent(
    model="gemini-3.1-flash-lite",
    name="translator_agent",
    description=(
        "Translates English into Spanish."
    ),
    instruction=(
        "Translate English text into "
        "natural Spanish. Return only "
        "the translation."
    ),
)
```

Expose:

```python
from google.adk.a2a.utils.agent_to_a2a import (
    to_a2a,
)

a2a_app = to_a2a(
    root_agent,
    port=8001,
)
```

ผลลัพธ์คือ ASGI Application ที่สามารถรันผ่าน Uvicorn

---

# 45. Run the A2A Server

```powershell
uv run uvicorn server:a2a_app `
  --host localhost `
  --port 8001
```

Port ใน:

```python
to_a2a(
    root_agent,
    port=8001,
)
```

ต้องตรงกับ Uvicorn Port

เพราะ Port นี้ถูกประกาศใน Agent Card

---

# 46. Agent Card

Agent Card อยู่ที่:

```text
/.well-known/agent-card.json
```

ตรวจ:

```powershell
curl `
  http://localhost:8001/.well-known/agent-card.json
```

Agent Card อธิบาย:

```text
Name
Description
Service URL
Capabilities
Skills
Input formats
Output formats
Protocol metadata
```

Mental Model:

```text
Tool Schema
= Contract ของ Tool

Agent Card
= Contract ของ Remote Agent
```

---

# 47. Agent Card Is Not Trust

Agent Card ช่วย Discovery

แต่ไม่ได้พิสูจน์ว่า:

```text
Service เป็นของใคร
Agent ปลอดภัย
Agent ทำตามคำอธิบายจริง
ข้อมูลไม่ถูกดัดแปลง
```

Production ต้องเพิ่ม:

```text
TLS
Authentication
Authorization
Service identity
Agent-card validation
Audit logging
```

---

# 48. `RemoteA2aAgent`

Local Concierge เชื่อม Remote Translator:

```python
from google.adk.agents.remote_a2a_agent import (
    AGENT_CARD_WELL_KNOWN_PATH,
    RemoteA2aAgent,
)

translator = RemoteA2aAgent(
    name="translator_agent",
    description=(
        "Remote Spanish translator."
    ),
    agent_card=(
        "http://localhost:8001"
        f"{AGENT_CARD_WELL_KNOWN_PATH}"
    ),
    use_legacy=False,
)
```

จากนั้นใช้เป็น Sub-agent:

```python
root_agent = LlmAgent(
    model="gemini-3.1-flash-lite",
    name="spanish_concierge",
    instruction=(
        "Delegate Spanish translation "
        "requests to translator_agent."
    ),
    sub_agents=[
        translator
    ],
)
```

---

# 49. A2A Execution Flow

```text
User
→ Spanish Concierge
→ Decide to delegate
→ RemoteA2aAgent
→ Fetch/Use Agent Card
→ Send A2A Request
→ Remote Translator
→ Spanish Result
→ Concierge
→ User
```

Local Agent ใช้ Remote Agent คล้าย Local Sub-agent

แต่การทำงานจริงผ่าน Network Boundary

---

# 50. Local Sub-agent vs Remote A2A Agent

## Local Sub-agent

```text
อยู่ Runtime หรือ Process เดียวกัน
Latency ต่ำกว่า
Deployment ง่ายกว่า
Failure Surface น้อยกว่า
```

## Remote A2A Agent

```text
อยู่คนละ Process หรือ Service
Scale แยกได้
Owner แยกได้
Language หรือ Framework ต่างได้
มี Network และ Security Boundary
```

---

# 51. A2A Failure Surfaces

```text
Remote service offline
Agent Card fetch failed
Protocol version mismatch
Timeout
Authentication failure
Malformed remote response
Remote model failure
```

ดังนั้น Local Agent ต้องมี:

```text
Timeout
Retry policy
Fallback
Clear error reporting
Circuit breaker
```

---

# 52. Multiple State Surfaces

Lab มี State อย่างน้อย:

```text
ADK Session
SQLite Board
Filesystem Workspace
Remote A2A Service
```

แต่ละ State อาจไม่ตรงกัน

ตัวอย่าง:

```text
Board says done
แต่ spanish.txt ไม่มี

File exists
แต่ Goal ยัง pending

Remote translator succeeded
แต่ Local request timed out

Session lost
แต่ Board ยัง in_progress
```

ระบบ Production ต้องมี Reconciliation

---

# 53. Model Portability

`SWAP_AI.md` แสดงการเปลี่ยน Model ผ่าน LiteLLM

```python
from google.adk.models.lite_llm import (
    LiteLlm,
)

model = LiteLlm(
    model="openai/gpt-5.4-mini"
)
```

แล้วใช้:

```python
root_agent = LlmAgent(
    model=model,
    ...
)
```

ข้อสังเกต:

```text
Model เปลี่ยนได้

Board contract
Tools
MCP server
Workspace
ยังเหมือนเดิม
```

นี่คือการแยก:

```text
Reasoning provider
ออกจาก
Application capabilities
```

---

# 54. Google ADK vs LangChain `create_agent`

## Google ADK

```text
LlmAgent
Runner
Session service
Function tools
McpToolset
ADK Web
A2A support
```

## LangChain

```text
create_agent
LangGraph runtime
Middleware
Checkpointer
MCP adapters
```

ทั้งสองมี Pattern คล้ายกัน:

```text
Model
+ Instructions
+ Tools
+ Runtime
→ Agent loop
```

---

# 55. Google ADK vs LangGraph

LangGraph เปิดเผย:

```text
State
Nodes
Edges
Routing
```

ADK ซ่อน Control Flow มากกว่า

Developer มองผ่าน:

```text
Agent
Runner
Session
Events
Tools
Sub-agents
```

ดังนั้น ADK เริ่มต้นง่ายกว่า แต่ Custom Control Flow เชิงลึกอาจต้องใช้ APIs ระดับ Workflow หรือ Application Code เพิ่ม

---

# 56. Connection to Week 4 Deep Agents

Deep Agents ใช้:

```text
Todo planning
Filesystem
Sub-agents
Skills
```

ADK Lab สร้างสิ่งใกล้เคียงด้วย:

```text
SQLite Board
plan_steps tool
Filesystem MCP
Remote A2A Agent
```

ความต่าง:

```text
Deep Agents
→ Harness ให้ Planning Capability

ADK Lab
→ Application สร้าง Planning State เองใน SQLite
```

---

# 57. Misconceptions Corrected

## ความเข้าใจคลาดเคลื่อนที่ 1

> `LlmAgent` เป็น Autonomous Agent ทันที

**ข้อเท็จจริง:**
หากไม่มี Tools และ Goal Loop พฤติกรรมยังเป็น LLM Call

---

## ความเข้าใจคลาดเคลื่อนที่ 2

> Runner คือ Model

**ข้อเท็จจริง:**
Runner จัดการ Execution, Sessions และ Events

---

## ความเข้าใจคลาดเคลื่อนที่ 3

> `InMemoryRunner` ให้ Durable Memory

**ข้อเท็จจริง:**
State หายเมื่อ Process ปิด

---

## ความเข้าใจคลาดเคลื่อนที่ 4

> SQLite Board คือ Agent Memory

**ข้อเท็จจริง:**
เป็น Application Work State

---

## ความเข้าใจคลาดเคลื่อนที่ 5

> WAL ป้องกันงานซ้ำ

**ข้อเท็จจริง:**
WAL ช่วย Database Concurrency ไม่ได้บังคับ Task Ownership

---

## ความเข้าใจคลาดเคลื่อนที่ 6

> MCP เป็น Multi-agent Protocol

**ข้อเท็จจริง:**
MCP เชื่อม Tools และ Resources

---

## ความเข้าใจคลาดเคลื่อนที่ 7

> A2A เหมือน Tool Calling ทุกอย่าง

**ข้อเท็จจริง:**
A2A มอบหมายงานให้ Remote Agent ที่อาจมี Loop ของตนเอง

---

## ความเข้าใจคลาดเคลื่อนที่ 8

> Agent Card พิสูจน์ความน่าเชื่อถือ

**ข้อเท็จจริง:**
เป็น Discovery Metadata

---

## ความเข้าใจคลาดเคลื่อนที่ 9

> Goal เป็น Done หมายถึงงานสำเร็จ

**ข้อเท็จจริง:**
ต้องตรวจ Artifact และ Invariants

---

## ความเข้าใจคลาดเคลื่อนที่ 10

> Workspace คือ Security Sandbox

**ข้อเท็จจริง:**
เป็น Root Directory restriction ไม่ใช่ Container Isolation

---

# 58. Risks Identified

## 58.1 Premature Completion

Agent ปิด Goal ก่อนงานเสร็จ

## 58.2 Missing Artifact

Board Done แต่ `spanish.txt` ไม่มี

## 58.3 Incorrect Translation

ไฟล์ถูกสร้างแต่เนื้อหาผิด

## 58.4 Duplicate Steps

Agent วางแผนซ้ำหลายครั้ง

## 58.5 Invalid IDs

Model ส่ง Task ID ผิด

## 58.6 Duplicate Claim

Workers หลายตัวทำ Goal เดียวกัน

## 58.7 Filesystem Mutation

Agent เขียนทับหรือลบไฟล์

## 58.8 File Prompt Injection

Agent ทำตามคำสั่งที่อยู่ในไฟล์

## 58.9 Side-effect Tool Risk

Agent ส่ง Notification โดยไม่มี Approval

## 58.10 MCP Failure

Node Process หรือ Tool Server เริ่มไม่ได้

## 58.11 A2A Failure

Remote Agent ไม่ตอบหรือ Protocol ไม่ตรงกัน

## 58.12 State Divergence

Session, Board, Files และ Remote State ไม่ตรงกัน

---

# 59. Production Improvements

```text
Atomic task claiming
Goal-completion gate
Required artifact validation
Structured tool results
Workspace per task
Filesystem permissions
Tool timeouts
Retry limits
Human approval
Durable session service
A2A authentication
TLS
Agent-card validation
Distributed tracing
Audit logs
Acceptance tests
```

---

# 60. Suggested Exercise — Goal Gate

เพิ่ม Rule:

```text
Goal ปิดได้เมื่อ:
ทุก Step เป็น done
spanish.txt มีอยู่
spanish.txt ไม่ว่าง
```

---

# 61. Suggested Exercise — Atomic Claim

แก้ `claim_todo()` ให้:

```sql
UPDATE todos
SET status = 'in_progress'
WHERE id = ?
  AND status = 'pending'
```

แล้วตรวจจำนวน Rows

---

# 62. Suggested Exercise — Structured Artifact Result

สร้าง Validator:

```python
def inspect_output() -> dict:
    path = WORKSPACE / "spanish.txt"

    return {
        "path": str(path),
        "exists": path.exists(),
        "size": (
            path.stat().st_size
            if path.exists()
            else 0
        ),
        "non_empty": (
            bool(
                path.read_text(
                    encoding="utf-8"
                ).strip()
            )
            if path.exists()
            else False
        ),
    }
```

---

# 63. Suggested Exercise — A2A Failure

ปิด Translator Service แล้วลองเรียก Concierge

สังเกต:

```text
Error type
Timeout behavior
Retry behavior
User-facing message
```

---

# 64. Suggested Exercise — Swap Model

เปลี่ยน Gemini เป็น OpenAI-compatible Model ผ่าน LiteLLM

เปรียบเทียบ:

```text
Tool choice
Plan quality
Number of calls
Board updates
Translation quality
Completion behavior
```

---

# 65. Patterns Learned

## Agent–Runner Separation

```text
Agent definition
≠ Execution runtime
```

## Application Work Queue

```text
Goal
→ Steps
→ Status
→ Result
```

## Function Tool Pattern

```text
Typed Python function
→ Tool schema
→ Model call
```

## MCP Adapter Pattern

```text
External tool server
→ Protocol
→ ADK toolset
```

## Goal-driven Loop

```text
Read
→ Plan
→ Act
→ Complete
→ Repeat
```

## Remote Agent Pattern

```text
Agent card
→ Remote agent discovery
→ A2A delegation
```

## Model Portability Pattern

```text
Model adapter changes
Tools and application contract remain
```

---

# 66. Lab 1 Mental Model

```text
Application
    ↓
Seed goal on SQLite board
    ↓
Runner starts Agent
    ↓
Agent reads Board
    ↓
Agent plans Steps
    ↓
Agent uses MCP File Tools
    ↓
Agent writes spanish.txt
    ↓
Agent completes Steps
    ↓
Agent completes Goal
    ↓
Application verifies Board and File
```

A2A extension:

```text
Local Concierge
    ↓
RemoteA2aAgent
    ↓
Agent Card
    ↓
HTTP/A2A
    ↓
Remote Translator
    ↓
Result returned
```

---

# 67. Final Lessons

## Lesson 1

Agent Definition และ Execution Runtime เป็นคนละส่วน

## Lesson 2

Agent ที่ไม่มี Tools ยังใกล้เคียง LLM Call

## Lesson 3

Agent Loop เริ่มชัดเมื่อ Model เลือก Tools หลายครั้งเพื่อทำ Goal

## Lesson 4

Python Function Tools ถูกอธิบายผ่าน Name, Docstring และ Type Hints

## Lesson 5

SQLite Board เป็น Shared Application State ไม่ใช่ Agent Memory

## Lesson 6

Goal และ Steps ทำให้ Plan เป็น External State ที่ตรวจสอบได้

## Lesson 7

MCP ทำให้ Agent ใช้ Tools จาก External Process ได้

## Lesson 8

Filesystem Root ลดพื้นที่เข้าถึง แต่ไม่ใช่ Security Sandbox สมบูรณ์

## Lesson 9

Application ควรจัดการ Lifecycle ส่วน Agent จัดการ Flexible Execution

## Lesson 10

Prompt บอกพฤติกรรม แต่ Code ต้องบังคับ Invariants

## Lesson 11

WAL ช่วย Concurrency แต่ไม่ป้องกัน Duplicate Work

## Lesson 12

A2A ทำให้ Remote Agent ถูกใช้คล้าย Local Sub-agent

## Lesson 13

Agent Card ช่วย Discovery แต่ไม่สร้าง Trust

## Lesson 14

MCP เชื่อม Tools ส่วน A2A เชื่อม Agents

## Lesson 15

Model สามารถเปลี่ยนได้โดยไม่ต้องเปลี่ยน Board, Tools และ MCP Contract

---

# 68. Memory Summary

```text
Week 5 Lab 1 ใช้:
Google ADK

Folder:
5_agent_frameworks/1_google_adk_a2a

Core agent:
LlmAgent

Runtime:
InMemoryRunner

run_debug:
ทดลองและดู Trace

run_async:
ควบคุม Session และ Events เอง

Agent without tools:
ใกล้กับ LLM call

Agent with tools:
Model
→ Tool
→ Result
→ Model

Shared work state:
SQLite todo board

Goal:
parent_id = None

Step:
parent_id = goal.id

Statuses:
pending
in_progress
done

Board:
Application database

ไม่ใช่:
Agent memory

Function tools:
show_todos
plan_steps
complete_task

Tool schema:
Function name
Docstring
Type hints

MCP toolset:
Filesystem server

Architecture:
ADK Agent
→ stdio
→ Node MCP Server
→ Workspace

Workspace:
task_worker/workspace

Files:
notes.txt
spanish.txt

Agent loop:
Read
Plan
Act
Complete
Repeat

worker.py:
Seed
Claim
Run
Inspect result

Application:
Owns lifecycle

Agent:
Owns flexible execution

InMemoryRunner:
State หายหลัง Process ปิด

Board and files:
อยู่บน Disk

WAL:
ช่วย Database concurrency

ไม่ช่วย:
Business task ownership

A2A:
Agent2Agent Protocol

Expose agent:
to_a2a()

Consume remote agent:
RemoteA2aAgent

Discovery:
Agent Card

MCP:
Agent → Tools

A2A:
Agent → Remote Agent

Agent Card:
Metadata

ไม่ใช่:
Trust proof

Model swap:
LiteLlm

Tools, Board and MCP:
Model-agnostic

ต้องเพิ่ม:
Atomic claiming
Goal validators
Artifact validation
Timeouts
Approval
Authentication
Tracing
```

---

# 69. Week 5 Direction

Lab 1 สร้าง Contract ที่ Framework อื่นใน Week 5 จะต้องทำซ้ำ:

```text
Shared board
One pending goal
Worker creates steps
Worker uses filesystem
Worker completes steps
Worker closes goal
```

Days ถัดไปจะเปลี่ยน Framework แต่คง Contract เดิม เพื่อให้เปรียบเทียบได้ว่าแต่ละ Framework:

```text
สร้าง Agent อย่างไร
รัน Loop อย่างไร
ผูก Tools อย่างไร
เชื่อม MCP อย่างไร
จัดการ State อย่างไร
ให้ Observability อย่างไร
```

คำถามสำคัญสำหรับ Lab ถัดไปคือ:

> เมื่อใช้ Framework อื่นสร้าง Worker ที่ทำ Contract เดียวกัน ความแตกต่างที่เห็นมาจากความสามารถของ Model จริง หรือมาจาก Runtime, Tool Interface, Session Model และ Developer Experience ของ Framework?
