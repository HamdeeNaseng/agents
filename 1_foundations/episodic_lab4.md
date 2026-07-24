# Episodic Learning Artifact

## Week 1 — Lab 4: Multiple Tools, Tool Dispatch และ Deployment

**หลักสูตร:** AI Engineer Agentic Track
**ผู้สอน:** Ed Donner
**Repository:** `ed-donner/agents`
**ไฟล์เรียน:** `1_foundations/4_lab4.ipynb`
**โปรเจกต์:** `1_foundations/twin`
**หัวข้อหลัก:** Multiple Tools, Tool Registry, External Actions, Modularization และ Deployment
**สถานะ:** เรียนและสรุปแนวคิดพื้นฐานแล้ว

---

# 1. Context

Lab 4 พัฒนา Digital Twin จาก Lab 3 ให้รองรับ Tool มากกว่าหนึ่งตัว และเปลี่ยนจาก Agent ที่ทำงานอยู่ใน Notebook ไปเป็น Application ที่มีโครงสร้างชัดเจนและสามารถ Deploy ได้

พัฒนาการของ Week 1 ถึงจุดนี้เป็นดังนี้:

```text
Lab 1
LLM Calls
→ Chained Workflow
→ Generator–Evaluator

Lab 2
หลายโมเดล
→ Fan-out/Fan-in
→ LLM-as-a-Judge

Lab 3
Conversation Context
→ Tool Calling
→ Agent Loop

Lab 4
Multiple Tools
→ Tool Dispatch
→ External Side Effects
→ Modular Application
→ Deployment
```

Lab 4 แสดงให้เห็นว่า Agent ที่ใช้ Tool จริงไม่ได้มีเพียงปัญหาเรื่องการตัดสินใจ แต่ยังมีปัญหาเรื่องการจับคู่ชื่อ Tool กับ Function, การควบคุมสิทธิ์, การจัดการ Error และการแบ่งโครงสร้าง Code

---

# 2. Learning Objectives

หลังจบ Lab 4 สามารถอธิบายได้ว่า:

1. External side effect คืออะไร
2. Agent ที่มีหลาย Tools แตกต่างจาก Agent ที่มี Tool เดียวอย่างไร
3. Tool Schema และ Tool Registry ทำหน้าที่ต่างกันอย่างไร
4. Tool Dispatch คืออะไร
5. `if/elif`, `globals()` และ Explicit Tool Map แตกต่างกันอย่างไร
6. ทำไม `globals()` จึงไม่เหมาะกับ Production
7. Agent รองรับ Tool Calls หลายรายการอย่างไร
8. เหตุใด Tool Result ต้องมี `tool_call_id`
9. ทำไมควรแยก Agent Code ออกจาก Notebook
10. `context.py`, `tools.py`, `app.py` และ `styles.py` รับผิดชอบอะไร
11. Local Environment Variables แตกต่างจาก Deployment Secrets อย่างไร
12. Deploy สำเร็จไม่ได้หมายความว่า Production Ready

---

# 3. Project Situation

Digital Twin ใน Lab 4 มีสองความสามารถใหม่:

```text
record_user_details
→ บันทึกข้อมูลของผู้สนใจติดต่อ

record_unknown_question
→ บันทึกคำถามเชิงวิชาชีพที่ Agent ตอบไม่ได้
```

ทั้งสอง Tool เชื่อมกับ Pushover เพื่อส่ง Notification ไปยังเจ้าของ Digital Twin

ภาพรวมระบบ:

```text
Website Visitor
       ↓
Gradio Interface
       ↓
Agent Application
       ↓
LLM Decision
       ├── Final Response
       ├── record_user_details
       └── record_unknown_question
                    ↓
              Pushover API
                    ↓
            Mobile Notification
```

Agent จึงไม่ได้เพียงสร้างข้อความ แต่สามารถสร้างผลกระทบในระบบภายนอกได้จริง

---

# 4. External Side Effects

