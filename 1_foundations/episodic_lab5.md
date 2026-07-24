# Episodic Learning Artifact

## Week 1 — Lab 5: The Unreasonable Effectiveness of the Agent Loop

**หลักสูตร:** AI Engineer Agentic Track
**ผู้สอน:** Ed Donner
**Repository:** `ed-donner/agents`
**ไฟล์เรียน:** `1_foundations/5_extra.ipynb`
**หัวข้อหลัก:** Planning, Todo State, Agent Loop และ Plan-and-Execute Pattern
**สถานะ:** เรียนและสรุป Week 1 แล้ว

---

# 1. Context

Lab 5 เป็น Lab สุดท้ายของ Week 1 และทำหน้าที่รวบรวมแนวคิดจาก Lab 1–4 ให้กลายเป็น Agent ที่สามารถ:

```text
รับเป้าหมาย
→ สร้างแผน
→ ติดตามงาน
→ ทำงานทีละขั้น
→ อัปเดตสถานะ
→ ตอบเมื่อเสร็จ
```

พัฒนาการของ Week 1:

```text
Lab 1
LLM Calls
→ Chained Workflow
→ Generator–Evaluator

Lab 2
Multi-Model Workflow
→ Fan-out/Fan-in
→ LLM-as-a-Judge

Lab 3
Conversation Context
→ Tool Calling
→ Agent Loop

Lab 4
Multiple Tools
→ Tool Registry
→ External Actions
→ Modular Application

Lab 5
Planning
→ External Working State
→ Plan-and-Execute
→ Goal-Directed Agent Loop
```

Lab นี้แสดงให้เห็นว่า Agent Loop ที่เรียบง่ายสามารถก่อให้เกิดพฤติกรรมหลายขั้นที่ดูซับซ้อนได้ โดยไม่ต้องใช้ Agent Framework

---

# 2. Learning Objectives

หลังจบ Lab 5 สามารถอธิบายได้ว่า:

1. Agent สามารถนิยามได้หลายระดับ
2. Agent Loop ทำให้ LLM ทำงานหลายขั้นได้อย่างไร
3. Todo List ทำหน้าที่เป็น External Working State อย่างไร
4. Planning Tool และ Execution Tool แตกต่างกันอย่างไร
5. `create_todos` และ `mark_complete` ทำหน้าที่อะไร
6. `for` และ `while` มีบทบาทต่างกันอย่างไร
7. การ Mark Task ว่าเสร็จไม่ได้พิสูจน์ว่าผลลัพธ์ถูกต้อง
8. Agent Loop ต้องมี Maximum Steps และ Error Handling
9. Global State มีข้อจำกัดอย่างไร
10. Framework ในบทต่อไปซ่อน Logic ใดไว้

---

# 3. Agent Definitions

Lab เสนอความหมายของ Agent หลายระดับ

## Product-Level Definition

```text
ระบบ AI ที่สามารถทำงานแทนผู้ใช้
ด้วยความเป็นอิสระระดับหนึ่ง
```

นิยามนี้เน้นผลลัพธ์และประสบการณ์ของผู้ใช้

---

## Workflow-Level Definition

```text
ระบบที่ LLM เป็นผู้ตัดสินใจ
หรือควบคุม Workflow
```

Programmer กำหนดกรอบและ Tools แต่ LLM ตัดสินใจว่าควรทำอะไรต่อ

---

## Engineering-Level Definition

```text
LLM
+ Tools
+ State
+ Repeated Loop
+ Goal
```

รูปแบบ:

```text
Goal
 ↓
Observe
 ↓
Decide
 ↓
Call Tool
 ↓
Receive Result
 ↓
Decide Again
 ↓
Final Answer
```

---

# 4. The Unreasonable Effectiveness of the Agent Loop

Logic หลักอาจดูเรียบง่าย:

```python
while not done:
    response = call_model()
    execute_requested_tools()
    send_results_back()
```

