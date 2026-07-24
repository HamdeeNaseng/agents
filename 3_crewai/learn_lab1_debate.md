# Week 3 — Lab 1: CrewAI Debate

ก่อนเริ่ม มีการเปลี่ยนแปลงจาก Course เวอร์ชันเดิมที่ต้องแก้ให้ถูกก่อน:

```text
โฟลเดอร์ปัจจุบัน:
3_crewai/

Reference implementation:
3_crewai/reference/debate/

พื้นที่สำหรับทำแบบฝึกหัด:
3_crewai/coursework/
```

Repository รุ่น Summer 2026 กำหนด CrewAI `1.14.4` และแนะนำให้ศึกษาโปรเจกต์ใน `reference/` แต่สร้างหรือแก้เวอร์ชันของเราใน `coursework/` เพื่อไม่ทำ Reference ต้นฉบับเสียหาย. ([GitHub][1])

---

## เป้าหมายของ Lab

เราจะสร้างทีมอภิปรายที่รับหัวข้อหนึ่งหรือ **motion** แล้วทำงานสามขั้น:

```text
Motion
  ↓
Propose
สร้างเหตุผลสนับสนุน
  ↓
Oppose
สร้างเหตุผลคัดค้าน
  ↓
Decide
Judge เลือกฝ่ายที่โน้มน้าวใจกว่า
```

โปรเจกต์มี Agent จริงสองประเภท:

```text
Debater Agent
├── Propose Task
└── Oppose Task

Judge Agent
└── Decide Task
```

จุดสำคัญคือ **Debater Agent ตัวเดียวทำงานทั้งฝ่ายสนับสนุนและฝ่ายคัดค้าน** โดย Task เป็นตัวกำหนดว่าในรอบนั้น Agent ต้องยืนอยู่ฝ่ายใด. ([GitHub][2])

---

# Learning Objectives

เมื่อจบ Lab นี้ คุณควรอธิบายได้ว่า:

1. `Agent`, `Task`, `Crew` และ `Process` ต่างกันอย่างไร
2. `role`, `goal` และ `backstory` มีผลต่อ Agent อย่างไร
3. `description` กับ `expected_output` ของ Task ต่างกันอย่างไร
4. YAML Configuration ถูกเชื่อมกับ Python Code อย่างไร
5. Decorators `@CrewBase`, `@agent`, `@task` และ `@crew` ทำอะไร
6. `{motion}` ถูกแทนค่าจาก Runtime Input อย่างไร
7. `Process.sequential` ควบคุมลำดับงานอย่างไร
8. `kickoff()` เริ่ม Crew Execution อย่างไร
9. ทำไม Judge จึงเป็น Evaluator แต่ไม่ใช่ Ground Truth
10. ทำไม Workflow นี้ยังไม่ใช่การ Debate หลายรอบอย่างแท้จริง

---

# 1. โครงสร้างโปรเจกต์

```text
3_crewai/reference/debate/
├── knowledge/
├── output/
├── src/
│   └── debate/
│       ├── config/
│       │   ├── agents.yaml
│       │   └── tasks.yaml
│       ├── tools/
│       ├── crew.py
│       ├── main.py
│       └── __init__.py
├── pyproject.toml
├── uv.lock
└── README.md
```

ไฟล์สำคัญสี่ตัวคือ:

```text
agents.yaml
กำหนดว่าใครทำงาน

tasks.yaml
กำหนดว่าต้องทำอะไร

crew.py
ประกอบ Agents, Tasks และ Process

main.py
รับ Runtime Input และเริ่ม Crew
```

โปรเจกต์ปัจจุบันใช้โครงสร้าง Python/YAML แบบ Classic CrewAI และมีคำสั่ง `crewai run` สำหรับเริ่มระบบ. ([GitHub][3])

---

# 2. Mental Model ของ CrewAI

```text
Agent
= สมาชิกในทีม

Task
= ใบมอบหมายงาน

Crew
= ทีมและรายการงานทั้งหมด

Process
= กฎว่าทีมทำงานตามลำดับอย่างไร
```

CrewAI อธิบาย Agent ว่าเป็นหน่วยเฉพาะทางที่ทำงานตาม `role` และ `goal` สามารถใช้ Tools และร่วมมือกับ Agent อื่นได้ ส่วน Task คือ Assignment ที่ระบุงาน Agent ผู้รับผิดชอบ ผลลัพธ์ที่คาดหวัง และตัวเลือกอื่น ๆ. ([CrewAI Documentation][4])

