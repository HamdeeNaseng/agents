# Episodic Learning Artifact

## Week 1 — Lab 3: Digital Twin, Tool Calling และ Agent Loop

**หลักสูตร:** AI Engineer Agentic Track
**ผู้สอน:** Ed Donner
**Repository:** `ed-donner/agents`
**ไฟล์เรียน:** `1_foundations/3_lab3.ipynb`
**หัวข้อหลัก:** Digital Twin, System Prompt, Conversation History, Tool Calling และ Agent Loop
**สถานะ:** เรียนและสรุปแนวคิดพื้นฐานแล้ว

---

# 1. Context

Lab 3 เป็นจุดเปลี่ยนจากระบบที่เรียก LLM ตามลำดับแบบตายตัว ไปสู่ระบบที่โมเดลสามารถตัดสินใจขอใช้เครื่องมือได้

พัฒนาการจาก Lab ก่อนหน้าเป็นดังนี้:

```text
Lab 1
LLM Call
→ Chained Calls
→ Generator–Evaluator

Lab 2
หลายโมเดล
→ Fan-out/Fan-in
→ LLM-as-a-Judge

Lab 3
Conversation Context
→ Tool Calling
→ Tool Result
→ Agent Loop
```

Lab นี้เริ่มสร้าง **Career Digital Twin** ซึ่งเป็น AI ตัวแทนของบุคคล สามารถตอบคำถามเกี่ยวกับประสบการณ์ ความสามารถ และประวัติการทำงาน รวมถึงบันทึกข้อมูลติดต่อของผู้สนใจผ่าน Tool

---

# 2. Learning Objectives

หลังจบ Lab 3 สามารถอธิบายได้ว่า:

1. Digital Twin ใช้ข้อมูลส่วนตัวเป็น Context อย่างไร
2. System Prompt และ Conversation History แตกต่างกันอย่างไร
3. เหตุใด LLM จึงดูเหมือนมี Memory
4. Gradio เชื่อม User Interface กับ LLM Function อย่างไร
5. Python Function และ Tool Definition แตกต่างกันอย่างไร
6. โมเดลไม่ได้ Execute Tool ด้วยตนเอง
7. `tool_calls` และ `tool_call_id` ทำหน้าที่อะไร
8. Tool Result ต้องถูกส่งกลับไปหาโมเดลอย่างไร
9. Agent Loop เกิดขึ้นจากการเรียก Model และ Tool ซ้ำอย่างไร
10. Agent Framework ช่วยซ่อนขั้นตอนใดบ้าง

---

# 3. Project Situation

ระบบ Digital Twin มีโครงสร้างพื้นฐานดังนี้:

```text
ผู้เยี่ยมชมเว็บไซต์
        ↓
Gradio Chat Interface
        ↓
Python chat function
        ↓
System Prompt + Conversation History
        ↓
LLM
        ↓
ตอบคำถาม หรือ ขอเรียก Tool
```

ข้อมูลที่ใช้สร้างตัวแทนบุคคลมาจาก:

```text
twin/
├── linkedin.pdf
└── summary.txt
```

`linkedin.pdf` ให้ข้อมูลประวัติและประสบการณ์ ส่วน `summary.txt` ใช้กำหนดคำอธิบายตนเองในรูปแบบที่ควบคุมได้มากกว่า

---

# 4. Episode: Preparing Personal Context

## 4.1 อ่านข้อมูลจาก PDF

โปรแกรมใช้ Python อ่านข้อความจาก PDF:

```text
linkedin.pdf
    ↓
PdfReader
    ↓
extract_text()
    ↓
ข้อความธรรมดา
```

LLM ไม่ได้เปิด PDF ด้วยตัวเอง โปรแกรมเป็นผู้ Extract ข้อความออกมาแล้วใส่ใน Prompt

ลำดับที่ถูกต้องคือ:

```text
ไฟล์ PDF
→ Python อ่านไฟล์
→ Extract Text
→ รวมข้อความ
→ ใส่ใน System Prompt
→ ส่งให้ LLM
```

## ข้อจำกัดที่พบ

การ Extract PDF อาจมีปัญหาเมื่อ:

* PDF เป็นภาพ Scan
* ไม่มี Text Layer
* เอกสารมีหลายคอลัมน์
* Layout ซับซ้อน
* ตัวอักษรหรือ Encoding ผิด

ดังนั้น Code ไม่ Error ไม่ได้หมายความว่าข้อมูลถูกอ่านถูกต้อง ต้องตรวจข้อความที่ Extract ออกมาด้วย

---

## 4.2 อ่าน `summary.txt`

ไฟล์ข้อความถูกอ่านด้วย UTF-8:

```python
with open("twin/summary.txt", encoding="utf-8") as file:
    summary = file.read()
```

ข้อมูลในไฟล์นี้อาจประกอบด้วย:

* ตำแหน่งงาน
* ประสบการณ์
* ความเชี่ยวชาญ
* ผลงานสำคัญ
* เป้าหมายทางอาชีพ
* รูปแบบการแนะนำตัว

## สิ่งที่เรียนรู้

`summary.txt` ไม่ได้เป็น Memory ของโมเดล แต่เป็นข้อมูลที่ Application ส่งให้โมเดลทุกครั้งผ่าน System Prompt

---

# 5. System Prompt

System Prompt ใช้กำหนด:

```text
Identity
+ Role
+ Context
+ Rules
+ Behavioral Boundaries
```

ตัวอย่างหน้าที่ของ System Prompt:

* บอกว่า AI เป็น Digital Twin ของใคร
* ให้ข้อมูลเกี่ยวกับประสบการณ์และความสามารถ
* กำหนดให้ตอบเฉพาะเรื่องวิชาชีพ
* ระบุว่าไม่ควรแต่งข้อมูล
* กำหนดวิธีตอบเมื่อไม่มีข้อมูล
* บอกให้เชิญผู้สนใจฝากข้อมูลติดต่อ

Mental Model:

```text
System Prompt
=
Job Description
+ Operating Instructions
+ Knowledge Context
+ Policy
```

## สิ่งที่ต้องไม่เข้าใจผิด

System Prompt ไม่ใช่ระบบรักษาความปลอดภัยที่สมบูรณ์

แม้จะเขียนว่า:

```text
Never make up an answer.
```

โมเดลก็ยังสามารถ Hallucinate ได้

ระบบจริงต้องเสริมด้วย:

* Output validation
* Grounding verification
* Evaluator
* Domain restriction
* Guardrails
* Human review
* Source attribution

---

# 6. Conversation History

Gradio ส่งข้อมูลสองส่วนให้ฟังก์ชันแชต:

```text
message = ข้อความล่าสุด
history = บทสนทนาก่อนหน้า
```

Application นำข้อมูลมาประกอบเป็น:

```text
System Prompt
+
Previous Conversation
+
Latest User Message
```

ตัวอย่าง:

```text
System: คุณเป็น Digital Twin
User: ฉันชื่อสมชาย
Assistant: ยินดีที่ได้รู้จัก
User: ฉันชื่ออะไร
```

โมเดลตอบชื่อได้เพราะข้อความเดิมถูกส่งกลับไปใน Request ใหม่

---

# 7. Illusion of Memory

LLM API โดยพื้นฐานเป็น Stateless

แต่ละ Request ไม่ได้จำ Request ก่อนหน้าโดยอัตโนมัติ

ระบบทำให้โมเดลดูเหมือนจำได้ด้วยวิธี:

```text
Request 1
User Message
→ LLM
→ Response

Request 2
System Prompt
+ User Message เดิม
+ Assistant Response เดิม
+ User Message ใหม่
→ LLM
```

ดังนั้นสิ่งที่ดูเหมือน Memory คือ:

> Application นำ Conversation History กลับมาใส่ Context ในแต่ละ Request