แต่เมื่อนำมารวมกับ:

```text
LLM capability
Tool descriptions
External state
System instructions
Repeated observations
```

ระบบสามารถแสดงพฤติกรรม เช่น:

* วางแผน
* แบ่งงาน
* ติดตามความคืบหน้า
* ทำงานตามลำดับ
* ปรับการตัดสินใจ
* สรุปผลเมื่อภารกิจเสร็จ

คำว่า “unreasonable effectiveness” จึงสื่อว่า Logic ที่เล็กและตรงไปตรงมากลับสร้างพฤติกรรมที่มีความซับซ้อนได้มากกว่าที่คาด

ไม่ได้หมายความว่า Agent Loop แก้ปัญหาได้ทุกประเภทหรือรับประกันความถูกต้อง

---

# 5. Todo State

Lab ใช้ List สองตัวเก็บสถานะ:

```python
todos = []
completed = []
```

ตัวอย่าง:

```python
todos = [
    "Estimate the distance",
    "Calculate the first train's progress",
    "Determine the meeting time"
]

completed = [
    True,
    False,
    False
]
```

ความสัมพันธ์อาศัยตำแหน่งเดียวกัน:

```text
todos[0] ↔ completed[0]
todos[1] ↔ completed[1]
todos[2] ↔ completed[2]
```

---

# 6. External Working State

Todo List เป็น State ที่อยู่นอกโมเดล

```text
LLM
= ผู้คิดและตัดสินใจ

Todo List
= กระดานติดตามงาน

Agent Loop
= วงจรที่ตรวจบอร์ดและเลือกงานถัดไป
```

ประโยชน์:

```text
ลดโอกาสลืมขั้นตอน
ทำให้ลำดับงานชัดเจน
แสดงว่างานใดเสร็จแล้ว
ช่วยเลือกงานถัดไป
เพิ่ม Observability
```

Todo State ไม่ใช่ Memory ที่อยู่ภายใน LLM

---

# 7. Parallel Lists

การใช้:

```python
todos = []
completed = []
```

เรียกว่า Parallel Lists เพราะข้อมูลเชื่อมกันด้วย Index

## ความเสี่ยง

หากขนาดไม่ตรงกัน:

```python
todos = ["A", "B", "C"]
completed = [True, False]
```

การเข้าถึง:

```python
completed[2]
```

จะเกิด `IndexError`

โครงสร้างที่แข็งแรงกว่า:

```python
todos = [
    {
        "description": "A",
        "completed": True
    },
    {
        "description": "B",
        "completed": False
    }
]
```

อย่างไรก็ตาม Lab ใช้ Parallel Lists เพื่อให้แนวคิด State Management เข้าใจง่าย

---

# 8. `get_todo_report()`

Function นี้อ่านสถานะ Todo และสร้างรายงาน:

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

หน้าที่:

```text
อ่าน Todo
ตรวจสถานะ
สร้างข้อความรายงาน
แสดงผล
คืน State ให้ Agent Observe
```

---

# 9. `enumerate()`

```python
for index, todo in enumerate(todos):
```

ให้ทั้ง:

```text
index
ข้อมูล Todo
```

Python Index เริ่มจาก `0` แต่หมายเลขสำหรับผู้ใช้เริ่มจาก `1` จึงใช้:

```python
index + 1
```

ตัวอย่าง:

```text
index 0 → Todo #1
index 1 → Todo #2
index 2 → Todo #3
```

---

# 10. Rich Console Output

Lab ใช้ Rich markup เช่น:

```text
[green]
[strike]
```

เพื่อแสดง Todo ที่เสร็จแล้วเป็นสีเขียวและขีดฆ่า

```python
"[green][strike]Finish task[/strike][/green]"
```

Function `show()` มี Fallback:

```python
def show(text):
    try:
        Console().print(text)
    except Exception:
        print(text)
```

Rich เป็น Presentation Layer ไม่ได้เปลี่ยน Logic ของ Agent