Side effect คือการที่ Function ทำให้สถานะภายนอกเปลี่ยนแปลง ไม่ได้เพียงรับ Input แล้วคืน Output

ตัวอย่าง Side Effects:

```text
ส่ง Push Notification
ส่ง Email
เขียนไฟล์
เพิ่มข้อมูลใน Database
สร้าง Git Commit
เปิด Pull Request
Deploy Application
ลบ Resource
```

ใน Lab นี้:

```python
def push(message):
    requests.post(
        pushover_url,
        data={
            "user": pushover_user,
            "token": pushover_token,
            "message": message
        }
    )
```

เมื่อ Function ทำงาน จะเกิด Notification จริงนอกระบบสนทนา

## Key Insight

ความผิดพลาดของ Chatbot อาจเป็นเพียงข้อความผิด แต่ความผิดพลาดของ Tool-Using Agent อาจทำให้เกิด Action ผิดในโลกจริง

ดังนั้น Tool ที่มี Side Effect ต้องมี:

```text
Validation
Permission
Error handling
Rate limiting
Audit logging
Human approval เมื่อมีความเสี่ยงสูง
```

---

# 5. Pushover Configuration

Credential ถูกเก็บใน `.env`:

```env
PUSHOVER_USER=...
PUSHOVER_TOKEN=...
```

จากนั้นอ่านผ่าน:

```python
pushover_user = os.getenv("PUSHOVER_USER")
pushover_token = os.getenv("PUSHOVER_TOKEN")
```

ลำดับการทำงาน:

```text
.env
  ↓
Environment Variables
  ↓
Python Tool
  ↓
HTTP Request
  ↓
Pushover Server
  ↓
Mobile Device
```

## Misconception Corrected

### ความเข้าใจคลาดเคลื่อน

> ถ้าตัวแปร Credential มีค่า แสดงว่า API ใช้งานได้

### ข้อเท็จจริง

การตรวจว่าตัวแปรมีค่าเป็นเพียง Configuration Check การตรวจจริงเกิดเมื่อส่ง Request และตรวจ HTTP Response

---

# 6. Tool: `record_user_details`

Tool นี้ใช้บันทึกข้อมูลผู้ที่ต้องการติดต่อ:

```python
def record_user_details(
    email,
    name="Name not provided",
    notes="not provided"
):
    push(
        f"Recording interest from {name} "
        f"with email {email} and notes {notes}"
    )
    return "OK"
```

Arguments:

```text
email
จำเป็น

name
ไม่จำเป็น

notes
ไม่จำเป็น
```

ตัวอย่าง Tool Arguments:

```json
{
  "email": "user@example.com",
  "name": "Somchai",
  "notes": "Interested in discussing an AI project"
}
```

Tool นี้เป็นตัวอย่างของ **Lead Capture Tool**

```text
User expresses interest
→ Agent extracts contact details
→ Application executes Tool
→ Owner receives notification
```

---

# 7. Tool: `record_unknown_question`

Tool นี้ใช้บันทึกคำถามที่ Digital Twin ตอบไม่ได้:

```python
def record_unknown_question(question):
    push(
        f"Recording {question} asked "
        "that I couldn't answer"
    )
    return "OK"
```

Tool นี้เป็นตัวอย่างของ **Knowledge Gap Capture**

```text
User asks professional question
        ↓
Context does not contain answer
        ↓
Agent avoids hallucination
        ↓
record_unknown_question
        ↓
Owner receives knowledge gap
```

ข้อมูลนี้สามารถนำไปใช้:

```text
ปรับ Summary
เพิ่มข้อมูลใน Context
สร้าง FAQ
เพิ่มเอกสาร
ออกแบบ Tool ใหม่
วิเคราะห์ความต้องการของผู้ใช้
```

---

# 8. Unknown Question กับ Unrelated Question

สองสถานการณ์นี้ต้องแยกจากกัน

## Unknown Professional Question

```text
คำถามเกี่ยวกับประสบการณ์หรือทักษะ
แต่ Context ไม่มีข้อมูลเพียงพอ
```