## Misconception Corrected

### ความเข้าใจคลาดเคลื่อน

> โมเดลจำทุกข้อความใน Session ไว้ภายในตัวเอง

### ข้อเท็จจริง

Application เป็นผู้เก็บและส่ง History ส่วนโมเดลเห็นเฉพาะ Context ที่ได้รับใน Request ปัจจุบัน

---

# 8. Gradio Chat Interface

Gradio ทำหน้าที่เป็น User Interface:

```text
Browser
   ↓
Gradio Chat
   ↓
chat(message, history)
   ↓
LLM API
   ↓
Response
   ↓
Browser
```

Gradio ไม่ใช่ Agent Framework

มันช่วยจัดการ:

* กล่องข้อความ
* การแสดงบทสนทนา
* การส่ง `message`
* การส่ง `history`
* การเปิด Web Interface

Agent Logic ยังคงอยู่ใน Python Function ที่ผู้พัฒนาเขียน

---

# 9. Python Function และ Tool Definition

Lab สร้าง Function เช่น:

```text
record_email(email)
```

Function นี้ทำงานจริง เช่น:

```text
รับ Email
→ เปิดไฟล์
→ เขียนข้อมูล
→ ส่งผลลัพธ์กลับ
```

แต่ LLM ยังไม่รู้ว่า Function นี้มีอยู่ จึงต้องสร้าง Tool Definition

---

## 9.1 Python Function

Python Function คือ Code ที่ Execute จริง:

```python
def record_email(email):
    ...
```

---

## 9.2 Tool Definition

Tool Definition คือ Metadata ที่อธิบายให้โมเดลรู้ว่า:

* Tool ชื่ออะไร
* ใช้ทำอะไร
* รับ Arguments อะไร
* Argument แต่ละตัวเป็นชนิดใด
* Field ใดจำเป็น

ตัวอย่างแนวคิด:

```json
{
  "name": "record_email",
  "description": "Record the email address of an interested visitor.",
  "parameters": {
    "type": "object",
    "properties": {
      "email": {
        "type": "string"
      }
    },
    "required": ["email"]
  }
}
```

Mental Model:

```text
Python Function = เครื่องมือจริง
Tool Definition = คู่มือเครื่องมือที่ให้ LLM อ่าน
```

---

# 10. Tool Calling Workflow

จุดสำคัญที่สุดคือ LLM ไม่ได้ Execute Python Function เอง

ลำดับจริงคือ:

```text
1. Application ส่ง Messages และ Tool Definitions
2. LLM วิเคราะห์ว่าควรเรียก Tool หรือไม่
3. LLM ส่ง Tool Call Request
4. Application อ่านชื่อ Tool และ Arguments
5. Application Execute Python Function
6. Application สร้าง Tool Result Message
7. Application ส่ง Result กลับให้ LLM
8. LLM สร้างคำตอบสุดท้ายหรือขอ Tool เพิ่ม
```

ดังนั้นข้อความที่ถูกต้องคือ:

> LLM ตัดสินใจร้องขอการใช้ Tool แต่ Application เป็นผู้ Execute Tool

---

# 11. `tool_calls`

Response จากโมเดลอาจเป็น:

## กรณีตอบข้อความโดยตรง

```text
finish_reason = stop
```

## กรณีต้องการใช้ Tool

```text
finish_reason = tool_calls
```

Tool Call ประกอบด้วยข้อมูลสำคัญ:

```text
Tool Name
Arguments
Tool Call ID
```

ตัวอย่างเชิงแนวคิด:

```json
{
  "id": "call_123",
  "function": {
    "name": "record_email",
    "arguments": "{\"email\":\"user@example.com\"}"
  }
}
```

Application ต้อง Parse Arguments ก่อนใช้

---

# 12. Tool Arguments ต้องได้รับการตรวจสอบ

Arguments มาจาก LLM จึงไม่ควรถูก Trust โดยตรง

