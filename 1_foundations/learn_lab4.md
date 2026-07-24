# Week 1 — Lab 4: Multiple Tools, Tool Dispatch และ Deployment

ไฟล์เรียน:

```text
1_foundations/4_lab4.ipynb
```

Lab 4 นำ Agent Loop จาก Lab 3 มาพัฒนาเป็นโปรเจกต์ที่ใช้งานได้มากขึ้น โดยเพิ่ม:

```text
Agent Loop เดิม
+ หลาย Tools
+ Tool Dispatch
+ External Action ผ่าน Pushover
+ Modular Python Project
+ Gradio UI
+ Deployment บน Hugging Face Spaces
```

จุดสำคัญที่สุดของ Lab นี้ไม่ใช่การส่ง Notification แต่คือการเรียนว่า **เมื่อ Agent มีหลาย Tools โปรแกรมจะจับคู่ Tool Call จากโมเดลกับ Python Function ที่ถูกต้องได้อย่างไร**

---

## 1. เชื่อมจาก Episodic Artifact Lab 3

Lab 3 สร้างวงจรพื้นฐาน:

```text
LLM ตัดสินใจ
   ↓
ร้องขอ Tool
   ↓
Application เรียก Function
   ↓
ส่ง Tool Result กลับ
   ↓
LLM ตัดสินใจต่อ
```

Lab 4 เพิ่มปัญหาใหม่:

```text
ถ้ามีหลาย Tools
Application จะรู้ได้อย่างไรว่า
ต้องเรียก Function ตัวไหน?
```

ใน Lab นี้ Agent มีสอง Tools:

```text
record_user_details
บันทึกข้อมูลผู้สนใจติดต่อ

record_unknown_question
บันทึกคำถามที่ Digital Twin ตอบไม่ได้
```

ทั้งสอง Tool ส่ง Push Notification ไปยังโทรศัพท์ผ่าน Pushover. ([GitHub][1])

---

# Learning Objectives

เมื่อจบ Lab 4 คุณควรอธิบายได้ว่า:

1. External side effect คืออะไร
2. Pushover API เชื่อมกับ Python อย่างไร
3. Function, Tool Schema และ Tool Registry ต่างกันอย่างไร
4. Agent รองรับ Tool Calls หลายรายการอย่างไร
5. Manual `if/elif` dispatch ทำงานอย่างไร
6. Dynamic dispatch ผ่าน `globals()` ทำงานอย่างไร
7. เหตุใด Explicit Tool Map ปลอดภัยกว่า `globals()`
8. ทำไมควรแยก Notebook เป็น Python Modules
9. `context.py`, `tools.py`, `app.py` และ `styles.py` รับผิดชอบอะไร
10. Environment Variables และ Secrets ต่างกันอย่างไรเมื่อ Deploy
11. ข้อจำกัดของโค้ด Lab ก่อนนำไป Production

---

# 2. ภาพรวมสถาปัตยกรรม

```text
Website Visitor
       ↓
Gradio UI
       ↓
app.py
       ↓
OpenAI Model
       ↓
ตอบโดยตรง หรือสร้าง Tool Call
       ↓
tools.py เลือก Function
       ↓
Pushover API
       ↓
Push Notification บนโทรศัพท์
       ↓
Tool Result ส่งกลับให้โมเดล
       ↓
Final Response
```

เมื่อผู้ใช้สนใจติดต่อ:

```text
User:
My email is user@example.com

LLM:
ขอเรียก record_user_details

Application:
ส่ง Push Notification

LLM:
ขอบคุณครับ ผมบันทึกข้อมูลติดต่อแล้ว
```

เมื่อ Digital Twin ไม่รู้คำตอบ:

```text
User:
คุณเคยใช้เทคโนโลยี X หรือไม่?

LLM:
ไม่มีข้อมูลใน Context
ขอเรียก record_unknown_question

Application:
ส่งคำถามไปยังเจ้าของ Digital Twin

LLM:
ผมไม่มีข้อมูลเพียงพอที่จะตอบคำถามนี้
```

---

# 3. Pushover คือ External Tool

Lab ให้สร้างบัญชีและ Application Token ใน Pushover แล้วเพิ่มสองค่าใน `.env`:

```env
PUSHOVER_USER=...
PUSHOVER_TOKEN=...
```

จากนั้นโหลดค่าเข้าโปรแกรม:

```python
load_dotenv(override=True)

pushover_user = os.getenv("PUSHOVER_USER")
pushover_token = os.getenv("PUSHOVER_TOKEN")
```