ควร:

```text
เรียก record_unknown_question
แล้วแจ้งว่าไม่มีข้อมูลเพียงพอ
```

## Unrelated Question

```text
คำถามไม่เกี่ยวกับบทบาทหรือวิชาชีพของ Digital Twin
```

ควร:

```text
Redirect กลับไปยังหัวข้อที่เกี่ยวข้อง
โดยไม่จำเป็นต้องเรียก Tool
```

## Risk

หาก System Prompt ระบุเพียงว่า:

```text
ถ้าไม่รู้คำตอบ ให้บันทึกคำถาม
```

Agent อาจบันทึกทุกคำถามนอกขอบเขต ทำให้เกิด Notification Spam

System Prompt จึงควรกำหนด Scope ให้ชัดเจน

---

# 9. Tool Schema

Tool Schema อธิบายให้โมเดลรู้ว่า Agent มี Tool อะไรและรับ Arguments แบบใด

ตัวอย่าง:

```json
{
  "name": "record_user_details",
  "description": "Record a user's contact details",
  "parameters": {
    "type": "object",
    "properties": {
      "email": {
        "type": "string"
      },
      "name": {
        "type": "string"
      },
      "notes": {
        "type": "string"
      }
    },
    "required": ["email"],
    "additionalProperties": false
  }
}
```

Tool Schema มีหน้าที่:

```text
บอกชื่อ Tool
บอกวัตถุประสงค์
บอก Arguments
บอก Data Types
บอก Required Fields
จำกัด Field ที่อนุญาต
```

---

# 10. `additionalProperties: false`

ค่าดังกล่าวบอกว่า Arguments ไม่ควรมี Field ที่ไม่ได้ประกาศไว้ใน Schema

ตัวอย่างที่ตรง Schema:

```json
{
  "email": "user@example.com",
  "name": "Somchai"
}
```

ตัวอย่างที่ไม่ตรง Schema:

```json
{
  "email": "user@example.com",
  "name": "Somchai",
  "admin": true
}
```

เพราะ `admin` ไม่ได้อยู่ใน `properties`

## Key Insight

Tool Schema เป็น Contract ระหว่าง Model กับ Application แต่ไม่ควรถูกใช้แทน Runtime Validation

Application ยังต้องตรวจ Arguments ก่อน Execute เสมอ

---

# 11. Function, Tool Schema และ Tool Registry

สามแนวคิดนี้เป็นคนละชั้น

## Python Function

Code ที่ทำงานจริง:

```python
def record_user_details(...):
    ...
```

## Tool Schema

คำอธิบายที่ส่งให้ LLM:

```text
Tool นี้ชื่ออะไร
ทำอะไร
รับค่าอะไร
```

## Tool Registry

Mapping ภายใน Application:

```python
tool_map = {
    "record_user_details": record_user_details,
    "record_unknown_question": record_unknown_question
}
```

Mental Model:

```text
Tool Schema
= เมนูที่ให้ LLM อ่าน

Tool Registry
= รายชื่อเครื่องมือที่ Application อนุญาต

Python Function
= เครื่องมือจริงที่ถูก Execute
```

---

# 12. Tool Dispatch

Tool Dispatch คือกระบวนการแปลงชื่อ Tool ที่โมเดลส่งมาให้เป็น Function ที่ต้อง Execute

```text
"record_user_details"
        ↓
Tool Registry
        ↓
record_user_details(...)
```

ลำดับ:

```text
1. อ่านชื่อ Tool
2. Parse Arguments
3. ตรวจว่า Tool อยู่ใน Allowlist
4. Validate Arguments
5. Execute Function
6. สร้าง Tool Result
```

---

# 13. Manual `if/elif` Dispatch

วิธีแรก:

```python
if tool_name == "record_user_details":
    result = record_user_details(**arguments)

elif tool_name == "record_unknown_question":
    result = record_unknown_question(**arguments)
```

## ข้อดี