ต้องตรวจ:

* JSON ถูกต้องหรือไม่
* Tool มีอยู่จริงหรือไม่
* Required fields ครบหรือไม่
* Data type ถูกต้องหรือไม่
* Email อยู่ในรูปแบบที่สมเหตุสมผลหรือไม่
* Input มีอันตรายหรือไม่
* Application มีสิทธิ์ Execute หรือไม่

ตัวอย่าง Validation:

```text
Tool Requested: record_email
Argument exists: yes
Argument type: string
Email format: valid
Permission: allowed
→ Execute
```

หาก Validation ไม่ผ่าน:

```text
Do not execute
→ Return tool error
→ ให้โมเดลอธิบายหรือขอข้อมูลใหม่
```

---

# 13. `tool_call_id`

`tool_call_id` ใช้เชื่อมผลลัพธ์กลับไปยัง Tool Call ที่เกี่ยวข้อง

ตัวอย่าง:

```text
Tool Call A
ID: call_001
Email: a@example.com

Tool Call B
ID: call_002
Email: b@example.com
```

เมื่อส่งผลลัพธ์กลับ:

```text
Tool Result for call_001
→ บันทึก a@example.com สำเร็จ

Tool Result for call_002
→ บันทึก b@example.com สำเร็จ
```

หากไม่มี `tool_call_id` โมเดลอาจไม่รู้ว่า Result ใดเป็นของ Request ใด

---

# 14. Tool Result Message

หลัง Execute Tool แล้ว Application ต้องเพิ่มข้อความสองส่วนใน Context:

```text
Assistant Message
→ ข้อความที่มี Tool Call

Tool Message
→ ผลลัพธ์จาก Function
```

Conversation Context จะกลายเป็น:

```text
User:
ติดต่อผมที่ user@example.com

Assistant:
ขอเรียก record_email

Tool:
บันทึก Email สำเร็จแล้ว
```

จากนั้นเรียกโมเดลอีกครั้ง เพื่อให้ตอบผู้ใช้ว่า:

```text
ขอบคุณครับ ผมบันทึกข้อมูลติดต่อไว้แล้ว
```

## สิ่งที่เรียนรู้

Tool Result เป็น Observation ใหม่ของ Agent

หากไม่ส่ง Tool Result กลับ โมเดลจะไม่ทราบว่า Tool:

* สำเร็จ
* ล้มเหลว
* ได้ข้อมูลอะไร
* ต้องทำขั้นตอนต่อไปหรือไม่

---

# 15. จาก Tool Calling ไปสู่ Agent Loop

ในเวอร์ชันง่าย ระบบอาจใช้:

```text
if model requests tool:
    execute tool
    call model again
```

แต่รองรับ Tool Call เพียงรอบเดียว

Lab เปลี่ยนเป็น:

```text
while model requests tools:
    execute tools
    append results
    call model again
```

นี่คือจุดเกิด Agent Loop

---

# 16. Agent Loop Mental Model

```text
User Goal
   ↓
LLM Observes Context
   ↓
LLM Decides
   ├── Final Answer
   └── Tool Call
          ↓
Application Executes Tool
          ↓
Tool Result
          ↓
LLM Observes Result
          ↓
LLM Decides Again
```

วงจรนี้ทำซ้ำจนกว่าโมเดลจะ:

* ให้ Final Answer
* ไม่มี Tool Call เพิ่ม
* ถึง Maximum Steps
* Encounter Error
* ถูก Human Interrupt

---

# 17. Agent Loop Pseudocode

```python
def run_agent(message, history):
    messages = build_context(message, history)

    response = call_model(messages, tools)

    steps = 0

    while response_requests_tools(response):
        steps += 1

        if steps > MAX_STEPS:
            return "Agent stopped because the step limit was reached."

        messages.append(response.message)

        for tool_call in response.tool_calls:
            name = tool_call.name
            arguments = parse_and_validate(tool_call.arguments)

            result = execute_tool(name, arguments)

            messages.append(
                create_tool_result(
                    tool_call_id=tool_call.id,
                    result=result
                )
            )

        response = call_model(messages, tools)

    return response.text
```

