## เริ่ม Week 2 — OpenAI Agents SDK
---

# ภาพรวม Week 2

Repository ปัจจุบันแบ่ง Week 2 อยู่ในโฟลเดอร์ `2_openai` มี Notebook หลักสี่ Lab พร้อมโปรเจกต์ `deep_research`, โฟลเดอร์ `code` และ `messenger.py` สำหรับส่วนประกอบของโปรเจกต์. ([GitHub][1])

| Lab   | หัวข้อหลัก                                                     |
| ----- | -------------------------------------------------------------- |
| Lab 1 | Agent, Runner, Tracing, Streaming, Function Tools และ Sessions |
| Lab 2 | Orchestration ด้วย Code, Agent-as-Tool และ Handoffs            |
| Lab 3 | หลายโมเดล, Structured Outputs และ Guardrails                   |
| Lab 4 | Deep Research Multi-Agent Project                              |

Week 2 จึงไม่ได้สอน Agent Loop ใหม่ตั้งแต่ศูนย์ แต่เปลี่ยนจากการเขียน Loop และ Tool Dispatch ด้วยมือใน Week 1 ไปใช้ **OpenAI Agents SDK เป็น abstraction layer**. ([GitHub][2])

---

# Mental Model: Week 1 เทียบกับ Week 2

```text
Week 1: เขียนเอง                  Week 2: Agents SDK

System Prompt                    Agent.instructions
ชื่อและบทบาท                     Agent.name
เรียกโมเดล                       Runner.run()
Manual while loop                Runner จัดการ Agent Loop
Tool Schema + Tool Map           @function_tool
Message History                  Session
Print Debug                      Trace
ส่งข้อความทีละก้อน              Streaming
```

สิ่งสำคัญคือ Framework ไม่ได้เปลี่ยนหลักการทำงานของ Agent แต่ช่วยจัดการรายละเอียดซ้ำ ๆ เช่น Agent Loop, Tool Schema, Tool Calls, Message History และ Tracing

---

# Week 2 — Lab 1

ไฟล์:

```text
2_openai/1_lab1.ipynb
```

Notebook แบ่งเป็นสามส่วน:

```text
Part 1
Agent + Runner + Trace + Streaming

Part 2
Function Tool

Part 3
Sessions หรือ Conversation Memory
```

Notebook ปัจจุบันใช้ `Agent`, `Runner`, `trace`, `function_tool` และ `SQLiteSession` จาก OpenAI Agents SDK. ([GitHub][2])

---

## Learning Objectives

เมื่อจบ Lab 1 คุณควรอธิบายได้ว่า:

1. `Agent` และ `Runner` ต่างกันอย่างไร
2. Runner จัดการ Agent Loop ส่วนใดให้เรา
3. `final_output` และ `to_input_list()` ใช้ทำอะไร
4. Trace ช่วย Debug Agent อย่างไร
5. Streaming แตกต่างจากการรอ Final Response อย่างไร
6. `@function_tool` เปลี่ยน Python Function ให้เป็น Tool ได้อย่างไร
7. เหตุใด `Runner.run()` แต่ละครั้งจึงไม่จำการสนทนาเอง
8. การส่ง History ด้วยมือและการใช้ `SQLiteSession` ต่างกันอย่างไร
9. Session เป็น Context Management ไม่ใช่ความทรงจำของโมเดล

---

# 1. Package ที่ต้องติดตั้งให้ถูก

ชื่อ Package บน PyPI คือ:

```text
openai-agents
```

ติดตั้งด้วย:

```powershell
uv add openai-agents
```

แต่ตอน Import ใช้:

```python
from agents import Agent, Runner, trace
```

อย่าใช้:

```powershell
pip install agents
```

เพราะ `agents` เป็น Package คนละตัวที่เก่ากว่าและเกี่ยวกับ Reinforcement Learning ไม่ใช่ OpenAI Agents SDK ตามที่ Notebook เตือนไว้. ([GitHub][2])

ใน Repository หลัก Dependency ถูกจัดการให้แล้ว จึงเริ่มจาก:

```powershell
cd <agents2>
uv sync
```

จากนั้นเปิด:

```text
2_openai/1_lab1.ipynb
```

---

# 2. สร้าง Agent ตัวแรก

```python
from agents import Agent

agent = Agent(
    name="Jokester",
    instructions="You are a joke teller",
    model="gpt-5.4-mini"
)
```

## `Agent` คืออะไร

`Agent` คือ Configuration Object ที่รวบรวม:

```text
Identity
+ Instructions
+ Model
+ Tools
+ Guardrails
+ Handoffs
+ Output Type
```

ในตัวอย่างนี้ Agent มีเพียง:

```text
Name         = Jokester
Instructions = You are a joke teller
Model        = gpt-5.4-mini
```

การสร้าง `Agent(...)` ยังไม่ได้ส่ง Request ไปยังโมเดล และยังไม่ได้ทำงานใด ๆ

Mental Model:

```text
Agent
=
Job description และ configuration
```

---

# 3. `Runner` เป็นผู้เริ่ม Agent Loop

```python
from agents import Runner

result = await Runner.run(
    agent,
    "Tell a joke about Autonomous AI Agents"
)
```

`Runner.run()` รับ:

```text
Agent Configuration
+
User Input
```

จากนั้น Framework จะจัดการ Run Lifecycle และ Agent Loop จนได้ Final Output

```text
Input
 ↓
Runner
 ↓
Call Model
 ↓
Model ตอบหรือเรียก Tool
 ↓
ถ้ามี Tool → Execute และกลับไปหา Model
 ↓
Final Output
```

ในตัวอย่างง่ายที่ไม่มี Tool การ Run อาจมีเพียง Model Call เดียว แต่ Runner ยังเป็น abstraction เดียวกับที่รองรับ Tool Calls และ Agent Loop ในภายหลัง. ([GitHub][2])

## สิ่งที่ต้องไม่เข้าใจผิด

```text
Agent ≠ Agent Loop
Runner = ตัวดำเนิน Agent Loop
```

Agent บอกว่า “ใครทำงานและทำอย่างไร” ส่วน Runner คือ “กลไกที่ทำให้งานนั้นเกิดขึ้น”

---

# 4. `await` หมายถึงอะไร

```python
result = await Runner.run(...)
```

`Runner.run()` เป็น Async Operation เพราะต้องรอ Network และอาจต้องรอ Tool หรือ Agent อื่น

คำว่า `await` หมายถึง:

> หยุดรอผลของงาน Async นี้โดยไม่บล็อกระบบ Async ทั้งหมด

ใน Jupyter Notebook สามารถใช้ `await` ที่ระดับ Cell ได้โดยตรง แต่ในไฟล์ Python ปกติมักอยู่ใน:

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

# 5. `result.final_output`

```python
print(result.final_output)
```

`result` ไม่ใช่ String แต่เป็น Result Object ที่มีรายละเอียดของ Run

```text
Run Result
├── final_output
├── Agent ที่ทำงานสุดท้าย
├── Input/Output Items
├── Tool Calls
├── Usage
└── Run Context
```

`final_output` คือผลลัพธ์สุดท้ายที่เหมาะสำหรับส่งให้ผู้ใช้

```python
answer = result.final_output
```

แต่ถ้าต้องการตรวจรายละเอียดของกระบวนการ ต้องอ่านส่วนอื่นใน Result

---

# 6. `to_input_list()`

```python
result.to_input_list()
```

Function นี้แปลงรายการเหตุการณ์และข้อความจาก Run ให้กลับมาเป็น Input Format ที่สามารถส่งเข้า Run ต่อไปได้

ภาพง่าย ๆ:

```text
Run แรก
User → Assistant

result.to_input_list()
        ↓
รายการข้อความที่เกิดขึ้นแล้ว
        ↓
นำไปต่อกับข้อความใหม่
```

ตัวอย่าง:

```python
first = await Runner.run(
    agent,
    "My name is Ed."
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

นี่คือวิธีจัด Conversation History ด้วยมือ ซึ่งต่อยอดตรงจากแนวคิด `messages` ใน Week 1. ([GitHub][2])

---

# 7. Tracing

```python
from agents import trace

