# Week 1 — Lab 3: Digital Twin, Tool Calling และ Agent Loop

ไฟล์เรียน:

```text
1_foundations/3_lab3.ipynb
```

Lab 3 เป็นจุดเปลี่ยนสำคัญจาก Lab 1–2:

```text
Lab 1: เรียก LLM และต่อหลาย Calls
Lab 2: หลายโมเดลสร้างคำตอบและมี Judge
Lab 3: LLM สนทนา ใช้ Context และตัดสินใจเรียก Tool
```

Notebook ระบุชัดว่าเราจะสร้าง **Agent Loop ด้วยมือโดยไม่ใช้ Agent Framework** และเริ่มต้นโปรเจกต์ Digital Twin ซึ่งจะเรียนต่อเนื่องสองวัน. ([GitHub][1])

---

## Learning Objectives

เมื่อจบ Lab 3 คุณควรเข้าใจว่า:

1. Digital Twin ใช้ข้อมูลบุคคลเป็น Context อย่างไร
2. System Prompt ต่างจาก Conversation History อย่างไร
3. เหตุใด LLM จึงดูเหมือนมี Memory ทั้งที่แต่ละ Request เป็น Stateless
4. Gradio ส่ง `message` และ `history` เข้าฟังก์ชันอย่างไร
5. Function กับ Tool Definition เป็นคนละสิ่งกันอย่างไร
6. โมเดลไม่ได้ Execute Tool เอง
7. `tool_calls`, `tool_call_id` และ Tool Result เชื่อมกันอย่างไร
8. เหตุใดการเปลี่ยน `if` เป็น `while` จึงทำให้เกิด Agent Loop
9. Agent Framework ซ่อนขั้นตอนใดไว้ให้เรา

---

# 1. โปรเจกต์ที่กำลังสร้าง

เป้าหมายของ Lab คือสร้างเว็บไซต์แชตที่ทำหน้าที่เป็นตัวแทนบุคคล:

```text
ผู้เยี่ยมชมเว็บไซต์
        ↓
ถามเรื่องประวัติ ทักษะ และประสบการณ์
        ↓
Digital Twin ตอบจาก LinkedIn และ Summary
        ↓
ผู้เยี่ยมชมให้ Email
        ↓
Digital Twin ขอเรียก Tool
        ↓
โปรแกรมบันทึก Email ลงไฟล์
```

ข้อมูลหลักมาจากสองไฟล์ในโฟลเดอร์ `twin`:

```text
twin/
├── linkedin.pdf
└── summary.txt
```

ผู้สอนแนะนำให้แทนที่ `linkedin.pdf` ด้วย LinkedIn PDF หรือ Resume ของผู้เรียน และแก้ `summary.txt` ให้เป็นข้อมูลของตนเอง. ([GitHub][1])

---

# 2. อ่านข้อมูลจาก PDF

Notebook ใช้ `PyPDF` เพื่ออ่าน PDF:

```text
PDF
 ↓
PdfReader
 ↓
วนอ่านทุกหน้า
 ↓
extract_text()
 ↓
รวมเป็นข้อความ linkedin
```

แนวคิดสำคัญคือ LLM ไม่ได้เปิดไฟล์ PDF เอง โปรแกรม Python เป็นผู้:

1. เปิดไฟล์
2. Extract ข้อความ
3. รวมข้อความ
4. ใส่ข้อความเข้า Prompt

ดังนั้นลำดับจริงคือ:

```text
linkedin.pdf
    ↓ Python
ข้อความธรรมดา
    ↓ System Prompt
LLM
```

## จุดที่อาจเกิดปัญหา

PDF บางไฟล์เป็นภาพ Scan ซึ่งไม่มี Text Layer ทำให้ `extract_text()` ได้ข้อความว่างหรืออ่านผิด นอกจากนี้ Layout แบบหลายคอลัมน์อาจทำให้ลำดับข้อความสับสนได้

ลองตรวจด้วย:

```python
print(linkedin[:2000])
```

อย่าเพิ่งเชื่อว่าอ่านสำเร็จเพียงเพราะ Code ไม่ Error ต้องอ่านผลที่ Extract ออกมาด้วย

---

# 3. อ่าน `summary.txt`

โปรแกรมเปิดไฟล์ด้วย UTF-8:

```python
with open("twin/summary.txt", encoding="utf-8") as file:
    summary = file.read()
```