---

# 18. Agent Framework Abstraction

Agent Framework ช่วยซ่อนงานประกอบ เช่น:

* Agent Loop
* Message construction
* Tool registration
* Argument parsing
* Tool dispatch
* Tool result formatting
* State management
* Retry
* Error handling
* Maximum iterations
* Tracing
* Guardrails
* Human approval

Lab 3 จงใจให้เขียน Loop ด้วยมือ เพื่อให้เห็นว่า Agent Framework ไม่ใช่เวทมนตร์ แต่เป็น Abstraction เหนือ Logic พื้นฐาน

---

# 19. Why Lab 3 Qualifies as an Agent

Lab 1 และ Lab 2 ยังเป็น Workflow ที่กำหนดขั้นตอนล่วงหน้าเป็นหลัก

Lab 3 เริ่มมีคุณสมบัติของ Agent:

```text
Goal-oriented interaction
+ LLM decision
+ Tool selection
+ External action
+ Observation
+ Repeated decision loop
```

โมเดลสามารถเลือกได้ว่า:

* ตอบโดยตรง
* ขอใช้ Tool
* ขอ Tool ต่อหลังเห็น Result
* หยุดเมื่อมีคำตอบสุดท้าย

ดังนั้น Lab 3 เป็น Agent แบบพื้นฐาน แม้ยังไม่มีความสามารถระดับ Production

---

# 20. Misconceptions Corrected

## ความเข้าใจคลาดเคลื่อนที่ 1

> Tool Definition คือ Function ที่ทำงานจริง

**ข้อเท็จจริง:**
Tool Definition เป็น Schema ที่อธิบาย Function ให้โมเดล ส่วน Python Function คือ Code ที่ Execute จริง

---

## ความเข้าใจคลาดเคลื่อนที่ 2

> LLM เรียก Python Function โดยตรง

**ข้อเท็จจริง:**
LLM ส่ง Tool Call Request ส่วน Application อ่าน Request และ Execute Function

---

## ความเข้าใจคลาดเคลื่อนที่ 3

> เมื่อ Tool ทำงานแล้ว Agent จะทราบผลโดยอัตโนมัติ

**ข้อเท็จจริง:**
Application ต้องส่ง Tool Result กลับเข้า Conversation Context

---

## ความเข้าใจคลาดเคลื่อนที่ 4

> Conversation History คือ Long-term Memory

**ข้อเท็จจริง:**
History เป็น Context ที่ถูกส่งซ้ำใน Session ไม่ใช่ Memory ถาวร

---

## ความเข้าใจคลาดเคลื่อนที่ 5

> System Prompt ป้องกัน Hallucination และ Prompt Injection ได้ทั้งหมด

**ข้อเท็จจริง:**
System Prompt เป็น Instruction Layer ไม่ใช่ Security Boundary ที่สมบูรณ์

---

## ความเข้าใจคลาดเคลื่อนที่ 6

> การมี Tool Calling อย่างเดียวเพียงพอสำหรับ Production Agent

**ข้อเท็จจริง:**
Production Agent ต้องมี Validation, permission, error handling, observability, limits และ security controls เพิ่มเติม

---

# 21. Risks Identified

## 21.1 Infinite Loop

โมเดลอาจขอใช้ Tool ซ้ำไม่สิ้นสุด

ควรมี:

```text
MAX_TOOL_ROUNDS
MAX_TOTAL_STEPS
TIMEOUT
```

---

## 21.2 Invalid Arguments

โมเดลอาจสร้าง:

* JSON ผิด
* Field ขาด
* Type ผิด
* Tool name ที่ไม่มีอยู่
* Argument ที่ไม่ปลอดภัย

---

## 21.3 Tool Execution Failure