Notebook ตรวจเพียงว่าค่ามีอยู่และ Prefix ดูสมเหตุสมผล ไม่ได้ยืนยันว่า Token ใช้งานได้จริง จนกว่าจะส่ง Request ไปยัง API. ([GitHub][1])

## Mental Model

```text
.env
  ↓
Environment Variables
  ↓
Python Function
  ↓
HTTP Request
  ↓
Pushover Server
  ↓
โทรศัพท์
```

Pushover จึงเป็นตัวอย่างของ Tool ที่ทำ **External Action** หรือการเปลี่ยนแปลงนอกโลกของบทสนทนา

---

# 4. External Side Effect คืออะไร

Side effect คือการที่ Function ไม่ได้เพียงคำนวณแล้วคืนค่า แต่ทำให้สิ่งภายนอกเปลี่ยนแปลง เช่น:

```text
ส่ง Notification
เขียนไฟล์
ส่ง Email
แก้ Database
สร้าง Git Commit
Deploy Application
```

ตัวอย่าง Function ใน Lab:

```python
def push(message):
    payload = {
        "user": pushover_user,
        "token": pushover_token,
        "message": message
    }

    requests.post(pushover_url, data=payload)
```

เมื่อ Function นี้ทำงาน จะมี Notification จริงเกิดขึ้นบนอุปกรณ์ของผู้ใช้. ([GitHub][1])

## ทำไมเรื่องนี้สำคัญ

คำตอบข้อความผิดสามารถอ่านแล้วแก้ได้ แต่ Tool Action ผิดอาจสร้างผลกระทบจริง เช่น:

* ส่งข้อความหาคนผิด
* เขียนข้อมูลผิดลง Database
* ลบไฟล์
* Deploy Code ผิด
* ส่ง Email ซ้ำ
* ใช้เงินผ่าน API

ดังนั้น Agent ที่มี Tools ต้องควบคุมเข้มกว่า Chatbot ธรรมดา

---

# 5. Tool ที่หนึ่ง: `record_user_details`

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

Function รับ:

| Argument | ความหมาย         | จำเป็นหรือไม่ |
| -------- | ---------------- | ------------: |
| `email`  | Email ของผู้สนใจ |        จำเป็น |
| `name`   | ชื่อผู้สนใจ      |     ไม่จำเป็น |
| `notes`  | บริบทเพิ่มเติม   |     ไม่จำเป็น |

ชื่อและ Notes มี Default Value ดังนั้นโมเดลสามารถเรียก Tool โดยส่งเพียง Email ได้. ([GitHub][2])

## ตัวอย่าง Tool Call

```json
{
  "email": "user@example.com",
  "name": "Somchai",
  "notes": "Interested in discussing an AI engineering role"
}
```

Python จะเรียก:

```python
record_user_details(
    email="user@example.com",
    name="Somchai",
    notes="Interested in discussing an AI engineering role"
)
```

---

# 6. Tool ที่สอง: `record_unknown_question`

```python
def record_unknown_question(question):
    push(
        f"Recording {question} asked "
        "that I couldn't answer"
    )
    return "OK"
```

Tool นี้มีเป้าหมายที่ต่างออกไป:

```text
record_user_details
จับ Business Lead

record_unknown_question
จับ Knowledge Gap
```

นี่เป็นแนวคิดที่มีคุณค่ามาก เพราะคำถามที่ Agent ตอบไม่ได้สามารถนำไป:

* ปรับปรุง Context
* เพิ่มเอกสาร
* สร้าง FAQ
* ปรับ Prompt
* เพิ่ม Tool
* ตรวจสอบความต้องการของผู้ใช้

Lab กำหนดใน System Prompt ว่า เมื่อไม่รู้คำตอบ ให้เรียก Tool นี้แล้วจึงแจ้งผู้ใช้ว่าไม่รู้ แทนการแต่งคำตอบขึ้นมา. ([GitHub][3])

---

# 7. Tool Schema ของหลาย Tools

## Schema ของ `record_user_details`

```python
record_user_details_json = {
    "name": "record_user_details",
    "description": (
        "Use this tool to record that a user "
        "is interested in being in touch "
        "and provided an email address"
    ),
    "parameters": {
        "type": "object",
        "properties": {
            "email": {
                "type": "string",
                "description": "The email address of this user"
            },
            "name": {
                "type": "string",
                "description": "The user's name, if provided"
            },
            "notes": {
                "type": "string",
                "description": "Additional conversation context"
            }
        },
        "required": ["email"],
        "additionalProperties": False
    }
}
```

