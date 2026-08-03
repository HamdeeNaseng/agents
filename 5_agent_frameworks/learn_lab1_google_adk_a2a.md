# Week 5 — Lab 1: Google ADK และ A2A

ตำแหน่ง Lab:

```text
5_agent_frameworks/
└── 1_google_adk_a2a/
    ├── lab.ipynb
    ├── board.py
    ├── worker.py
    ├── task_worker/
    │   ├── agent.py
    │   └── workspace/
    ├── a2a_demo/
    │   ├── server.py
    │   └── spanish_concierge/
    │       └── agent.py
    └── SWAP_AI.md
```

Week 5 เปลี่ยนมุมมองจากการศึกษา Framework เดียวอย่างลึก ไปเป็นการเปรียบเทียบ Agent Framework หลายตัวผ่านโจทย์เดียวกัน โดยทุก Framework จะถูกมองผ่าน Pattern ห้าขั้น:

```text
1. Create the agent
2. Run it
3. Add tools
4. Add MCP
5. Put it in a loop with a goal
```

Lab แรกใช้ **Google Agent Development Kit — ADK** และปิดท้ายด้วยภาพรวมของ **Agent2Agent Protocol — A2A**.

---

## 1. Learning Objectives

หลังจบ Lab นี้ควรอธิบายได้ว่า:

1. `LlmAgent` ประกอบด้วยอะไร
2. Agent ที่ยังไม่มี Tools ต่างจาก Agent Loop อย่างไร
3. `InMemoryRunner` ทำหน้าที่อะไร
4. `run_debug()` และ `run_async()` ต่างกันอย่างไร
5. Python Function กลายเป็น ADK Tool อย่างไร
6. SQLite todo board ทำหน้าที่เป็น Shared Work Queue อย่างไร
7. Goal และ Step มีความสัมพันธ์กันอย่างไร
8. `McpToolset` เชื่อม Agent กับ Filesystem MCP Server อย่างไร
9. MCP Tool ต่างจาก Python Function Tool อย่างไร
10. Agent Loop ใน Lab อ่าน วางแผน ลงมือ และปิดงานอย่างไร
11. `adk web` ช่วย Debug Agent อย่างไร
12. A2A ต่างจาก MCP อย่างไร
13. `to_a2a()` เปิด Local Agent เป็น Remote Service อย่างไร
14. `RemoteA2aAgent` ใช้ Remote Agent เสมือน Local Sub-agent อย่างไร
15. Agent Card ใช้เพื่อ Discovery อย่างไร
16. Board State, Session State และ File State เป็นคนละสิ่งกันอย่างไร
17. จุดใดใน Lab เป็น Prompt Guidance และจุดใดเป็น Code Enforcement

---

# 2. Prerequisites

ควรเข้าใจแนวคิดจาก Week 4:

```text
Model
Messages
Tool schema
Tool call
Tool result
Agent loop
External state
MCP
Sub-agent
Persistence
```

Environment ของ Repository ปัจจุบันกำหนด:

```text
Python >= 3.12
google-adk[a2a,mcp] >= 2.2.0
```

และ Lab ต้องใช้ Node.js สำหรับรัน Filesystem MCP Server ผ่าน `npx`.

ใน `.env` ต้องมี:

```env
GOOGLE_API_KEY=...
```

Course ตั้งค่า:

```python
os.environ.setdefault(
    "GOOGLE_GENAI_USE_VERTEXAI",
    "FALSE",
)
```

จึงใช้ API-key setup ตามแนวทาง Gemini API ของ ADK แทนการตั้งค่า Vertex AI ใน Lab นี้.

ติดตั้ง Environment จาก Repository Root:

```powershell
uv sync
```

ตรวจ Node:

```powershell
node --version
npx --version
```

Warm Filesystem MCP Server หนึ่งครั้ง:

```powershell
npx -y @modelcontextprotocol/server-filesystem .
```

เมื่อ Server เริ่มทำงานแล้วสามารถหยุดด้วย `Ctrl+C` จากนั้นจึงเปิด Notebook และเลือก Python Kernel ของ Repository

ณ วันที่ 3 สิงหาคม 2026 เอกสาร API ทางการแสดง ADK Python 2.3.0 ขณะที่ ADK Python 2.0 เปิดตัวแบบ GA เมื่อวันที่ 19 พฤษภาคม 2026 ดังนั้น Code เก่าจาก Blog หรือ Tutorial รุ่น 1.x อาจมี Import Path และ Runtime Behavior ต่างจาก Course ปัจจุบัน. ([Agent Development Kit][1])

