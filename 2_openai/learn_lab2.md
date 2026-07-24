# Week 2 — Lab 2: Agent Orchestration

ไฟล์เรียน:

```text
2_openai/2_lab2.ipynb
```

Lab นี้เป็นโปรเจกต์ Agentic Framework ตัวแรกของ Week 2 โดยสร้างทีม Sales Agents สำหรับเขียน เลือก และส่ง Cold Email โครงสร้างบทเรียนแบ่งเป็นสามส่วน:

```text
Part 1: Email Setup
Part 2: Orchestrating by Code
Part 3: Orchestrating by LLM
        ├── Agents as Tools
        └── Handoffs
```

Notebook ใช้ OpenAI Agents SDK ร่วมกับ `asyncio`, Function Tools, Tracing และการแสดงกราฟความสัมพันธ์ของ Agents. ([GitHub][1])

---

## Learning Objectives

เมื่อจบ Lab 2 คุณควรอธิบายได้ว่า:

1. Agent orchestration หมายถึงอะไร
2. Code orchestration กับ LLM orchestration ต่างกันอย่างไร
3. `asyncio.gather()` ช่วยรัน Agents พร้อมกันอย่างไร
4. Generator, Picker และ Sender Agents มีหน้าที่ต่างกันอย่างไร
5. `ModelSettings(tool_choice="required")` มีผลอย่างไร
6. Agent-as-a-Tool ทำงานอย่างไร
7. Handoff ต่างจาก Agent-as-a-Tool อย่างไร
8. ใครเป็นเจ้าของ Final Answer ในแต่ละ Pattern
9. ทำไม Multi-Agent ไม่ได้รับประกันว่าระบบจะทำงานน่าเชื่อถือขึ้น
10. Trace และ Agent Graph ช่วยตรวจสอบ Orchestration อย่างไร

---

# 1. Orchestration คืออะไร

**Orchestration** คือการกำหนดว่า:

```text
Agent ใดทำงาน
ทำงานเมื่อใด
ทำตามลำดับใด
ผลลัพธ์ส่งไปให้ใคร
ใครตัดสินใจขั้นตอนถัดไป
ใครเป็นผู้ตอบผู้ใช้สุดท้าย
```

OpenAI Agents SDK แบ่งแนวทางหลักออกเป็นสองแบบ:

```text
1. Orchestration by Code
   โปรแกรมเป็นผู้กำหนด Flow

2. Orchestration by LLM
   LLM เป็นผู้เลือก Agent หรือเส้นทาง
```

สองแนวทางนี้สามารถผสมกันได้ Code orchestration มักควบคุมความเร็ว ค่าใช้จ่าย และลำดับงานได้คาดเดาง่ายกว่า ส่วน LLM orchestration เหมาะกับงานเปิดกว้างที่ต้องตัดสินใจตามบริบท. ([OpenAI][2])

---

# 2. ภาพรวมโปรเจกต์

Lab สร้าง Sales Team ที่มีบทบาทดังนี้:

```text
Professional Sales Agent
เขียนอีเมลแบบจริงจังและน่าเชื่อถือ

Humorous Sales Agent
เขียนแบบเป็นกันเองและมีอารมณ์ขัน

Executive Sales Agent
เขียนแบบกระชับและตรงประเด็น

Sales Picker / Sales Sender
เลือกอีเมลที่ดีที่สุดและส่งออก
```

ทั้งสาม Writer Agents ใช้บริบทธุรกิจเดียวกัน แต่ได้รับ Instructions ด้าน Style ต่างกัน. ([GitHub][1])

นี่คือ **Role Specialization**:

```text
เป้าหมายเดียวกัน
+
บทบาทต่างกัน
=
Candidate ที่หลากหลาย
```

---

# Part 1 — Email Setup

## 3. Email เป็น Optional Side Effect

Notebook ต้องการสาธิต Agent ที่ทำ Action จริงด้วยการส่ง Email ผ่าน SMTP แต่ระบุว่าส่วนสำคัญคือ Agent Workflow ไม่ใช่ระบบ Email ผู้เรียนสามารถใช้ Pushover แทนได้. ([GitHub][1])