`summary` เป็นข้อมูลที่ผู้เรียนเขียนเอง เช่น:

* บทบาทปัจจุบัน
* ความเชี่ยวชาญ
* เป้าหมาย
* ประสบการณ์สำคัญ
* วิธีที่ต้องการให้ Digital Twin แนะนำตัว

## ความแตกต่างระหว่างสองแหล่ง

```text
linkedin.pdf
ข้อมูลค่อนข้างละเอียดและมาจากเอกสาร

summary.txt
ข้อมูลย่อที่ควบคุม Narrative ได้มากกว่า
```

ทั้งสองยังไม่ใช่ RAG เพราะระบบโหลดเอกสารทั้งหมดเข้า Prompt ทุกครั้ง ไม่ได้ Search หรือ Retrieve เฉพาะส่วนที่เกี่ยวข้อง

---

# 4. สามแนวคิดสำคัญ

Notebook หยุดอธิบายสามเรื่องโดยเฉพาะ ได้แก่ System Prompt, Conversation History และภาพลวงตาของ Memory. ([GitHub][1])

## 4.1 System Prompt

System Prompt กำหนด:

* Agent คือใคร
* ต้องทำหน้าที่อะไร
* มีข้อมูลอะไร
* ตอบเรื่องใดได้
* เรื่องใดไม่ควรตอบ
* ต้องปฏิบัติตามกฎอะไร

ภาพง่าย ๆ:

```text
System Prompt = Job Description + Operating Policy + Context
```

ตัวอย่างโครงสร้าง:

```text
Role
- คุณเป็น Digital Twin

Context
- ข้อมูลประวัติ
- ประสบการณ์
- ความสามารถ

Rules
- ตอบเฉพาะเรื่องวิชาชีพ
- ไม่รู้ให้บอกว่าไม่รู้
- ห้ามแต่งข้อมูล
```

## จุดที่ต้องเข้าใจ

System Prompt ไม่ใช่ Security Boundary ที่แข็งแรง ผู้ใช้อาจพยายาม Prompt Injection หรือทำให้โมเดลออกนอกบทบาทได้ จึงต้องใช้ Validation และ Application Logic เสริมในระบบจริง

---

## 4.2 Conversation History

สมมติบทสนทนาเป็น:

```text
User: ผมชื่อสมชาย
Assistant: ยินดีที่ได้รู้จัก
User: ผมชื่ออะไร
```

โมเดลตอบได้ก็ต่อเมื่อโปรแกรมส่งข้อความก่อนหน้ากลับไปด้วย:

```text
Request ใหม่
├── System Prompt
├── User: ผมชื่อสมชาย
├── Assistant: ยินดีที่ได้รู้จัก
└── User: ผมชื่ออะไร
```

ถ้าส่งเพียงข้อความสุดท้าย:

```text
User: ผมชื่ออะไร
```

โมเดลก็ไม่มีหลักฐานว่าชื่ออะไร

---

## 4.3 Illusion of Memory

จาก Episodic Artifact ของ Lab 1 เราจำไว้ว่า:

> `messages` เป็น Context ของ Request ไม่ใช่ Memory ถาวรของโมเดล

Lab 3 แสดงให้เห็นด้วยการทดลองจริงว่า เมื่อตัดข้อความเดิมออก โมเดลจำชื่อไม่ได้ แต่เมื่อนำ Conversation History กลับมา โมเดลจึงตอบได้

ภาพที่ถูกต้อง:

```text
Request 1
Program → LLM → Response

Request 2
Program ส่ง History ทั้งหมด → LLM → Response
```

ไม่ใช่:

```text
Request 1 → LLM จำเองถาวร → Request 2
```

---

# 5. สร้าง System Prompt ของ Digital Twin

System Prompt ใน Notebook รวมข้อมูลจาก:

```text
summary
+
linkedin
+
role
+
rules
```

แล้วระบุให้ Digital Twin:

* ตอบเกี่ยวกับอาชีพและประสบการณ์
* พูดคุยอย่างมืออาชีพ
* อธิบายได้ว่าตนเป็น AI Digital Twin
* ดึงบทสนทนากลับสู่เรื่องวิชาชีพ
* ไม่แต่งคำตอบเมื่อไม่มีข้อมูล