with trace("Telling a joke"):
    result = await Runner.run(
        agent,
        "Tell a joke about Autonomous AI Agents"
    )
```

Trace บันทึกเหตุการณ์ที่เกิดขึ้นระหว่าง Run เช่น:

```text
Model Calls
Tool Calls
Handoffs
Guardrails
Timing
Run Relationships
```

เอกสารทางการระบุว่า Agents SDK มี Built-in Tracing สำหรับบันทึกและแสดง Workflow เพื่อใช้ Debug, Visualize และ Monitor ระบบ Agent. ([OpenAI GitHub][3])

Mental Model:

```text
Log
มักเป็นข้อความเหตุการณ์แยก ๆ

Trace
แสดงความสัมพันธ์ของทั้ง Workflow
```

Trace จึงตอบคำถามได้ เช่น:

* Agent ตัวใดทำงาน
* เรียกโมเดลกี่ครั้ง
* Tool ใดถูกเรียก
* Tool คืนอะไร
* Agent เปลี่ยนมือหรือไม่
* ขั้นตอนไหนใช้เวลานาน

## Trace ไม่ใช่ Memory

Trace เก็บข้อมูลเพื่อ Observability แต่ไม่ได้ถูกส่งกลับให้ Agent เป็น Context โดยอัตโนมัติ

```text
Trace = ระบบสังเกตการณ์
Session = ระบบจัดการ Conversation History
```

---

# 8. Streaming

```python
from openai.types.responses import ResponseTextDeltaEvent

result = Runner.run_streamed(
    agent,
    input="Please tell me 5 jokes about AI Agents."
)

async for event in result.stream_events():
    if (
        event.type == "raw_response_event"
        and isinstance(
            event.data,
            ResponseTextDeltaEvent
        )
    ):
        print(
            event.data.delta,
            end="",
            flush=True
        )
```

`Runner.run_streamed()` ให้ Result แบบ Streaming และ `stream_events()` ส่ง Event ออกมาระหว่างที่ Agent กำลังทำงาน เอกสารทางการอธิบายว่า Streaming เหมาะสำหรับแสดง Partial Responses หรือ Progress ให้ผู้ใช้เห็นระหว่าง Run. ([OpenAI GitHub][4])

## Streaming ไม่ได้แปลว่าโมเดลทำงานเสร็จเร็วขึ้น

ความแตกต่างคือประสบการณ์ของผู้ใช้:

```text
ไม่ Streaming:
รอทั้งหมด → เห็นคำตอบพร้อมกัน

Streaming:
คำตอบค่อย ๆ ปรากฏระหว่างสร้าง
```

Time to First Token อาจดีขึ้น แต่เวลาที่ใช้สร้าง Output ทั้งหมดไม่ได้จำเป็นต้องลดลง

---

# 9. Function Tool

ใน Week 1 เราต้องเขียนเอง:

```text
Python Function
+ JSON Schema
+ Tool Map
+ Dispatch
```

Agents SDK ลดงานส่วนนี้ด้วย Decorator:

```python
from agents import function_tool

@function_tool
def push_tool(message: str) -> str:
    """Send the given message to the user
    as a push notification."""
    
    payload = {
        "user": pushover_user,
        "token": pushover_token,
        "message": message
    }

    status = requests.post(
        pushover_url,
        data=payload
    ).status_code

    return (
        f"Push sent with API "
        f"status code {status}"
    )
```

`function_tool` ใช้ Python Function, Type Annotations และคำอธิบายเพื่อสร้าง Function Tool ที่โมเดลมองเห็นได้ เอกสารทางการแนะนำให้ใช้ Helper นี้เพื่อ Wrap Python Function เป็น `FunctionTool`. ([OpenAI GitHub][5])

ตรวจรายละเอียดได้ด้วย:

```python
push_tool.description
```

หรือข้อมูล Schema/Metadata อื่นของ Tool

---

# 10. เพิ่ม Tool ให้ Agent

```python
notifier = Agent(
    name="Notifier",
    model="gpt-5.4-mini",
    instructions=(
        "You notify the user upon request"
    ),
    tools=[push_tool]
)
```

จากนั้น Run:

```python
with trace("Pizza has arrived"):
    result = await Runner.run(
        notifier,
        "Notify the user that the pizza is here"
    )