Environment Variables ที่ใช้คือ:

```env
EMAIL_ADDRESS=...
EMAIL_SMTP_SERVER=...
EMAIL_APP_PASSWORD=...
```

โปรแกรมตรวจว่าค่าครบหรือไม่:

```python
USE_EMAIL = (
    EMAIL_ADDRESS
    and EMAIL_SMTP_SERVER
    and EMAIL_APP_PASSWORD
)
```

จากนั้นเลือกช่องทาง:

```text
ถ้า Email Config ครบ
→ ส่งผ่าน SMTP

ถ้า Config ไม่ครบ
→ ส่ง Pushover Notification
```

---

## 4. `send_message()` เป็น Abstraction

```python
def send_message(subject, text_body, html_body):
    if USE_EMAIL:
        send_email(subject, text_body, html_body)
    else:
        push(f"Subject: {subject}\n\n{text_body}")
```

Agent ไม่ต้องรู้ว่าระบบส่งผ่าน SMTP หรือ Pushover

Agent เห็นเพียงความสามารถระดับธุรกิจ:

```text
send_message
```

ส่วนรายละเอียด Infrastructure ถูกซ่อนไว้ด้านหลัง

Mental Model:

```text
Agent Intent
   ↓
Send Message
   ↓
Infrastructure Decision
   ├── SMTP
   └── Pushover
```

นี่เป็นตัวอย่างของ **Separation between business capability and implementation**.

---

## 5. ความเสี่ยงก่อนทดลอง

Email และ Push Notification เป็น Side Effects จริง จึงควรเริ่มจาก:

```python
USE_EMAIL = False
```

หรือเปลี่ยน Function ให้เพียงพิมพ์ข้อความ:

```python
def send_message(subject, text_body, html_body):
    print(subject)
    print(text_body)
```

ก่อนเปิดการส่งจริง

เหตุผลคือ Agent อาจ:

* ส่งข้อความซ้ำ
* สร้าง Subject ผิด
* ส่งเนื้อหาที่ยังไม่ได้ตรวจ
* ถูกเรียกหลายครั้งระหว่าง Debug
* ส่งข้อมูลที่ไม่ควรออกจากระบบ

---

# Part 2 — Orchestrating by Code

## 6. สร้าง Writer Agents สามตัว

```python
sales_agent1 = Agent(
    name="Professional Sales Agent",
    instructions=instructions1,
    model=MODEL_NAME
)

sales_agent2 = Agent(
    name="Humorous Sales Agent",
    instructions=instructions2,
    model=MODEL_NAME
)

sales_agent3 = Agent(
    name="Executive Sales Agent",
    instructions=instructions3,
    model=MODEL_NAME
)
```

Agent ทั้งสามรับ Input เดียวกัน:

```text
Write a cold sales email
```

แต่จะสร้าง Output คนละ Style

---

## 7. รัน Agents พร้อมกันด้วย `asyncio.gather()`

```python
results = await asyncio.gather(
    Runner.run(sales_agent1, message),
    Runner.run(sales_agent2, message),
    Runner.run(sales_agent3, message),
)
```

นี่คือ Parallel หรือ Concurrent Orchestration ผ่าน Code

Flow:

```text
                    ┌─ Professional Agent ─┐
User Request ───────├─ Humorous Agent ─────┼─ Results
                    └─ Executive Agent ────┘
```

ต่างจาก Week 1 Lab 2 ที่เรียกหลายโมเดลทีละตัว Lab นี้ใช้ `asyncio.gather()` เพื่อเริ่ม Agent Runs หลายตัวโดยไม่ต้องรอให้ตัวแรกเสร็จก่อนเริ่มตัวถัดไป. ([GitHub][1])

## ใช้ Parallel เมื่อใด

ใช้ได้เมื่อ Tasks:

```text
ไม่พึ่งพาผลของกันและกัน
สามารถเริ่มพร้อมกันได้
ผลลัพธ์นำมารวมทีหลังได้
```

ไม่ควรใช้ Parallel ถ้า:

```text
Task B ต้องใช้ผลจาก Task A
Task C ต้องรอการอนุมัติจาก Task B
```

---

# 8. Fan-out/Fan-in กลับมาอีกครั้ง

จาก Episodic Artifact Week 1:

```text
Fan-out
กระจายงานไปหลาย Worker

Fan-in
รวมผลลัพธ์กลับมา
```

ใน Lab นี้:

```text
Fan-out
→ Writer Agents สามตัว

Fan-in
→ Sales Picker
```

แตกต่างจาก Week 1 ตรงที่ตอนนี้ Worker แต่ละตัวเป็น `Agent` ของ SDK ไม่ใช่เพียงการเรียก API โดยตรง

---

# 9. Sales Picker Agent

```python
sales_picker = Agent(
    name="Sales_picker",
    instructions="""
    Pick the best cold sales email.
    Reply with the selected email only.
    """,
    model=MODEL_NAME
)
```

หลัง Writer Agents ทำงาน:

```python
outputs = [
    result.final_output
    for result in results
]
```

โปรแกรมรวม Candidate Emails:

```python
emails = (
    "Cold sales emails:\n\n"
    + "\n\nEmail:\n\n".join(outputs)
)
```

แล้วส่งให้ Picker:

```python
best = await Runner.run(
    sales_picker,
    emails
)
```

Flow สมบูรณ์:

```text
Writer 1 ─┐
Writer 2 ─┼─► Picker ─► Best Email
Writer 3 ─┘
```

---

## 10. Picker ไม่ใช่ Ground Truth

Sales Picker เป็น LLM Evaluator เช่นเดียวกับ LLM-as-a-Judge ใน Week 1

มันอาจเลือกจาก:

* Style
* ความยาว
* ความมั่นใจของภาษา
* ลำดับของ Candidate
* ความคล้ายกับรูปแบบที่ตนชอบ

ไม่ได้รับประกันว่า Email ที่เลือก:

* ได้ Conversion สูงสุดจริง
* สอดคล้องกับ Brand
* ไม่มี Claim เกินจริง
* ถูกต้องตามกฎหมาย
* เหมาะกับผู้รับทุกกลุ่ม

ดังนั้น Production Evaluation ควรเพิ่ม:

```text
Brand Compliance
Factual Accuracy
Tone Policy
Legal Constraints
Spam Risk
Human Approval
A/B Test Results
```

---

# 11. Function Tool สำหรับส่ง Email

```python
@function_tool
def send_email_tool(
    subject: str,
    text_body: str,
    html_body: str
) -> str:
    """
    Send out an email with the given subject and body.
    """
    send_message(subject, text_body, html_body)
    return "Email sent successfully"
```

Agents SDK สร้าง Tool Schema จาก:

```text
Function Name
Type Annotations
Docstring
Arguments
```

Notebook ตรวจ Schema ผ่าน:

```python
send_email_tool.params_json_schema
```

และให้ Sales Sender ใช้ Tool นี้. ([GitHub][1])

---

# 12. `tool_choice="required"`

```python
require_tool = ModelSettings(
    tool_choice="required"
)
```

แล้วตั้งค่า Agent:

```python
sales_sender = Agent(
    name="Sales Sender",
    instructions=decision,
    model=MODEL_NAME,
    tools=[send_email_tool],
    model_settings=require_tool
)
```

ความหมายคือโมเดลต้องเลือกใช้ Tool แทนที่จะตอบข้อความธรรมดาเพียงอย่างเดียว

Mental Model:

```text
tool_choice="auto"
โมเดลตัดสินใจว่าจะใช้ Tool หรือไม่

tool_choice="required"
โมเดลต้องสร้าง Tool Call
```

Notebook ใช้วิธีนี้เพราะ Agent อาจเลือก Email ได้ แต่ไม่ยอมเรียก Tool เพื่อส่งจริง. ([GitHub][1])

## ข้อควรระวัง

`required` รับประกันเพียงว่าโมเดลต้องเรียก Tool ไม่ได้รับประกันว่า:

* Arguments ถูกต้อง
* Email เหมาะสม
* Tool ทำงานสำเร็จ
* ควรส่ง Email ในสถานการณ์นั้นจริง

---

# 13. Code Orchestration ฉบับเต็ม

```text
1. Code เริ่ม Writer Agents สามตัวพร้อมกัน
2. Code รวบรวม Outputs
3. Code ส่ง Outputs ให้ Sales Sender
4. Sales Sender เลือก Email
5. Sales Sender เรียก send_email_tool
6. SDK Execute Tool
7. Agent ส่ง Final Response
```

ผู้ควบคุม Flow หลักคือ Python Code

```text
Code decides:
Writer Agents ใดต้องทำงาน
เมื่อใดต้องรวมผล
Agent ใดต้องเลือก
ขั้นตอนไหนต้องส่ง Email
```

---

# จุดแข็งของ Code Orchestration

* ลำดับงานชัดเจน
* Predictable กว่า
* Test แยกแต่ละ Step ได้
* บังคับ Parallel ได้
* ควบคุมจำนวน Model Calls ได้
* ใส่ Validation Gate ระหว่างขั้นได้ง่าย
* ประเมิน Cost และ Latency ง่ายกว่า

OpenAI ระบุว่า Code orchestration เหมาะกับการ Chain Agents, Loop กับ Evaluator และรันงานอิสระแบบ Parallel ผ่าน Python เช่น `asyncio.gather()`. ([OpenAI][2])

---

# ข้อจำกัดของ Code Orchestration

* Workflow แข็งและเปลี่ยนยากกว่า
* Programmer ต้องรู้เส้นทางล่วงหน้า
* รองรับงานเปิดกว้างได้น้อยกว่า
* Branch จำนวนมากทำให้ Code ซับซ้อน
* ต้องเขียน Routing Logic เอง

---

# Part 3 — Orchestrating by LLM

LLM Orchestration หมายถึงการให้โมเดลตัดสินใจว่า:

```text
ควรเรียก Specialist ตัวใด
ควรเรียกกี่ครั้ง
ควรส่งงานต่อหรือไม่
ขั้นตอนใดควรเกิดถัดไป
```

Lab แสดงสอง Pattern:

```text
3a. Agent as a Tool
3b. Handoff
```

---

# 14. Agent as a Tool

Agent สามารถแปลงเป็น Tool ผ่าน:

```python
tool1 = sales_agent1.as_tool(
    tool_name="sales_email_writer_1",
    tool_description=description
)
```

แล้วรวม Specialist Agents เป็น Tools:

```python
tools = [
    sales_agent1.as_tool(...),
    sales_agent2.as_tool(...),
    sales_agent3.as_tool(...),
    send_email_tool
]
```

OpenAI อธิบายว่า Agent-as-a-Tool ใช้เมื่อ Agent กลางยังต้องควบคุมบทสนทนาและเรียก Specialist เพื่อทำ Subtask ที่มีขอบเขตชัดเจน. ([OpenAI][2])

---

# 15. Flow ของ Agent-as-a-Tool

```text
Sales Manager
      ↓
เรียก Professional Agent เป็น Tool
      ↓
Specialist สร้าง Draft
      ↓
ผล Draft กลับไป Sales Manager
      ↓
Sales Manager เรียก Specialist ตัวอื่น
      ↓
Sales Manager รวมผลและตอบสุดท้าย
```

ย่อเป็น:

```text
Agent A → Agent B → Agent A
```

ตัว Manager ยังคงเป็นเจ้าของ Run และ Final Answer

---

# 16. Nested Agent Run

เมื่อ Manager เรียก Agent-as-a-Tool:

```text
Parent Run: Sales Manager
    ↓
Nested Run: Professional Writer
    ↓
Tool Output กลับ Parent Run
```

Specialist ไม่ได้กลายเป็น Agent หลักของบทสนทนา แต่ทำงานเหมือน Function ที่มี LLM และ Agent Loop อยู่ภายใน

Metaphor:

```text
Manager จ้างผู้เชี่ยวชาญเขียนรายงานย่อย
ผู้เชี่ยวชาญส่งรายงานกลับ
Manager ยังเป็นคนคุยกับลูกค้า
```

---

# 17. Sales Manager Agent

```python
sales_manager = Agent(
    name="Sales Manager",
    instructions=instructions,
    tools=[tool1, tool2, tool3, send_email_tool],
    model=MODEL_NAME
)
```

Instructions กำหนดให้ Manager:

```text
1. ใช้ Writer Agent ทั้งสาม
2. รวบรวม Drafts
3. เลือก Email ที่ดีที่สุด
4. ใช้ Tool ส่ง Email
```

แต่แตกต่างจาก Code Orchestration ตรงที่ Python ไม่ได้เรียก Agents ตามลำดับเอง LLM ของ Sales Manager เป็นผู้เลือก Tool Calls

---

# 18. จุดแข็งของ Agent-as-a-Tool

* Manager ควบคุม Final Answer
* Specialist มี Instructions เฉพาะทาง
* Manager สามารถรวมหลาย Outputs
* เหมาะกับ Planning Agent
* ใช้ Specialist แบบ Subtask ได้
* Shared policy สามารถรวมไว้ที่ Manager

Official SDK แนะนำ Pattern นี้เมื่อ Agent กลางต้องเป็นเจ้าของคำตอบสุดท้าย รวมผลจากหลาย Specialists หรือบังคับใช้ Guardrails ส่วนกลาง. ([OpenAI][2])

---

# 19. ข้อจำกัดของ Agent-as-a-Tool

* Manager อาจไม่เรียก Tool ครบ
* Manager อาจเรียก Tool ซ้ำ
* Manager ต้องรับ Context และ Outputs จำนวนมาก
* มี Nested Model Calls เพิ่ม Cost
* Specialist Result อาจถูก Manager สรุปผิด
* Context ของ Nested Run ไม่ได้สืบทอดทั้งหมดโดยอัตโนมัติในทุกกรณี

เอกสารปัจจุบันระบุว่า Nested Agent Run ของ `as_tool()` ไม่ได้รับ Conversation State ของ Parent โดยอัตโนมัติ หากต้องแชร์ Client-managed History ต้องกำหนด State Strategy อย่างชัดเจน. ([OpenAI][3])

---

# 20. Handoff

Handoff คือการที่ Agent หนึ่งโอนความรับผิดชอบให้ Agent อื่น

```python
sales_manager = Agent(
    name="Sales Manager",
    tools=[tool1, tool2, tool3],
    handoffs=[sales_sender],
    model=MODEL_NAME
)
```

เมื่อเกิด Handoff:

```text
Sales Manager
      ↓ transfer
Sales Sender
      ↓
Sales Sender กลายเป็น Active Agent
```

OpenAI ระบุว่า Agent ผู้รับ Handoff จะได้รับ Conversation History และรับช่วงการสนทนาต่อ โดย Handoff ถูกนำเสนอให้โมเดลในรูป Tool เช่น `transfer_to_refund_agent`. ([OpenAI][4])

---

# 21. Flow ของ Handoff

```text
Agent A → Agent B
```

ไม่ใช่:

```text
Agent A → Agent B → Agent A
```

โดยปกติ Agent B จะเป็นเจ้าของขั้นตอนต่อไปและ Final Output ของ Run

Metaphor:

```text
Agent-as-a-Tool:
หัวหน้าขอคำปรึกษาจากผู้เชี่ยวชาญ
แล้วกลับมาคุยกับลูกค้าต่อ

Handoff:
หัวหน้าโอนเคสให้ผู้เชี่ยวชาญ
ผู้เชี่ยวชาญรับผิดชอบลูกค้าต่อ
```

---

# 22. Handoff ในโปรเจกต์นี้

Final Pattern ของ Notebook คือ:

```text
Sales Manager
├── Professional Writer เป็น Tool
├── Humorous Writer เป็น Tool
├── Executive Writer เป็น Tool
└── Handoff ไป Sales Sender
```

Task กำหนดให้ Manager:

```text
1. เรียก Writer Tools ทั้งสาม
2. รอให้ Draft ทั้งหมดพร้อม
3. Handoff ไป Sales Sender
4. Sales Sender เลือกและส่ง Email
```

Notebook ใช้ Pattern ผสมระหว่าง Agent-as-a-Tool และ Handoff. ([GitHub][1])

---

# 23. ทำไม Writer เป็น Tools แต่ Sender เป็น Handoff

Writer Agents ทำ Subtask แบบจำกัด:

```text
เขียน Draft หนึ่งฉบับ
→ คืน Draft ให้ Manager
```

จึงเหมาะกับ Agent-as-a-Tool

Sales Sender รับผิดชอบ Stage ถัดไปทั้งหมด:

```text
รับ Drafts
→ เลือก
→ ส่ง
→ ตอบผลสุดท้าย
```

จึงเหมาะกับ Handoff

นี่เป็นหลักเลือก Pattern ที่สำคัญ:

```text
Specialist ช่วยงานย่อยแล้วคืนผล
→ Agent-as-a-Tool

Specialist ควรรับช่วงความรับผิดชอบ
→ Handoff
```

---

# 24. Agent Graph

Notebook ใช้:

```python
draw_graph(sales_manager)
```

เพื่อแสดงความสัมพันธ์ระหว่าง:

```text
Agents
Agent Tools
Function Tools
Handoffs
```

Graph ช่วยให้เห็น Static Architecture ส่วน Trace แสดง Runtime Execution

```text
Graph
= ระบบสามารถไปทางไหนได้

Trace
= Run นี้เดินทางไหนจริง
```

---

# 25. Graph กับ Trace ต่างกันอย่างไร

## Graph

ตอบว่า:

```text
Agent ใดเชื่อมกับใคร
มี Tool อะไร
Handoff ไปที่ใดได้
```

## Trace

ตอบว่า:

```text
Run นี้เรียก Agent ใดจริง
เรียก Tool ใด
เกิด Handoff หรือไม่
ใช้เวลานานเท่าไร
```

Graph คือแผนที่ ส่วน Trace คือประวัติการเดินทาง

---

# 26. ตารางเปรียบเทียบสาม Pattern

| ประเด็น          | Code Orchestration | Agent-as-a-Tool     | Handoff                     |
| ---------------- | ------------------ | ------------------- | --------------------------- |
| ผู้เลือก Flow    | Python Code        | Manager LLM         | Routing LLM                 |
| ผู้ถือ Control   | Application        | Manager Agent       | Specialist หลังรับงาน       |
| Final Answer     | Code กำหนด Agent   | Manager             | Agent ผู้รับ Handoff        |
| Specialist คืนผล | ให้ Code           | ให้ Manager         | ไม่จำเป็นต้องคืน Manager    |
| ความคาดเดาได้    | สูง                | ปานกลาง             | ต่ำกว่า                     |
| ความยืดหยุ่น     | ปานกลาง            | สูง                 | สูง                         |
| เหมาะกับ         | Fixed Workflow     | Subtasks            | Routing/Ownership Transfer  |
| Debug            | ง่ายกว่า           | ต้องดู Nested Trace | ต้องดู Handoff Boundary     |
| Context Load     | Code จัดการ        | Manager อาจรับมาก   | Specialist รับ Conversation |
| Failure Mode     | Logic Bug          | Tool ไม่ถูกเรียก    | Handoff ผิด Agent           |

หลักการนี้สอดคล้องกับคู่มือ SDK: Agent-as-a-Tool เหมาะเมื่อ Orchestrator ต้องรักษาการควบคุม ส่วน Handoff เหมาะเมื่อ Specialist ควรรับช่วงการสนทนา. ([OpenAI][2])

---

# 27. Misconceptions ที่ต้องแก้

## “Multi-Agent ทำงานดีกว่า Single Agent เสมอ”

ไม่จริง

Multi-Agent เพิ่ม:

```text
Model Calls
Prompt Complexity
Context Transfer
Failure Points
Latency
Cost
Debug Difficulty
```

