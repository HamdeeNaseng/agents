# เริ่ม Week 1: Foundations

Week 1 จะเป็นฐานรากของทั้งหลักสูตร เป้าหมายไม่ใช่จำคำสั่ง OpenAI API แต่ต้องเข้าใจว่า **LLM ธรรมดาค่อย ๆ ถูกประกอบให้กลายเป็น Agent ได้อย่างไร**

Repository ล่าสุดแบ่ง Week 1 เป็น Lab 1–4, Extra Lab และโปรเจกต์ `twin` โดยเริ่มจากการเรียก LLM ผ่าน API ไปจนถึงการสร้าง Digital Twin ที่มีเครื่องมือและ Agent Loop แบบเขียนเอง ([GitHub][1])

## แผนที่ Week 1

| Lab   | สิ่งที่จะเรียน                                     | แก่นที่ต้องเข้าใจ                      |
| ----- | -------------------------------------------------- | -------------------------------------- |
| Lab 1 | Environment, API, messages, การเรียก LLM หลายรอบ   | LLM call และการส่ง Output ไปเป็น Input |
| Lab 2 | หลาย Model Provider และ LLM Judge                  | Parallelization และ Evaluator Pattern  |
| Lab 3 | Digital Twin, System Prompt, History, Tool Calling | Memory illusion และ Tool Definition    |
| Lab 4 | Agent Loop, หลาย Tools, Modularization             | Agent ทำงานวนรอบอย่างไร                |
| Extra | เนื้อหาเสริม                                       | ใช้หลังจากพื้นฐานแน่นแล้ว              |

Lab 1 ใช้โมเดลสร้างคำถาม ให้โมเดลอีกครั้งตอบ แล้วส่งคำถามและคำตอบให้โมเดลที่สามประเมิน ส่วน Lab 2 ขยายไปสู่การเรียกหลาย Provider และใช้โมเดลหนึ่งเป็นกรรมการจัดอันดับคำตอบ ([GitHub][2])

---

# Mental Model สำคัญที่สุด

ก่อนเปิด Notebook ให้จำภาพนี้:

```text
User
  ↓
Messages
  ↓
LLM
  ↓
Response
```

นี่เป็นเพียง **LLM Application** ยังไม่ใช่ Agent เต็มรูปแบบ

Agent จะเริ่มเป็นแบบนี้:

```text
Goal
  ↓
LLM ตัดสินใจ
  ↓
เลือก Action หรือ Tool
  ↓
โปรแกรมรัน Tool
  ↓
ส่งผลลัพธ์กลับให้ LLM
  ↓
LLM ตัดสินใจอีกครั้ง
  ↓
จบเมื่อบรรลุเป้าหมาย
```

## Metaphor: ครัวของ Imed Transform

* **LLM** คือพ่อครัวที่คิดและให้คำแนะนำ
* **Prompt** คือใบสั่งอาหาร
* **Context** คือข้อมูลวัตถุดิบและสูตรที่วางอยู่ตรงหน้า
* **Tool** คือมีด เตา เครื่องชั่ง หรือเครื่องตรวจอาหาร
* **Agent Loop** คือวงจรที่พ่อครัวตรวจงาน ใช้เครื่องมือ ดูผล แล้วตัดสินใจทำขั้นต่อไป
* **Agent Framework** คือระบบจัดการครัวที่ซ่อนขั้นตอนยิบย่อยไว้ให้

ข้อสำคัญคือ พ่อครัวไม่ได้ใช้เครื่องมือจริงด้วยตัวเอง โปรแกรมของเราเป็นผู้หยิบเครื่องมือไปทำงานตามคำขอของ LLM

---

# Lab 1: First LLM Calls

ไฟล์:

```text
1_foundations/1_lab1.ipynb
```

## Learning Objectives

เมื่อจบ Lab 1 คุณควรสามารถ:

1. อธิบายว่า `.env` และ Environment Variable ใช้ทำอะไร
2. สร้าง API Client
3. อธิบายโครงสร้าง `messages`
4. เรียก LLM และดึงข้อความจาก Response
5. ส่ง Output ของ LLM หนึ่งไปเป็น Input ของ LLM อีกครั้ง
6. แยกความแตกต่างระหว่าง LLM Call, Workflow และ Agent
7. เข้าใจว่าผลลัพธ์จาก LLM ต้องได้รับการประเมิน ไม่ควรเชื่อทันที

## Prerequisites

ก่อนเริ่มให้แน่ใจว่า:

```powershell
cd D:\path\to\agents2
uv sync
uv run python --version
```

จากนั้นเปิด:

```text
1_foundations/1_lab1.ipynb
```

เลือก Kernel ที่เป็น `.venv` ซึ่ง Notebook ล่าสุดระบุ Python 3.12.x และให้รันแต่ละ Cell ด้วย `Shift+Enter` ([GitHub][2])

---

# ส่วนที่ 1: โหลด `.env`

```python
from dotenv import load_dotenv

load_dotenv(override=True)
```

## `load_dotenv()` ทำอะไร

ไฟล์ `.env` อาจมี:

```env
OPENAI_API_KEY=sk-...
```

`load_dotenv()` จะอ่านไฟล์นี้และนำค่าเข้า Environment ของ Python Process

หลังจากนั้น Python จึงสามารถอ่านได้ด้วย:

```python
import os

openai_api_key = os.getenv("OPENAI_API_KEY")
```

## `override=True` หมายถึงอะไร

หมายถึง:

> ถ้า Environment มีค่าเดิมอยู่แล้ว ให้นำค่าจาก `.env` เขียนทับได้

สมมติว่าเครื่องมี:

```text
OPENAI_API_KEY=old-key
```

แต่ `.env` มี:

```text
OPENAI_API_KEY=new-key
```

เมื่อใช้:

```python
load_dotenv(override=True)
```

โปรแกรมจะใช้ `new-key`

## จุดที่มักเข้าใจผิด

`.env` ไม่ได้ส่ง Key ไปยัง OpenAI ทันที มันเพียงทำให้โปรแกรมของเราอ่าน Key ได้

ลำดับจริงคือ:

```text
.env
  ↓ load_dotenv()
Environment Variable
  ↓ OpenAI()
API Client
  ↓ API Request
OpenAI Server
```

---

# ส่วนที่ 2: ตรวจสอบ API Key

```python
import os

openai_api_key = os.getenv("OPENAI_API_KEY")

if openai_api_key:
    print(f"OpenAI API Key exists and begins {openai_api_key[:8]}")
else:
    print("OpenAI API Key not set")
```

โค้ดนี้ไม่ได้ตรวจว่า Key ใช้งานได้จริง เพียงตรวจว่า:

* ตัวแปรมีค่าอยู่หรือไม่
* ชื่อ Environment Variable สะกดถูกหรือไม่
* `.env` ถูกโหลดหรือไม่

การตรวจว่า Key ใช้ได้จริงจะเกิดขึ้นเมื่อส่ง API Request

## ข้อควรระวัง

อย่าพิมพ์ Key เต็ม:

```python
print(openai_api_key)  # ไม่ควรทำ
```

Notebook จึงแสดงเพียงอักขระช่วงต้นเพื่อช่วย Debug ([GitHub][2])

---

# ส่วนที่ 3: สร้าง API Client

```python
from openai import OpenAI

openai = OpenAI()
```

คำว่า:

```python
OpenAI()
```

คือการสร้าง **Instance** จาก Class `OpenAI`

คิดเหมือน:

```text
OpenAI Class
    ↓ สร้าง
openai Object
```

ตัวแปร `openai` จะมี Method สำหรับติดต่อ API เช่น:

```python
openai.chat.completions.create(...)
```