print(result.final_output)
```

Flow ที่เกิดขึ้น:

```text
User asks for notification
        ↓
Runner calls Model
        ↓
Model requests push_tool
        ↓
SDK executes Python Function
        ↓
Tool Result returns to Model
        ↓
Model creates Final Response
```

นี่คือ Agent Loop เดียวกับ Lab 3–4 ของ Week 1 แต่ SDK จัดการ Tool Schema, Tool Dispatch, Tool Result และ Loop ให้เรา. ([GitHub][2])

## จุดที่ต้องจำ

`@function_tool` ทำให้สร้าง Tool ง่ายขึ้น แต่ไม่ได้ทำให้ Tool ปลอดภัยโดยอัตโนมัติ

เรายังต้องเพิ่ม:

```text
Input Validation
Timeout
Error Handling
Permission
Rate Limit
Human Approval
Audit Log
```

---

# 11. Runner แต่ละ Call เริ่มใหม่

Notebook ทดลอง:

```python
agent = Agent(
    name="Assistant",
    model="gpt-5.4-mini"
)

response = await Runner.run(
    agent,
    "Hi there. My name is Ed."
)

response = await Runner.run(
    agent,
    "What's my name?"
)
```

Call ที่สองไม่จำเป็นต้องรู้ชื่อ เพราะ `Runner.run()` ใหม่เป็น Run ใหม่ หากไม่ได้ส่ง History หรือ Session เข้าไป. ([GitHub][2])

นี่สอดคล้องกับ Episodic Artifact ของ Week 1:

> ตัวโมเดลไม่ได้จำ Conversation เดิมเอง Application ต้องจัด Context ให้

---

# 12. Memory Approach 1: ส่ง History ด้วยมือ

```python
first = await Runner.run(
    agent,
    "Hi there. My name is Ed."
)

next_input = first.to_input_list() + [
    {
        "role": "user",
        "content": "What's my name?"
    }
]

second = await Runner.run(
    agent,
    next_input
)
```

ข้อดีคือ Control ชัดเจน:

* เลือกข้อความที่จะส่งได้
* ลบข้อความบางส่วนได้
* Summarize History ได้
* Debug ง่าย

ข้อเสียคือ Application ต้องจัดการเองทุก Turn

---

# 13. Memory Approach 2: `SQLiteSession`

```python
from agents import SQLiteSession

session = SQLiteSession("12346")
```

ใช้กับ Runner:

```python
first = await Runner.run(
    agent,
    "Hi there. My name is Ed.",
    session=session
)

second = await Runner.run(
    agent,
    "What's my name?",
    session=session
)
```

Session จะดึง History ก่อน Run และเก็บข้อความใหม่หลัง Run ให้โดยอัตโนมัติ ทำให้หลาย Calls ใช้ Conversation Context เดียวกันได้. ([OpenAI GitHub][6])

## In-memory Session

```python
SQLiteSession("12346")
```

โดย Default ใช้ SQLite แบบ In-memory ข้อมูลจะหายเมื่อ Process สิ้นสุด. ([OpenAI GitHub][7])

## Persistent Session

```python
SQLiteSession(
    "12346",
    "memory.db"
)
```

เมื่อระบุไฟล์ Database ข้อมูลสามารถอยู่ต่อหลัง Process Restart ได้ ตามตัวอย่างใน Notebook. ([GitHub][2])

---

# Session ไม่เท่ากับ Long-term Semantic Memory

`SQLiteSession` เก็บ Conversation Items เพื่อใช้เป็น Context ต่อเนื่อง

มันไม่ได้ทำสิ่งเหล่านี้โดยอัตโนมัติ:

```text
ค้นหาความทรงจำที่เกี่ยวข้อง
สรุปประสบการณ์ระยะยาว
แยก Fact ออกจากความคิดเห็น
ลืมข้อมูลที่หมดอายุ
จัดลำดับความสำคัญของ Memory
```

ดังนั้นคำที่แม่นยำกว่าคือ:

```text
Session
=
Conversation History Management
```

ไม่ใช่ระบบ Long-term Memory ที่มี Retrieval และ Memory Policy ครบถ้วน

---

# จุดที่มักเข้าใจผิดใน Lab 1

| ความเข้าใจคลาดเคลื่อน             | ความหมายที่ถูกต้อง                        |
| --------------------------------- | ----------------------------------------- |
| `Agent` ทำงานทันที                | Agent เป็น Configuration                  |
| `Runner` คือ Model                | Runner เป็น Runtime/Loop                  |
| Trace ทำให้ Agent จำได้           | Trace ใช้ Observability                   |
| Streaming ทำให้งานทั้งหมดเร็วขึ้น | ทำให้เห็น Output ก่อนครบ                  |
| Function Tool ปลอดภัยแล้ว         | ยังต้อง Validation และ Permission         |
| Session คือ Memory ภายใน LLM      | Session เก็บ History ภายนอกโมเดล          |
| SQLiteSession แบบ Default ถาวร    | Default แบบ In-memory หายเมื่อ Process จบ |

---

# แบบฝึกหัด Lab 1

Notebook ให้เลือกหนึ่งโปรเจกต์จาก Week 1 แล้วเขียนใหม่ด้วย Agents SDK เช่น Digital Twin หรือ Checklist Agent. ([GitHub][2])

แบบฝึกหัดที่เหมาะที่สุดคือสร้าง Checklist Agent:

```python
from agents import Agent, Runner, function_tool