```text
Explicit
อ่านง่าย
Debug ง่าย
ควบคุม Tool ที่อนุญาตชัดเจน
```

## ข้อเสีย

```text
Code ยาวเมื่อ Tools เพิ่ม
ต้องแก้ Dispatcher ทุกครั้ง
เกิด Duplicate Logic ได้ง่าย
```

Manual Dispatch ยังเหมาะกับระบบขนาดเล็กหรือ Tool ที่ต้องควบคุมอย่างเข้มงวด

---

# 14. Dictionary Unpacking

```python
tool(**arguments)
```

สมมติว่า:

```python
arguments = {
    "email": "user@example.com",
    "name": "Somchai"
}
```

จะเทียบเท่ากับ:

```python
tool(
    email="user@example.com",
    name="Somchai"
)
```

`**` ใช้แตก Key-Value จาก Dictionary ให้เป็น Keyword Arguments

## Risk

ถ้า Dictionary มี Field ผิดหรือขาด Function อาจเกิด `TypeError`

จึงต้อง Validate ก่อน Unpacking

---

# 15. Dynamic Dispatch ผ่าน `globals()`

Notebook สาธิต:

```python
tool = globals().get(tool_name)
```

`globals()` คืน Dictionary ของชื่อ Global Objects ใน Module

ตัวอย่าง:

```text
{
  "record_user_details": <function>,
  "record_unknown_question": <function>,
  "push": <function>,
  ...
}
```

จากนั้นสามารถเรียก:

```python
tool(**arguments)
```

## ประโยชน์

```text
Code สั้น
ไม่ต้องเขียน if/elif จำนวนมาก
ค้นหา Function จากชื่อ String ได้
```

## ความเสี่ยง

`globals()` อาจทำให้เข้าถึง Function หรือ Object ที่ไม่ได้ตั้งใจให้เป็น Tool

ตัวอย่าง:

```python
def delete_all_records():
    ...
```

ถ้า Function นี้อยู่ใน Global Scope การ Lookup แบบเปิดอาจสร้างความเสี่ยง

---

# 16. Explicit Tool Map

วิธีที่เหมาะสมกว่า:

```python
tool_map = {
    "record_user_details": record_user_details,
    "record_unknown_question": record_unknown_question
}
```

Dispatch:

```python
tool = tool_map.get(tool_name)

if tool is None:
    result = {
        "success": False,
        "error": f"Unknown tool: {tool_name}"
    }
else:
    result = tool(**arguments)
```

## ข้อดี

```text
เป็น Allowlist
เพิ่ม Tool ได้ง่าย
อ่านง่าย
ลด Attack Surface
ไม่เปิด Function อื่นใน Module
```

## Comparison

```text
if/elif
Explicit แต่ขยายยาก

globals()
ขยายง่ายแต่เปิดกว้างเกินไป

tool_map
Explicit และขยายได้ดี
```

---

# 17. Multiple Tool Calls

โมเดลอาจร้องขอ Tool มากกว่าหนึ่งรายการใน Response เดียว

```text
Tool Call 1
record_user_details

Tool Call 2
record_unknown_question
```

Application จึงต้องวน:

```python
for tool_call in tool_calls:
    ...
```

แต่ละ Tool Result ต้องเก็บแยกกัน:

```python
results.append({
    "role": "tool",
    "content": json.dumps(result),
    "tool_call_id": tool_call.id
})
```

---

# 18. `tool_call_id`

`tool_call_id` ใช้จับคู่ Result กับ Tool Call ที่เกี่ยวข้อง

```text
call_001
→ record_user_details
→ Result A

call_002
→ record_unknown_question
→ Result B
```

เมื่อส่งกลับ:

```text
Tool Result A ต้องใช้ call_001
Tool Result B ต้องใช้ call_002
```

หากจับคู่ผิด โมเดลอาจตีความว่า Result มาจาก Action คนละตัว

---

# 19. Tool Result

Tool Result ควรบอกผลจริงของการ Execute

รูปแบบพื้นฐาน:

```python
{
  "role": "tool",
  "content": json.dumps(result),
  "tool_call_id": tool_call.id
}
```

รูปแบบที่เหมาะกับ Production มากกว่า:

```json
{
  "success": true,
  "data": {
    "message": "Notification sent"
  }
}
```

หรือ:

```json
{
  "success": false,
  "error": {
    "type": "HTTPError",
    "message": "Pushover request failed"
  }
}
```

## Misconception Corrected

### ความเข้าใจคลาดเคลื่อน

> Function คืน `"OK"` แปลว่า Action สำเร็จ

### ข้อเท็จจริง

ถ้า Function ไม่ตรวจ HTTP Status อาจคืน `"OK"` แม้ External API ล้มเหลว

Tool Result ต้องอิงจากผล Execute จริง

---

# 20. Agent Loop with Multiple Tools

Agent Loop ใน Lab 4:

```text
User Message
   ↓
Build Context
   ↓
Call Model with Tool Schemas
   ↓
Model Decision
   ├── Final Response
   └── Tool Calls
           ↓
     Dispatch Tools
           ↓
     Collect Results
           ↓
     Append Tool Results
           ↓
     Call Model Again
```

Pseudocode:

```python
response = call_model(messages, tools)

while response_requests_tools(response):
    assistant_message = response.message
    tool_calls = assistant_message.tool_calls

    tool_results = handle_tool_calls(tool_calls)

    messages.append(assistant_message)
    messages.extend(tool_results)

    response = call_model(messages, tools)

return response.text
```

---

# 21. Why `while` Still Matters

แม้ Model จะเรียก Tools หลายตัวในหนึ่งรอบ แต่หลังเห็น Tool Results อาจยังต้องเรียก Tool เพิ่ม

ตัวอย่าง:

```text
Round 1
ค้นหาข้อมูลผู้ใช้

Round 2
ใช้ข้อมูลที่พบเพื่อบันทึก Lead

Round 3
ตอบ Final Response
```

ดังนั้น:

```text
for
รองรับหลาย Tools ในรอบเดียว

while
รองรับหลาย Tool Rounds
```

ทั้งสอง Loop ทำหน้าที่ต่างกัน

---

# 22. Modularization

Notebook เหมาะสำหรับเรียนและทดลอง แต่เมื่อ Code เริ่มมีหลายความรับผิดชอบ ควรแยกเป็น Modules

โครงสร้าง:

```text
twin/
├── app.py
├── context.py
├── tools.py
├── styles.py
├── linkedin.pdf
├── summary.txt
└── requirements.txt
```

---

# 23. `context.py`

หน้าที่:

```text
อ่าน LinkedIn PDF
อ่าน Summary
สร้าง System Prompt
Export Context
```

ควรรู้เรื่อง:

```text
Identity
Knowledge
Rules
Behavior
Domain boundaries
```

ไม่ควรรับผิดชอบ:

```text
Tool execution
Gradio UI
HTTP notifications
Agent loop
```

---

# 24. `tools.py`

หน้าที่:

```text
โหลด Tool Credentials
ประกาศ Python Functions
ประกาศ Tool Schemas
สร้าง Tools List
สร้าง Tool Map
Dispatch Tool Calls
```

ไฟล์นี้เป็น Action Layer ของ Agent

Mental Model:

```text
context.py
Agent รู้อะไร

tools.py
Agent ทำอะไรได้
```

---

# 25. `app.py`

หน้าที่:

```text
สร้าง Model Client
Import Context
Import Tools
ประกอบ Messages
รัน Agent Loop
เปิด Gradio Application
```

Mental Model:

```text
app.py
เป็น Orchestrator ของ Application
```

---

# 26. `styles.py`

หน้าที่:

```text
CSS
JavaScript
UI Examples
Presentation Configuration
```

การแยก Styling ออกจาก Agent Logic ทำให้เปลี่ยน UI ได้โดยไม่กระทบ Agent Loop

---

# 27. Separation of Concerns