Notebook ใช้โมเดลขนาดเล็กสำหรับการทดลองพื้นฐาน และใช้โมเดลระดับ `mini` สำหรับ Digital Twin และ Tool Calling. ([GitHub][1])

## จุดสำคัญ

การใส่คำว่า:

```text
Never make up an answer
```

ช่วยกำหนดพฤติกรรม แต่ไม่ได้รับประกันว่าโมเดลจะไม่ Hallucinate

ระบบจริงควรเพิ่ม:

* แสดงแหล่งข้อมูล
* ตรวจว่าคำตอบอ้างอิง Context
* ใช้ Evaluator จาก Pattern ของ Lab 1
* จำกัด Domain
* มีข้อความปฏิเสธมาตรฐาน

---

# 6. ฟังก์ชัน `chat(message, history)`

Gradio จะส่งข้อมูลสองส่วนเข้า Function:

```text
message = ข้อความล่าสุดของผู้ใช้
history = บทสนทนาก่อนหน้า
```

ฟังก์ชันสร้าง Messages ตามลำดับ:

```text
System Prompt
+
Conversation History
+
ข้อความล่าสุดของ User
```

จากนั้นเรียกโมเดลและคืนข้อความตอบกลับให้ Gradio

Mental Model:

```text
Browser UI
   ↓ message + history
chat()
   ↓ assemble messages
OpenAI API
   ↓ response text
Gradio แสดงผล
```

Notebook ใช้ `gr.ChatInterface(...).launch(inbrowser=True)` เพื่อเปิด UI แชตใน Browser. ([GitHub][1])

## Gradio ไม่ใช่ Agent Framework

Gradio ทำหน้าที่สร้าง User Interface เป็นหลัก ส่วน Agent Logic ยังอยู่ใน Function `chat()` ที่เราเขียนเอง

---

# 7. จาก Function ธรรมดาเป็น Tool

Notebook สร้าง Python Function สำหรับบันทึก Email:

```text
รับ email
  ↓
เปิด emails.txt แบบ append
  ↓
เขียน email ลงไฟล์
  ↓
คืนข้อความยืนยัน
```

Python Function นี้ทำงานได้ทันทีเมื่อเราเรียกจาก Python

แต่ LLM ยังไม่รู้ว่า Function นี้มีอยู่

เราจึงต้องสร้าง **Tool Definition** เพื่ออธิบายให้โมเดลรู้ว่า:

* Tool ชื่ออะไร
* ใช้ทำอะไร
* รับ Argument อะไร
* Argument ประเภทใด
* Field ใดจำเป็น

---

# 8. Function กับ Tool Schema ต่างกันอย่างไร

## Python Function

เป็น Code ที่ทำงานจริง:

```text
record_email(...)
```

## Tool Schema

เป็นคำอธิบายความสามารถให้โมเดลอ่าน:

```json
{
  "name": "record_email",
  "description": "...",
  "parameters": {
    "type": "object",
    "properties": {
      "email": {"type": "string"}
    },
    "required": ["email"]
  }
}
```

Metaphor:

```text
Python Function = เครื่องจักรจริง
Tool Schema = คู่มือและป้ายชื่อเครื่องจักร
```

LLM เห็นคู่มือ แต่ไม่ได้เข้าไปกดเครื่องจักรเอง

---

# 9. Tool Calling ไม่ได้แปลว่า LLM รัน Function

นี่เป็นจุดสำคัญที่สุดของ Lab 3

ลำดับจริงของ Function Calling คือ:

```text
1. โปรแกรมส่ง Messages และ Tool Definitions ให้โมเดล
2. โมเดลตอบว่าต้องการใช้ Tool
3. โปรแกรมอ่านชื่อ Tool และ Arguments
4. โปรแกรมเรียก Python Function จริง
5. โปรแกรมส่ง Tool Result กลับให้โมเดล
6. โมเดลสร้างคำตอบสุดท้าย
```

เอกสาร OpenAI อธิบาย Tool Calling เป็นบทสนทนาหลายขั้นในรูปแบบเดียวกัน: ส่งรายการ Tools, รับ Tool Call, ให้ Application Execute Code, ส่งผล Tool กลับ แล้วรับคำตอบหรือ Tool Call เพิ่มเติม. ([OpenAI Developers][2])

ดังนั้นประโยคที่ถูกต้องคือ:

> โมเดลเสนอหรือร้องขอให้เรียก Tool ส่วน Application เป็นผู้ Execute Tool

ไม่ใช่:

> โมเดลเข้าไปเรียก Python Function โดยตรง

---

# 10. `finish_reason == "tool_calls"`

หลังเรียกโมเดล Response อาจมีสองรูปแบบหลัก:

## กรณีตอบข้อความธรรมดา

```text
finish_reason = stop
```

โปรแกรมสามารถคืนข้อความให้ผู้ใช้ได้

## กรณีต้องการใช้ Tool

```text
finish_reason = tool_calls
```

ในกรณีนี้ Assistant Message จะมีข้อมูลประมาณ:

```text
Tool name
Arguments
Tool-call ID
```

โปรแกรมจึงต้อง:

1. อ่าน `tool_calls`
2. Parse Arguments
3. Execute Function
4. ส่ง Tool Result กลับ

---

# 11. ทำไมต้องใช้ `tool_call_id`

สมมติ Agent ขอเรียก Tool สองครั้ง:

```text
Tool Call A: บันทึก a@example.com
Tool Call B: บันทึก b@example.com
```

เมื่อส่งผลลัพธ์กลับ โมเดลต้องรู้ว่า Result ใดตอบ Tool Call ใด

```text
tool_call_id A ↔ ผลของ Tool A
tool_call_id B ↔ ผลของ Tool B
```

จึงไม่ควรสร้าง ID ใหม่เอง แต่ต้องใช้ ID ที่โมเดลส่งมา

Official API Reference ระบุว่า Tool Message ต้องมี `tool_call_id` เพื่อเชื่อม Tool Result กับ Tool Call ที่กำลังตอบกลับ. ([OpenAI Developers][3])

---

# 12. Tool Message ต้องถูกเพิ่มใน History

หลัง Execute Tool โปรแกรมต้องเพิ่มสองส่วน:

```text
Assistant Message ที่ขอเรียก Tool
Tool Message ที่บอกผลลัพธ์
```

ประวัติจะกลายเป็น:

```text
User: ติดต่อผมที่ a@example.com
Assistant: ต้องการเรียก record_email
Tool: บันทึก Email สำเร็จแล้ว
```

จากนั้นจึงเรียกโมเดลอีกครั้ง เพื่อให้โมเดลตอบข้อความที่มนุษย์อ่านได้ เช่น:

```text
ขอบคุณครับ ผมบันทึกอีเมลของคุณแล้ว
```

ถ้าไม่ส่ง Tool Result กลับ โมเดลจะไม่รู้ว่าการทำงานสำเร็จ ล้มเหลว หรือเกิดอะไรขึ้น

---

# 13. จาก `if` ไปเป็น `while`: จุดเกิด Agent Loop

เวอร์ชันแรกใช้:

```text
ถ้ามี Tool Call
    เรียก Tool หนึ่งครั้ง
    เรียกโมเดลอีกครั้ง
    จบ
```

รูปแบบนี้รองรับ Tool Round เดียว

ต่อมา Notebook เปลี่ยนเป็น:

```text
ตราบใดที่โมเดลยังขอ Tool
    Execute Tool ทุกตัว
    ส่งผลกลับ
    เรียกโมเดลใหม่
```

Notebook ระบุว่าการเปลี่ยนจาก `if` เป็น `while` และวนผ่าน Tool Calls ทั้งหมดคือการสร้าง Agent Loop ด้วยมือ. ([GitHub][1])

Agent Loop จึงมีรูปแบบ:

```text
Observe
   ↓
Reason / Decide
   ↓
Act through Tool
   ↓
Observe Tool Result
   ↓
Reason / Decide again
   ↓
หยุดเมื่อมี Final Answer
```

---

# 14. Agent Loop แบบ Pseudocode

```python
def run_agent(user_message, history):
    context = build_messages(user_message, history)
    response = call_model(context, available_tools)

    step_count = 0

    while model_requested_tools(response):
        step_count += 1

        if step_count > MAX_STEPS:
            return "ระบบหยุดเพื่อป้องกันการวนซ้ำ"

        context.append(response.message)

        for request in response.tool_calls:
            arguments = validate_arguments(request.arguments)
            result = execute_tool(request.name, arguments)

            context.append(
                make_tool_result(
                    call_id=request.id,
                    result=result
                )
            )

        response = call_model(context, available_tools)

    return response.text
```