tasks: list[dict] = []


@function_tool
def create_task(description: str) -> str:
    """Create a task in the checklist."""
    tasks.append({
        "description": description,
        "completed": False
    })
    return f"Created: {description}"


@function_tool
def complete_task(index: int) -> str:
    """Mark a checklist task as completed."""
    if index < 1 or index > len(tasks):
        return "Invalid task index"

    tasks[index - 1]["completed"] = True
    return (
        f"Completed: "
        f"{tasks[index - 1]['description']}"
    )


planner = Agent(
    name="Checklist Planner",
    instructions=(
        "Break the user's goal into tasks. "
        "Create each task using create_task. "
        "Complete tasks only after reasoning "
        "through the required work."
    ),
    model="gpt-5.4-mini",
    tools=[create_task, complete_task]
)

result = await Runner.run(
    planner,
    "Plan the steps required to test "
    "a small Python API."
)

print(result.final_output)
```

สิ่งที่ควรสังเกตระหว่าง Run:

```text
Runner เรียกโมเดลกี่ครั้ง
Tool ใดถูกเรียกก่อน
Arguments ถูกสร้างอย่างไร
Tool Result ถูกส่งกลับอย่างไร
Agent หยุดเมื่อใด
Trace แสดง Agent Loop แบบใด
```

แก่นของ Lab 1 คือ:

> Week 1 ทำให้เราเข้าใจกลไกภายใน ส่วน OpenAI Agents SDK นำกลไก Agent Loop, Tools, Tracing, Streaming และ Session มาห่อเป็น Primitive ที่เล็กและใช้งานง่ายขึ้น โดยเรายังต้องเข้าใจข้อจำกัดและควบคุมความเสี่ยงของระบบเอง.

[1]: https://github.com/ed-donner/agents/tree/main/2_openai "agents/2_openai at main · ed-donner/agents · GitHub"
[2]: https://github.com/ed-donner/agents/raw/refs/heads/main/2_openai/1_lab1.ipynb "raw.githubusercontent.com"
[3]: https://openai.github.io/openai-agents-python/tracing/?utm_source=chatgpt.com "Tracing - OpenAI Agents SDK"
[4]: https://openai.github.io/openai-agents-python/streaming/?utm_source=chatgpt.com "Streaming - OpenAI Agents SDK"
[5]: https://openai.github.io/openai-agents-python/ref/tool/?utm_source=chatgpt.com "Tools - OpenAI Agents SDK"
[6]: https://openai.github.io/openai-agents-python/sessions/?utm_source=chatgpt.com "Overview - OpenAI Agents SDK"
[7]: https://openai.github.io/openai-agents-python/ref/memory/?utm_source=chatgpt.com "Memory - OpenAI Agents SDK"