---

# 11. Tool: `create_todos`

```python
def create_todos(descriptions: list[str]) -> str:
    todos.extend(descriptions)
    completed.extend([False] * len(descriptions))
    return get_todo_report()
```

หน้าที่:

```text
รับรายการงาน
→ เพิ่มลง Todo State
→ สร้างสถานะ False
→ คืนรายงานทั้งหมด
```

ตัวอย่าง Input:

```json
{
  "descriptions": [
    "Estimate the distance",
    "Calculate travel times",
    "Determine the final time"
  ]
}
```

---

# 12. `append()` และ `extend()`

## `append()`

```python
todos.append(descriptions)
```

เพิ่ม List ทั้งก้อนเป็นสมาชิกหนึ่งตัว:

```python
[
    [
        "Task A",
        "Task B"
    ]
]
```

## `extend()`

```python
todos.extend(descriptions)
```

เพิ่มสมาชิกแต่ละตัว:

```python
[
    "Task A",
    "Task B"
]
```

Lab ต้องการ Todo แยกเป็นรายการ จึงใช้ `extend()`

---

# 13. Creating Completion State

```python
completed.extend([False] * len(descriptions))
```

หากมีสาม Tasks:

```python
[False] * 3
```

ได้:

```python
[False, False, False]
```

Todo ทุกตัวจึงเริ่มในสถานะ Pending

---

# 14. Tool: `mark_complete`

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

หน้าที่:

```text
รับหมายเลข Todo
ตรวจว่ามีอยู่จริง
เปลี่ยนสถานะเป็น True
แสดง Completion Notes
คืน Todo Report
```

---

# 15. Human Index กับ Python Index

Agent เห็น:

```text
Todo #1
```

แต่ Python ใช้:

```python
completed[0]
```

จึงต้องแปลง:

```python
completed[index - 1] = True
```

ตัวอย่าง:

```text
Todo #1 → completed[0]
Todo #2 → completed[1]
Todo #3 → completed[2]
```

---

# 16. Planning Tool ไม่ใช่ Execution Tool

นี่คือประเด็นสำคัญที่สุดของ Lab 5

Tools ที่มีคือ:

```text
create_todos
mark_complete
```

ทั้งสองเป็น Planning และ Tracking Tools

ไม่มี Tool สำหรับ:

```text
ค้นหาข้อมูล
คำนวณ
รัน Python
เปิดเว็บไซต์
ตรวจสอบคำตอบ
```

ดังนั้นเมื่อ Agent เรียก:

```python
mark_complete(
    index=2,
    completion_notes="The calculation is complete."
)
```

Tool ไม่ได้คำนวณอะไร

ลำดับจริงคือ:

```text
LLM คิดหรือสร้างผลลัพธ์
        ↓
LLM บอกว่างานเสร็จแล้ว
        ↓
Tool เปลี่ยนสถานะเป็น Completed
```

---

# 17. Claim of Completion

การ Mark Task ว่าเสร็จหมายความเพียงว่า:

```text
Agent ระบุว่า Task เสร็จ
```

ไม่ได้พิสูจน์ว่า:

```text
Task ถูก Execute จริง
ผลลัพธ์ถูกต้อง
ข้อสมมติฐานสมเหตุสมผล
ข้อกำหนดทั้งหมดได้รับการทำตาม
```

ดังนั้น:

```text
Marked Complete
≠
Verified Complete
```

---

# 18. Planning, Execution และ Verification

ระบบ Agent ที่แข็งแรงควรแยกสามชั้น:

```text
Planning
→ กำหนดสิ่งที่ต้องทำ

Execution
→ ทำงานหรือ Action จริง

Verification
→ ตรวจว่าผลถูกต้อง
```

ตัวอย่าง Tools:

| ประเภท     | ตัวอย่าง                                 |
| ---------- | ---------------------------------------- |
| Planning   | `create_todos`                           |
| Tracking   | `mark_complete`                          |
| Execution  | Calculator, Web Search, Python Runner    |
| Validation | Unit Test, Constraint Checker, Evaluator |