---

# 3. `agents.yaml`

โปรเจกต์กำหนด Agents สองตัว:

```yaml
debater:
  role: >
    A compelling debater

  goal: >
    Present a clear argument either in favor of
    or against the motion. The motion is: {motion}.
    You will be successful if a judge agrees
    with your argument.

  backstory: >
    You're an experienced debator with a knack
    for giving concise but convincing arguments.
    The motion is: {motion}

  llm: openai/gpt-5.4-mini
```

และ:

```yaml
judge:
  role: >
    Decide the winner of the debate based
    on the arguments presented

  goal: >
    Given arguments for and against this motion:
    {motion}, decide which side is more convincing,
    based purely on the arguments presented.

  backstory: >
    You are a fair judge with a reputation for
    weighing up arguments without factoring in
    your own views, and making a decision based
    purely on the merits of the argument.
    The motion is: {motion}

  llm: openai/gpt-5.4-mini
```

ทั้ง Debater และ Judge ใช้โมเดลเดียวกันคือ `openai/gpt-5.4-mini` แต่ได้รับ Role, Goal และ Backstory ต่างกัน. ([GitHub][2])

---

## `role`

บอกว่า Agent เป็นใครและเชี่ยวชาญด้านใด:

```text
Debater:
A compelling debater

Judge:
ผู้ตัดสินข้อโต้แย้ง
```

`role` เป็นกรอบความรับผิดชอบ ไม่ใช่ Task ปัจจุบัน

---

## `goal`

บอกว่า Agent ต้องพยายามบรรลุอะไร

Debater:

```text
สร้างข้อโต้แย้งที่ทำให้ Judge เห็นด้วย
```

Judge:

```text
เลือกฝ่ายที่โน้มน้าวใจกว่า
จากข้อโต้แย้งที่ได้รับ
```

Goal ช่วยให้ Agent ตัดสินใจภายใต้สถานการณ์ที่ Prompt ไม่ได้ระบุรายละเอียดทุกขั้น

---

## `backstory`

Backstory ให้บริบทด้านประสบการณ์และลักษณะการทำงาน:

```text
Debater
มีประสบการณ์สร้างข้อโต้แย้งที่กระชับและน่าเชื่อถือ

Judge
ยุติธรรมและพยายามไม่นำความคิดเห็นส่วนตัวมาตัดสิน
```

Backstory ไม่ใช่ข้อมูลจริงเกี่ยวกับโมเดล แต่เป็น Prompt Context ที่ใช้ชี้นำพฤติกรรม

CrewAI ระบุว่า `role` กำหนดหน้าที่, `goal` กำหนดวัตถุประสงค์ และ `backstory` เพิ่มบริบทกับบุคลิกของ Agent. ([CrewAI Documentation][4])

---

# 4. `{motion}` คือ Runtime Placeholder

ใน YAML มี:

```text
{motion}
```

ค่านี้ยังไม่มีตอนเขียน Configuration แต่จะถูกแทนเมื่อเริ่ม Crew เช่น:

```python
inputs = {
    "motion": "AI agents should make hiring decisions"
}
```

หลัง Interpolation Agent จะเห็น Prompt ประมาณ:

```text
The motion is:
AI agents should make hiring decisions
```

ดังนั้น Key ใน `inputs` ต้องตรงกับชื่อ Placeholder:

```python
{"motion": "..."}  # ถูก
```

ไม่ใช่:

```python
{"topic": "..."}   # ไม่ตรงกับ {motion}
```

ในโปรเจกต์ปัจจุบัน `main.py` รับ Motion จากผู้ใช้แล้วส่ง Dictionary ที่มี Key ชื่อ `motion` เข้า `kickoff()`. ([GitHub][5])

---

# 5. `tasks.yaml`

## Task 1: `propose`

```yaml
propose:
  description: >
    You are proposing the motion: {motion}.
    Come up with a clear argument in favor
    of the motion.
    Be very convincing.

  expected_output: >
    Your clear argument in favor of the motion,
    in a concise manner.

  agent: debater

  output_file: output/propose.md
```

หน้าที่:

```text
รับ Motion
→ สร้างข้อสนับสนุน
→ บันทึกใน output/propose.md
```

---

## Task 2: `oppose`

```yaml
oppose:
  description: >
    You are in opposition to the motion: {motion}.
    Come up with a clear argument against
    the motion.
    Be very convincing.

  expected_output: >
    Your clear argument against the motion,
    in a concise manner.

  agent: debater

  output_file: output/oppose.md
```