## Schema ของ `record_unknown_question`

```python
record_unknown_question_json = {
    "name": "record_unknown_question",
    "description": (
        "Always use this tool to record any "
        "question that couldn't be answered"
    ),
    "parameters": {
        "type": "object",
        "properties": {
            "question": {
                "type": "string",
                "description": (
                    "The question that couldn't be answered"
                )
            }
        },
        "required": ["question"],
        "additionalProperties": False
    }
}
```

จากนั้นรวม Tool Definitions เป็น List:

```python
tools = [
    {
        "type": "function",
        "function": record_user_details_json
    },
    {
        "type": "function",
        "function": record_unknown_question_json
    }
]
```

โมเดลจะเห็น List นี้และเลือกว่าจะใช้ Tool ตัวใด. ([GitHub][2])

---

# 8. `additionalProperties: False`

บรรทัดนี้หมายความว่า Tool Schema ไม่ต้องการ Arguments ที่อยู่นอกเหนือจาก Field ที่กำหนด

สำหรับ `record_unknown_question`:

```json
{
  "question": "..."
}
```

เหมาะสม แต่รูปแบบนี้ไม่ตรง Schema:

```json
{
  "question": "...",
  "priority": "high",
  "send_to": "admin"
}
```

เพราะ `priority` และ `send_to` ไม่ได้ประกาศใน `properties`

## สิ่งที่ต้องเข้าใจ

Schema ช่วยกำหนดรูปแบบที่โมเดลควรสร้าง แต่ Application ยังต้อง Validate Arguments ก่อน Execute เสมอ

---

# 9. ปัญหาใหม่: Tool Dispatch

โมเดลอาจคืนชื่อ Tool:

```text
record_user_details
```

หรือ:

```text
record_unknown_question
```

Application จึงต้องแปลงชื่อที่เป็น String ไปเป็น Python Function จริง:

```text
"record_user_details"
        ↓
record_user_details(...)
```

กระบวนการนี้เรียกว่า **Tool Dispatch**

---

# 10. วิธีแรก: Manual `if/elif`

Notebook เริ่มจากวิธีที่ตรงไปตรงมา:

```python
def handle_tool_calls_with_manual_if(tool_calls):
    results = []

    for tool_call in tool_calls:
        tool_name = tool_call.function.name
        arguments = json.loads(
            tool_call.function.arguments
        )

        if tool_name == "record_user_details":
            result = record_user_details(**arguments)

        elif tool_name == "record_unknown_question":
            result = record_unknown_question(**arguments)

        results.append({
            "role": "tool",
            "content": json.dumps(result),
            "tool_call_id": tool_call.id
        })

    return results
```

Notebook เรียกส่วนนี้ว่า “The big IF statement” เพราะทุกครั้งที่เพิ่ม Tool ใหม่ เราต้องเพิ่มเงื่อนไขใหม่. ([GitHub][1])

## `**arguments` ทำอะไร

สมมติว่า:

```python
arguments = {
    "email": "user@example.com",
    "name": "Somchai"
}
```

คำสั่ง:

```python
record_user_details(**arguments)
```

เทียบเท่ากับ:

```python
record_user_details(
    email="user@example.com",
    name="Somchai"
)
```

เครื่องหมาย `**` เรียกว่า Dictionary Unpacking

---

# 11. ข้อดีและข้อเสียของ Manual Dispatch

## ข้อดี

* อ่านง่าย
* เห็น Control Flow ชัดเจน
* Debug ง่ายสำหรับ Tool จำนวนน้อย
* Explicit ว่า Tool ใดได้รับอนุญาต

## ข้อเสีย

เมื่อมี 20 Tools จะกลายเป็น:

```python
if tool_name == "tool_a":
    ...
elif tool_name == "tool_b":
    ...
elif tool_name == "tool_c":
    ...
```

Code จะยาวและแก้ไขยาก

อย่างไรก็ตาม Manual Dispatch ไม่ใช่สิ่งผิด สำหรับระบบเล็กหรือ Tool ที่มีความเสี่ยงสูง การเขียนเงื่อนไขอย่าง Explicit อาจปลอดภัยกว่าการ Dispatch แบบเปิดกว้าง

---

# 12. วิธีที่สอง: `globals()`

Notebook สาธิตว่า Python มี Dictionary ของชื่อ Global Objects:

```python
globals()
```

จึงสามารถเรียก Function จากชื่อ String ได้:

```python
tool = globals().get(tool_name)

result = (
    tool(**arguments)
    if tool
    else "No tool found"
)
```

ตัวอย่าง:

```python
globals()["record_unknown_question"](
    "This is a hard question"
)
```

Notebook ใช้วิธีนี้เพื่อหลีกเลี่ยง `if/elif` จำนวนมาก แต่ก็ระบุว่าเมื่อ Deploy จริงควรใช้วิธีที่ป้องกันมากกว่านี้. ([GitHub][1])

## Mental Model

```text
globals()
=
{
    "record_user_details": <function>,
    "record_unknown_question": <function>,
    "push": <function>,
    ...
}
```

เมื่อใช้:

```python
globals().get("record_user_details")
```

จะได้ Function Object กลับมา

---

# 13. ความเสี่ยงของ `globals()`

`globals()` อาจเปิดให้เข้าถึง Function หรือ Object อื่นที่ไม่ตั้งใจให้เป็น Tool

สมมติ Module มี:

```python
def delete_all_records():
    ...
```

แม้ไม่ได้ประกาศเป็น Tool Schema แต่ถ้า Tool Name ที่ถูกส่งเข้ามาควบคุมไม่ดี การใช้ `globals()` อาจทำให้ Function ที่ไม่ควรเปิดเผยถูกค้นพบและเรียกใช้ได้

ดังนั้นหลักการสำคัญคือ:

> Tool Schema ไม่ควรเป็น Permission System เพียงชั้นเดียว

Application ต้องมี Allowlist ของ Function ที่อนุญาตจริง

---

# 14. วิธีที่เหมาะกว่า: Explicit Tool Map

ในโค้ด Module เวอร์ชันสมบูรณ์ของ Repository ผู้สอนไม่ได้ใช้ `globals()` แต่สร้าง Mapping ที่ชัดเจน:

```python
tool_map = {
    "record_user_details": record_user_details,
    "record_unknown_question": record_unknown_question
}
```

แล้ว Dispatch:

```python
tool = tool_map.get(tool_name)

result = (
    tool(**arguments)
    if tool
    else "Unknown tool: " + tool_name
)
```

นี่เป็นจุดที่ควรสังเกต: Notebook ใช้ `globals()` เพื่อสาธิต Dynamic Lookup แต่โปรเจกต์ที่จัด Module แล้วเปลี่ยนไปใช้ Explicit `tool_map` ซึ่งจำกัดเฉพาะ Function ที่ได้รับอนุญาต. ([GitHub][2])

## เปรียบเทียบ

| วิธี                | อ่านง่าย | เพิ่ม Tool ง่าย | ความปลอดภัย |
| ------------------- | -------: | --------------: | ----------: |
| `if/elif`           |      สูง |             ต่ำ |         สูง |
| `globals()`         |  ปานกลาง |             สูง |         ต่ำ |
| Explicit `tool_map` |      สูง |             สูง |     สูงกว่า |

ในงานจริง `tool_map` เป็นตัวเลือกที่สมดุลที่สุดของทั้งสามวิธี

---

# 15. รองรับ Tool Calls หลายรายการ

ฟังก์ชันไม่ได้รับ Tool Call เพียงตัวเดียว แต่รับ List:

```python
def handle_tool_calls(tool_calls):
    results = []

    for tool_call in tool_calls:
        ...
        results.append(tool_result)

    return results
```

จึงรองรับสถานการณ์ที่โมเดลร้องขอหลาย Actions ใน Response เดียว เช่น:

```text
Tool Call 1:
บันทึก Email

Tool Call 2:
บันทึกคำถามที่ตอบไม่ได้
```

แต่ละ Result ต้องมี `tool_call_id` ของตัวเองเพื่อจับคู่กลับไปยัง Request ที่ถูกต้อง. ([GitHub][2])

---

# 16. Tool Result Format

แต่ละ Tool Result ถูกสร้างเป็น:

```python
{
    "role": "tool",
    "content": json.dumps(result),
    "tool_call_id": tool_call.id
}
```

องค์ประกอบสำคัญคือ:

| Field          | หน้าที่                  |
| -------------- | ------------------------ |
| `role: tool`   | บอกว่านี่คือผลจาก Tool   |
| `content`      | ผลลัพธ์จาก Function      |
| `tool_call_id` | เชื่อมกับ Tool Call เดิม |

ถึงแม้ Function คืน `"OK"` แต่ระบบยังใช้ `json.dumps()` เพื่อให้ Content อยู่ในรูปแบบที่ส่งเข้า API ได้อย่างสม่ำเสมอ