นี่คือแก่นที่ Framework อย่าง OpenAI Agents SDK, LangGraph หรือ CrewAI ช่วยจัดการให้ในระดับ abstraction ที่สูงขึ้น

---

# 15. Agent Framework ซ่อนอะไรไว้

Notebook เปรียบ Tool Calling ว่าเบื้องหลังเป็น Logic ธรรมดา เช่น `if`, `while` และการจัด Messages ส่วน Framework ทำหน้าที่เป็น Abstraction Layer เพื่อไม่ให้ผู้พัฒนาต้องเขียนงานประกอบเหล่านี้ซ้ำเอง. ([GitHub][1])

Framework มักช่วยจัดการ:

* Agent Loop
* Tool Registry
* Argument Parsing
* Tool Dispatch
* Tool Result Messages
* Retry
* Error Handling
* Tracing
* State
* Guardrails
* Maximum Steps

แต่การเรียน Lab 3 ด้วยมือมีคุณค่ามาก เพราะทำให้เราไม่เข้าใจผิดว่า Framework มีเวทมนตร์บางอย่าง

---

# 16. จุดอ่อนของ Code ใน Lab

Code นี้ออกแบบเพื่อการเรียน ไม่ใช่ Production

## 16.1 Hardcode Tool

Logic สมมติว่า Tool ทุกตัวเป็น Email Tool จึงยัง Dispatch Tool ตามชื่อไม่ได้

ระบบที่ดีควรมี Registry:

```python
tool_registry = {
    "record_email": record_email,
    "search_profile": search_profile,
}
```

---

## 16.2 ไม่ Validate Email

String อย่างนี้อาจถูกบันทึก:

```text
not-an-email
```

ควรตรวจรูปแบบก่อนเขียนไฟล์

---

## 16.3 Trust Arguments จากโมเดลมากเกินไป

Official API Reference เตือนว่า Arguments จาก Tool Call อาจไม่ใช่ JSON ที่ถูกต้องหรืออาจมี Parameter ที่ไม่ได้กำหนดไว้ จึงต้อง Validate ก่อน Execute. ([OpenAI Developers][3])

ควรตรวจ:

* JSON Parse ได้หรือไม่
* Tool มีอยู่จริงหรือไม่
* Field ครบหรือไม่
* Type ถูกต้องหรือไม่
* Value ปลอดภัยหรือไม่

---

## 16.4 ไม่มี Maximum Iterations

`while` อาจวนต่อเนื่องหากโมเดลขอ Tool ซ้ำ

ควรมี:

```python
MAX_TOOL_ROUNDS = 5
```

---

## 16.5 ไม่มี Error Handling

ไฟล์อาจเขียนไม่ได้ หรือ Tool อาจล้มเหลว

Tool Result ควรแจ้งความจริง:

```text
success: true
```

หรือ:

```text
success: false
error: permission denied
```

ห้ามส่งกลับว่า “บันทึกแล้ว” หาก Tool ทำงานไม่สำเร็จ

---

## 16.6 Email เป็นข้อมูลส่วนบุคคล

ระบบ Production ควรมี:

* Consent
* Privacy notice
* Secure storage
* Access control
* Data retention policy
* ป้องกันการบันทึกซ้ำ
* ไม่เก็บลง Text File แบบเปิดเผย

---

# 17. เชื่อมกับ Episodic Artifact ของ Lab 1–2

## จาก Lab 1

```text
Lab 1:
Output A → Input B

Lab 3:
Model Tool Call → Tool Result → Model Call ใหม่
```

นี่คือ Chaining ที่มี Action เพิ่มเข้ามา

## จาก Lab 2

```text
Lab 2:
LLM Judge ประเมินคำตอบ

Lab 3 Exercise:
ให้ LLM อีกตัวตรวจว่า Digital Twin ตอบเฉพาะเรื่องงานหรือไม่
```

จึงสามารถประกอบ Pattern ได้เป็น:

```text
Digital Twin สร้างคำตอบ
        ↓
Evaluator ตรวจ Domain
        ↓
ผ่าน → ส่งให้ผู้ใช้
ไม่ผ่าน → แก้หรือปฏิเสธ
```

แต่ Evaluator ยังไม่ใช่ Ground Truth ตามบทเรียน Lab 2

