# Episodic Learning Artifact

## Week 2 — Lab 1: OpenAI Agents SDK Foundations

**หลักสูตร:** AI Engineer Agentic Track
**ผู้สอน:** Ed Donner
**Repository:** `ed-donner/agents`
**ไฟล์เรียน:** `2_openai/1_lab1.ipynb`
**หัวข้อหลัก:** Agent, Runner, Tracing, Streaming, Function Tools และ Sessions
**สถานะ:** เรียนและสรุปพื้นฐานแล้ว

---

# 1. Context

Week 1 สร้าง Agent Loop ด้วยตนเอง โดยต้องจัดการ:

```text
Messages
Tool Schema
Tool Dispatch
Tool Results
Agent Loop
Conversation History
State
```

Week 2 เริ่มใช้ **OpenAI Agents SDK** เพื่อห่อหุ้มกลไกเหล่านี้ให้อยู่ใน Primitive ที่ใช้งานง่ายขึ้น

การเปลี่ยนผ่านสำคัญคือ:

```text
Week 1
เขียน Agent Runtime เอง

Week 2
ใช้ SDK จัดการ Agent Runtime
```

อย่างไรก็ตาม SDK ไม่ได้เปลี่ยนหลักการพื้นฐานของ Agent

```text
LLM
+ Instructions
+ Tools
+ Context
+ Agent Loop
+ Runtime Controls
```

สิ่งที่เปลี่ยนคือผู้พัฒนาไม่ต้องเขียนรายละเอียดซ้ำทั้งหมดด้วยตนเอง

---

# 2. Learning Objectives

หลังจบ Lab 1 สามารถอธิบายได้ว่า:

1. `Agent` และ `Runner` มีหน้าที่ต่างกันอย่างไร
2. Runner จัดการ Agent Loop ส่วนใด
3. `final_output` และ `to_input_list()` ใช้ทำอะไร
4. Tracing ช่วยสังเกตและ Debug Agent อย่างไร
5. Streaming เปลี่ยนประสบการณ์ของผู้ใช้อย่างไร
6. `@function_tool` เปลี่ยน Python Function เป็น Agent Tool อย่างไร
7. ทำไม Agent จึงไม่จำบทสนทนาระหว่าง Runner Calls โดยอัตโนมัติ
8. การส่ง History ด้วยตนเองและการใช้ Session แตกต่างกันอย่างไร
9. `SQLiteSession` แบบ In-memory และ Persistent ต่างกันอย่างไร
10. Session, Trace และ Long-term Memory เป็นคนละแนวคิดกันอย่างไร

---

# 3. Week 1 กับ Week 2

ความสัมพันธ์ระหว่างโค้ดที่เขียนเองกับ Agents SDK:

```text
Week 1                         Week 2

System Prompt                 Agent.instructions
ชื่อและบทบาท                  Agent.name
เลือกโมเดล                    Agent.model
Manual Agent Loop             Runner.run()
Tool Schema                   @function_tool
Tool Registry                 SDK Tool Management
Tool Dispatch                 Runner Runtime
Tool Result Messages          SDK Runtime
Conversation History          Session
Manual Debug Logs             Trace
Partial Output Handling       Streaming
```

## Key Insight

Framework ไม่ได้สร้างกลไกใหม่ทั้งหมด แต่ซ่อนรายละเอียดการจัดการวงจร Agent ไว้หลัง API ที่เรียบง่ายกว่า

---

# 4. Package และ Import

Package ที่ถูกต้องคือ:

```text
openai-agents
```

ติดตั้งผ่าน:

```powershell
uv add openai-agents
```

แต่ Import ผ่าน Namespace:

```python
from agents import Agent, Runner, trace
```

## Misconception Corrected

ชื่อ Package ตอนติดตั้งกับชื่อ Module ตอน Import ไม่จำเป็นต้องเหมือนกัน

```text
Install:
openai-agents

Import:
agents
```

ไม่ควรติดตั้ง Package ชื่อ `agents` โดยตรง เพราะเป็น Package คนละโครงการ

---

# 5. Agent

ตัวอย่าง:

```python
from agents import Agent

agent = Agent(
    name="Jokester",
    instructions="You are a joke teller",
    model="gpt-5.4-mini"
)
```

