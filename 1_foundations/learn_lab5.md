# Week 1 — Extra Lab: The Unreasonable Effectiveness of the Agent Loop

ไฟล์สุดท้ายของ Week 1 คือ:

```text
1_foundations/5_extra.ipynb
```

Notebook ใช้ชื่อว่า **“A little extra!”** และเน้นหัวข้อ **“The Unreasonable Effectiveness of the Agent Loop”** โดยให้เราสร้าง Agent ที่วางแผนผ่าน Todo List แล้วทำงานทีละขั้นจนตอบโจทย์ได้สำเร็จ. ([GitHub][1])

บทนี้เป็นการรวบยอดของ Lab 1–4:

```text
LLM Call
    ↓
Tool Calling
    ↓
External State
    ↓
Repeated Agent Loop
    ↓
Planning and Execution
```

---

## 1. Learning Objectives

เมื่อจบ Lab นี้ คุณควรเข้าใจว่า:

1. นิยามของ Agent มีได้หลายระดับ
2. Agent Loop ทำให้ LLM ดูมีความสามารถมากขึ้นอย่างไร
3. Todo List ทำหน้าที่เป็น External Working State อย่างไร
4. Planning Tool ต่างจาก Execution Tool อย่างไร
5. Agent สามารถใช้ Tool หลายรอบเพื่อรักษาความต่อเนื่องของงานได้อย่างไร
6. การที่ Agent ระบุว่า Task เสร็จ ไม่ได้พิสูจน์ว่างานถูกต้องจริง
7. Agent Loop แบบพื้นฐานสามารถสร้างได้โดยไม่ใช้ Framework
8. Framework ในสัปดาห์ถัดไปกำลังห่อหุ้ม Logic ใดไว้

---

# 2. Agent คืออะไร

Notebook เสนอนิยามสามแบบ:

### นิยามที่ 1

> ระบบ AI ที่สามารถทำงานให้เราได้อย่างเป็นอิสระ

นิยามนี้เน้นผลลัพธ์เชิงผลิตภัณฑ์ เช่น Agent สามารถรับเป้าหมายแล้วจัดการงานให้ผู้ใช้ได้

### นิยามที่ 2

> ระบบที่ LLM เป็นผู้ควบคุม Workflow

จุดสำคัญคือ Programmer ไม่ได้กำหนดทุก Step แบบตายตัว แต่ LLM ตัดสินใจว่าควรทำอะไรต่อ

### นิยามที่ 3

> LLM Agent ใช้ Tools ซ้ำเป็น Loop เพื่อบรรลุเป้าหมาย

Notebook มองว่านิยามที่สามกำลังกลายเป็นนิยามเชิงวิศวกรรมที่สำคัญ:

```text
Goal
 ↓
LLM decides
 ↓
Tool call
 ↓
Tool result
 ↓
LLM decides again
 ↓
Final answer
```

Lab นี้จะทำให้นิยามที่สามเห็นเป็น Code จริง. ([GitHub][2])

---

# 3. ความหมายของ “Unreasonable Effectiveness”

คำนี้ไม่ได้หมายความว่า Agent Loop สามารถแก้ทุกปัญหาได้

ความหมายคือ Logic ที่ดูเรียบง่ายมาก:

```python
while not done:
    call_model()
    execute_tools()
    return_results()
```

กลับสามารถสร้างพฤติกรรมที่ซับซ้อนได้ เช่น:

* วางแผน
* แบ่งงาน
* ทำงานทีละขั้น
* ติดตามความคืบหน้า
* แก้ปัญหาหลายขั้น
* ปรับการตัดสินใจหลังเห็นผลลัพธ์

ความสามารถส่วนใหญ่ไม่ได้มาจาก Loop เพียงอย่างเดียว แต่เกิดจากการประกอบ:

```text
LLM capability
+ Tool descriptions
+ External state
+ Prompt instructions
+ Repeated observations
```

---

# 4. Todo State

Notebook สร้าง List สองตัว:

```python
todos = []
completed = []
```

ตัวอย่างสถานะ:

```python
todos = [
    "Estimate the distance",
    "Calculate the first train's progress",
    "Calculate the meeting time"
]

completed = [
    True,
    False,
    False
]
```