โดยปกติ `OpenAI()` จะอ่าน Key จาก:

```text
OPENAI_API_KEY
```

จึงไม่จำเป็นต้องเขียน:

```python
OpenAI(api_key="sk-...")
```

และไม่ควรฝัง Secret ไว้ใน Source Code

---

# ส่วนที่ 4: Messages

```python
messages = [
    {
        "role": "user",
        "content": "Tell me a fun fact"
    }
]
```

`messages` เป็น List ของ Dictionary

โครงสร้างคือ:

```text
messages
└── message
    ├── role
    └── content
```

## `role`

Role ที่พบบ่อย:

| Role        | หน้าที่                    |
| ----------- | -------------------------- |
| `system`    | กำหนดบทบาท กฎ และบริบทหลัก |
| `user`      | ข้อความจากผู้ใช้           |
| `assistant` | คำตอบก่อนหน้าของโมเดล      |
| `tool`      | ผลลัพธ์ที่ได้จาก Tool      |

ตัวอย่าง:

```python
messages = [
    {
        "role": "system",
        "content": "You are an expert Java modernization assistant."
    },
    {
        "role": "user",
        "content": "Explain the difference between CMP Entity Bean and JPA."
    }
]
```

## Mental Model

`messages` คือเอกสารทั้งหมดที่นำไปวางตรงหน้าโมเดลในครั้งนั้น

LLM ไม่ได้เปิดอ่านข้อความจากตัวแปร Python เอง มันเห็นเฉพาะข้อมูลที่เราส่งไปใน Request

---

# ส่วนที่ 5: เรียกโมเดล

Notebook ใช้รูปแบบ:

```python
response = openai.chat.completions.create(
    model="gpt-5.4-nano",
    messages=messages
)
```

และดึงคำตอบด้วย:

```python
answer = response.choices[0].message.content
print(answer)
```

Repository ปัจจุบันใช้ `gpt-5.4-nano`, `gpt-5.4-mini` และ `gpt-5.4` ใน Lab 1 โดยเลือกโมเดลที่แรงขึ้นสำหรับงานประเมินคำตอบ ([GitHub][2])

## Response ไม่ใช่ String

`response` เป็น Object ที่มีข้อมูลหลายอย่าง เช่น:

```text
response
├── id
├── model
├── choices
│   └── message
│       ├── role
│       └── content
└── usage
```

ดังนั้นจึงต้องเข้าไปตามเส้นทาง:

```python
response.choices[0].message.content
```

แปลว่า:

1. เอา `choices`
2. เลือกผลลัพธ์ตัวแรก `[0]`
3. เข้าไปที่ `message`
4. ดึง `content`

---

# ส่วนที่ 6: การต่อ LLM หลายครั้ง

Lab 1 ทำประมาณนี้:

```text
LLM 1 สร้างคำถามยาก
        ↓
LLM 2 ตอบคำถาม
        ↓
LLM 3 ประเมินว่าคำตอบถูกหรือไม่
```

นี่เป็นแนวคิดสำคัญมาก เพราะโปรแกรมไม่ได้ใช้ LLM เพียงครั้งเดียว แต่จัดบทบาทให้แต่ละ Call ต่างกัน

ตัวอย่าง:

```python
question_request = """
Create a difficult question that tests reasoning.
Respond only with the question.
"""

messages = [
    {"role": "user", "content": question_request}
]

response = openai.chat.completions.create(
    model="gpt-5.4-mini",
    messages=messages
)

question = response.choices[0].message.content
```

จากนั้นส่ง `question` ไปอีกครั้ง:

```python
messages = [
    {"role": "user", "content": question}
]

response = openai.chat.completions.create(
    model="gpt-5.4-mini",
    messages=messages
)

answer = response.choices[0].message.content
```

แล้วสร้างข้อความสำหรับ Evaluator:

```python
evaluation_prompt = f"""
Here is a question:

{question}

Here is a proposed answer:

{answer}

Evaluate whether the answer is correct.
Explain any important mistakes.
"""
```

---

# นี่เป็น Agent แล้วหรือยัง

**ยังไม่ใช่ Agent เต็มรูปแบบ**

สิ่งที่ Lab 1 สร้างคือ:

```text
Fixed LLM Workflow
```

หรือ Pipeline ที่กำหนดไว้ล่วงหน้า:

```text
Step 1 → Step 2 → Step 3
```

โมเดลไม่ได้เลือกเองว่าจะ:

* เรียก Tool ไหน
* ทำขั้นตอนอะไรต่อ
* ทำซ้ำหรือไม่
* หยุดเมื่อใด

ดังนั้นควรเรียกว่า:

* LLM Pipeline
* Chained LLM Calls
* Fixed Orchestration
* Evaluator Workflow

แต่ Lab นี้เป็นก้าวแรกของ Agentic AI เพราะเริ่มมี:

* การแบ่งบทบาท
* การส่งผลลัพธ์ระหว่างขั้นตอน
* การตรวจคำตอบของโมเดลด้วยโมเดลอีกตัว
* โปรแกรมภายนอกเป็น Orchestrator

---

# Pattern ที่ซ่อนอยู่ใน Lab 1

## 1. Generator Pattern

โมเดลสร้างคำถามหรือคำตอบ:

```text
Prompt → Generated Output
```

## 2. Chaining Pattern

ผลลัพธ์หนึ่งกลายเป็น Input ถัดไป:

```text
Output A → Input B
```

## 3. Evaluator Pattern

โมเดลอีกตัวตรวจงาน:

```text
Generator → Evaluator
```

## 4. Role Specialization

แต่ละ LLM Call มีหน้าที่ต่างกัน:

```text
Question Creator
Answerer
Judge
```

แม้ใช้ Model ตัวเดียวกัน แต่เมื่อ Prompt และบทบาทต่างกัน ก็ถือว่าเป็นคนละ “หน้าที่เชิงตรรกะ”

---

# สิ่งที่ห้ามเข้าใจผิด

## “โมเดลแรงกว่าเป็นผู้ตัดสิน จึงถูกเสมอ”

ไม่จริง

Evaluator ก็เป็น LLM จึงอาจ:

* เข้าใจโจทย์ผิด
* ให้เหตุผลผิด
* ชอบคำตอบที่เขียนดูดี
* ให้คะแนนตัวเองหรือ Model บางตัวสูงกว่า
* ถูก Prompt Injection จากคำตอบที่นำมาตรวจ

ในระบบจริงต้องเสริม:

```text
LLM Evaluator
+ Deterministic Test
+ Human Review
+ Ground Truth
```

สำหรับ Imed Transform:

```text
Reviewer Agent บอกว่าโค้ดดี
```

ยังไม่เพียงพอ ต้องมี:

```text
mvn compile
Unit Test
Schema Validation
Field Coverage
Static Analysis
```

---

# แบบฝึกหัดสำหรับ Imed Transform

แทนที่จะใช้โจทย์ธุรกิจทั่วไป ให้ทำ Pipeline นี้:

```text
Call 1: ระบุปัญหา Legacy Migration
Call 2: วิเคราะห์ Pain Point
Call 3: เสนอ Agentic Solution
Call 4: วิจารณ์ว่าส่วนใดควรใช้ LLM และส่วนใดควรใช้โปรแกรมปกติ
```

ตัวอย่าง Code:

```python
from dotenv import load_dotenv
from openai import OpenAI

load_dotenv(override=True)
client = OpenAI()

model = "gpt-5.4-mini"


def ask(prompt: str) -> str:
    response = client.chat.completions.create(
        model=model,
        messages=[
            {"role": "user", "content": prompt}
        ]
    )
    return response.choices[0].message.content


migration_problem = ask("""
Identify one difficult problem commonly encountered when migrating
an enterprise Java application from EJB 2.x CMP Entity Beans to JPA.

Respond with only the problem.
""")

pain_point = ask(f"""
Here is an enterprise Java migration problem:

{migration_problem}

Explain the technical and business pain caused by this problem.
Be specific and concise.
""")

agentic_solution = ask(f"""
Problem:

{migration_problem}

Pain point:

{pain_point}

Propose an Agentic AI solution for this problem.
Describe the agents, tools and validation steps.
""")

evaluation = ask(f"""
Evaluate the following Agentic AI proposal:

{agentic_solution}

Identify:

1. Tasks appropriate for an LLM
2. Tasks that must be deterministic
3. Major technical risks
4. Missing validation mechanisms
""")

print("PROBLEM")
print(migration_problem)

print("\nPAIN POINT")
print(pain_point)

print("\nSOLUTION")
print(agentic_solution)

print("\nEVALUATION")
print(evaluation)
```

---

# สิ่งที่ต้องสังเกตระหว่างรัน

อย่าดูแค่ว่า “Code ผ่านหรือไม่” ให้สังเกตว่า:

1. ผลลัพธ์แต่ละครั้งเหมือนกันหรือไม่
2. ถ้ารันซ้ำ คำตอบเปลี่ยนตรงไหน
3. Pain Point อิงข้อมูลจาก Call แรกจริงหรือไม่
4. Evaluator ตรวจพบจุดอ่อนจริง หรือเพียงสรุปคำตอบเดิม
5. มีงานใดที่ LLM เสนอให้ Agent ทำ แต่จริง ๆ ควรใช้ AST, Compiler หรือ Test
6. Prompt ที่คลุมเครือทำให้ผลลัพธ์กว้างแค่ไหน

---

# แบบตรวจความเข้าใจ Lab 1

ก่อนเดินต่อ Lab 2 คุณควรตอบคำถามเหล่านี้ได้:

### 1. `.env` แตกต่างจากตัวแปร Python อย่างไร

คำตอบที่ควรได้:

> `.env` เป็นไฟล์เก็บค่า Configuration ส่วน `load_dotenv()` นำค่าเหล่านั้นเข้า Environment และ `os.getenv()` ใช้อ่านค่าจาก Environment

### 2. `messages` คือ Memory ของโมเดลหรือไม่

คำตอบ:

> ไม่ใช่ Memory ภายในโมเดล แต่เป็น Context ที่โปรแกรมส่งไปใน Request ครั้งนั้น

### 3. ทำไมต้องใช้ `response.choices[0].message.content`

คำตอบ:

> เพราะ API Response เป็น Object ที่มีข้อมูลหลายส่วน ข้อความของโมเดลอยู่ใน Message ของ Choice แรก

### 4. Lab 1 เป็น Agent เต็มรูปแบบหรือยัง

คำตอบ:

> ยัง เป็น Fixed Workflow ที่เรียก LLM หลายครั้งตามลำดับที่ Programmer กำหนด

### 5. Evaluator LLM เชื่อถือได้ 100% หรือไม่

คำตอบ:

> ไม่ได้ ต้องมี Test, Ground Truth หรือ Human Review ประกอบ

---

## ภารกิจแรก

เปิด `1_foundations/1_lab1.ipynb` แล้วรันตั้งแต่ Cell แรกจนถึงการตอบ `Tell me a fun fact` จากนั้นส่งผลลัพธ์หรือ Error ที่เกิดขึ้นมา เราจะอ่าน Code และแก้ความเข้าใจทีละ Cell โดยไม่ข้ามจุดสำคัญครับ

[1]: https://github.com/ed-donner/agents/tree/main/1_foundations "agents/1_foundations at main · ed-donner/agents · GitHub"
[2]: https://raw.githubusercontent.com/ed-donner/agents/main/1_foundations/1_lab1.ipynb "raw.githubusercontent.com"