---

# 3. ภาพรวมของ Lab

```text
SQLite board
    ↓
Pending goal
    ↓
ADK Worker
├── show_todos
├── plan_steps
├── complete_task
└── Filesystem MCP tools
        ↓
Read notes.txt
        ↓
Translate content
        ↓
Write spanish.txt
        ↓
Complete steps and goal
```

Task หลักคือ:

```text
Read notes.txt,
translate its contents into natural Spanish,
and write the Spanish to spanish.txt.
```

Worker จะไม่ได้รับ Step-by-step Plan จาก Code แต่ต้องสร้าง Steps ลง Board ด้วยตัวเอง แล้วใช้ File Tools ทำงานตาม Plan. `worker.py` เป็นตัว Seed Goal, Claim งาน, รัน ADK Agent และแสดง Board กับไฟล์หลังจบงาน.

---

# 4. Step 1 — Create the Agent

Code พื้นฐาน:

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

องค์ประกอบสำคัญ:

```text
model
→ Model ที่ใช้ตัดสินใจและสร้างคำตอบ

name
→ Identifier ของ Agent

instruction
→ System-level behavioral guidance
```

`LlmAgent` เป็น Core Agent ของ ADK และอาจถูก Import ในชื่อ `Agent` ได้ในบางตัวอย่างทางการ. ([Agent Development Kit][2])

Mental model:

```text
LlmAgent
= Model
+ Identity
+ Instructions
+ Optional tools
+ Optional sub-agents
```

แต่ Agent ที่มีเพียง Model และ Instruction ยังไม่มี External Action Loop

```text
User
→ Model
→ Text response
```

จึงยังใกล้เคียง LLM Call มากกว่า Autonomous Agent

---

# 5. Step 2 — Run the Agent

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

`InMemoryRunner` ให้ Runtime Services ที่เก็บอยู่ใน Memory เช่น Session, Artifact และ Memory Services จึงเหมาะกับ Notebook, การทดลอง และ Debug แต่ข้อมูลจะหายเมื่อ Process หรือ Kernel ปิด. Course แยกสิ่งนี้ออกจาก Production `Runner` ที่ต้องเชื่อม Persistent Services เอง.

---

## `run_debug()` คืออะไร

`run_debug()` เป็น Convenience Method ที่ช่วย:

```text
สร้าง Session
ส่ง Message
รัน Agent
พิมพ์ Trace
คืน Events
```

เหมาะกับ:

```text
Notebook
ทดลอง Prompt
ดู Tool Calls
เรียนรู้ Execution Flow
```

ไม่ควรตีความว่าเป็น Application Runtime สำหรับ Production

---

## `run_async()` คืออะไร

เมื่อควบคุม Session และ Event Stream เอง:

```python
from google.genai import types

runner = InMemoryRunner(agent=agent)

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
    print(event.author, event.content)
```

`run_async()` คือ Entry Point ที่เหมาะกับ Application มากกว่า เพราะ Developer เป็นผู้กำหนด:

```text
User identity
Session identity
Input message
Event handling
Tracing
UI updates
Artifact handling
```

ADK สร้าง Invocation Context เมื่อเริ่ม `run_async()` และส่ง Context ที่เกี่ยวข้องให้ Agents, Tools และ Callbacks ระหว่าง Execution. ([Agent Development Kit][3])

---

# 6. SQLite Todo Board

Board เป็น Shared Substrate ของ Week 5

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

ความสัมพันธ์:

```text
Goal
parent_id = NULL

Step
parent_id = goal.id
```

สถานะหลัก:

```text
pending
→ in_progress
→ done
```

Functions สำคัญ:

```text
reset_board()
add_goal()
add_step()
list_todos()
claim_todo()
complete_todo()
show_board()
```

Board ใช้ SQLite WAL Mode และ `busy_timeout=5000` เพื่อให้หลาย Processes อ่านและเขียนได้สะดวกขึ้นโดยลดปัญหา Database Lock ระยะสั้น.

---

## Board ไม่ใช่ Agent Memory

ต้องแยกให้ชัด:

```text
ADK Session State
→ Conversation และ Events ของ Agent

SQLite Board
→ Application work queue

Workspace Files
→ External artifacts

Model Context
→ ข้อมูลที่ถูกส่งให้ Model ในแต่ละ Call
```

แม้ใช้ `InMemoryRunner` แล้ว Session หายเมื่อ Process ปิด แต่:

```text
board.sqlite
notes.txt
spanish.txt
```

ยังคงอยู่บน Disk จนกว่า Code จะ Reset หรือลบไฟล์