---

# 18. Exercise ของ Lab 3

Notebook ให้แบบฝึกหัดสองข้อ:

1. เพิ่ม LLM Call เพื่อตรวจว่าคำตอบเกี่ยวข้องกับงานเท่านั้น
2. ประยุกต์ Digital Twin เข้ากับธุรกิจของผู้เรียน และบันทึก Email ของผู้ที่ต้องการติดต่อ. ([GitHub][1])

## Exercise ที่ควรทำตามลำดับ

### Exercise A: ตรวจ Domain

สร้าง Evaluator ให้คืน Structured Result:

```json
{
  "work_related": true,
  "reason": "The response discusses professional experience."
}
```

Flow:

```text
User
 ↓
Digital Twin
 ↓
Work-domain Evaluator
 ├── ผ่าน → ส่งคำตอบ
 └── ไม่ผ่าน → เปลี่ยนเป็นข้อความปฏิเสธ
```

### Exercise B: ทดสอบ Tool Calling

ลองข้อความ:

```text
I would like to contact you. My email is test@example.com
```

ตรวจสามจุด:

1. โมเดลสร้าง Tool Call หรือไม่
2. `emails.txt` มี Email หรือไม่
3. โมเดลตอบยืนยันหลัง Tool Result หรือไม่

### Exercise C: ทดสอบว่าไม่เรียก Tool โดยไม่จำเป็น

ลองข้อความ:

```text
Tell me about your professional experience.
```

กรณีนี้ควรตอบข้อความโดยไม่เรียก Email Tool

---

# 19. Checklist ตรวจความเข้าใจ

ก่อนจบ Lab 3 คุณควรตอบได้ว่า:

### System Prompt กับ History ต่างกันอย่างไร

System Prompt กำหนดบทบาทและกฎ ส่วน History คือบทสนทนาที่เกิดขึ้นแล้ว

### LLM มี Memory ใน Lab นี้หรือไม่

ไม่มี Memory ถาวร โปรแกรมส่ง History กลับไปทุก Request เพื่อสร้างภาพว่าจำได้

### Tool Schema ใช้ทำอะไร

อธิบายชื่อ วัตถุประสงค์ และ Arguments ของ Tool ให้โมเดลรู้

### ใคร Execute Python Function

Application Code ไม่ใช่ LLM

### ทำไมต้องส่ง Tool Result กลับให้โมเดล

เพื่อให้โมเดลรู้ผลของ Action และสร้างคำตอบหรือเลือก Action ถัดไป

### ทำไม `while` จึงสำคัญ

เพราะ Agent สามารถเรียก Tool ทำงาน ดูผล และเรียก Tool ต่อได้หลายรอบจนกว่าจะสร้าง Final Answer

### Lab 3 เป็น Agent แล้วหรือยัง

เป็น Agent แบบพื้นฐาน เพราะมี:

```text
LLM decision
+ Tool use
+ Result observation
+ Repeated loop
```

แต่ยังมี Tool เดียว State แบบง่าย และไม่มี Production Guardrails

---

## ภารกิจ Lab 3

รันตามลำดับนี้:

```text
1. แทนที่ twin/linkedin.pdf
2. แก้ twin/summary.txt
3. ตรวจข้อความที่ Extract จาก PDF
4. ทดลอง System Prompt
5. เปิด Gradio Chat
6. ทดลองถามต่อเนื่องเพื่อดู Conversation History
7. ทดลองให้ Email
8. ตรวจ emails.txt
9. ทดลอง Agent Loop เวอร์ชัน while
```

แก่นที่ต้องจำจาก Lab 3 คือ:

> Agent ไม่ใช่แค่ LLM ที่ตอบเก่ง แต่เป็น Loop ที่ให้โมเดลตัดสินใจเลือก Action โปรแกรม Execute Action ส่งผลกลับ แล้วโมเดลจึงตัดสินใจต่อจนงานเสร็จ.

[1]: https://github.com/ed-donner/agents/raw/refs/heads/main/1_foundations/3_lab3.ipynb "raw.githubusercontent.com"
[2]: https://developers.openai.com/api/docs/guides/function-calling?utm_source=chatgpt.com "Function calling | OpenAI API"
[3]: https://developers.openai.com/api/reference/ruby/resources/chat/?utm_source=chatgpt.com "Chat | OpenAI API Reference"