---

# 17. Agent Loop ฉบับ Lab 4

```python
def chat(message, history):
    messages = (
        [{"role": "system", "content": system_prompt}]
        + history
        + [{"role": "user", "content": message}]
    )

    response = openai.chat.completions.create(
        model="gpt-5.4-mini",
        messages=messages,
        tools=tools
    )

    while (
        response.choices[0].finish_reason
        == "tool_calls"
    ):
        assistant_message = (
            response.choices[0].message
        )

        tool_calls = assistant_message.tool_calls
        results = handle_tool_calls(tool_calls)

        messages.append(assistant_message)
        messages.extend(results)

        response = openai.chat.completions.create(
            model="gpt-5.4-mini",
            messages=messages,
            tools=tools
        )

    return response.choices[0].message.content
```

Agent สามารถเรียก Tool หลายรอบจนกว่า Model จะให้ Final Response. ([GitHub][1])

---

# 18. อ่าน Flow ทีละขั้น

## ขั้นที่ 1: ประกอบ Context

```python
messages = system + history + latest_message
```

## ขั้นที่ 2: ส่ง Tools ให้โมเดล

```python
response = openai.chat.completions.create(
    messages=messages,
    tools=tools
)
```

## ขั้นที่ 3: ตรวจ Finish Reason

```python
while finish_reason == "tool_calls":
```

## ขั้นที่ 4: Dispatch Tools

```python
results = handle_tool_calls(tool_calls)
```

## ขั้นที่ 5: เพิ่ม Request และ Results ลง Context

```python
messages.append(assistant_message)
messages.extend(results)
```

## ขั้นที่ 6: ให้โมเดล Observe ผล

```python
response = call_model_again()
```

## ขั้นที่ 7: จบเมื่อได้ข้อความธรรมดา

```python
return response.choices[0].message.content
```

นี่คือ Agent Loop เดิมจาก Lab 3 แต่ถูกขยายให้รองรับหลาย Tools อย่างเป็นระบบ

---

# 19. การแยก Notebook เป็น Modules

Notebook เหมาะกับ:

* ทดลองแนวคิด
* รันทีละ Cell
* ดูค่าระหว่างทาง
* เรียนรู้และ Debug

แต่เมื่อ Logic เริ่มนิ่ง ควรแยกเป็น Python Modules

Repository แบ่งโครงสร้างเป็น:

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

ผู้สอนอธิบายว่า `context.py` จัดการข้อมูลและ System Prompt, `tools.py` จัดการ Tool Logic, `app.py` จัดการ Gradio และ Model Call ส่วน `styles.py` จัดการหน้าตา UI. ([GitHub][1])

---

# 20. `context.py`

หน้าที่:

```text
อ่าน linkedin.pdf
อ่าน summary.txt
สร้าง TWIN_SYSTEM_PROMPT
```

ไฟล์นี้ไม่ควรรับผิดชอบ:

* เรียก LLM
* ส่ง Notification
* เปิด Gradio
* Dispatch Tool

Repository สร้างตัวแปร:

```python
TWIN_SYSTEM_PROMPT = f"""
...
{summary}
...
{linkedin}
...
"""
```

แล้ว Export ให้ `app.py` Import ไปใช้. ([GitHub][3])

## จุดที่ต้องระวัง

โค้ดใช้ Path แบบ:

```python
PdfReader("linkedin.pdf")
open("summary.txt")
```

ดังนั้น Current Working Directory ต้องอยู่ใน `twin` ตอนรัน:

```powershell
cd 1_foundations
cd twin
uv run app.py
```

ถ้ารันจาก Root Repository โดยตรง อาจหาไฟล์ไม่พบ เพราะ Path เป็น Relative Path. ([GitHub][3])

---

# 21. `tools.py`

หน้าที่:

```text
โหลด Pushover Credentials
ส่ง Push Notification
ประกาศ Python Functions
ประกาศ Tool Schemas
ประกอบ tools List
สร้าง tool_map
Dispatch Tool Calls
```

จุดสำคัญคือ Module นี้รวมทุกสิ่งที่เกี่ยวกับ Action ไว้ด้วยกัน และ Final Version ใช้ Explicit `tool_map` แทน `globals()`. ([GitHub][2])

---

# 22. `app.py`

หน้าที่:

```text
โหลด Environment
สร้าง OpenAI Client
Import System Prompt
Import Tools
ประกอบ Agent Loop
เปิด Gradio UI
```