ดังนั้น Board คือ **Persistent Application State** ไม่ใช่ Semantic Memory ของ Model

---

# 7. Step 3 — Add Function Tools

Tools ของ Board:

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
        board.add_step(goal_id, step)
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
    """Mark a todo as done and record a result."""

    board.complete_todo(
        task_id,
        result,
    )

    return {
        "task_id": task_id,
        "status": "done",
    }
```

ADK ตรวจ Python Function โดยอัตโนมัติจาก:

```text
Function name
Docstring
Parameters
Type hints
Default values
```

แล้วสร้าง Function Tool Schema ที่ Model ใช้ตัดสินใจเรียก Tool. ([Agent Development Kit][4])

Mental model:

```text
Python function
        ↓
ADK FunctionTool schema
        ↓
Model sees capability
        ↓
Model proposes function call
        ↓
ADK executes function
        ↓
Result returns to Model
```

---

## Docstring สำคัญอย่างไร

Model ไม่ได้อ่าน Implementation ทั้งหมดเป็นหลัก แต่มอง Tool ผ่าน Schema และ Description

```text
show_todos
→ อ่าน Board

plan_steps
→ แตก Goal เป็น Steps

complete_task
→ ปิด Step หรือ Goal
```

ถ้า Docstring ไม่ชัด Model อาจ:

```text
เลือก Tool ผิด
ส่ง ID ผิด
ปิด Goal ก่อนเวลา
สร้าง Steps ไม่ตรงงาน
```

แต่ต้องจำว่า:

```text
Docstring
= Guidance

Function implementation
= Authority
```

---

# 8. Real-world Side-effect Tool

Notebook ยังแสดง Tool สำหรับ Push Notification:

```python
def send_push_notification(
    message: str,
) -> str:
    """Send a short push notification."""

    ...
```

จุดประสงค์คือแสดงว่า Function Tool ไม่ได้จำกัดอยู่ที่ Database หรือ Calculation แต่สามารถเรียก External API และเปลี่ยนโลกภายนอกได้.

ความเสี่ยง:

```text
ส่งผิดข้อความ
ส่งซ้ำ
API timeout
Status code ไม่สำเร็จ
ข้อมูลลับหลุด
Model เรียกโดยไม่จำเป็น
```

Lab ยังไม่ได้เพิ่ม Human Approval แบบ Week 4 Sidekick ดังนั้น Side-effect Tool ใน Production ควรมี:

```text
Timeout
Status validation
Idempotency
Audit log
Human approval
```

---

# 9. Step 4 — Add MCP

MCP ถูกนำเสนอใน Lab นี้ว่าเป็น:

> Tools ที่เราไม่ได้เขียนเอง แต่เชื่อมเข้ามาผ่าน Protocol มาตรฐาน

Agent ใช้ Filesystem MCP Server:

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

จากนั้นใส่ Toolset ลงใน `tools` List เดียวกับ Python Functions:

```python
root_agent = LlmAgent(
    model=MODEL,
    name="task_worker",
    instruction=INSTRUCTIONS,
    tools=[
        show_todos,
        plan_steps,
        complete_task,
        filesystem,
    ],
)
```

Course Agent จำกัด Filesystem Server ให้อยู่ภายใน `task_worker/workspace` โดยเฉพาะ จึงมองเห็นเฉพาะไฟล์ใน Directory นี้.

ADK `McpToolset` เชื่อม MCP Server และนำ Tools ที่ Server ประกาศมาแสดงเป็น ADK Tools ให้ Agent ใช้. ([Agent Development Kit][5])

---

## Function Tool กับ MCP Tool ต่างกันอย่างไร

### Function Tool

```text
Agent process
→ Python function
→ Result
```

Developer เขียน Implementation เอง

### MCP Tool

```text
Agent process
→ MCP client
→ Protocol transport
→ MCP server process
→ Tool implementation
```

Tool อาจถูกเขียนด้วยภาษาอื่นและรันใน Process อื่น

สำหรับ Lab:

```text
Python ADK Agent
    ↓ stdio
Node.js Filesystem MCP Server
    ↓