ความสัมพันธ์อาศัยตำแหน่งเดียวกัน:

```text
todos[0]     ↔ completed[0]
todos[1]     ↔ completed[1]
todos[2]     ↔ completed[2]
```

ดังนั้น State ของ Todo แรกอาจเป็น:

```text
Todo #1: completed
```

และ Todo ที่สอง:

```text
Todo #2: not completed
```

## สิ่งที่ต้องระวัง

นี่เรียกว่า **parallel lists** ซึ่งเข้าใจง่าย แต่มีความเสี่ยง หากสอง List ยาวไม่เท่ากัน

ตัวอย่างที่ผิด:

```python
todos = ["A", "B", "C"]
completed = [True, False]
```

เมื่อ Code เข้าถึง:

```python
completed[2]
```

จะเกิด `IndexError`

ระบบที่แข็งแรงกว่าอาจใช้:

```python
todos = [
    {"description": "A", "completed": True},
    {"description": "B", "completed": False}
]
```

แต่ Notebook ใช้สอง List เพื่อให้ผู้เรียนเห็น State Management อย่างตรงไปตรงมา

---

# 5. `get_todo_report()`

Function นี้อ่าน Todo ทั้งหมดแล้วสร้างรายงาน:

```python
def get_todo_report() -> str:
    result = ""

    for index, todo in enumerate(todos):
        if completed[index]:
            result += (
                f"Todo #{index + 1}: "
                f"[green][strike]{todo}[/strike][/green]\n"
            )
        else:
            result += f"Todo #{index + 1}: {todo}\n"

    show(result)
    return result
```

## `enumerate()` ทำหน้าที่อะไร

มันให้ทั้ง Index และข้อมูล:

```python
for index, todo in enumerate(todos):
```

ตัวอย่าง:

```text
index = 0, todo = "Estimate distance"
index = 1, todo = "Calculate travel"
```

แต่รายงานสำหรับมนุษย์ต้องการเริ่มจาก 1 จึงใช้:

```python
index + 1
```

---

# 6. Rich Console Markup

Notebook ใช้ Library `rich` เพื่อแสดงข้อความใน Terminal ให้มีรูปแบบ เช่น:

```text
[green]
```

ทำให้ข้อความเป็นสีเขียว และ:

```text
[strike]
```

ทำให้ข้อความมีเส้นขีดฆ่า

ดังนั้น:

```python
"[green][strike]Finish lab[/strike][/green]"
```

จะแสดง Todo ที่เสร็จแล้วเป็นสีเขียวและมีเส้นขีดฆ่า

Function `show()` มี Fallback:

```python
def show(text):
    try:
        Console().print(text)
    except Exception:
        print(text)
```

ถ้า Rich แสดงผลไม่ได้ ระบบจะกลับไปใช้ `print()` ธรรมดาแทน. ([GitHub][2])

---

# 7. Tool ที่หนึ่ง: `create_todos`

```python
def create_todos(descriptions: list[str]) -> str:
    todos.extend(descriptions)
    completed.extend([False] * len(descriptions))
    return get_todo_report()
```

สมมติ Agent ส่ง:

```python
descriptions = [
    "Estimate the distance",
    "Calculate the travel times",
    "Determine the meeting time"
]
```

บรรทัดนี้:

```python
todos.extend(descriptions)
```

จะเพิ่ม Todo ทั้งหมดเข้า List

ส่วน:

```python
completed.extend([False] * len(descriptions))
```

ถ้ามีสาม Todo จะสร้าง:

```python
[False, False, False]
```

ผลลัพธ์:

```python
todos = [
    "Estimate the distance",
    "Calculate the travel times",
    "Determine the meeting time"
]

completed = [
    False,
    False,
    False
]
```

## `append()` กับ `extend()` ต่างกันอย่างไร

ถ้าใช้:

```python
todos.append(descriptions)
```

ผลจะเป็น List ซ้อน List:

```python
[
    [
        "Estimate the distance",
        "Calculate travel"
    ]
]
```

แต่ `extend()` นำสมาชิกแต่ละตัวมาเพิ่ม:

```python
[
    "Estimate the distance",
    "Calculate travel"
]
```

---

# 8. Tool ที่สอง: `mark_complete`