Agent ใน Lab นี้มี Planning และ Tracking เป็นหลัก แต่ไม่มี Execution หรือ Verification Tool ที่เป็นอิสระ

---

# 19. Tool Schema: `create_todos`

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

จุดสำคัญคือ Argument เป็น Array:

```json
{
  "descriptions": [
    "Task A",
    "Task B"
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

หมายความว่าสมาชิกทุกตัวต้องเป็น String

---

# 20. Tool Schema: `mark_complete`

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

ตัวอย่างที่ถูกต้อง:

```json
{
  "index": 1,
  "completion_notes": "Estimated the distance."
}
```

---

# 21. Tool Handler

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

ลำดับ:

```text
อ่านชื่อ Tool
→ Parse Arguments
→ ค้นหา Function
→ Execute Function
→ สร้าง Tool Result
```

---

# 22. Tool Registry Improvement

จาก Lab 4 ทราบว่า `globals()` เปิดกว้างเกินไป

ควรใช้ Explicit Tool Map:

```python
TOOL_MAP = {
    "create_todos": create_todos,
    "mark_complete": mark_complete
}
```

Dispatch:

```python
tool = TOOL_MAP.get(tool_name)

if tool is None:
    result = {
        "success": False,
        "error": f"Unknown tool: {tool_name}"
    }
else:
    result = tool(**arguments)
```

ข้อดี:

```text
เป็น Allowlist
ลด Attack Surface
อ่านง่าย
ควบคุม Tool ที่อนุญาต
```

---

# 23. Agent Loop

```python
def loop(messages):
    done = False

    while not done:
        response = openai.chat.completions.create(
            model="...",
            messages=messages,
            tools=tools
        )

        choice = response.choices[0]

        if choice.finish_reason == "tool_calls":
            message = choice.message
            results = handle_tool_calls(
                message.tool_calls
            )

            messages.append(message)
            messages.extend(results)

        else:
            done = True
            show(choice.message.content)
```

หัวใจคือ:

```text
Call Model
→ Check Decision
→ Execute Tools
→ Append Results
→ Call Model Again
```

---

# 24. Agent Loop Execution

## Round 1: Planning

```text
User provides goal
→ Model calls create_todos
→ Todo State is created
```

## Round 2: First Task

```text
Model observes Todo State
→ Works on Task 1
→ Calls mark_complete
```

## Round 3: Next Task

```text
Model observes updated State
→ Works on Task 2
→ Calls mark_complete
```

## Final Round

```text
All required work appears complete
→ Model stops requesting Tools
→ Final Answer
```

---

# 25. `for` กับ `while`

## `for tool_call in tool_calls`

จัดการหลาย Tool Calls ภายใน Model Response เดียว

```text
หนึ่ง Model Round
→ หลาย Tool Calls
```

## `while not done`

จัดการหลายรอบการตัดสินใจ

```text
Model Round 1
→ Tool Results
→ Model Round 2
→ Tool Results
→ Model Round 3
```

ดังนั้น:

```text
for
= หลาย Tools ในรอบเดียว

while
= หลายรอบของ Agent
```

---

# 26. System Prompt

System Prompt กำหนดให้ Agent:

```text
สร้าง Todo Plan
ทำงานตามลำดับ
ใช้ Tool ติดตามความคืบหน้า
สร้าง Estimate เมื่อข้อมูลขาด
ไม่ถามผู้ใช้เพิ่มเติม
ตอบเมื่อทำงานเสร็จ
```

จุดสำคัญคือการกำหนดวิธีจัดการ Missing Information

แทนการซ่อน Assumption ระบบควรเพิ่ม Todo เช่น:

```text
Estimate missing distance
```

ทำให้ Assumption กลายเป็นขั้นตอนที่มองเห็นได้

---

# 27. Train Problem

โจทย์รถไฟมีข้อมูล:

```text
Train A:
ออกเวลา 2:00 pm
ความเร็ว 60 mph