Workspace files
```

จากมุมมอง Model ทั้งสองกลายเป็น Tools ใน Capability List เดียวกัน

---

## MCP Root ไม่ใช่ Security Sandbox ที่สมบูรณ์

การ Scope Server ไปยัง Workspace ช่วยลดพื้นที่ที่ Agent เข้าถึงได้ แต่ Directory นั้นยังเป็น Local Filesystem จริง

Agent อาจ:

```text
อ่านไฟล์ทั้งหมดใน Workspace
เขียนทับไฟล์
ลบหรือเปลี่ยน Artifact
ทำตามคำสั่งอันตรายที่พบในไฟล์
```

ดังนั้น:

```text
Filesystem root restriction
≠ Container isolation
≠ Content safety
```

---

# 10. Step 5 — Put It in a Loop with a Goal

Worker ตัวเต็ม:

```python
INSTRUCTIONS = """
You are a careful worker with a shared todo board
and a set of file tools.

Take the pending goal and see it through.

Begin by laying out a short plan:
add concrete steps under the goal.

Carry them out with your file tools,
mark each step done,
then close the goal.
"""

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

Agent Loop ที่คาดหวัง:

```text
1. show_todos()
        ↓
2. Identify pending/in-progress goal
        ↓
3. plan_steps(...)
        ↓
4. Read notes.txt
        ↓
5. Translate content
        ↓
6. Write spanish.txt
        ↓
7. complete_task(step...)
        ↓
8. Repeat until all steps done
        ↓
9. complete_task(goal...)
```

Course สรุป Loop ว่า:

```text
Read
→ Plan
→ Act
→ Check off
→ Repeat
```

นี่คือจุดที่ระบบเปลี่ยนจาก Tool-enabled Chatbot เป็น Agent ที่ทำงานหลายขั้นตาม Goal.

---

# 11. `worker.py` ทำอะไร

Execution flow:

```python
def seed() -> int:
    board.reset_board()
    WORKSPACE.mkdir(exist_ok=True)
    remove_old_spanish_file()
    return board.add_goal(TASK)
```

จากนั้น:

```python
goal_id = seed()
board.claim_todo(goal_id)
asyncio.run(run())
```

และ:

```python
async def run():
    runner = InMemoryRunner(
        agent=root_agent
    )

    await runner.run_debug(
        "Please work the pending task "
        "on the board.",
        verbose=True,
    )
```

สังเกตว่า Code ภายนอกเป็นผู้:

```text
Reset board
Seed goal
Claim goal
Start worker
```

ส่วน Agent เป็นผู้:

```text
Read goal
Create steps
Operate on files
Complete steps
Complete goal
```

การแบ่งนี้สำคัญมาก:

```text
Application
→ Owns lifecycle and assignment

Agent
→ Owns flexible execution
```

`worker.py` ใช้ Contract เดียวกับ Workers ของ Days 2–4 เพื่อให้เปรียบเทียบ Framework ได้อย่างเป็นธรรม.

---

# 12. ดู Event Stream

เมื่อใช้ `run_async()` สามารถตรวจ:

```python
async for event in runner.run_async(...):
    for call in event.get_function_calls():
        print(
            f"tool: {call.name}"
            f"({call.args})"
        )

    if any(
        response.name in (
            "plan_steps",
            "complete_task",
        )
        for response
        in event.get_function_responses()
    ):
        board.show_board()

    if event.is_final_response():
        print(event.content)
```

Event Stream ทำให้เห็นว่า Agent:

```text
เรียก Tool ใด
ส่ง Arguments อะไร
Tool ตอบอะไร
Board เปลี่ยนเมื่อใด
Final Response เกิดเมื่อใด
```

นี่คือ Observability ระดับ Runtime

แต่:

```text
Trace
≠ Correctness
```

ต้องตรวจ Board และ `spanish.txt` จริงหลังจบด้วย

---

# 13. วิธีรันสามแบบ

## แบบที่ 1 — Plain Worker Script

```powershell
uv run worker.py
```

ใช้เมื่ออยากเห็น Agent ทำงาน End-to-end ใน Terminal

---

## แบบที่ 2 — Seed อย่างเดียว

```powershell
uv run worker.py --seed-only
```

จากนั้น:

```powershell
uv run adk web
```

เลือก `task_worker` แล้วสั่ง:

```text
Please work the pending task on the board.
```

`adk web` เป็น Development UI สำหรับสนทนาและ Debug Agent ไม่ได้ออกแบบเป็น Production Frontend. ([Agent Development Kit][6])

---

## แบบที่ 3 — ADK CLI

จาก Parent Folder ที่ ADK มองเห็น Agent Package:

```powershell
uv run adk run task_worker
```

แล้วส่งงานผ่าน Interactive CLI

หลักสำคัญคือ ADK คาดหวังให้ Agent Package มีตัวแปร:

```python
root_agent
```

ซึ่งเป็น Entry Point หลักของ Agent Project. ([Agent Development Kit][6])