Task นี้ใช้ `debater` ตัวเดิม แต่เปลี่ยน Assignment เป็นการคัดค้าน

```text
Agent เดิม
+ Task ใหม่
=
พฤติกรรมในมุมมองใหม่
```

---

## Task 3: `decide`

```yaml
decide:
  description: >
    Review the arguments presented by the
    debaters and decide which side is more
    convincing.

  expected_output: >
    Your decision on which side is more
    convincing, and why.

  agent: judge

  output_file: output/decide.md
```

Judge ทำงานหลัง Propose และ Oppose เพื่อเลือกฝ่ายที่มีข้อโต้แย้งน่าเชื่อถือกว่า. ([GitHub][6])

---

# 6. `description` กับ `expected_output`

สอง Field นี้ไม่เหมือนกัน

## `description`

บอกว่า Agent ต้องทำอะไร:

```text
สร้างเหตุผลสนับสนุน Motion
```

## `expected_output`

บอกว่าผลสำเร็จควรมีลักษณะอย่างไร:

```text
ข้อโต้แย้งสนับสนุนที่ชัดเจน กระชับ
และโน้มน้าวใจ
```

Mental Model:

```text
description
= ใบสั่งงาน

expected_output
= เกณฑ์อธิบายว่างานที่เสร็จควรหน้าตาอย่างไร
```

CrewAI รองรับการตรวจ Output ที่แข็งแรงขึ้น เช่น Pydantic Output และ Task Guardrail แต่ Lab นี้ใช้เพียงข้อความ `expected_output` จึงเป็น Prompt Guidance ไม่ใช่ Runtime Validation ที่เข้มงวด. ([CrewAI Documentation][7])

---

# 7. ทำไม Agent เดียวทำสองฝ่ายได้

ใน Lab นี้:

```yaml
propose:
  agent: debater

oppose:
  agent: debater
```

จึงไม่ได้มี Proponent Agent และ Opponent Agent แยกคนกัน

สิ่งที่เปลี่ยนคือ Task:

```text
Propose Task
บอกให้สนับสนุน

Oppose Task
บอกให้คัดค้าน
```

นี่สอนแนวคิดสำคัญว่า:

> Agent กำหนดความสามารถและลักษณะของผู้ทำงาน ส่วน Task กำหนด Assignment ที่ผู้ทำงานต้องรับผิดชอบในรอบนั้น

แต่การใช้ Agent และโมเดลเดียวกันทั้งสองฝ่ายอาจทำให้ Style, สมมติฐาน และ Bias ของข้อโต้แย้งทั้งสองด้านมีความสัมพันธ์กันสูง

---

# 8. `crew.py`

ส่วน Import:

```python
from crewai import Agent, Crew, Process, Task
from crewai.project import CrewBase, agent, crew, task
```

Class หลัก:

```python
@CrewBase
class Debate:
    """Debate crew"""

    agents: list[BaseAgent]
    tasks: list[Task]
```

`@CrewBase` ช่วยโหลด Classic YAML Configuration และจัดการ Methods ที่ถูกตกแต่งด้วย `@agent`, `@task` และ `@crew`. ([GitHub][8])

---

## `@agent`

```python
@agent
def debater(self) -> Agent:
    return Agent(
        config=self.agents_config["debater"],
        verbose=True
    )
```

Flow:

```text
agents.yaml["debater"]
        ↓
Agent(config=...)
        ↓
Debater Agent Object
```

Judge ทำแบบเดียวกัน:

```python
@agent
def judge(self) -> Agent:
    return Agent(
        config=self.agents_config["judge"],
        verbose=True
    )
```

Decorator `@agent` บอก CrewAI ว่า Method นี้สร้างสมาชิก Agent ของ Crew. ([GitHub][8])

---

## `@task`

```python
@task
def propose(self) -> Task:
    return Task(
        config=self.tasks_config["propose"]
    )
```

เช่นเดียวกันกับ:

```python
@task
def oppose(self) -> Task:
    return Task(
        config=self.tasks_config["oppose"]
    )

@task
def decide(self) -> Task:
    return Task(
        config=self.tasks_config["decide"]
    )
```

Flow:

```text
tasks.yaml["propose"]
        ↓
Task(config=...)
        ↓
Propose Task Object
```

---

## `@crew`

```python
@crew
def crew(self) -> Crew:
    return Crew(
        agents=self.agents,
        tasks=self.tasks,
        process=Process.sequential,
        verbose=True,
        tracing=True
    )
```