Train B:
ออกเวลา 3:00 pm
ความเร็ว 80 mph
```

ข้อมูลที่ขาด:

```text
ระยะทางระหว่างเมือง
```

ให้ระยะทางเป็น `D`

รถไฟ A เดินทางก่อนหนึ่งชั่วโมง:

```text
60 miles
```

ระยะทางที่เหลือเวลา 3:00 pm:

```text
D - 60
```

Combined Speed:

```text
60 + 80 = 140 mph
```

เวลาหลัง 3:00 pm:

```text
(D - 60) / 140 ชั่วโมง
```

เวลาพบกัน:

```text
3:00 pm
+
(D - 60) / 140 ชั่วโมง
```

Agent ต้องประมาณ `D` และระบุ Assumption ให้ชัดเจน

---

# 28. External State และ Observability

Todo Report ทำให้มนุษย์เห็น:

```text
งานใดถูกสร้าง
งานใดเสร็จแล้ว
Agent กำลังทำงานลำดับใด
มีขั้นตอนใดเหลืออยู่
```

นี่คือ Observability ระดับพื้นฐาน

แต่:

```text
Visible Progress
≠
Correct Progress
```

การมองเห็นว่า Agent Mark Task แล้ว ไม่ได้ยืนยันว่าผลถูกต้อง

---

# 29. State Classification

Todo State เป็น:

```text
In-memory Application State
```

ไม่ใช่:

```text
Model Memory
Long-term Memory
Persistent Database
Cross-session Memory
```

เมื่อ Restart Kernel หรือ Process:

```text
State หาย
```

---

# 30. Global State Risk

ตัวแปร:

```python
todos = []
completed = []
```

เป็น Global State

หากมีผู้ใช้หลายคน:

```text
User A Todos
อาจปะปนกับ
User B Todos
```

ระบบจริงควรแยก State ตาม:

```text
Session ID
User ID
Task ID
Run ID
```

และควรใช้ State Object หรือ Persistent Store

---

# 31. Risks Identified

## 31.1 Infinite Loop

```python
while not done:
```

ไม่มี Limit อาจวนไม่สิ้นสุด

ควรมี:

```python
MAX_STEPS = 10
```

---

## 31.2 Completion Without Execution

Agent อาจ Mark Complete โดยไม่ทำงานจริง

---

## 31.3 No Verification

ไม่มี Validator อิสระเพื่อตรวจคำตอบ

---

## 31.4 Global State

หลาย Sessions อาจใช้ Todo เดียวกัน

---

## 31.5 Invalid Tool Arguments

อาจเกิด:

```text
JSON ผิด
Index ผิด
Field ขาด
Type ผิด
Unknown Tool
```

---

## 31.6 Tool Handler Error

ไม่มี Try/Except รอบการ Parse และ Execute

---

## 31.7 No Return Value

Loop แสดง Final Answer แต่ไม่ Return ทำให้ Test และนำไปใช้ต่อยาก

---

## 31.8 Unbounded Cost

ไม่มีข้อจำกัดด้าน:

```text
Model Calls
Tokens
Execution Time
Tool Calls
```

---

# 32. Stronger Agent Loop

```python
def loop(
    messages,
    max_steps: int = 10
) -> str:
    for step in range(max_steps):
        response = call_model(
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
        "Agent stopped because the "
        "maximum step limit was reached."
    )
```

สิ่งที่เพิ่ม:

```text
Maximum Steps
Return Final Answer
Error Handling
None Safety
Controlled Termination
```

---

# 33. Production Improvements

## Structured Task State

```python
tasks = [
    {
        "id": 1,
        "description": "Estimate distance",
        "status": "pending",
        "result": None,
        "evidence": None
    }
]
```

สถานะอาจมี:

```text
pending
in_progress
completed
failed
blocked
needs_review
```

---

## Execution Evidence

ก่อน Mark Complete ควรมี:

```text
Result
Evidence
Tool output
Validation result
```

ตัวอย่าง:

```json
{
  "task_id": 2,
  "status": "completed",
  "result": "2.35 hours",
  "evidence": {
    "calculator_output": "2.35"
  }
}
```

---

## Verification Gate

```text
Agent claims completion
        ↓