```python
def mark_complete(
    index: int,
    completion_notes: str
) -> str:
    if 1 <= index <= len(todos):
        completed[index - 1] = True
    else:
        return "No todo at this index."

    Console().print(completion_notes)
    return get_todo_report()
```

Tool รับ:

* `index` — หมายเลข Todo แบบเริ่มจาก 1
* `completion_notes` — คำอธิบายว่างานนั้นทำเสร็จอย่างไร

## ทำไมใช้ `index - 1`

ผู้ใช้และ LLM เห็น:

```text
Todo #1
```

แต่ Python ใช้ Index:

```text
todos[0]
```

ดังนั้น:

```python
completed[index - 1] = True
```

---

# 9. จุดสำคัญ: Todo Tool ไม่ได้ทำงานจริง

นี่เป็นประเด็นที่ควรเข้าใจให้ชัดที่สุด

Tools ใน Lab มีเพียง:

```text
create_todos
mark_complete
```

ไม่มี Tool สำหรับ:

* ค้นหาระยะทาง
* ใช้ Calculator
* เปิดเว็บไซต์
* ตรวจคำตอบ
* Execute Code

ดังนั้นเมื่อ Agent เรียก:

```python
mark_complete(
    index=2,
    completion_notes="Calculated the travel time..."
)
```

การคำนวณไม่ได้เกิดขึ้นใน `mark_complete`

LLM เป็นผู้สร้างข้อความ `completion_notes` แล้ว Tool เพียง:

1. ทำเครื่องหมาย Todo ว่าเสร็จ
2. แสดงข้อความ
3. คืนรายการ Todo

ภาพที่ถูกต้อง:

```text
LLM reasons or calculates
        ↓
LLM says step completed
        ↓
mark_complete changes status
```

ไม่ใช่:

```text
mark_complete performs the calculation
```

---

# 10. Planning Tool กับ Execution Tool

Todo Tools เป็น **Planning and Tracking Tools**

```text
create_todos
สร้างแผน

mark_complete
ติดตามสถานะ
```

แต่ไม่ใช่ Execution Tools เช่น:

```text
calculator
คำนวณจริง

web_search
ค้นข้อมูลจริง

run_python
Execute Code จริง

query_database
อ่าน Database จริง
```

การแยกนี้สำคัญมาก:

| Tool ประเภท     | หน้าที่                      |
| --------------- | ---------------------------- |
| Planning Tool   | แบ่งงานและติดตามความคืบหน้า  |
| Execution Tool  | ทำ Action หรือคำนวณจริง      |
| Validation Tool | ตรวจว่าผลลัพธ์ถูกต้องหรือไม่ |

Agent ที่น่าเชื่อถือควรประกอบทั้งสามประเภท:

```text
Plan
 ↓
Execute
 ↓
Validate
 ↓
Complete
```

แต่ Agent ใน Lab นี้มีเพียง Planning/Tracking Tool ดังนั้นการ Mark ว่าเสร็จยังเป็น **Claim of completion** ไม่ใช่หลักฐานของ Completion

---

# 11. Tool Schema ของ `create_todos`

```python
create_todos_json = {
    "name": "create_todos",
    "description": (
        "Add new todos from a list of descriptions "
        "and return the full list"
    ),
    "parameters": {
        "type": "object",
        "properties": {
            "descriptions": {
                "type": "array",
                "items": {
                    "type": "string"
                }
            }
        },
        "required": ["descriptions"],
        "additionalProperties": False
    }
}
```

จุดใหม่จาก Lab ก่อนหน้าคือ Argument เป็น Array:

```json
{
  "descriptions": [
    "Step one",
    "Step two",
    "Step three"
  ]
}
```

Schema:

```json
"type": "array",
"items": {
  "type": "string"
}
```

หมายความว่า:

> ต้องเป็น List และสมาชิกแต่ละตัวต้องเป็น String

---

# 12. Tool Schema ของ `mark_complete`

```python
mark_complete_json = {
    "name": "mark_complete",
    "description": (
        "Mark complete the todo at the given position "
        "and return the full list"
    ),
    "parameters": {
        "type": "object",
        "properties": {
            "index": {
                "type": "integer"
            },
            "completion_notes": {
                "type": "string"
            }
        },
        "required": [
            "index",
            "completion_notes"
        ],
        "additionalProperties": False
    }
}
```