ควรใช้เมื่อการแยกบทบาทสร้างประโยชน์จริง

---

## “Parallel Agents หมายถึง Multi-Agent ที่ Autonomous”

ไม่จำเป็น

ใน Code Orchestration:

```python
asyncio.gather(...)
```

Code เป็นผู้กำหนดทุก Agent ที่ต้องรัน จึงเป็น Multi-Agent Workflow แต่ไม่ได้เป็น Autonomous Orchestration

---

## “Agent-as-a-Tool เหมือน Function Tool ทุกอย่าง”

ไม่เหมือนทั้งหมด

Function Tool:

```text
Python Function เป็นผู้ทำงาน
```

Agent Tool:

```text
Nested Agent Run ใช้ LLM และอาจใช้ Tools เพิ่ม
```

ทั้งคู่ปรากฏเป็น Tool ต่อ Manager แต่ Cost, Latency และ Failure Mode ต่างกัน

---

## “Handoff คือเรียก Agent แล้วกลับมาหา Manager”

ไม่ใช่

นั่นเป็น Agent-as-a-Tool

Handoff คือการโอน Active Agent และความรับผิดชอบไปให้ Specialist

---

## “Instructions บอกให้เรียกทุก Agent จึงรับประกันว่าจะเรียกครบ”

ไม่รับประกัน

LLM อาจ:

* ข้าม Tool
* เรียก Tool ซ้ำ
* Handoff ก่อนเวลา
* เลือก Draft ก่อนครบ
* ไม่เรียก Sender

Notebook เองเตือนว่า Handoff อาจไม่น่าเชื่อถือและต้องปรับ Prompt, บังคับ Tool Use หรือใช้โมเดลที่มีความสามารถสูงขึ้น. ([GitHub][1])

---

## “Email sent successfully หมายถึงส่งสำเร็จจริง”

ไม่เสมอไป

ถ้า Function ไม่ตรวจ Exception หรือ SMTP Response อย่างถูกต้อง ข้อความ Success อาจเป็นเพียง String ที่ Function คืนกลับ

Production Tool ควรคืน:

```json
{
  "success": true,
  "message_id": "...",
  "provider": "smtp"
}
```

หรือ Controlled Error

---

# 28. จุดอ่อนเชิงสถาปัตยกรรมของ Lab

## Picker ไม่มี Structured Output

Picker คืน Email เป็นข้อความธรรมดา ทำให้ตรวจยากว่าเลือก Candidate หมายเลขใด

ควรใช้:

```json
{
  "selected_candidate": 2,
  "reason": "...",
  "subject": "...",
  "text_body": "...",
  "html_body": "..."
}
```

## ไม่มี Validation Gate ก่อนส่ง

ควรเพิ่ม:

```text
Drafts
→ Picker
→ Compliance Reviewer
→ Human Approval
→ Sender
```

## ไม่มี Deduplication

Run ซ้ำอาจส่ง Email ซ้ำ

## ไม่มี Idempotency Key

Tool Call ซ้ำอาจสร้าง Side Effect ซ้ำ

## ไม่มี Recipient Management

Lab ส่งกลับมายัง Email ของผู้เรียนเองเพื่อ Demo แต่ระบบจริงต้องตรวจ Recipient, Consent และ Opt-out

## ไม่มี Cost/Turn Limits

Manager อาจเรียก Specialist หลายครั้งเกินความจำเป็น

---

# 29. Orchestration ที่แข็งแรงขึ้น

```text
User Goal
   ↓
Code validates request
   ↓
Run three Writer Agents concurrently
   ↓
Structured Draft Outputs
   ↓
Picker Agent
   ↓
Deterministic Policy Checks
   ↓
Human Approval
   ↓
Sender Tool
   ↓
Delivery Result
   ↓
Audit Log
```

สิ่งที่ Code ควรควบคุม:

```text
ต้องสร้างกี่ Draft
ต้องตรวจ Policy อะไร
ส่งได้เมื่อใด
จำนวน Retry
Budget
Approval
```

สิ่งที่ LLM เหมาะจะตัดสิน:

```text
ถ้อยคำ
Style
การจัดโครงสร้าง
Candidate ที่น่าสนใจ
การปรับตามบริบท
```

---

# 30. วิธีทดสอบ Lab อย่างปลอดภัย

## Test 1: ปิด Side Effect

ให้ `send_message()` พิมพ์ข้อความแทนส่งจริง

## Test 2: Code Orchestration

ตรวจว่า Writer Agents ทั้งสามคืน Output และ Trace แสดง Parallel Runs

## Test 3: Picker

สลับลำดับ Emails แล้วดูว่า Winner เปลี่ยนหรือไม่ เพื่อตรวจ Position Bias

## Test 4: Agent-as-a-Tool

ตรวจ Trace ว่า Manager เรียก Writer Tools ครบหรือไม่

## Test 5: Handoff

ตรวจว่า Final Agent เป็น Sales Sender ไม่ใช่ Sales Manager

## Test 6: Failure

ทำให้ Writer Agent ตัวหนึ่ง Error แล้วดูว่า Workflow จัดการอย่างไร

## Test 7: Duplicate Protection

เรียก Workflow สองครั้ง แต่ไม่ให้เกิดการส่งจริงซ้ำ

---

# 31. Checklist ความเข้าใจ

### Code Orchestration คืออะไร

Python เป็นผู้กำหนด Agents ลำดับ Parallelism และการส่ง Output ต่อ

### `asyncio.gather()` ใช้ทำอะไร

เริ่ม Async Agent Runs หลายตัวที่ไม่พึ่งพากัน และรอผลทั้งหมด

### Agent-as-a-Tool คืออะไร

ทำให้ Specialist Agent ปรากฏเป็น Tool ของ Manager และคืนผลกลับให้ Manager

### Handoff คืออะไร

โอน Active Agent และความรับผิดชอบการสนทนาไปยัง Specialist

### ใครตอบสุดท้ายใน Agent-as-a-Tool

Manager Agent

### ใครตอบสุดท้ายหลัง Handoff

Agent ผู้รับ Handoff

### `tool_choice="required"` รับประกันอะไร

รับประกันว่าต้องเกิด Tool Call แต่ไม่รับประกันความถูกต้องหรือความเหมาะสมของ Action

### Graph กับ Trace ต่างกันอย่างไร

Graph แสดงเส้นทางที่เป็นไปได้ ส่วน Trace แสดงเส้นทางที่เกิดขึ้นจริงใน Run

---

# 32. แก่นของ Lab 2

```text
Orchestration by Code
=
Application เลือก Flow
Predictable และควบคุมง่ายกว่า

Agent as a Tool
=
Manager รักษา Control
Specialist ทำ Subtask แล้วคืนผล

Handoff
=
Specialist รับช่วง Control
และเป็นเจ้าของขั้นตอนถัดไป
```

สิ่งสำคัญที่สุดไม่ใช่การสร้าง Agent จำนวนมาก แต่คือการตอบคำถามว่า:

> **ใครควรเป็นผู้ควบคุม Workflow และใครควรเป็นเจ้าของ Final Answer ในแต่ละช่วงของงาน**

ในระบบที่ต้องการความแน่นอน ควรให้ Code ควบคุมโครงหลัก แล้วเปิดพื้นที่ให้ LLM ตัดสินใจเฉพาะส่วนที่ต้องใช้ความเข้าใจและความยืดหยุ่น ส่วน Handoff ควรใช้เมื่อการโอนความรับผิดชอบไปยัง Specialist มีความหมายจริง ไม่ใช่เพียงเพราะ Framework รองรับครับ

[1]: https://raw.githubusercontent.com/ed-donner/agents/main/2_openai/2_lab2.ipynb "raw.githubusercontent.com"
[2]: https://openai.github.io/openai-agents-python/multi_agent/ "Agent orchestration - OpenAI Agents SDK"
[3]: https://openai.github.io/openai-agents-python/tools/ "Tools - OpenAI Agents SDK"
[4]: https://openai.github.io/openai-agents-python/handoffs/ "Handoffs - OpenAI Agents SDK"