Function อาจล้มเหลวเพราะ:

* Permission
* Disk full
* Network failure
* Invalid input
* External API error

Tool Result ต้องสะท้อนผลจริง ห้ามบอกว่าสำเร็จเมื่อระบบล้มเหลว

---

## 21.4 Privacy

Email เป็นข้อมูลส่วนบุคคล

ระบบจริงควรมี:

* Consent
* Privacy notice
* Secure storage
* Access control
* Encryption
* Retention policy
* Delete request mechanism
* Audit logs

การเขียนลง `emails.txt` เหมาะสำหรับ Lab แต่ไม่เหมาะกับ Production

---

## 21.5 Prompt Injection

ผู้ใช้อาจพยายามให้ Digital Twin:

* เปิดเผย System Prompt
* เปิดเผยข้อมูลส่วนตัว
* เรียก Tool โดยไม่เหมาะสม
* บันทึกข้อมูลที่เป็นอันตราย
* ออกจากขอบเขตวิชาชีพ

---

## 21.6 Overloaded Context

การใส่ LinkedIn และ Summary ทั้งหมดทุก Request ทำให้:

* ใช้ Token มาก
* ค่าใช้จ่ายเพิ่ม
* Latency สูง
* Context อาจมี Noise
* ข้อมูลสำคัญอาจถูกกลบ

นี่เป็นเหตุผลที่ระบบขั้นสูงอาจใช้ Retrieval หรือ RAG แทนการใส่เอกสารทั้งหมด

---

# 22. Production Improvements

## Tool Registry

แทนการ Hardcode:

```python
tool_registry = {
    "record_email": record_email,
    "search_profile": search_profile,
}
```

---

## Input Validation

ใช้ Schema หรือ Model ตรวจ Arguments:

```text
Required fields
Type constraints
String length
Email format
Allowed values
```

---

## Structured Tool Result

```json
{
  "success": true,
  "message": "Email recorded",
  "record_id": "123"
}
```

หรือ:

```json
{
  "success": false,
  "error": "Invalid email format"
}
```

---

## Execution Limit

```text
Maximum tool rounds
Maximum total calls
Maximum execution time
Maximum token usage
```

---

## Permissions

แบ่ง Tool ตามระดับความเสี่ยง:

```text
Read-only tools
Low-risk write tools
High-risk write tools
Deployment tools
Destructive tools
```

Tool ที่มีความเสี่ยงสูงควรต้องมี Human Approval

---

## Observability

ควรบันทึก:

* User request
* Model
* Prompt version
* Tool requested
* Arguments
* Tool result
* Latency
* Token usage
* Errors
* Final response

---

# 23. Connection to Lab 1

Lab 1 สอน Chaining:

```text
Output A
→ Input B
```

Lab 3 ขยายเป็น:

```text
Model Output: Tool Call
→ Application Action
→ Tool Result
→ Model Input
```

ดังนั้น Tool Calling คือ Chaining ที่มี External Action อยู่ตรงกลาง

---

# 24. Connection to Lab 2

Lab 2 สอน Evaluator Pattern:

```text
Generator
→ Judge
```

สามารถนำมาเสริม Digital Twin:

```text
Digital Twin Response
        ↓
Domain Evaluator
        ↓
ผ่าน → ส่งให้ผู้ใช้
ไม่ผ่าน → ปฏิเสธหรือสร้างใหม่
```

แต่ต้องจำว่า Evaluator LLM ไม่ใช่ Ground Truth จึงควรมี Rule-based Validation ร่วมด้วย

---

# 25. Agent Pattern Learned

## Tool-Using Agent Pattern

```text
Reason
→ Select Tool
→ Execute
→ Observe
→ Continue
```

## ReAct-like Pattern

แม้ Notebook จะไม่ได้แสดง Chain of Thought แต่โครงสร้างการทำงานคล้าย:

```text
Decision
→ Action
→ Observation
→ Next Decision
```

## Human-Facing Agent