---

# 14. จุดอ่อนเชิง Deterministic ของ Board

Board ใช้ Prompt ควบคุมลำดับงานมากกว่า Code Enforcement

ตัวอย่าง:

```python
def complete_todo(
    task_id,
    result,
):
    UPDATE todos
    SET status = "done"
```

Function ไม่ได้ตรวจว่า:

```text
Task ID มีอยู่จริงหรือไม่
Step ทั้งหมดเสร็จหรือยัง
Goal ถูกปิดก่อน Steps หรือไม่
Result ว่างหรือไม่
spanish.txt ถูกสร้างจริงหรือไม่
```

ดังนั้น Agent สามารถปิด Goal ได้แม้ Deliverable ยังไม่สำเร็จ

Safer implementation:

```python
def complete_goal(
    goal_id: int,
    result: str,
) -> dict:
    unfinished = find_unfinished_steps(
        goal_id
    )

    if unfinished:
        raise ValueError(
            "Cannot close goal while "
            "steps remain unfinished"
        )

    output = WORKSPACE / "spanish.txt"

    if not output.exists():
        raise ValueError(
            "Required output is missing"
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

หลัก:

```text
Prompt
→ บอกสิ่งที่ควรทำ

Code
→ บังคับสิ่งที่ห้ามผิด
```

---

# 15. Database Concurrency ไม่เท่ากับ Work Coordination

SQLite WAL และ Busy Timeout ช่วยให้หลาย Processes ใช้ Database ได้ราบรื่นขึ้น แต่ `claim_todo()` ปัจจุบันเพียง Update Status โดยไม่ตรวจว่า Task ยังเป็น `pending`

```sql
UPDATE todos
SET status = 'in_progress'
WHERE id = ?
```

ถ้ามี Workers สองตัว ทั้งสองอาจเชื่อว่าตน Claim Goal เดียวกันได้

Safer atomic claim:

```sql
UPDATE todos
SET status = 'in_progress'
WHERE id = ?
  AND status = 'pending'
```

จากนั้นตรวจ:

```text
affected_rows == 1
```

สรุป:

```text
Database locking
≠ Business-level exclusivity
```

Lab 1 มี Worker เดียว จึงยังไม่เกิดปัญหาชัด แต่แนวคิดนี้จะสำคัญเมื่อ Board ถูกขยายเป็น Multi-worker System ในภายหลัง.

---

# 16. Quick Look at A2A

A2A หรือ Agent2Agent เป็น Protocol สำหรับให้ Agents ที่อยู่คนละ Process หรือคนละ Service:

```text
Discover each other
Understand capabilities
Send tasks/messages
Receive responses
```

Course Demo มีสองส่วน:

```text
Remote translator service
        ↑ A2A over HTTP
Local Spanish concierge
```

Notebook สรุป API หลักสองตัว:

```text
to_a2a()
→ Expose an ADK agent as a remote service

RemoteA2aAgent
→ Consume a remote A2A agent
```

A2A Support ใน Python ADK ยังถูกระบุเป็น Experimental ในเอกสารทางการปัจจุบัน จึงต้องระวัง API และ Protocol Version Drift. ([Agent Development Kit][7])

---

# 17. Expose Agent ด้วย `to_a2a()`

Remote Translator:

```python
from google.adk.agents import LlmAgent
from google.adk.a2a.utils.agent_to_a2a import (
    to_a2a,
)

root_agent = LlmAgent(
    model="gemini-3.1-flash-lite",
    name="translator_agent",
    description=(
        "Translates English text "
        "into Spanish."
    ),
    instruction=(
        "Translate the user's English "
        "text into natural Spanish. "
        "Return only the translation."
    ),
)

a2a_app = to_a2a(
    root_agent,
    port=8001,
)
```

`to_a2a()` คืน Starlette ASGI Application และสร้าง Agent Card จาก Metadata ของ Agent ให้โดยอัตโนมัติ. Port ที่ระบุต้องตรงกับ Port ที่ Uvicorn เปิด เพราะข้อมูลนี้จะถูกประกาศใน Agent Card.  ([Agent Development Kit][7])

รัน Service:

```powershell
cd 5_agent_frameworks/1_google_adk_a2a/a2a_demo

uv run uvicorn server:a2a_app `
  --host localhost `
  --port 8001
```

---

# 18. Agent Card

ตรวจ Agent Card:

```powershell
curl `
  http://localhost:8001/.well-known/agent-card.json
```

Agent Card ทำหน้าที่คล้าย Business Card หรือ Service Contract:

```text
Agent name
Description
Service URL
Capabilities
Input modes
Output modes
Skills
Protocol metadata
```

Mental model:

```text
MCP Tool Schema
→ อธิบาย Tool

A2A Agent Card
→ อธิบาย Remote Agent
```

แต่:

```text
Agent Card
= Discovery metadata

ไม่ใช่
= Proof of trust
```

Production ยังต้องมี Authentication, Authorization, TLS และ Service Identity

---

# 19. Consume Agent ด้วย `RemoteA2aAgent`

```python
from google.adk.agents.remote_a2a_agent import (
    AGENT_CARD_WELL_KNOWN_PATH,
    RemoteA2aAgent,
)

translator = RemoteA2aAgent(
    name="translator_agent",
    description=(
        "Remote agent that translates "
        "English into Spanish."
    ),
    agent_card=(
        "http://localhost:8001"
        f"{AGENT_CARD_WELL_KNOWN_PATH}"
    ),
    use_legacy=False,
)
```

จากนั้นใส่ Remote Agent เป็น Sub-agent:

```python
root_agent = LlmAgent(
    model="gemini-3.1-flash-lite",
    name="spanish_concierge",
    description=(
        "Delegates Spanish translation."
    ),
    instruction=(
        "When translation into Spanish "
        "is requested, delegate to "
        "translator_agent."
    ),
    sub_agents=[translator],
)
```

`RemoteA2aAgent` อ่าน Agent Card แล้วแปลง Interaction ระหว่าง ADK Session กับ A2A Requests/Responses ทำให้ Remote Service มี Interface ใกล้เคียง Local Sub-agent.  ([Agent Development Kit][8])

---

# 20. รัน A2A Demo

Terminal 1:

```powershell
cd 5_agent_frameworks/1_google_adk_a2a/a2a_demo

uv run uvicorn server:a2a_app `
  --host localhost `
  --port 8001
```

Terminal 2:

```powershell
cd 5_agent_frameworks/1_google_adk_a2a/a2a_demo

curl `
  http://localhost:8001/.well-known/agent-card.json

uv run adk web
```

ใน ADK Web:

```text
เลือก spanish_concierge
```

แล้วถาม:

```text
Translate "The agent finished the task"
into Spanish.
```

Execution:

```text
User
→ Spanish Concierge
→ RemoteA2aAgent
→ HTTP/A2A request
→ Translator Service
→ Spanish result
→ Concierge
→ User
```

---

# 21. MCP, A2A และ Local Sub-agent ต่างกันอย่างไร

| Mechanism       | เชื่อมอะไร                        | สิ่งที่ปลายทางทำ         |
| --------------- | --------------------------------- | ------------------------ |
| Function Tool   | Agent → Function                  | Action เฉพาะอย่าง        |
| MCP             | Agent → Tool server               | Expose tools/resources   |
| Local Sub-agent | Agent → Agent ใน Runtime เดียวกัน | ทำงานย่อยหลายขั้น        |
| A2A             | Agent → Remote agent service      | Delegate งานให้อีก Agent |

Mental model:

```text
MCP
= ขอใช้มือหรือเครื่องมือของระบบอื่น

A2A
= ขอให้ผู้เชี่ยวชาญอีกคนรับงานไปทำ
```

MCP Server ไม่จำเป็นต้องมี Reasoning Loop ส่วน A2A Endpoint โดยปกติเป็น Agent ที่อาจวางแผน ใช้ Tools และทำงานหลายขั้นของตนเอง

---

# 22. State Surfaces ใน Lab

Lab นี้มี State อย่างน้อยสี่พื้นผิว:

```text
1. ADK Session
   → Messages และ Events

2. SQLite Board
   → Goals, Steps, Status, Results

3. MCP Workspace
   → notes.txt และ spanish.txt

4. A2A Remote Service
   → Remote agent execution/session state
```

ปัญหาที่อาจเกิด:

```text
Board บอก done
แต่ไฟล์ไม่มี

ไฟล์มี
แต่ Session หาย

Remote Agent ตอบสำเร็จ
แต่ Local Agent timeout