การแบ่ง Module สะท้อนหลัก:

```text
context.py
Knowledge and instructions

tools.py
Actions and integrations

app.py
Orchestration

styles.py
Presentation
```

ประโยชน์:

```text
อ่านง่าย
Test ง่าย
แก้ไขแยกส่วนได้
ลด Coupling
Deploy ง่าย
รองรับการเพิ่ม Tools
```

## Misconception Corrected

การแยก Module ไม่ได้ทำให้ Agent ฉลาดขึ้น แต่ทำให้ระบบดูแลและขยายได้ดีขึ้น

---

# 28. Relative Path Risk

ตัวอย่าง:

```python
PdfReader("linkedin.pdf")
open("summary.txt")
```

เป็น Relative Path ที่อิงจาก Current Working Directory

หากรันจาก Directory ผิด อาจเกิด:

```text
FileNotFoundError
```

ควรรันจาก:

```powershell
cd 1_foundations/twin
uv run app.py
```

หรือสร้าง Path จากตำแหน่งของไฟล์ Python:

```python
from pathlib import Path

BASE_DIR = Path(__file__).resolve().parent
linkedin_path = BASE_DIR / "linkedin.pdf"
```

---

# 29. Local Environment กับ Deployment Secrets

Local Development:

```text
.env
```

Deployment:

```text
Hugging Face Settings
→ Secrets
```

Application อ่านทั้งสองแบบผ่าน:

```python
os.getenv("OPENAI_API_KEY")
```

หลักการสำคัญ:

```text
Secret ต้องอยู่นอก Source Code
Secret ไม่ควรถูก Commit
Secret ไม่ควรถูกแสดงใน Logs
```

---

# 30. Deployment

Lab Deploy Gradio Application ไปยัง Hugging Face Spaces

Deployment ทำให้:

```text
Local Application
→ Public Web Application
```

แต่ไม่ได้รับประกัน:

```text
Security
Reliability
Privacy
Scalability
Compliance
Cost control
```

## Misconception Corrected

### ความเข้าใจคลาดเคลื่อน

> Deploy สำเร็จเท่ากับ Production Ready

### ข้อเท็จจริง

Deployment เป็นเพียงขั้นตอนทำให้ระบบเข้าถึงได้ Production Readiness ต้องมี Controls เพิ่มเติม

---

# 31. Risks Identified

## 31.1 External API Failure

Pushover อาจ:

```text
Timeout
คืน HTTP Error
Token ผิด
Quota เต็ม
Network ใช้งานไม่ได้
```

ควรใช้:

```python
response = requests.post(
    url,
    data=payload,
    timeout=10
)
response.raise_for_status()
```

---

## 31.2 Invalid Tool Arguments

โมเดลอาจสร้าง:

```text
JSON ผิด
Required Field ขาด
Type ผิด
Email ไม่ถูกต้อง
ข้อความยาวเกินไป
Unknown Field
```

---

## 31.3 Unknown Tool

โมเดลอาจขอ Tool ที่ไม่ได้ประกาศหรือไม่มีใน Registry

ต้องคืน Controlled Error แทนการ Crash

---

## 31.4 Infinite Tool Loop

โมเดลอาจขอ Tools ซ้ำ

ควรมี:

```text
MAX_TOOL_ROUNDS
MAX_TOTAL_TOOL_CALLS
TIMEOUT
TOKEN BUDGET
```

---

## 31.5 Notification Spam

ผู้ใช้สามารถกระตุ้น Tool ซ้ำจำนวนมาก

ควรมี:

```text
Rate limit
Deduplication
Cooldown
User/session quota
Abuse detection
```

---

## 31.6 Privacy

Email, Name และ Notes เป็นข้อมูลส่วนบุคคล

ควรมี:

```text
Consent
Privacy notice
Purpose limitation
Secure storage
Retention policy
Deletion process
Access control
```

---

## 31.7 Prompt Injection

ผู้ใช้อาจพยายาม:

```text
สั่งให้เรียก Tool โดยไม่เหมาะสม
เปลี่ยน Tool Arguments
เปิดเผย Secrets
บังคับให้ Agent ส่ง Notification
```

Tool Execution ต้องอิง Allowlist และ Runtime Policy ไม่ควรเชื่อข้อความจากโมเดลเพียงอย่างเดียว

---

# 32. Production Improvements

## Explicit Tool Registry

```python
TOOL_MAP = {
    "record_user_details": record_user_details,
    "record_unknown_question": record_unknown_question
}
```

## Structured Validation

```text
Schema validation
Email validation
Length validation
Allowed values
Business rules
```

## Structured Tool Results

```json
{
  "success": false,
  "error": "Invalid email address"
}
```

## Execution Limits

```text
Maximum rounds
Maximum calls
Timeout
Rate limit
Cost limit
```

## Permission Levels

```text
Read-only Tool
Low-risk Write Tool
High-risk Write Tool
Destructive Tool
```

## Human Approval

Tool ที่มีผลกระทบสูงควรถูกหยุดก่อน Execute:

```text
Agent proposes action
→ Human reviews
→ Approve or reject
→ Application executes
```

## Observability

ควรบันทึก:

```text
Conversation ID
Model
Tool name
Arguments
Result
Latency
Error
Token usage
Timestamp
User approval
```

---

# 33. Patterns Learned

## Multiple-Tool Agent Pattern

```text
LLM
├── Tool A
├── Tool B
└── Tool C
```

## Tool Registry Pattern

```text
Tool Name
→ Allowlisted Mapping
→ Function
```

## External Action Pattern

```text
Agent Decision
→ External API
→ Real-world Side Effect
```

## Lead Capture Pattern

```text
User Interest
→ Extract Details
→ Record Lead
→ Notify Owner
```

## Knowledge Gap Capture Pattern

```text
Unknown Domain Question
→ Record Gap
→ Improve Knowledge Base
```

## Modular Agent Application Pattern

```text
Context
Tools
Orchestration
Presentation
```

---

# 34. Connection to Lab 3

Lab 3:

```text
Tool Definition
→ Tool Call
→ Function
→ Tool Result
→ Agent Loop
```

Lab 4:

```text
Multiple Tool Definitions
→ Multiple Tool Calls
→ Tool Registry
→ External Side Effects
→ Modular Application
```

Lab 4 ไม่ได้เปลี่ยนหลัก Agent Loop แต่ทำให้ Agent Loop รองรับ Application ที่มีความสามารถหลายด้าน

---

# 35. Connection to Lab 2

Lab 2 สอนให้ไม่เชื่อ LLM Judge เป็น Ground Truth

หลักเดียวกันใช้กับ Tool Calling:

```text
LLM เลือก Tool
ไม่ได้แปลว่า Tool นั้นควรถูก Execute เสมอ
```

ต้องมี:

```text
Policy validation
Permission validation
Argument validation
Risk classification
Human approval
```

---

# 36. Key Distinction

```text
Model Capability
โมเดลสามารถเสนอ Tool Call

Application Authority
ระบบเป็นผู้ตัดสินว่า Tool Call ได้รับอนุญาตให้ Execute หรือไม่
```

LLM มีสิทธิ์เสนอ Action แต่ไม่ควรมีสิทธิ์ Execute ทุก Action โดยอัตโนมัติ

นี่เป็นหลักสำคัญของ Agent Security

---

# 37. Retrieval Cues

เมื่อพบคำต่อไปนี้ ให้นึกถึง Lab 4:

```text
Multiple Tools
Tool Dispatch
Tool Registry
Tool Map
globals()
if/elif dispatcher
Dictionary unpacking
External side effect
Pushover
Knowledge gap capture
Lead capture
Modularization
Separation of concerns
Deployment secrets
Hugging Face Spaces
Production readiness
```

---

# 38. Final Lessons

## Lesson 1