Validator checks output
        ├── Pass → Completed
        └── Fail → Retry or Failed
```

---

## Persistent State

เก็บ State ใน:

```text
Database
Checkpoint Store
File
Redis
Workflow State
```

เพื่อรองรับ Resume หลัง Process หยุด

---

## Retry Policy

```text
Tool fails
→ Retry under policy
→ Escalate
→ Mark Task failed
```

---

## Human-in-the-Loop

สำหรับ Tasks ที่เสี่ยง:

```text
Agent proposes completion
→ Human reviews evidence
→ Approve or reject
```

---

# 34. Patterns Learned

## Plan-and-Execute Pattern

```text
Plan
→ Execute Step
→ Update State
→ Continue
```

## Externalized Working Memory

```text
Task State อยู่นอก LLM
และถูกส่งกลับเป็น Observation
```

## Progress Tracking Pattern

```text
Pending
→ In Progress
→ Completed
```

## Goal-Directed Agent Loop

```text
Agent ทำงานต่อ
จนกว่าจะถึง Final Answer
หรือเงื่อนไขหยุด
```

## Iterative Tool-Using Agent

```text
Decide
→ Act
→ Observe
→ Decide Again
```

---

# 35. Misconceptions Corrected

## ความเข้าใจคลาดเคลื่อนที่ 1

> Todo Tool ทำงานใน Todo ให้เสร็จ

**ข้อเท็จจริง:**
Todo Tool สร้างและอัปเดตสถานะ แต่ไม่ได้ Execute เนื้อหาของงาน

---

## ความเข้าใจคลาดเคลื่อนที่ 2

> Mark Complete แปลว่างานถูกต้อง

**ข้อเท็จจริง:**
เป็นเพียง Claim ของ Agent ต้องมี Verification แยก

---

## ความเข้าใจคลาดเคลื่อนที่ 3

> Todo State เป็น Memory ของ LLM

**ข้อเท็จจริง:**
เป็น Application State ที่อยู่นอกโมเดล

---

## ความเข้าใจคลาดเคลื่อนที่ 4

> Agent Loop ที่มี Todo พร้อมใช้งาน Production

**ข้อเท็จจริง:**
ยังขาด Limits, Persistence, Validation, Retry, Security และ Session Isolation

---

## ความเข้าใจคลาดเคลื่อนที่ 5

> Planning เท่ากับ Execution

**ข้อเท็จจริง:**
Planning ระบุสิ่งที่ต้องทำ ส่วน Execution ทำงานจริง และ Verification ตรวจผล

---

# 36. Connection to Lab 1

Lab 1:

```text
Output A
→ Input B
```

Lab 5:

```text
Model Tool Call
→ State Update
→ New Observation
→ Next Model Decision
```

Agent Loop เป็น Chaining ที่ทำซ้ำแบบ Dynamic

---

# 37. Connection to Lab 2

Lab 2 สอนว่า Evaluator ไม่ใช่ Ground Truth

ใน Lab 5:

```text
Agent บอกว่า Todo เสร็จ
ก็ไม่ใช่ Ground Truth เช่นกัน
```

ต้องเสริม Independent Validation

---

# 38. Connection to Lab 3

Lab 3:

```text
LLM
→ Tool Call
→ Tool Result
→ LLM
```

Lab 5 เพิ่ม:

```text
External Task State
+ Planning
+ Progress Tracking
```

---

# 39. Connection to Lab 4

Lab 4 สอน:

```text
Multiple Tools
Tool Registry
Application Authority
```

Lab 5 ใช้หลักเดียวกันกับ:

```text
create_todos
mark_complete
```

และควรใช้ Explicit `TOOL_MAP` แทน `globals()`

---

# 40. Week 1 Complete Mental Model

```text
User provides a goal
        ↓