`Agent` เป็น Configuration Object ที่รวม:

```text
Name
Instructions
Model
Tools
Guardrails
Handoffs
Output Type
Model Settings
```

การสร้าง:

```python
Agent(...)
```

ยังไม่ได้ส่ง Request ไปยังโมเดลและยังไม่ได้เริ่ม Agent Loop

Mental Model:

```text
Agent
=
บทบาท
+ กฎการทำงาน
+ ความสามารถที่อนุญาต
+ Model Configuration
```

---

# 6. Agent ไม่เท่ากับ Runtime

`Agent` บอกว่า:

```text
ใครกำลังทำงาน
มีบทบาทอะไร
ใช้โมเดลใด
ใช้ Tools ใดได้
```

แต่ Agent ไม่ได้ดำเนินงานด้วยตัวเอง

ตัวที่เริ่มและควบคุม Run คือ:

```text
Runner
```

ดังนั้น:

```text
Agent = Configuration
Runner = Execution Runtime
```

---

# 7. Runner

ตัวอย่าง:

```python
from agents import Runner

result = await Runner.run(
    agent,
    "Tell a joke about Autonomous AI Agents"
)
```

Runner รับ:

```text
Agent
+
Input
```

แล้วจัดการ:

```text
สร้าง Model Request
→ รับ Model Response
→ ตรวจ Tool Calls
→ Execute Tools
→ ส่ง Tool Results
→ เรียก Model อีกครั้ง
→ หยุดเมื่อได้ Final Output
```

นี่คือ Agent Loop แบบเดียวกับ Week 1 แต่ SDK เป็นผู้จัดการรายละเอียด

---

# 8. Runner Lifecycle

ภาพรวม:

```text
User Input
    ↓
Runner
    ↓
Load Agent Configuration
    ↓
Call Model
    ↓
Model Decision
    ├── Final Answer
    ├── Tool Call
    └── Handoff
           ↓
Runtime executes action
           ↓
Result becomes observation
           ↓
Call Model again
```

Runner ทำงานจนกระทั่ง:

```text
ได้ Final Output
เกิด Error
ถึง Guardrail
ถึง Run Limit
หรือถูกยกเลิก
```

---

# 9. Async และ `await`

```python
result = await Runner.run(...)
```

Runner เป็น Async Operation เพราะอาจต้องรอ:

```text
Model API
Network
External Tools
Agent Handoffs
Streaming Events
```

`await` หมายถึงรอผลของ Async Task โดยเปิดโอกาสให้ Event Loop จัดการงานอื่นได้

ใน Notebook สามารถใช้ `await` ได้โดยตรง

ใน Python Script ปกติ:

```python
import asyncio

async def main():
    result = await Runner.run(
        agent,
        "Tell me a joke"
    )
    print(result.final_output)

asyncio.run(main())
```

---

# 10. Run Result

`Runner.run()` ไม่ได้คืน String โดยตรง แต่คืน Result Object

```text
Run Result
├── final_output
├── final agent
├── generated items
├── tool calls
├── handoffs
├── usage
└── run context
```

ดึงคำตอบสุดท้ายด้วย:

```python
result.final_output
```

## Key Insight

```text
final_output
=
ผลลัพธ์ที่พร้อมส่งให้ผู้ใช้

result
=
ข้อมูลทั้งหมดของการทำงานหนึ่ง Run
```

---

# 11. `to_input_list()`

```python
result.to_input_list()
```

ใช้แปลงเหตุการณ์ใน Run ให้กลับมาเป็น Input Items ที่ส่งเข้า Runner ครั้งต่อไปได้

ตัวอย่าง:

```python
first = await Runner.run(
    agent,
    "My name is Dee."
)

next_input = first.to_input_list() + [
    {
        "role": "user",
        "content": "What is my name?"
    }
]

second = await Runner.run(
    agent,
    next_input
)
```

Flow:

```text
Run 1
User → Assistant
       ↓
to_input_list()
       ↓
Conversation Items
       ↓
เพิ่ม User Message ใหม่
       ↓
Run 2
```

นี่คือการจัด Conversation History ด้วยตนเอง

---

# 12. Tracing

ตัวอย่าง:

```python
from agents import trace

with trace("Telling a joke"):
    result = await Runner.run(
        agent,
        "Tell a joke about AI agents"
    )
```

Trace เก็บความสัมพันธ์ของเหตุการณ์ใน Workflow เช่น:

```text
Model Calls
Tool Calls
Tool Results
Handoffs
Guardrails
Timing
Run Hierarchy
Errors
```

Mental Model:

```text
Trace
=
กล้องวงจรปิดของ Agent Workflow
```

Trace ช่วยตอบว่า:

```text
Agent ใดทำงาน
โมเดลถูกเรียกกี่ครั้ง
Tool ใดถูกเรียก
Arguments คืออะไร
Tool ใช้เวลานานเท่าไร
เกิด Error ที่ขั้นตอนไหน
```

---

# 13. Trace ไม่ใช่ Memory

Trace ใช้เพื่อ:

```text
Observability
Debugging
Monitoring
Performance Analysis
```

มันไม่ได้ถูกนำกลับเข้า Context ของ Agent โดยอัตโนมัติ

ดังนั้น:

```text
Trace
= สิ่งที่นักพัฒนาใช้สังเกตระบบ

Session
= สิ่งที่ Application ใช้เก็บ Conversation Context
```

---

# 14. Logging กับ Tracing

## Logging

บันทึกเหตุการณ์แยกเป็นข้อความ:

```text
Model call started
Tool executed
Request failed
```

## Tracing

แสดงความสัมพันธ์ของเหตุการณ์ทั้ง Run:

```text
Parent Run
├── Model Call
├── Tool Call
│   └── Tool Result
└── Final Model Call
```

Tracing จึงเหมาะกับระบบ Agent เพราะ Agent มักมีการเรียกหลาย Model และหลาย Tools ต่อหนึ่ง User Request

---

# 15. Streaming

ตัวอย่าง:

```python
result = Runner.run_streamed(
    agent,
    input="Tell me five jokes about AI agents."
)

async for event in result.stream_events():
    ...
```

Streaming ส่ง Events ออกมาขณะที่ Agent ยังทำงาน

ตัวอย่าง Event:

```text
Text Delta
Tool Call Started
Tool Call Completed
Agent Changed
Final Response
```

## ไม่ Streaming

```text
เริ่ม Run
→ รอทั้งหมด
→ แสดง Final Output
```

## Streaming

```text
เริ่ม Run
→ แสดงข้อความบางส่วน
→ แสดง Progress
→ แสดง Final Output
```

---

# 16. Streaming ไม่ได้ทำให้การคำนวณเร็วขึ้นเสมอ

Streaming ช่วยลด:

```text
Perceived Latency
Time to First Visible Output
```

แต่ไม่ได้จำเป็นต้องลด:

```text
Total Generation Time
Total Token Usage
Model Computation
```

ประโยชน์หลักคือ User Experience และ Progress Visibility

---

# 17. Function Tool

ใน Week 1 การสร้าง Tool ต้องเขียน:

```text
Python Function
Tool JSON Schema
Tool Map
Tool Handler
Tool Result Message
```

Agents SDK ใช้:

```python
from agents import function_tool

@function_tool
def notify_user(message: str) -> str:
    """Send a notification message to the user."""
    return f"Notification sent: {message}"
```

Decorator จะอ่าน:

```text
ชื่อ Function
Docstring
Type Annotations
Arguments
Return Value
```

แล้วสร้าง Function Tool ที่ Agent สามารถมองเห็นได้

---

# 18. Type Annotations มีความสำคัญ

ตัวอย่าง:

```python
def notify_user(message: str) -> str:
```

`message: str` ช่วย SDK สร้าง Schema ว่า:

```json
{
  "message": {
    "type": "string"
  }
}
```

ถ้า Function มี:

```python
def calculate_total(
    price: float,
    quantity: int
) -> float:
```

SDK สามารถสร้าง Schema จาก Type Hints ได้

ดังนั้น Type Annotation ไม่ได้มีประโยชน์เพียงต่อ IDE แต่ยังเป็นส่วนหนึ่งของ Tool Contract

---

# 19. Docstring เป็น Tool Description

```python
@function_tool
def notify_user(message: str) -> str:
    """Send a notification message to the user."""
```

Docstring ช่วยให้โมเดลเข้าใจว่า:

```text
Tool ใช้เมื่อใด
Tool ทำอะไร
ผลลัพธ์ที่คาดหวังคืออะไร
```

Tool Description ที่คลุมเครืออาจทำให้ Agent:

```text
ไม่เรียก Tool ตอนที่ควรเรียก
เรียก Tool ผิดสถานการณ์
สร้าง Arguments ไม่เหมาะสม
```

---

# 20. เพิ่ม Tool ให้ Agent

```python
notifier = Agent(
    name="Notifier",
    instructions=(
        "Notify the user when requested."
    ),
    model="gpt-5.4-mini",
    tools=[notify_user]
)
```

Run:

```python
result = await Runner.run(
    notifier,
    "Notify me that the pizza has arrived."
)
```

Flow:

```text
User Request
    ↓
Runner calls Model
    ↓
Model requests notify_user
    ↓
SDK executes Python Tool
    ↓
Tool Result returned to Model
    ↓
Model generates Final Response
```

---

# 21. SDK ซ่อนอะไรไว้ให้

เมื่อใช้ `@function_tool` และ Runner ผู้พัฒนาไม่ต้องเขียนเองทั้งหมด:

```text
JSON Schema Generation
Tool Call Parsing
Tool Dispatch
Tool Result Formatting
Repeated Model Calls
Basic Run Lifecycle
```

แต่นักพัฒนายังต้องรับผิดชอบ:

```text
Tool Safety
Argument Validation
Business Rules
Permissions
Timeouts
Retries
Error Handling
Rate Limits
Human Approval
```

---

# 22. Function Tool ไม่ได้ปลอดภัยโดยอัตโนมัติ

Decorator ช่วยสร้าง Interface แต่ไม่ได้ควบคุมความเสี่ยงทั้งหมด

ตัวอย่าง Tool ที่ส่ง Notification อาจถูกเรียกซ้ำจน Spam

Tool ที่ลบไฟล์อาจสร้างความเสียหายถ้า Agent เลือกผิด

ดังนั้นต้องเพิ่ม:

```text
Input Validation
Authorization
Side-effect Classification
Execution Limits
Audit Logs
Approval Gates
```

หลักจาก Week 1 ยังคงใช้:

```text
LLM เสนอ Action
Application ถือ Authority
```

---

# 23. Runner Call ไม่จำ Run ก่อนหน้า

ตัวอย่าง:

```python
await Runner.run(
    agent,
    "My name is Dee."
)

await Runner.run(
    agent,
    "What is my name?"
)
```

Run ที่สองเป็น Run ใหม่ และไม่ได้รับ Conversation History จาก Run แรกโดยอัตโนมัติ

ดังนั้น Agent อาจไม่ทราบชื่อ

## Key Insight

```text
Agent Object
ไม่ได้เก็บ Conversation History เอง

Runner Call
ไม่ได้เชื่อมกับ Runner Call ก่อนหน้าโดยอัตโนมัติ
```

---

# 24. Manual Conversation History

สามารถส่ง History ด้วยตนเองผ่าน:

```python
result.to_input_list()
```

ข้อดี:

```text
ควบคุม Context ได้ละเอียด
ลบข้อความได้
สรุป History ได้
เลือกเฉพาะข้อมูลที่เกี่ยวข้องได้
```

ข้อเสีย:

```text
ต้องเขียน Logic เอง
จัดการหลาย Sessions ยาก
เสี่ยงส่งข้อมูลปะปน
```

---

# 25. SQLiteSession

ตัวอย่าง:

```python
from agents import SQLiteSession

session = SQLiteSession("session-123")
```

จากนั้น:

```python
first = await Runner.run(
    agent,
    "My name is Dee.",
    session=session
)

second = await Runner.run(
    agent,
    "What is my name?",
    session=session
)
```

Session ช่วย:

```text
อ่าน History ก่อน Run
เพิ่ม History ลง Input
บันทึกข้อความใหม่หลัง Run
เชื่อมหลาย Runner Calls
```

---

# 26. Session ID

```python
SQLiteSession("session-123")
```

String นี้ใช้แยก Conversation

ตัวอย่าง:

```text
session-user-a
session-user-b
session-support-001
```