Agent ตอบว่าปิดงานแล้ว
แต่ Steps ยัง pending
```

ระบบ Production ต้องมี Reconciliation และ Validation ระหว่าง State Surfaces เหล่านี้

---

# 23. เปรียบเทียบกับ Week 4

## Week 4 Lab 1

```text
LangChain:
ChatOpenAI
@tool
Manual Tool Loop
```

## Week 5 Lab 1

```text
Google ADK:
LlmAgent
Python function tools
Runner-managed loop
```

---

## Week 4 Lab 2

```text
LangGraph:
StateGraph
Nodes
Edges
Checkpointer
```

ADK ซ่อน Control Flow มากกว่า แต่ ADK 2.x ใช้ Workflow Runtime แบบ Graph-based ภายใน และยังคง Compatibility กับ Agent APIs จาก 1.x ในระดับหนึ่ง. ([Agent Development Kit][1])

---

## Week 4 Lab 3

```text
LangChain create_agent()
≈
ADK LlmAgent + Runner
```

ทั้งสองจัด Model–Tool Loop ให้ Framework เป็นผู้ควบคุม

---

## Week 4 Lab 4

```text
Deep Agents:
Planning
Filesystem
Artifacts
Sub-agents
```

ADK Lab นี้สร้างสิ่งคล้ายกันด้วย:

```text
plan_steps tool
SQLite board
Filesystem MCP
Remote A2A sub-agent
```

ต่างกันตรงที่ Planning State ถูกออกแบบใน Application Database อย่างชัดเจน ไม่ได้พึ่ง Built-in Todo Middleware

---

# 24. Model Portability

Course มี `SWAP_AI.md` สำหรับเปลี่ยนจาก Gemini ไปยัง OpenAI-compatible Endpoint ผ่าน LiteLLM:

```python
from google.adk.models.lite_llm import (
    LiteLlm,
)

model = LiteLlm(
    model="openai/gpt-5.4-mini",
)
```

แล้วส่งให้:

```python
root_agent = LlmAgent(
    model=model,
    ...
)
```

Tools, Board และ MCP ไม่จำเป็นต้องเปลี่ยน เพราะถูกแยกออกจาก Model Adapter. การใช้งานนี้ต้องติดตั้ง ADK Extensions เพิ่มตามไฟล์ Course.

บทเรียน:

```text
Model
ควรถูกแยกจาก
Tools, State และ Workflow Contract
```

---

# 25. Misconceptions ที่ต้องแก้

### “สร้าง `LlmAgent` แล้วได้ Autonomous Agent ทันที”

ไม่จริง หากไม่มี Tools และ Goal Loop พฤติกรรมยังเป็น LLM Call

### “Runner คือ Model”

ไม่ใช่ Runner จัดการ Execution, Sessions และ Events

### “InMemoryRunner คือ Persistent Memory”

ไม่ใช่ State หายเมื่อ Process ปิด

### “Board คือ ADK Memory”

ไม่ใช่ เป็น Application Database

### “WAL ป้องกัน Workers ทำงานซ้ำ”

ไม่จริง WAL ช่วย Database Concurrency แต่ไม่บังคับ Business Lock

### “MCP คือ Multi-agent Protocol”

ไม่ใช่ MCP เชื่อม External Tools/Resources

### “A2A คือ Tool Calling”

ไม่ทั้งหมด A2A ใช้ Delegate งานไปยัง Remote Agent

### “Agent Card ทำให้ Remote Agent เชื่อถือได้”

ไม่จริง มันเป็น Discovery Metadata

### “Goal ถูก Mark Done หมายถึงงานสำเร็จ”

ไม่จริง ต้องตรวจ Artifact และ Acceptance Criteria

### “Workspace Scope คือ Security Sandbox”

ไม่สมบูรณ์ มันเพียงจำกัด Root Directory

---

# 26. Risks Identified

```text
Agent closes goal too early
Missing output file
Incorrect translation
Duplicate steps
Invalid task IDs
Two workers claiming same goal
Filesystem overwrite
Prompt injection from files
External side effects without approval
MCP server startup failure
A2A remote service unavailable
Agent-card version mismatch
No authentication in local A2A demo
Session and board state divergence
```

---

# 27. Production Improvements

```text
Atomic task claiming
Goal completion validator
Required artifact checks
Structured tool results
Workspace per task
Filesystem write policy
Tool timeouts
Retry limits
Human approval for side effects
Persistent session service
A2A authentication
TLS
Agent-card version validation
Distributed tracing
Audit logs
Acceptance tests
```

---

# 28. Exercises

## Exercise 1 — Deterministic Goal Gate

ห้ามปิด Goal จนกว่า:

```text
ทุก Step เป็น done
spanish.txt มีอยู่
spanish.txt ไม่ว่าง
```

---

## Exercise 2 — Atomic Claim

แก้ `claim_todo()` ให้ Claim ได้เฉพาะ Task ที่ยังเป็น `pending`

แล้วทดลอง Workers สองตัวพร้อมกัน

---

## Exercise 3 — Structured File Result

ให้ File Tool หรือ Validator คืน:

```json
{
  "path": "spanish.txt",
  "exists": true,
  "bytes": 128,
  "encoding": "utf-8"
}
```

---

## Exercise 4 — A2A Failure

ปิด Translator Server แล้วสั่ง Concierge แปลภาษา

สังเกตว่า:

```text
Error ถูกแสดงอย่างไร
Local Agent retry หรือไม่
User ได้ข้อความที่เข้าใจได้หรือไม่
```

---

## Exercise 5 — Swap Model

ใช้ `LiteLlm` ตาม `SWAP_AI.md` แล้วตรวจว่า:

```text
Board Contract เหมือนเดิมหรือไม่
Tool-selection behavior ต่างหรือไม่
จำนวน Calls ต่างหรือไม่
Final artifact ต่างหรือไม่
```

---

# 29. Checklist ก่อนจบ Lab

### ADK Agent หลักคืออะไร

```text
LlmAgent
```

### Runtime ที่ใช้ใน Notebook คืออะไร

```text
InMemoryRunner
```

### Tool Function ถูกสร้างจากอะไร

```text
Function name
Docstring
Type hints
Parameters
```

### Work Queue อยู่ที่ไหน

```text
board.sqlite
```

### File Tools มาจากไหน

```text
Filesystem MCP Server
```

### Agent สร้าง Plan ที่ไหน

```text
Steps ใต้ Goal ใน SQLite Board
```

### Agent Loop คืออะไร

```text
Read
→ Plan
→ Act
→ Complete
→ Repeat
```

### `run_debug()` เหมาะกับอะไร

```text
Learning และ Debugging
```

### `run_async()` เหมาะกับอะไร

```text
Application-controlled execution
```

### A2A Server สร้างอย่างไร

```python
to_a2a(root_agent)
```

### Remote Agent เชื่อมอย่างไร

```python
RemoteA2aAgent(...)
```

### Agent Discovery ใช้อะไร

```text
Agent Card
```

### MCP กับ A2A ต่างกันอย่างไร

```text
MCP → Tools
A2A → Remote agents
```

---

# แก่นของ Week 5 Lab 1

```text
LlmAgent
= Model-driven worker