Application builds context
        ↓
LLM observes messages and state
        ↓
LLM decides:
   ├── Create a plan
   ├── Request a tool
   ├── Update progress
   └── Give final answer
        ↓
Application validates the request
        ↓
Application executes the tool
        ↓
Tool result becomes observation
        ↓
State is updated
        ↓
Loop continues under controls
```

Agent จึงไม่ใช่โมเดลชนิดใหม่ แต่เป็นระบบประกอบด้วย:

```text
LLM
+ Context
+ Tools
+ State
+ Application Execution
+ Observation
+ Repeated Loop
+ Termination Conditions
+ Safety Controls
```

---

# 41. Final Lessons

## Lesson 1

Agent Loop ที่เรียบง่ายสามารถสร้างพฤติกรรมหลายขั้นได้

## Lesson 2

Todo List ช่วย Externalize Planning และ Working State

## Lesson 3

Planning Tool ไม่ได้ Execute งานจริง

## Lesson 4

Completion State เป็น Claim ไม่ใช่ Proof

## Lesson 5

Agent ที่น่าเชื่อถือต้องแยก Planning, Execution และ Verification

## Lesson 6

External State เพิ่ม Observability แต่ไม่ได้รับประกัน Correctness

## Lesson 7

Agent Loop ต้องมี Maximum Steps, Error Handling และ Cost Limits

## Lesson 8

Global State ไม่เหมาะกับ Multi-user Application

## Lesson 9

Explicit Tool Registry ควรถูกใช้แทน `globals()`

## Lesson 10

Framework ในบทต่อไปจะห่อหุ้ม Tool Loop, State, Dispatch และ Lifecycle Management เหล่านี้

---

# 42. Memory Summary

```text
Lab 5 แสดง Plan-and-Execute Agent
ผ่าน Todo State และ Agent Loop

Tools:
create_todos
mark_complete

Todo State เป็น External Working State
ไม่ใช่ Model Memory หรือ Long-term Memory

create_todos สร้างแผน
mark_complete อัปเดตสถานะ

mark_complete ไม่ได้ Execute งานจริง

Marked Complete
ไม่เท่ากับ Verified Complete

Planning
Execution
Verification
ต้องแยกจากกัน

Agent Loop:
Observe
→ Decide
→ Tool
→ Result
→ State Update
→ Decide Again

for loop:
หลาย Tool Calls ในหนึ่งรอบ

while loop:
หลายรอบการตัดสินใจ

Global State มีความเสี่ยงต่อหลาย Sessions

Production Agent ต้องเพิ่ม:
Maximum Steps
Tool Allowlist
Validation
Error Handling
Persistent State
Verification
Retry Policy
Human Approval
Observability
Cost Control
```

---

# 43. Week 1 Completion Summary

```text
Lab 1
LLM Calls และ Chained Workflow

Lab 2
Multi-Model Evaluation และ LLM-as-a-Judge

Lab 3
Tool Calling และ Agent Loop

Lab 4
Multiple Tools, Tool Registry และ Deployment

Lab 5
Planning, External Working State และ Plan-and-Execute
```

แก่นรวมของ Week 1:

```text
Agent
=
LLM-controlled decision loop
+ Context
+ Tools
+ External State
+ Application-controlled execution
+ Repeated observations
+ Controlled termination
```

คำถามสำคัญก่อนเข้าสู่ Week 2 คือ:

> เมื่อ Agent Loop, Tool Dispatch, State, Tracing และ Error Handling ซับซ้อนขึ้น เราควรใช้ Framework ช่วยลดภาระส่วนใด โดยไม่สูญเสียการควบคุมและความสามารถในการตรวจสอบระบบ?