หากใช้ Session ID เดียวกันกับหลายผู้ใช้ ข้อมูลสนทนาอาจปะปนกัน

ดังนั้น Production ต้องสร้าง Session ID ที่:

```text
ไม่ซ้ำ
ผูกกับผู้ใช้หรือบทสนทนาอย่างถูกต้อง
ไม่เดาง่ายเมื่อเกี่ยวข้องกับ Security
```

---

# 27. In-memory Session

```python
SQLiteSession("session-123")
```

เมื่อไม่ระบุ Database File Session อาจทำงานใน Memory ของ Process

ลักษณะ:

```text
รวดเร็ว
เหมาะกับ Demo
ข้อมูลหายเมื่อ Process จบ
ไม่เหมาะกับหลาย Instance
```

---

# 28. Persistent SQLite Session

```python
SQLiteSession(
    "session-123",
    "memory.db"
)
```

Conversation Items จะถูกเก็บใน SQLite File

ข้อดี:

```text
Process Restart แล้วข้อมูลยังอยู่
ตรวจสอบข้อมูลได้
เหมาะกับ Local Prototype
```

ข้อจำกัด:

```text
ไม่เหมาะกับระบบกระจายหลาย Server
ต้องจัดการ Privacy
ต้องจัดการ Retention
ต้องป้องกัน Concurrent Access
```

---

# 29. Session ไม่ใช่ Model Memory

Session เป็น Storage ภายนอกโมเดล

Flow:

```text
Session Store
    ↓
Load Previous Items
    ↓
Send Items to Model
    ↓
Receive New Items
    ↓
Save Back to Session
```

โมเดลยังคง Stateless ในแต่ละ API Request

---

# 30. Session ไม่ใช่ Long-term Semantic Memory

Session เก็บ Conversation History ตามลำดับเวลา

แต่ไม่ได้ทำโดยอัตโนมัติ:

```text
เลือก Memory ที่เกี่ยวข้อง
ค้นหาแบบ Semantic Search
สรุป Fact ระยะยาว
ลืมข้อมูลที่หมดอายุ
แก้ Fact ที่ขัดแย้งกัน
จัดระดับความสำคัญ
```

ดังนั้น:

```text
Session
=
Conversation History Management

Long-term Memory
=
Storage + Retrieval + Selection + Update Policy
```

---

# 31. Trace, Session และ State

สามแนวคิดนี้ต้องแยกจากกัน

## Trace

```text
ใช้สังเกตว่า Agent ทำอะไร
```

## Session

```text
เก็บ Conversation Items ระหว่าง Runs
```

## State

```text
ข้อมูลการทำงานของ Task หรือ Workflow
```

ตัวอย่าง State:

```text
Todo List
Current Step
Selected Plan
Tool Result
Approval Status
```

Session อาจมีข้อความเกี่ยวกับ State แต่ไม่ได้แปลว่าเป็น Workflow State ที่มีโครงสร้าง

---

# 32. Misconceptions Corrected

## ความเข้าใจคลาดเคลื่อนที่ 1

> Agent Object คือ Agent ที่กำลังทำงาน

**ข้อเท็จจริง:**
Agent เป็น Configuration ส่วน Runner เป็น Runtime

---

## ความเข้าใจคลาดเคลื่อนที่ 2

> Runner คือโมเดล

**ข้อเท็จจริง:**
Runner เป็นตัวจัดการวงจรการทำงานและเรียกโมเดล

---

## ความเข้าใจคลาดเคลื่อนที่ 3

> Trace ทำให้ Agent จำข้อมูล

**ข้อเท็จจริง:**
Trace ใช้ Observability ไม่ใช่ Context Memory

---

## ความเข้าใจคลาดเคลื่อนที่ 4

> Streaming ทำให้ทุกอย่างเสร็จเร็วขึ้น

**ข้อเท็จจริง:**
Streaming ทำให้เห็นผลบางส่วนเร็วขึ้น แต่ Total Run Time อาจเท่าเดิม

---

## ความเข้าใจคลาดเคลื่อนที่ 5

> `@function_tool` ทำให้ Tool ปลอดภัย

**ข้อเท็จจริง:**
มันช่วยสร้าง Tool Interface แต่ยังต้องมี Security Controls

---

## ความเข้าใจคลาดเคลื่อนที่ 6