Code หลัก Import:

```python
from context import TWIN_SYSTEM_PROMPT
from tools import tools, handle_tool_calls
from styles import CSS, JS, EXAMPLES
```

จากนั้นสร้าง `chat()` และเปิด `gr.ChatInterface` เมื่อไฟล์ถูกเรียกโดยตรง. ([GitHub][4])

## `if __name__ == "__main__":`

```python
if __name__ == "__main__":
    gr.ChatInterface(...).launch(...)
```

หมายความว่า Gradio จะเปิดเมื่อรัน:

```powershell
uv run app.py
```

แต่จะไม่เปิดโดยอัตโนมัติเมื่อ Module ถูก Import จากไฟล์อื่น

---

# 23. `styles.py`

หน้าที่คือเก็บ:

* CSS
* JavaScript
* ตัวอย่างคำถามใน UI

การแยก Styling ออกจาก Agent Logic ทำให้ `app.py` อ่านง่าย และการเปลี่ยนหน้าตาไม่กระทบ Tool หรือ Context Logic. ([GitHub][4])

---

# 24. Separation of Concerns

การแยก Module สะท้อนหลัก **Separation of Concerns**

```text
context.py
รู้อะไร

tools.py
ทำอะไรได้

app.py
ควบคุมการสนทนาอย่างไร

styles.py
แสดงผลอย่างไร
```

Metaphor:

```text
context.py = แฟ้มข้อมูลพนักงาน
tools.py   = แผนกปฏิบัติการ
app.py     = ผู้จัดการงาน
styles.py  = ฝ่ายออกแบบหน้าร้าน
```

หากรวมทุกอย่างไว้ไฟล์เดียว การแก้ส่วนหนึ่งอาจกระทบอีกส่วนและทำให้ Test ยากขึ้น

---

# 25. Deployment บน Hugging Face Spaces

Lab ให้ Deploy Gradio Application ด้วย:

```powershell
cd 1_foundations
cd twin
uv run gradio deploy
```

ขั้นตอนแยก Secret ออกจาก Source Code โดยตั้ง `OPENAI_API_KEY`, `PUSHOVER_USER` และ `PUSHOVER_TOKEN` ใน Settings ของ Space แทนการ Upload `.env`. ([GitHub][1])

## Local `.env` กับ Deployment Secret

```text
เครื่อง Local
.env

Hugging Face Space
Settings → Secrets
```

ทั้งสองมีเป้าหมายเดียวกันคือให้ Application อ่านผ่าน:

```python
os.getenv(...)
```

แต่ไม่ควร Commit Secret ลง Git

---

# 26. Misconceptions ที่ต้องแก้

## “เมื่อมี Tool Schema แล้ว Tool ปลอดภัย”

ไม่จริง

Tool Schema ช่วยบอกโมเดลว่าจะสร้าง Arguments แบบใด แต่ Application ยังต้อง:

* Allowlist Tool
* Validate Arguments
* ตรวจ Permission
* ตรวจ Input
* จำกัดจำนวนครั้ง
* บันทึก Audit Log

---

## “`globals()` เป็นวิธีที่ดีที่สุดเพราะสั้น”

ไม่จริง

`globals()` สะดวกสำหรับการสาธิต แต่เปิดพื้นที่เข้าถึงมากเกินไป Final Module จึงเปลี่ยนเป็น Explicit `tool_map`

---

## “Tool คืน `OK` หมายความว่าทำงานสำเร็จ”

ไม่จำเป็น

Function ใน Lab ไม่ตรวจ HTTP Status ของ Pushover ดังนั้น `requests.post()` อาจล้มเหลว แต่ Function ยังคืน `"OK"` ได้

---

## “การแยก Module ทำให้ Agent ฉลาดขึ้น”

ไม่จริง

โมเดลและพฤติกรรมอาจเหมือนเดิม แต่ Code:

* อ่านง่ายขึ้น
* Test ง่ายขึ้น
* เปลี่ยนส่วนต่าง ๆ ได้ง่ายขึ้น
* Deploy ง่ายขึ้น
* ลด Coupling

---

## “Deploy แล้วเป็น Production Ready”

ไม่จริง

Deployment หมายถึง Application เข้าถึงผ่าน Internet ได้ แต่ไม่ได้รับประกัน Security, Reliability หรือ Privacy

---

# 27. ช่องโหว่ของ Code Lab

## 27.1 ไม่มี HTTP Error Handling