Agent สื่อสารกับผู้ใช้ผ่าน UI และสามารถทำ Action ภายนอกบทสนทนา

## Context-Augmented Agent

Agent ได้รับข้อมูลบุคคลผ่าน Prompt Context

---

# 26. Retrieval Cues

เมื่อพบคำต่อไปนี้ ให้นึกถึง Lab 3:

* Digital Twin
* System Prompt
* Conversation History
* Illusion of Memory
* Gradio
* Tool Schema
* Function Calling
* `tool_calls`
* `tool_call_id`
* Tool Result
* Agent Loop
* `while`
* Maximum iterations
* Tool Registry
* Application executes tools

---

# 27. Knowledge Transfer to Enterprise Agents

แนวคิดจาก Digital Twin สามารถนำไปสร้าง Enterprise Agent:

```text
Domain Documents
→ System Context
→ User Request
→ Agent Decision
→ Enterprise Tool
→ Tool Result
→ Final Response
```

ตัวอย่าง:

```text
User asks about a software module
        ↓
Agent reads available context
        ↓
Agent requests search_code
        ↓
Application searches codebase
        ↓
Result returned to Agent
        ↓
Agent explains findings
```

หลักการเดียวกันใช้กับ:

* Source code analysis
* Database inspection
* Build execution
* Git operations
* Test execution
* Document retrieval
* Support automation

---

# 28. Final Lessons

## Lesson 1

Conversation History เป็น Context ที่ Application จัดการ ไม่ใช่ Memory ถาวรของ LLM

## Lesson 2

System Prompt กำหนดบทบาทและกฎ แต่ไม่ใช่ Security Boundary ที่สมบูรณ์

## Lesson 3

Tool Schema เป็นคำอธิบาย ส่วน Python Function เป็นการทำงานจริง

## Lesson 4

LLM ตัดสินใจขอใช้ Tool แต่ Application เป็นผู้ Execute

## Lesson 5

Tool Result ต้องถูกส่งกลับให้โมเดลเพื่อให้ Agent Observe ผลลัพธ์

## Lesson 6

`while` Loop ทำให้ Agent สามารถตัดสินใจและใช้ Tool หลายรอบได้

## Lesson 7

Agent Framework คือ Abstraction ที่จัดการ Loop, Tools, State และ Error Handling

## Lesson 8

Agent Production ต้องมี Validation, Limits, Security, Observability และ Human Approval

---

# 29. Memory Summary

```text
Lab 3 สร้าง Career Digital Twin จาก LinkedIn PDF และ Summary Text

System Prompt กำหนดบทบาท กฎ และข้อมูลพื้นฐาน

Conversation History ทำให้โมเดลดูเหมือนจำได้
แต่ Application เป็นผู้ส่ง History กลับทุก Request

Tool Definition บอกโมเดลว่ามีความสามารถอะไร
Python Function คือ Code ที่ทำงานจริง

LLM ไม่ Execute Tool เอง
Application อ่าน Tool Call, Validate Arguments และ Execute Function

Tool Result ต้องเชื่อมด้วย tool_call_id
และส่งกลับให้โมเดล

การเปลี่ยนจาก if เป็น while ทำให้เกิด Agent Loop

Agent Loop:
Decide
→ Act
→ Observe
→ Decide again
→ Final Answer

Lab 3 เป็น Agent ขั้นพื้นฐาน
แต่ยังต้องเพิ่ม Security, Validation, Limits และ Observability
ก่อนใช้ใน Production
```

---

# 30. Next Episode

หัวข้อถัดไปคือ:

```text
Week 1 — Lab 4
Multiple Tools
Tool Dispatch
Modular Agent Code
More Complete Agent Loop
Career Digital Twin Packaging
```

Lab 4 จะนำ Logic ที่เขียนรวมกันใน Notebook มาแยกเป็นส่วนที่ชัดเจนและรองรับ Agent ที่มีหลาย Tools มากขึ้น