> Agent จำบทสนทนาเพราะใช้ Agent Object เดิม

**ข้อเท็จจริง:**
ต้องส่ง History หรือ Session เข้า Runner

---

## ความเข้าใจคลาดเคลื่อนที่ 7

> SQLiteSession คือ Memory ภายใน LLM

**ข้อเท็จจริง:**
เป็น Storage ภายนอกที่นำ History กลับเข้าสู่ Context

---

## ความเข้าใจคลาดเคลื่อนที่ 8

> Session คือ Long-term Memory ที่สมบูรณ์

**ข้อเท็จจริง:**
Session เป็น Conversation History Store ยังไม่มี Semantic Retrieval หรือ Memory Policy

---

# 33. Risks Identified

## 33.1 Tool Side Effects

Tools อาจ:

```text
ส่ง Notification
เขียน Database
ส่ง Email
แก้ไฟล์
```

ต้องควบคุมผลกระทบจริง

---

## 33.2 Session Data Leakage

ใช้ Session ID ผิดอาจส่ง History ของผู้ใช้หนึ่งไปให้อีกผู้ใช้

---

## 33.3 Unbounded Context

Conversation History ยาวขึ้นเรื่อย ๆ ทำให้:

```text
Token usage เพิ่ม
ค่าใช้จ่ายเพิ่ม
Latency เพิ่ม
Context Noise เพิ่ม
```

---

## 33.4 Persistent Sensitive Data

SQLite อาจเก็บ:

```text
ชื่อ
Email
ข้อมูลส่วนตัว
เนื้อหาสนทนา
Tool Results
```

ต้องมี Privacy และ Retention Policy

---

## 33.5 Streaming Error Handling

UI ต้องรองรับกรณี Stream หยุดกลางทางหรือ Tool Error

---

## 33.6 Trace Privacy

Trace อาจบันทึก Inputs, Outputs และ Tool Arguments ที่มีข้อมูลละเอียดอ่อน

---

# 34. Production Improvements

## Session Isolation

```text
หนึ่ง User/Conversation
→ หนึ่ง Session ID ที่เหมาะสม
```

## Context Compaction

```text
เก็บข้อความล่าสุด
+ สรุปข้อความเก่า
+ Retrieve เฉพาะข้อมูลที่เกี่ยวข้อง
```

## Tool Validation

```text
Type Validation
Business Validation
Permission Check
Rate Limit
Timeout
```

## Trace Redaction

ลบหรือปิดบัง:

```text
API Keys
Passwords
Personal Data
Medical Data
Confidential Code
```

## Persistent Store ที่เหมาะสม

ระบบขนาดใหญ่ควรพิจารณา:

```text
Database
Distributed Session Store
Encrypted Storage
Retention and Deletion Controls
```

---

# 35. Patterns Learned

## Agent–Runner Separation

```text
Agent
= Configuration

Runner
= Execution
```

## Managed Agent Loop

```text
Runner
→ Model
→ Tool
→ Observation
→ Model
```

## Function Tool Pattern

```text
Python Function
→ Decorator
→ Model-visible Tool
```

## Trace Pattern

```text
Run
→ Spans
→ Workflow Visualization
```

## Streaming Pattern

```text
Run Events
→ Partial UI Updates
```

## Session Pattern

```text
Run 1
→ Store History
→ Run 2 loads History
```

---

# 36. Connection to Week 1 Lab 3

Week 1 Lab 3:

```text
Manual Tool Schema
Manual Tool Handler
Manual while Loop
```

Week 2 Lab 1:

```text
@function_tool
Runner
SDK-managed Tool Loop
```

กลไกพื้นฐานเหมือนเดิม แต่ระดับ Abstraction สูงขึ้น

---

# 37. Connection to Week 1 Lab 4

Week 1 Lab 4:

```text
Explicit Tool Map
Multiple Tools
External Side Effects
```

Week 2 Lab 1:

```text
Agent.tools
Function Tools
Runner dispatches Tools
```

แม้ SDK จัดการ Dispatch แต่ Application ยังต้องกำหนดว่า Tools ใดถูกส่งให้ Agent

---

# 38. Connection to Week 1 Lab 5

Week 1 Lab 5:

```text
Agent Loop
+ External State
+ Planning
```