Crew ประกอบด้วย:

```text
Agents:
- debater
- judge

Tasks:
- propose
- oppose
- decide

Process:
- sequential
```

`self.agents` และ `self.tasks` ถูกสร้างจาก Methods ที่มี Decorators โดยอัตโนมัติ. ([GitHub][8])

---

# 9. `Process.sequential`

```python
process=Process.sequential
```

ทำให้ Tasks ทำงานตามลำดับที่ประกาศ:

```text
propose
  ↓
oppose
  ↓
decide
```

Sequential Process รับประกันลำดับ Execution แต่ไม่ได้รับประกันว่า:

```text
ข้อโต้แย้งถูกต้อง
Judge ยุติธรรมจริง
ข้อมูลเป็นความจริง
การตัดสินไม่มี Bias
```

CrewAI รองรับทั้ง Sequential Process ซึ่งรัน Tasks ตามลำดับ และ Hierarchical Process ซึ่งใช้ Manager ควบคุมการมอบหมายงาน แต่ Debate Lab เลือก Sequential. ([CrewAI Documentation][7])

---

# 10. `verbose=True` และ `tracing=True`

Agents และ Crew เปิด:

```python
verbose=True
tracing=True
```

`verbose=True` ทำให้เห็นรายละเอียดการทำงานใน Terminal มากขึ้น ส่วน `tracing=True` เปิดการเก็บ Trace ของ Crew Execution. ([GitHub][8])

สิ่งที่ควรสังเกตตอนรัน:

```text
Task ใดกำลังเริ่ม
Agent ใดกำลังทำงาน
Prompt ถูกประกอบอย่างไร
Output แต่ละ Task คืออะไร
จำนวน Model Calls
ลำดับ Task Execution
```

Trace และ Verbose Output ช่วย Observability แต่ไม่ได้ตรวจ Correctness โดยอัตโนมัติ

---

# 11. `main.py`

จุดเริ่มต้นของ Crew:

```python
def run():
    motion = input("Enter the motion: ")

    inputs = {
        "motion": motion,
    }

    try:
        Debate().crew().kickoff(inputs=inputs)
    except Exception as e:
        raise Exception(
            f"An error occurred while running the crew: {e}"
        )
```

ลำดับ:

```text
1. รับ Motion จาก Terminal
2. สร้าง inputs Dictionary
3. สร้าง Debate Class
4. สร้าง Crew
5. kickoff(inputs=inputs)
6. Interpolate {motion}
7. Execute Tasks ตาม Process
```

`kickoff()` เป็น Method ที่เริ่มการทำงานของ Crew ตาม Process ที่กำหนด. ([GitHub][5])

---

# 12. วิธีรัน

ตรวจ CrewAI Version จาก Root Repository:

```powershell
uv tool list
```

Course ปัจจุบันใช้:

```text
crewai 1.14.4
```

หากยังไม่มี:

```powershell
uv tool install crewai==1.14.4
```

จากนั้นเข้าโปรเจกต์:

```powershell
cd 3_crewai\reference\debate
crewai install
crewai run
```

ระบบจะถาม:

```text
Enter the motion:
```

ตัวอย่าง:

```text
AI agents should be allowed to make
high-impact business decisions without human approval.
```

เมื่อทำงานเสร็จ ให้ตรวจ:

```text
output/propose.md
output/oppose.md
output/decide.md
```

Repository และ `pyproject.toml` ปัจจุบันกำหนด Python `>=3.10,<3.14`, ใช้ `crewai[tools]==1.14.4` และเชื่อมคำสั่งรันเข้ากับ `debate.main:run`. ([GitHub][1])

---

# 13. Flow แบบละเอียด

```text
User enters Motion
        ↓
main.run()
        ↓
inputs = {"motion": motion}
        ↓
Debate().crew()
        ↓
@CrewBase loads YAML
        ↓
Create Debater Agent
Create Judge Agent
        ↓
Create Propose Task
Create Oppose Task
Create Decide Task
        ↓
Crew(Process.sequential)
        ↓
Propose Task
        ↓
output/propose.md
        ↓
Oppose Task
        ↓
output/oppose.md
        ↓
Decide Task
        ↓
output/decide.md
```

---

# 14. Pattern ที่กำลังเรียน

## Role-based Agent Pattern

```text
Debater
Judge
```

Agents ถูกแยกตามความรับผิดชอบ

## Task Separation Pattern

```text
Propose
Oppose
Decide
```