Agent ที่มีหลาย Tools ต้องมี Tool Dispatch เพื่อจับคู่ชื่อ Tool กับ Function จริง

## Lesson 2

Tool Schema เป็นสิ่งที่โมเดลเห็น ส่วน Tool Registry เป็นสิ่งที่ Application ใช้ควบคุมสิทธิ์

## Lesson 3

Explicit Tool Map เหมาะกว่า `globals()` เพราะเป็น Allowlist ที่ชัดเจน

## Lesson 4

`for` รองรับหลาย Tool Calls ในหนึ่งรอบ ส่วน `while` รองรับหลายรอบของ Agent Loop

## Lesson 5

Tool ที่สร้าง Side Effect ต้องมี Controls มากกว่า Tool ที่เพียงอ่านข้อมูล

## Lesson 6

Tool Result ต้องสะท้อนผลการ Execute จริง ไม่ควรคืน Success โดยไม่มีการตรวจสอบ

## Lesson 7

Modularization แยก Knowledge, Actions, Orchestration และ Presentation ออกจากกัน

## Lesson 8

การ Deploy ทำให้ระบบเข้าถึงได้ แต่ไม่ได้ทำให้ระบบปลอดภัยหรือพร้อมใช้งานระดับ Production โดยอัตโนมัติ

## Lesson 9

LLM ควรมีสิทธิ์เสนอ Action ส่วน Application ต้องถือ Authority ในการอนุญาตและ Execute

---

# 39. Memory Summary

```text
Lab 4 ขยาย Agent Loop ให้รองรับหลาย Tools

Agent มี Tools หลัก:
record_user_details
record_unknown_question

Tool Schema บอกโมเดลว่า Agent ทำอะไรได้

Tool Registry บอก Application
ว่าชื่อ Tool เชื่อมกับ Function ใด

Tool Dispatch:
อ่านชื่อ
→ Parse Arguments
→ ตรวจ Allowlist
→ Validate
→ Execute
→ Return Result

if/elif อ่านง่ายแต่ขยายยาก

globals() สั้นแต่เปิด Function มากเกินไป

Explicit tool_map เป็นวิธีที่สมดุลและปลอดภัยกว่า

for loop จัดการหลาย Tool Calls ในรอบเดียว

while loop จัดการหลาย Agent Rounds

Pushover เป็น External Tool
ที่สร้าง Side Effect จริง

Side-effect Tools ต้องมี:
Validation
Permission
Error handling
Rate limiting
Audit logging
Human approval เมื่อจำเป็น

Agent Code ถูกแยกเป็น:
context.py
tools.py
app.py
styles.py

Deployment Secrets ต้องเก็บนอก Source Code

Deploy สำเร็จไม่ได้หมายความว่า Production Ready
```

---

# 40. Week 1 Progress Summary

```text
Lab 1
เข้าใจ LLM Call และ Chained Workflow

Lab 2
เข้าใจ Multi-Model Evaluation และ LLM-as-a-Judge

Lab 3
เข้าใจ Tool Calling และ Agent Loop

Lab 4
เข้าใจ Multiple Tools, Tool Registry,
External Actions, Modularization และ Deployment
```

แก่นรวมของ Week 1:

```text
Agent
=
LLM Decision
+ Context
+ Tools
+ Application-controlled Execution
+ Observation
+ Repeated Loop
+ Safety Controls
```

---

# 41. Next Learning Episode

หลัง Lab 4 ควรสรุป Week 1 ในภาพรวมก่อนเข้าสู่ Framework ระดับสูง:

```text
LLM Application
→ Workflow
→ Multi-Model Evaluation
→ Tool-Using Agent
→ Multiple-Tool Agent
→ Deployable Agent Application
```

คำถามสำคัญสำหรับบทถัดไปคือ:

> เมื่อ Agent Loop, Tool Dispatch, State และ Error Handling เริ่มซับซ้อน Framework จะช่วยลดภาระส่วนใด และจะแลกมาด้วยข้อจำกัดด้านการควบคุมอะไรบ้าง?