Week 2 Lab 1:

```text
Runner manages Loop
Session manages Conversation History
```

แต่ Session ไม่ได้แทน Structured Task State เช่น Todo Status หรือ Workflow Checkpoint

---

# 39. Week 2 Lab 1 Mental Model

```text
Define Agent
    ↓
Assign Instructions, Model and Tools
    ↓
Runner receives Input
    ↓
Runner manages Agent Loop
    ↓
Tools are executed when requested
    ↓
Trace records workflow
    ↓
Streaming exposes live events
    ↓
Session maintains conversation continuity
    ↓
Final Output returned
```

---

# 40. Final Lessons

## Lesson 1

Agent เป็น Configuration ส่วน Runner เป็น Execution Runtime

## Lesson 2

Runner ห่อหุ้ม Agent Loop ที่เคยเขียนด้วยตนเองใน Week 1

## Lesson 3

`final_output` คือผลลัพธ์สุดท้าย แต่ Result Object มีข้อมูลของ Run มากกว่านั้น

## Lesson 4

Tracing ใช้สังเกต Workflow ไม่ใช่ทำให้ Agent จำข้อมูล

## Lesson 5

Streaming ช่วยให้ผู้ใช้เห็น Progress และ Partial Output เร็วขึ้น

## Lesson 6

`@function_tool` ลดงานสร้าง Schema และ Dispatch แต่ไม่แทน Security Controls

## Lesson 7

Runner Calls เป็นอิสระจากกัน เว้นแต่ส่ง History หรือ Session

## Lesson 8

`SQLiteSession` จัดการ Conversation History ภายนอกโมเดล

## Lesson 9

Session, Trace, Workflow State และ Long-term Memory เป็นคนละระบบ

## Lesson 10

Framework ช่วยลด Boilerplate แต่ผู้พัฒนายังรับผิดชอบความถูกต้อง ความปลอดภัย และการควบคุม Side Effects

---

# 41. Memory Summary

```text
Week 2 Lab 1 เริ่มใช้ OpenAI Agents SDK

Agent
= Configuration:
name, instructions, model, tools

Runner
= Runtime ที่จัดการ Agent Loop

Runner.run()
คืน Run Result

result.final_output
คือคำตอบสุดท้าย

result.to_input_list()
ใช้แปลง Run เดิมเป็น Input สำหรับ Run ถัดไป

trace()
ใช้ Observability และ Debugging
ไม่ใช่ Memory

run_streamed()
ส่ง Events และ Partial Output
ช่วย User Experience แต่ไม่จำเป็นต้องลด Total Time

@function_tool
แปลง Python Function เป็น Tool
จาก Type Hints และ Docstring

SDK จัดการ Tool Schema, Dispatch
และ Tool Loop หลายส่วน

แต่ Tool ยังต้องมี:
Validation
Permission
Timeout
Error Handling
Rate Limit
Human Approval

Runner Calls ไม่จำกันโดยอัตโนมัติ

Conversation Continuity ทำได้สองแบบ:
1. ส่ง History ด้วย to_input_list()
2. ใช้ SQLiteSession

SQLiteSession แบบ In-memory
หายเมื่อ Process จบ

SQLiteSession แบบใช้ไฟล์
เก็บข้อมูลหลัง Restart ได้

Session
= Conversation History Management

Session ไม่ใช่:
Model Memory
Workflow State
Semantic Long-term Memory

Agent Framework ลด Boilerplate
แต่ไม่ลดความรับผิดชอบด้าน
Security, Correctness และ Governance
```

---

# 42. Next Episode

หัวข้อถัดไป:

```text
Week 2 — Lab 2

Code-based Orchestration
Agent as a Tool
Handoffs
Multi-Agent Collaboration
Manager Pattern
Decentralized Delegation
```

คำถามสำคัญสำหรับ Lab ถัดไปคือ:

> เมื่อระบบมี Agent หลายตัว เราควรให้โปรแกรมเป็นผู้ควบคุมลำดับงาน ให้ Agent หนึ่งเรียก Agent อื่นเป็น Tool หรือให้ Agent ส่งมอบงานผ่าน Handoff และแต่ละวิธีส่งผลต่อ Control, Context และ Traceability อย่างไร?

```
```