จุดสำคัญคือ `index` เป็น Integer:

```json
{
  "index": 1,
  "completion_notes": "Estimated the distance."
}
```

ไม่ควรเป็น:

```json
{
  "index": "first",
  "completion_notes": "..."
}
```

---

# 13. Tool Handler

```python
def handle_tool_calls(tool_calls):
    results = []

    for tool_call in tool_calls:
        tool_name = tool_call.function.name
        arguments = json.loads(
            tool_call.function.arguments
        )

        tool = globals().get(tool_name)
        result = tool(**arguments) if tool else {}

        results.append({
            "role": "tool",
            "content": json.dumps(result),
            "tool_call_id": tool_call.id
        })

    return results
```

โครงสร้างเหมือน Lab 4:

```text
Read tool name
 ↓
Parse arguments
 ↓
Find function
 ↓
Execute
 ↓
Create tool result
```

Notebook ใช้ `globals()` เพื่อให้ Code สั้นและมองเห็น Agent Loop ง่ายขึ้น. ([GitHub][2])

## ข้อควรระวัง

จาก Episodic Artifact Lab 4 เราทราบแล้วว่า Production ควรใช้ Explicit Map:

```python
TOOL_MAP = {
    "create_todos": create_todos,
    "mark_complete": mark_complete
}
```

แทน:

```python
globals().get(tool_name)
```

---

# 14. Agent Loop

หัวใจของ Notebook คือ:

```python
def loop(messages):
    done = False

    while not done:
        response = openai.chat.completions.create(
            model="gpt-5.2",
            messages=messages,
            tools=tools,
            reasoning_effort="none"
        )

        finish_reason = (
            response.choices[0].finish_reason
        )

        if finish_reason == "tool_calls":
            message = response.choices[0].message
            tool_calls = message.tool_calls

            results = handle_tool_calls(tool_calls)

            messages.append(message)
            messages.extend(results)

        else:
            done = True
            show(
                response.choices[0].message.content
            )
```

Notebook กำหนดให้ Loop ทำงานจนโมเดลไม่ขอ Tool เพิ่ม แล้วจึงแสดง Final Answer. ([GitHub][2])

---

# 15. อ่าน Loop ทีละรอบ

## รอบที่ 1: สร้างแผน

```text
User gives problem
 ↓
Model calls create_todos
 ↓
Todo list created
```

ตัวอย่าง:

```text
1. Estimate distance
2. Determine first train progress
3. Solve combined-speed equation
4. Calculate clock time
```

## รอบที่ 2: ทำ Step แรก

```text
Model observes todo list
 ↓
Model reasons about distance estimate
 ↓
Model calls mark_complete(index=1)
```

## รอบที่ 3: ทำ Step ถัดไป

```text
Model observes updated list
 ↓
Model works on next calculation
 ↓
Model calls mark_complete(index=2)
```

## รอบสุดท้าย

เมื่อ Todo ทั้งหมดเสร็จ โมเดลไม่เรียก Tool เพิ่มและตอบ Final Answer

```text
finish_reason != tool_calls
 ↓
done = True
 ↓
show(final response)
```

---

# 16. System Prompt

Notebook กำหนดให้ Agent:

* ใช้ Todo Tools เพื่อวางแผน
* ทำแต่ละ Step ตามลำดับ
* ใช้ค่าประมาณเมื่อโจทย์ขาดข้อมูล
* ไม่ถามคำถามเพิ่มเติม
* ตอบหลังจากใช้ Tools แล้ว
* แสดงผลด้วย Rich markup. ([GitHub][2])

ส่วนสำคัญคือ:

```text
If any quantity isn't provided,
include a step to come up with
a reasonable estimate.
```

นี่เกิดจากโจทย์รถไฟไม่ได้ให้ระยะทาง Boston–New York

Agent จึงไม่ควรแอบใช้ค่าที่ไม่บอกโดยไม่มีคำอธิบาย แต่ต้องสร้าง Step ว่า:

```text
Estimate the distance between the cities
```

---

# 17. โจทย์รถไฟและข้อสมมติฐาน

โจทย์มีข้อมูล:

```text
Train A:
ออก Boston เวลา 2:00 pm
ความเร็ว 60 mph

Train B:
ออก New York เวลา 3:00 pm
ความเร็ว 80 mph
```

แต่ขาด:

```text
ระยะทางระหว่าง Boston และ New York
```

ดังนั้นไม่มีคำตอบที่แน่นอนได้จนกว่าจะกำหนดระยะทาง `D`

## วิธีแก้เชิงสัญลักษณ์

เวลา 2:00–3:00 pm รถไฟ A เดินทางก่อนหนึ่งชั่วโมง:

```text
60 miles
```

ระยะทางที่เหลือเวลา 3:00 pm:

```text
D - 60
```

หลัง 3:00 pm รถไฟสองขบวนเคลื่อนเข้าหากันด้วย Combined Speed:

```text
60 + 80 = 140 mph
```

เวลาหลัง 3:00 pm ที่ใช้พบกัน:

```text
(D - 60) / 140 ชั่วโมง
```

ดังนั้นเวลาเจอกันคือ:

```text
3:00 pm + (D - 60) / 140 ชั่วโมง
```

Agent ต้องเลือกค่าประมาณของ `D` แล้วแจ้ง Assumption อย่างชัดเจน

---

# 18. External State ทำให้ Agent มีประสิทธิภาพอย่างไร

Todo List ทำหน้าที่เป็น **Scratchpad ที่มีโครงสร้าง**

แทนที่ Agent ต้องเก็บทุกอย่างอยู่ในข้อความที่ไม่เป็นระเบียบ มันเห็น:

```text
Todo #1: completed
Todo #2: pending
Todo #3: pending
```

สิ่งนี้ช่วย:

* ลดการลืม Step
* ทำให้ลำดับงานชัดเจน
* ช่วยตัดสินใจว่าอะไรควรทำต่อ
* ทำให้มนุษย์สังเกตการทำงานได้
* รองรับงานหลายขั้น

Mental Model:

```text
LLM = ผู้ปฏิบัติงาน

Todo state = กระดานงาน

Agent loop = วงจรตรวจบอร์ดและทำงานต่อ
```

---

# 19. State นี้เป็น Memory ประเภทใด

Todo Lists อยู่ใน Python Process:

```python
todos = []
completed = []
```

จึงเป็น:

```text
In-memory application state
```

ไม่ใช่:

* Long-term memory
* Model memory
* Database persistence
* Cross-session memory

ถ้า Restart Kernel:

```text
todos และ completed หาย
```

ถ้ามีผู้ใช้หลายคนพร้อมกัน Global Lists ยังอาจปะปนกัน เพราะทุก Session ใช้ State เดียวกัน

ระบบจริงควรแยก State ตาม:

```text
User ID
Session ID
Task ID
Run ID
```

---

# 20. Observability กับ Verification

Todo Report ช่วยให้เห็นว่า Agentกำลังทำอะไร จึงเพิ่ม **Observability**

แต่ไม่ได้เพิ่ม Verification โดยอัตโนมัติ

ตัวอย่าง:

```text
Todo #2 marked complete:
"Calculated the result correctly"
```

สิ่งนี้พิสูจน์เพียงว่า:

```text
โมเดลบอกว่าคำนวณแล้ว
```

ไม่ได้พิสูจน์ว่า:

```text
ผลคำนวณถูกต้อง
```

ต้องเพิ่ม Verification Tool เช่น:

```text
Calculator
Python execution
Unit test
Constraint checker
Independent evaluator
```

ดังนั้น:

```text
Visible progress ≠ Correct progress
```

---

# 21. ความเสี่ยงของ Agent Loop นี้

## 21.1 ไม่มี Maximum Steps

ถ้าโมเดลเรียก Tool ซ้ำไม่หยุด:

```python
while not done:
```

อาจวนไม่สิ้นสุด

ควรเพิ่ม:

```python
MAX_STEPS = 10
```

---

## 21.2 Mark Complete ก่อนทำจริง

LLM สามารถเรียก:

```text
mark_complete
```

โดยไม่ได้คำนวณหรือ Execute ขั้นตอนจริง

---

## 21.3 ไม่มี Validation

ไม่มีตัวตรวจว่า:

* Todo ถูกทำตามลำดับหรือไม่
* Completion Note ถูกต้องหรือไม่
* ทุก Todo เสร็จแล้วหรือไม่
* Final Answer สอดคล้องกับ Steps หรือไม่

---

## 21.4 State เป็น Global

หากรันหลาย Agent พร้อมกัน State อาจปะปน

---

## 21.5 ไม่มี Tool Error Handling

`json.loads()` หรือ Function อาจ Error แล้วทำให้ Agent หยุดทั้งหมด

---

## 21.6 `globals()` เปิดกว้างเกินไป

ควรใช้ Explicit Tool Registry

---

## 21.7 ไม่มีคำตอบ Return จาก `loop()`

Function แสดงผลผ่าน:

```python
show(...)
```

แต่ไม่ได้ `return` Final Answer ทำให้นำผลไปใช้ต่อหรือ Test ยากกว่า

ควรเปลี่ยนเป็น:

```python
final_answer = response.choices[0].message.content
show(final_answer)
return final_answer
```

---

# 22. Agent Loop ที่แข็งแรงขึ้น

```python
def loop(messages, max_steps: int = 10) -> str:
    for step in range(max_steps):
        response = openai.chat.completions.create(
            model="...",
            messages=messages,
            tools=tools
        )

        choice = response.choices[0]
        message = choice.message

        if choice.finish_reason != "tool_calls":
            final_answer = message.content or ""
            show(final_answer)
            return final_answer

        messages.append(message)

        try:
            results = handle_tool_calls(
                message.tool_calls or []
            )
        except Exception as exc:
            return f"Agent stopped: {exc}"

        messages.extend(results)

    return (
        "Agent stopped because it reached "
        "the maximum step limit."
    )
```

สิ่งที่เพิ่ม:

* Maximum Steps
* Return Final Answer
* Tool Error Handling
* ป้องกัน `tool_calls` เป็น `None`

---

# 23. Pattern ที่ Lab นี้สอน

## Plan-and-Execute Pattern

```text
Create Plan
 ↓
Execute Step
 ↓
Mark Complete
 ↓
Execute Next Step
```

## Externalized Working Memory

```text
Todo State
=
ข้อมูลความคืบหน้าที่อยู่นอก LLM
```

## Iterative Tool-Using Agent

```text
Decide
→ Tool
→ Observe
→ Decide again
```

## Progress Tracking Pattern

```text
Pending
→ In progress
→ Completed
```

แม้ Lab จะมีเพียง Pending และ Completed

## Goal-Directed Loop

Agent ไม่ได้หยุดหลัง Tool Call เดียว แต่ทำงานต่อจนสร้าง Final Answer

---

# 24. สิ่งที่ Lab นี้ไม่ได้ทำ

Lab นี้ยังไม่มี:

* Persistent State
* Task Dependencies
* Retry Policy
* Step Failure State
* Calculator Tool
* Search Tool
* Deterministic Validator
* Human Approval
* Parallel Execution
* Cost Budget
* Timeout
* Checkpoint
* Resume หลังระบบล้ม

จึงเป็น Demonstration ของ Agent Loop ไม่ใช่ระบบจัดการงาน Production

---

# 25. Exercise ที่ผู้สอนต้องการ

Notebook แนะนำให้สร้าง Notebook ใหม่แล้วเขียน Agent Loop ตั้งแต่ต้นด้วยตนเอง โดยอ้างอิงกลับมาที่ Lab นี้เมื่อจำเป็น. ([GitHub][2])

นี่เป็นหนึ่งในครั้งที่ควรพิมพ์ Code ด้วยตัวเอง แทนการ Copy ทั้งหมด เพราะเป้าหมายคือให้เข้าใจความสัมพันธ์ระหว่าง:

```text
Messages
Tools
Tool Schema
Tool Handler
Tool Result
Loop
State
Final Answer
```

---

# 26. ลำดับสร้างจากศูนย์

สร้างไฟล์ เช่น:

```text
1_foundations/my_agent_loop.ipynb
```

แล้วทำตามลำดับ:

```text
1. Import Libraries
2. Load Environment
3. Create Client
4. Define Application State
5. Define Python Tools
6. Define Tool Schemas
7. Create Tool List
8. Create Explicit Tool Map
9. Write Tool Handler
10. Write Agent Loop
11. Write System Prompt
12. Send User Goal
13. Test Multiple Tool Rounds
14. Add Maximum Steps
15. Test Error Cases
```

---

# 27. แบบทดสอบที่ควรทำ

## Test 1: งานง่าย

```text
Plan three steps for preparing breakfast,
then carry them out conceptually.
```

ตรวจว่า:

* เรียก `create_todos`
* เรียก `mark_complete`
* ตอบหลัง Todo เสร็จ

## Test 2: Argument ผิด

ลองให้ Handler ได้ Index ที่ไม่มี:

```python
mark_complete(99, "Done")
```

ต้องคืน:

```text
No todo at this index.
```

## Test 3: ไม่มี Todo

```python
todos, completed = [], []
get_todo_report()
```

ควรทำงานโดยไม่ Error

## Test 4: Loop Limit

ตั้ง:

```python
MAX_STEPS = 2
```

ตรวจว่า Agent หยุดอย่างควบคุมได้

## Test 5: State Reset

รัน:

```python
todos, completed = [], []
```

ตรวจว่า State เดิมหายไปจริง

---

# 28. Checklist ความเข้าใจ

ก่อนจบ Week 1 คุณควรตอบได้ว่า:

### Todo Tools ช่วย Agent อย่างไร

ช่วยวางแผน ติดตามความคืบหน้า และเก็บ Working State นอก LLM

### `mark_complete` ทำงานจริงหรือไม่

มันเปลี่ยนสถานะ Todo แต่ไม่ได้ Execute เนื้อหาของ Task

### ทำไม Agent Loop จึงทรงพลัง

เพราะโมเดลสามารถตัดสินใจ ใช้ Tool เห็นผล และตัดสินใจต่อหลายครั้งจนบรรลุเป้าหมาย

### Todo State เป็น Long-term Memory หรือไม่

ไม่ เป็น In-memory State และหายเมื่อ Process หรือ Kernel Restart

### Mark Complete เป็นหลักฐานว่างานถูกต้องหรือไม่

ไม่ เป็นเพียงสถานะที่ Agent ระบุ ต้องมี Validation แยกต่างหาก

### `for` และ `while` ต่างกันอย่างไรในบริบทนี้

```text
for tool_call
จัดการหลาย Tools ใน Response เดียว

while/Agent loop
จัดการหลายรอบของการตัดสินใจ
```

### ทำไมต้องมี Maximum Steps

เพื่อป้องกัน Infinite Loop ค่าใช้จ่ายเกิน และการทำ Action ซ้ำ

---

# 29. Week 1 Mental Model ฉบับสมบูรณ์

```text
User provides a goal
        ↓
Application builds context
        ↓
LLM observes context and state
        ↓
LLM decides:
   ├── Create a plan
   ├── Call an execution tool
   ├── Update progress
   ├── Request more observations
   └── Give final answer
        ↓
Application validates and executes
        ↓
Tool result becomes new observation
        ↓
Loop continues under limits
```

แก่นสุดท้ายของ Week 1 คือ:

> Agent ไม่ใช่โมเดลชนิดใหม่ แต่เป็นระบบที่นำ LLM มาอยู่ในวงจรการตัดสินใจ เชื่อมกับ State และ Tools แล้วให้ Application ควบคุมการ Execute จนกว่าจะบรรลุเป้าหมายหรือถึงเงื่อนไขหยุด

Lab สุดท้ายนี้แสดงอีกชั้นหนึ่งว่า **การวางแผนและการเห็นความคืบหน้าสามารถเพิ่มประสิทธิภาพของ Agent ได้มาก แต่ Todo ที่ถูกขีดว่าเสร็จยังไม่ใช่หลักฐานว่างานสำเร็จจริง—ระบบที่น่าเชื่อถือยังต้องแยก Planning, Execution และ Verification ออกจากกัน**

[1]: https://github.com/ed-donner/agents/tree/main/1_foundations "agents/1_foundations at main · ed-donner/agents · GitHub"
[2]: https://github.com/ed-donner/agents/raw/refs/heads/main/1_foundations/5_extra.ipynb "raw.githubusercontent.com"