```python
requests.post(...)
```

ควรเพิ่ม:

```python
response = requests.post(
    pushover_url,
    data=payload,
    timeout=10
)
response.raise_for_status()
```

## 27.2 ไม่มี Argument Validation จริง

`json.loads()` ตรวจเพียงว่า JSON Parse ได้ แต่ไม่ได้ตรวจ Email Format หรือความยาวของข้อความ

## 27.3 ไม่มี Maximum Tool Rounds

`while` อาจวนซ้ำไม่สิ้นสุด

```python
MAX_TOOL_ROUNDS = 5
```

## 27.4 ไม่มี Try/Except

Tool Failure อาจทำให้ Chat ทั้งหมด Error

## 27.5 ไม่มี Confirmation

โมเดลสามารถส่ง Notification ได้ทันทีโดยไม่มี User หรือ Human Approval

## 27.6 อาจเกิด Notification Spam

ผู้ใช้อาจส่งคำถามที่ตอบไม่ได้จำนวนมาก ทำให้โทรศัพท์ได้รับข้อความต่อเนื่อง

## 27.7 ข้อมูลส่วนบุคคล

Email และ Conversation Notes ถูกส่งผ่าน External Service จึงควรมี Privacy Notice และ Consent

---

# 28. Tool Handler ที่แข็งแรงขึ้น

ตัวอย่างแนวคิด:

```python
import json
from typing import Any, Callable

ToolFunction = Callable[..., Any]

TOOL_MAP: dict[str, ToolFunction] = {
    "record_user_details": record_user_details,
    "record_unknown_question": record_unknown_question,
}


def execute_tool_call(tool_call) -> dict[str, str]:
    tool_name = tool_call.function.name

    tool = TOOL_MAP.get(tool_name)
    if tool is None:
        result = {
            "success": False,
            "error": f"Unknown tool: {tool_name}"
        }
    else:
        try:
            arguments = json.loads(
                tool_call.function.arguments
            )

            result = {
                "success": True,
                "data": tool(**arguments)
            }

        except json.JSONDecodeError:
            result = {
                "success": False,
                "error": "Invalid JSON arguments"
            }

        except TypeError as exc:
            result = {
                "success": False,
                "error": f"Invalid arguments: {exc}"
            }

        except Exception as exc:
            result = {
                "success": False,
                "error": f"Tool execution failed: {exc}"
            }

    return {
        "role": "tool",
        "content": json.dumps(result),
        "tool_call_id": tool_call.id
    }
```

หลักสำคัญคือ Tool Result ต้องบอกความจริงว่า Action สำเร็จหรือไม่

---

# 29. แบบทดสอบ Agent

ควรทดสอบอย่างน้อยสี่สถานการณ์

## กรณี 1: คำถามที่ตอบได้

```text
What experience do you have with AI?
```

ผลที่คาดหวัง:

```text
ตอบจาก Context
ไม่เรียก Tool
```

## กรณี 2: ต้องการติดต่อ

```text
I would like to discuss a role.
My email is test@example.com.
```

ผลที่คาดหวัง:

```text
เรียก record_user_details
ส่ง Push
ตอบยืนยัน
```

## กรณี 3: คำถามที่ไม่มีข้อมูล

```text
What was your exact salary in 2021?
```

ผลที่คาดหวัง:

```text
เรียก record_unknown_question
ไม่แต่งคำตอบ
แจ้งว่าไม่มีข้อมูล
```

## กรณี 4: นอกขอบเขต

```text
Who will win the next football match?
```

ผลที่คาดหวัง:

```text
ไม่จำเป็นต้องบันทึกเป็น Unknown เสมอไป
ชวนกลับมาคุยเรื่องอาชีพและประสบการณ์
```

จุดที่ต้องสังเกตคือ “ไม่เกี่ยวข้อง” กับ “เกี่ยวข้องแต่ไม่รู้คำตอบ” เป็นคนละกรณี

---

# 30. ประเด็นเชิงออกแบบที่สำคัญ

System Prompt ระบุว่า:

```text
If you don't know the answer,
use your tool to record the question.
```

แต่ต้องแยก:

```text
Unknown Professional Question
ควรบันทึก

Unrelated Question
ควร Redirect กลับ
```

ถ้า Prompt ไม่ชัด โมเดลอาจเรียก `record_unknown_question` กับทุกคำถามนอกขอบเขต ทำให้ Notification มากเกินไป

Prompt ที่แม่นขึ้นควรสื่อว่า:

```text
เรียก record_unknown_question เฉพาะคำถาม
เกี่ยวกับอาชีพ ประสบการณ์ ทักษะ หรือภูมิหลัง
ที่ข้อมูลปัจจุบันไม่เพียงพอตอบ

คำถามที่ไม่เกี่ยวกับงานให้ Redirect
โดยไม่ต้องเรียก Tool
```

---

# 31. Pattern ที่เรียนจาก Lab 4

## Multiple-Tool Agent

```text
LLM
├── Tool A
└── Tool B
```

## Tool Registry Pattern

```text
Tool Name
   ↓
Explicit Mapping
   ↓
Python Function
```

## External Action Pattern

```text
Agent Decision
→ API Call
→ Real-world Side Effect
```

## Knowledge Gap Capture

```text
Agent ตอบไม่ได้
→ บันทึกคำถาม
→ ปรับปรุง Knowledge
```

## Lead Capture

```text
ผู้ใช้สนใจ
→ เก็บข้อมูลติดต่อ
→ แจ้งเจ้าของ
```

## Modular Agent Application

```text
Context
Tools
Orchestration
Presentation
```

---

# 32. Checklist ก่อนจบ Lab 4

คุณควรตอบได้ว่า:

### Tool Dispatch คืออะไร

กระบวนการแปลงชื่อ Tool จาก Model Response ไปเป็น Python Function ที่ต้อง Execute

### ทำไม Explicit `tool_map` ดีกว่า `globals()`

เพราะจำกัด Function ที่อนุญาตและลดโอกาสเรียก Object อื่นโดยไม่ตั้งใจ

### Tool Schema กับ Tool Map ต่างกันอย่างไร

```text
Tool Schema
บอกโมเดลว่ามี Tool อะไร

Tool Map
บอก Application ว่าชื่อ Tool เชื่อมกับ Function ใด
```

### ทำไมต้องวน `for tool_call in tool_calls`

เพราะโมเดลอาจร้องขอหลาย Tools ใน Response เดียว

### ทำไมต้องมี `while`

เพราะหลังเห็น Tool Result โมเดลอาจร้องขอ Action ต่ออีกหนึ่งรอบ

### Pushover เป็นส่วนใดของ Agent

เป็น External Tool ที่สร้าง Side Effect จริง

### Module แต่ละไฟล์ทำอะไร

```text
context.py = Context และ System Prompt
tools.py   = Tool และ Dispatch
app.py     = Agent Loop และ UI
styles.py  = Presentation
```

---

# 33. ภารกิจ Lab 4

ดำเนินตามลำดับ:

```text
1. เปิด 1_foundations/4_lab4.ipynb
2. ตั้ง PUSHOVER_USER และ PUSHOVER_TOKEN
3. ทดสอบ push("HEY!!")
4. ทดสอบ record_user_details โดยตรง
5. ทดสอบ record_unknown_question โดยตรง
6. ตรวจ Tool Schemas ทั้งสอง
7. รัน Manual if/elif dispatcher
8. ทดลอง globals() เพื่อเข้าใจแนวคิด
9. เปรียบเทียบกับ tool_map ใน twin/tools.py
10. เปิด Digital Twin ผ่าน Notebook
11. ทดสอบ Tools ทั้งสอง
12. รัน Module Version จาก twin/app.py
13. ตรวจว่า Relative Paths ทำงาน
14. Deploy หลังจาก Local Test ผ่าน
```

แก่นของ Lab 4 ที่ต้องจำคือ:

> Tool Schema บอกโมเดลว่า Agent ทำอะไรได้ ส่วน Tool Registry บอก Application ว่า Tool Name ต้อง Execute Function ใด และ Agent ที่ทำ Action ในโลกจริงต้องมี Validation, Permission และ Error Handling มากกว่า Agent ที่เพียงสร้างข้อความ.

[1]: https://github.com/ed-donner/agents/raw/refs/heads/main/1_foundations/4_lab4.ipynb "raw.githubusercontent.com"
[2]: https://github.com/ed-donner/agents/blob/main/1_foundations/twin/tools.py "agents/1_foundations/twin/tools.py at main · ed-donner/agents · GitHub"
[3]: https://github.com/ed-donner/agents/blob/main/1_foundations/twin/context.py "agents/1_foundations/twin/context.py at main · ed-donner/agents · GitHub"
[4]: https://github.com/ed-donner/agents/blob/main/1_foundations/twin/app.py "agents/1_foundations/twin/app.py at main · ed-donner/agents · GitHub"