Runner
= Execution and session runtime

Function tools
= Application capabilities

MCP
= External tool interoperability

SQLite board
= Shared work state

Agent loop
= Repeated decision and action

A2A
= Remote agent interoperability
```

บทเรียนสำคัญที่สุดคือ:

> **Framework อาจเปลี่ยนชื่อ Class และ API แต่ Agent Pattern ยังเหมือนเดิม: Model เห็น Goal และ State เลือก Tool รับผลกลับมา แล้วทำซ้ำจนคิดว่างานเสร็จ**

และ:

> **การที่ Agent คิดว่างานเสร็จไม่เพียงพอ Application ต้องเป็นผู้ถือ Authority ผ่าน Board Invariants, Artifact Validation, Atomic Claiming และ Security Policies**

อีกแก่นหนึ่งของ Week 5 คือ:

> **MCP ทำให้ Agent ยืมความสามารถจากระบบอื่น ส่วน A2A ทำให้ Agent มอบหมายงานให้อีก Agent หนึ่ง ทั้งสองเป็น Interoperability Protocol แต่เชื่อมคนละระดับของระบบ**

[1]: https://adk.dev/2.0/?utm_source=chatgpt.com "Welcome to ADK 2.0 - Agent Development Kit (ADK)"
[2]: https://adk.dev/agents/llm-agents/?utm_source=chatgpt.com "Simple agents - Agent Development Kit (ADK)"
[3]: https://adk.dev/context/?utm_source=chatgpt.com "Context - Agent Development Kit (ADK)"
[4]: https://adk.dev/tools/function-tools/?utm_source=chatgpt.com "Overview - Agent Development Kit (ADK)"
[5]: https://adk.dev/tools-custom/mcp-tools/?utm_source=chatgpt.com "MCP tools - Agent Development Kit (ADK)"
[6]: https://adk.dev/get-started/python/?utm_source=chatgpt.com "Python - Agent Development Kit (ADK)"
[7]: https://adk.dev/a2a/quickstart-exposing/?utm_source=chatgpt.com "Python - Agent Development Kit (ADK)"
[8]: https://adk.dev/api-reference/java/com/google/adk/a2a/agent/RemoteA2AAgent.html?utm_source=chatgpt.com "RemoteA2AAgent (Google Agent Development Kit Maven Parent POM 1.6.0 API)"