งานถูกแยกออกจากตัว Agent

## Generator–Evaluator Pattern

```text
Debater generates arguments
        ↓
Judge evaluates arguments
```

## Sequential Pipeline

```text
Task A
→ Task B
→ Task C
```

## Artifact Output Pattern

```text
Task Result
→ Markdown File
```

---

# 15. นี่คือ “Debate” จริงหรือไม่

ในความหมายของโปรเจกต์ มันเป็น Debate Workflow เพราะมีฝ่ายสนับสนุน ฝ่ายคัดค้าน และ Judge

แต่ในเชิง Multi-round Debate มันยังไม่มี:

```text
Rebuttal
Counterargument
Cross-examination
Evidence challenge
Multiple debate rounds
Revision after feedback
```

Flow จริงคือ:

```text
Generate Pro
→ Generate Con
→ Judge
```

ไม่ใช่:

```text
Pro
→ Con rebuts Pro
→ Pro rebuts Con
→ Evidence verification
→ Judge
```

ชื่อที่แม่นยำในเชิง Architecture คือ:

```text
Pro/Con Generation
+ LLM Evaluation
```

---

# 16. Judge ไม่ใช่ Ground Truth

Judge ถูกสั่งให้เลือกฝ่ายที่ **โน้มน้าวใจกว่า**

ไม่ได้ถูกสั่งให้พิสูจน์ว่า:

```text
ฝ่ายใดถูกต้องตามข้อเท็จจริง
Evidence ใดน่าเชื่อถือ
ข้ออ้างใดมี Source
ข้อมูลใดเป็นปัจจุบัน
```

ดังนั้น Judge อาจเลือก Argument ที่:

```text
เขียนสวยกว่า
มั่นใจกว่า
ยาวกว่า
ใช้ภาษาน่าเชื่อถือกว่า
อยู่ในตำแหน่งที่ Model ชอบกว่า
```

คำตัดสินคือ LLM-as-a-Judge ไม่ใช่ข้อพิสูจน์ความจริง

---

# 17. ข้อจำกัดของ Lab

## ไม่มี Tools หรือ Evidence

Debater ไม่มี Web Search หรือ Knowledge Tool จึงสร้าง Argument จาก Model Context และ Prompt เป็นหลัก

## ใช้โมเดลเดียวกันทุก Agent

Debater และ Judge ใช้ `gpt-5.4-mini` เหมือนกัน อาจมี Bias และรูปแบบการให้เหตุผลที่สัมพันธ์กัน. ([GitHub][2])

## ไม่มี Structured Output

ผลลัพธ์เป็นข้อความและ Markdown Files จึงตรวจรูปแบบหรือคะแนนแบบ Deterministic ได้ยาก

## ไม่มี Rubric ที่ชัดเจน

Judge ถูกบอกเพียงให้เลือกฝ่ายที่น่าเชื่อถือกว่า แต่ไม่ได้มีคะแนนด้าน:

```text
Logical validity
Evidence quality
Relevance
Counterargument handling
Factual accuracy
```

## ไม่มี Human Review

คำตัดสินถูกสร้างและเขียนไฟล์ทันที

## Motion ถูกใส่ใน Prompt โดยตรง

Motion ที่มาจากผู้ใช้อาจมี Prompt Injection หรือข้อความที่พยายามเปลี่ยนคำสั่งของ Agent

---

# 18. แบบฝึกหัดที่ควรทำ

## Exercise 1 — Run ซ้ำด้วย Motion เดิม

รัน Motion เดิมสามครั้ง แล้วเปรียบเทียบ:

```text
ฝ่ายชนะเหมือนกันหรือไม่
ข้อโต้แย้งเปลี่ยนหรือไม่
Judge ใช้เกณฑ์เหมือนเดิมหรือไม่
```

เป้าหมายคือเห็นความไม่ Deterministic ของ LLM Workflow

---

## Exercise 2 — สลับลำดับ Task

ทดลองเปลี่ยน:

```text
propose → oppose → decide
```

เป็น:

```text
oppose → propose → decide
```

แล้วตรวจว่าฝ่ายชนะเปลี่ยนหรือไม่

นี่ใช้ตรวจ Position หรือ Recency Bias

---

## Exercise 3 — เพิ่ม Judge Rubric

แก้ `decide.expected_output` ให้ Judge ให้คะแนน:

```text
Logical consistency: /10
Evidence quality: /10
Relevance: /10
Persuasiveness: /10
Final winner:
Reason:
```

---

## Exercise 4 — แยก Debater เป็นสอง Agents

เปลี่ยนจาก Agent เดียว:

```text
debater
```

เป็น:

```text
proponent
opponent
judge
```

แล้วเปรียบเทียบความหลากหลายของ Argument

---

## Exercise 5 — เพิ่ม Rebuttal

เพิ่ม Tasks:

```text
propose
→ oppose
→ pro_rebuttal
→ con_rebuttal
→ decide
```

นี่จะทำให้ระบบใกล้ Multi-round Debate มากขึ้น

---

# 19. Checklist ก่อนจบ Lab

คุณควรตอบได้ว่า:

### Agent กับ Task ต่างกันอย่างไร

```text
Agent
= ผู้ทำงานและความสามารถ

Task
= งานเฉพาะที่มอบหมายในรอบนั้น
```

### ทำไม Debater ตัวเดียวอยู่ได้ทั้งสองฝ่าย

เพราะ Task Description กำหนดมุมมองปัจจุบัน

### Crew ทำอะไร

รวม Agents, Tasks และ Process เป็น Workflow เดียว

### Sequential Process ทำอะไร

กำหนดให้ Tasks ทำตามลำดับที่ประกาศ

### `{motion}` มาจากไหน

มาจาก `inputs={"motion": ...}` ตอน `kickoff()`

### `expected_output` เป็น Validator หรือไม่

ไม่ใช่ Validator ที่เข้มงวด เป็นคำอธิบายผลลัพธ์ที่ต้องการใน Prompt

### `output_file` ทำอะไร

บันทึก Task Output ลง Markdown File

### Judge พิสูจน์ความจริงหรือไม่

ไม่ Judge เลือกฝ่ายที่ดูโน้มน้าวใจกว่าจาก Arguments ที่ได้รับ

---

# แก่นของ Lab 1

```text
CrewAI แยก:

ใครทำงาน
→ Agent

ทำงานอะไร
→ Task

ทำตามลำดับใด
→ Process

รวมเป็นทีมอย่างไร
→ Crew

เริ่มระบบด้วยข้อมูลอะไร
→ kickoff(inputs)
```

บทเรียนที่สำคัญที่สุดคือ:

> **Agent ไม่ควรถูกผูกติดกับงานเดียว Agent แสดงบทบาท ความสามารถ และเป้าหมายทั่วไป ส่วน Task เป็นผู้กำหนด Assignment ปัจจุบัน จากนั้น Crew และ Process เป็นผู้จัดลำดับให้คนและงานทำงานร่วมกัน**

และต้องจำอีกด้านหนึ่งว่า:

> **การมีฝ่ายสนับสนุน ฝ่ายคัดค้าน และ Judge ทำให้ระบบมองปัญหาหลายมุม แต่ยังไม่ได้รับประกันความจริง เพราะทุกฝ่ายยังเป็น LLM ที่อาจแบ่งปัน Bias และ Judge กำลังประเมินความโน้มน้าวใจ ไม่ใช่ตรวจสอบ Evidence**

[1]: https://github.com/ed-donner/agents/tree/main/3_crewai "agents/3_crewai at main · ed-donner/agents · GitHub"
[2]: https://github.com/ed-donner/agents/blob/main/3_crewai/reference/debate/src/debate/config/agents.yaml "agents/3_crewai/reference/debate/src/debate/config/agents.yaml at main · ed-donner/agents · GitHub"
[3]: https://github.com/ed-donner/agents/tree/main/3_crewai/reference/debate "agents/3_crewai/reference/debate at main · ed-donner/agents · GitHub"
[4]: https://docs.crewai.com/v1.15.2/en/concepts/agents "Agents - CrewAI"
[5]: https://github.com/ed-donner/agents/blob/main/3_crewai/reference/debate/src/debate/main.py "agents/3_crewai/reference/debate/src/debate/main.py at main · ed-donner/agents · GitHub"
[6]: https://github.com/ed-donner/agents/blob/main/3_crewai/reference/debate/src/debate/config/tasks.yaml "agents/3_crewai/reference/debate/src/debate/config/tasks.yaml at main · ed-donner/agents · GitHub"
[7]: https://docs.crewai.com/v1.15.2/en/concepts/tasks "Tasks - CrewAI"
[8]: https://github.com/ed-donner/agents/blob/main/3_crewai/reference/debate/src/debate/crew.py "agents/3_crewai/reference/debate/src/debate/crew.py at main · ed-donner/agents · GitHub"
