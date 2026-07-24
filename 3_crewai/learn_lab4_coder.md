# Week 3 — Lab 4: CrewAI Coder

โปรเจกต์:

```text
3_crewai/reference/coder/
```

Lab นี้เปลี่ยนจาก Crew ที่ค้นข้อมูลและตัดสินใจ มาเป็น Agent ที่สามารถ:

```text
รับโจทย์
→ เขียนไฟล์ Python
→ รันโค้ด
→ ตรวจผลลัพธ์
→ แก้ไขหากจำเป็น
→ สรุปสิ่งที่ทำ
```

โปรเจกต์ใช้ CrewAI `1.14.4`, Python `>=3.10,<3.14` และมีโฟลเดอร์ `sandbox/` สำหรับเก็บโค้ดที่ Agent สร้างขึ้น. ([GitHub][1])

---

## Learning Objectives

หลังจบ Lab นี้ควรอธิบายได้ว่า:

1. Coder Agent แตกต่างจาก Agent ที่เพียงสร้าง Code Block อย่างไร
2. Custom Tools ทำให้ Agent อ่าน เขียน และรันไฟล์ได้อย่างไร
3. ทำไมโปรเจกต์นี้ไม่ใช้ `allow_code_execution=True`
4. Sandbox Tool Loop ทำงานอย่างไร
5. Docker ช่วยแยก Runtime ออกจาก Python Process หลักอย่างไร
6. ทำไม Tool Cache ต้องถูกปิด
7. `subprocess.run()` จัดการ Output และ Timeout อย่างไร
8. การรันสำเร็จต่างจากการพิสูจน์ว่าโปรแกรมถูกต้องอย่างไร
9. Path Traversal, Network Access และ Resource Exhaustion เกิดขึ้นได้อย่างไร
10. Code Execution Pipeline ที่ปลอดภัยกว่าควรเพิ่มอะไรบ้าง

---

# 1. Architecture Overview

```text
Runtime Assignment
        ↓
Coder Agent
        ↓
┌─────────────────────────┐
│ Custom Sandbox Tools    │
│                         │
│ List files              │
│ Read file               │
│ Write file              │
│ Run Python file         │
└─────────────────────────┘
        ↓
sandbox/solution.py
        ↓
Docker Python Runtime
        ↓
Program stdout
        ↓
Agent evaluates result
        ↓
output/solution.md
```

โปรเจกต์นี้มีเพียง:

```text
1 Agent
1 Task
1 Crew
4 Custom Tools
```

ดังนั้นมันเป็น **Single-Agent Tool-Using Workflow** ไม่ใช่ Multi-Agent Crew แม้ยังใช้ CrewAI เป็น Runtime Container. `Process.sequential` ถูกกำหนดไว้ แต่เพราะมี Task เดียว จึงแทบไม่มีผลต่อการจัดลำดับงาน. ([GitHub][2])

---

# 2. Coder Agent

ใน `agents.yaml` มี Agent เพียงตัวเดียว:

```yaml
coder:
  role: Python Developer
  goal: >
    Use sandbox tools to complete {assignment}.
    Write a Python file, run it and check the output.
  backstory: >
    An experienced Python developer who writes
    clean and efficient code.
```

โมเดลปัจจุบันคือ:

```text
openai/gpt-5.4-mini
```

เป้าหมายไม่ได้บอกเพียงให้ “เขียนโค้ด” แต่บังคับกระบวนการเชิงพฤติกรรมว่า:

```text
เขียนไฟล์
→ รัน
→ ตรวจผล
```

อย่างไรก็ตาม ข้อความใน Goal ยังเป็น Prompt Instruction ไม่ใช่ข้อบังคับเชิงโปรแกรม หาก Agent เลือกสรุปคำตอบโดยไม่เรียก Tool ระบบยังไม่มี Deterministic Gate ที่พิสูจน์ว่าไฟล์ถูกสร้างและรันจริง. ([GitHub][3])

---

# 3. Coding Task

`tasks.yaml` กำหนด Task เดียว:

```yaml
coding_task:
  description: >
    Use sandbox tools to write and run Python code
    to achieve {assignment}.

  expected_output: >
    A summary of what you did and the final result.

  agent: coder
  output_file: output/solution.md
```

Task จึงมีผลลัพธ์สองประเภท:

```text
Executable Artifact
→ sandbox/solution.py

Human-readable Artifact
→ output/solution.md
```

สิ่งสำคัญคือ `output/solution.md` เป็นเพียงรายงานจาก Agent ไม่ใช่หลักฐานการทดสอบแบบอิสระ หาก Agent รายงานว่า “รันสำเร็จ” ระบบยังต้องตรวจ Tool Result, Exit Code และ Tests เพื่อยืนยันอีกชั้นหนึ่ง. ([GitHub][4])

---

# 4. Runtime Assignment

`main.py` กำหนดโจทย์ตัวอย่างไว้โดยตรง:

```text
คำนวณ 1,000,000 พจน์แรกของอนุกรม

1 - 1/3 + 1/5 - 1/7 + ...

แล้วคูณผลรวมด้วย 4
```

โจทย์ถูกส่งผ่าน:

```python
inputs = {
    "assignment": assignment
}
```

แล้วแทนค่า `{assignment}` ใน Agent Goal และ Task Description ก่อนเริ่ม Crew. ([GitHub][5])

Flow:

```text
main.py assignment
        ↓
inputs["assignment"]
        ↓
{assignment} interpolation
        ↓
Coder Agent Prompt
```

---

# 5. นี่ไม่ใช่ Built-in Code Execution

สิ่งที่ควรแยกให้ออกคือ โปรเจกต์นี้ไม่ได้ตั้งค่า:

```python
allow_code_execution=True
```

และไม่ได้ใช้ Built-in Code Interpreter ของ CrewAI

แต่มันมอบ **Custom Tools สี่ตัว** ให้ Agent:

```python
tools=sandbox_tools
```

ดังนั้น Agent ไม่ได้รัน Python โดยตรงจากความสามารถภายใน Framework แต่ร้องขอให้ Application เรียก Python Functions ที่ผู้พัฒนาสร้างขึ้น. ([GitHub][2])

CrewAI รุ่นเอกสารปัจจุบันระบุว่า `allow_code_execution` และ `code_execution_mode` ถูกเลิกแนะนำแล้ว และเสนอให้ใช้ Dedicated Sandbox Service สำหรับ Secure Code Execution ดังนั้นแนวทางของ Lab ที่สร้าง Runtime Boundary แยกเองจึงต่างจาก API เก่าของ CrewAI. โปรเจกต์หลักสูตรยังล็อก CrewAI `1.14.4` ขณะที่เอกสารปัจจุบันเป็น `1.15.5` จึงควรยึดโค้ดใน Repository เป็นหลักสำหรับ Lab นี้. ([CrewAI Documentation][6])

---

# 6. Sandbox Tools

ไฟล์:

```text
src/coder/tools/sandbox_tools.py
```

กำหนด Sandbox Directory ไว้ที่:

```text
coder/sandbox/
```

และสร้าง Directory ให้อัตโนมัติหากยังไม่มี. ([GitHub][7])

## Tool 1 — List Sandbox Files

```text
List Sandbox Files
```

ทำหน้าที่แสดงชื่อไฟล์ที่อยู่ตรง Root ของ `sandbox/`

ตัวอย่างผล:

```text
solution.py
test_solution.py
```

Tool นี้ไม่ได้ค้นแบบ Recursive และไม่ได้แสดง Metadata เช่น:

```text
ขนาดไฟล์
วันที่แก้ไข
Directory
Hash
```

---

## Tool 2 — Read Sandbox File

```text
Read Sandbox File
```

รับ:

```text
filename
```

แล้วคืนเนื้อหา Text ของไฟล์

Agent สามารถใช้เพื่อ:

```text
ตรวจไฟล์เดิม
อ่านโค้ดที่เพิ่งเขียน
ดูว่าแก้ไขถูกหรือไม่
```

แต่ Tool อ่านด้วย `read_text()` โดยไม่มีการจัดการ Encoding หรือ Exception ที่ละเอียด จึงอาจล้มเหลวเมื่อเจอ Binary File หรือ Text Encoding ที่ไม่รองรับ. ([GitHub][7])

---

## Tool 3 — Write Sandbox File

```text
Write Sandbox File
```

รับ:

```text
filename
content
```

แล้วเขียนทับไฟล์เดิมทันที

```text
File exists
→ Overwrite

File does not exist
→ Create
```

ไม่มี:

```text
Diff
Backup
Confirmation
Versioning
Atomic write
```

ดังนั้น Agent สามารถทำลายไฟล์เดิมใน Sandbox ได้โดยไม่ตั้งใจ. ([GitHub][7])

---

## Tool 4 — Run Sandbox Python File

Tool นี้เรียกคำสั่งเชิงแนวคิด:

```bash
docker run --rm \
  -v <sandbox>:/workspace \
  -w /workspace \
  python:3.13-slim \
  python <filename>
```

และกำหนด Timeout ที่ Python Process ฝั่ง Host ไว้ 60 วินาที. ([GitHub][7])

Flow:

```text
Agent requests run
        ↓
Python subprocess
        ↓
Docker container starts
        ↓
Sandbox mounted at /workspace
        ↓
python filename
        ↓
stdout returned to Agent
```

---

# 7. Agentic Coding Loop

เมื่อรวม Tools ทั้งหมด Coder สามารถทำ Loop:

```text
1. List files
2. Read existing code
3. Write solution.py
4. Run solution.py
5. Inspect stdout
6. Detect problem
7. Rewrite solution.py
8. Run again
9. Summarize final result
```

นี่คือ Agent Loop ที่มี **External Working State**

```text
LLM Context
+
Sandbox Files
+
Execution Output
```

ไฟล์ใน Sandbox ทำหน้าที่คล้าย Working Memory ภายนอก แต่ไม่ใช่ CrewAI Memory System

```text
Sandbox State
= persistent working artifacts

CrewAI Memory
= retrieved historical context
```

---

# 8. ทำไมปิด Tool Cache

ท้ายไฟล์ Tool มีการกำหนด:

```python
def _never_cache(...):
    return False
```

แล้วนำไปใส่ใน `cache_function` ของ Tools ทั้งหมด

เหตุผลคือ Sandbox State เปลี่ยนตลอดเวลา:

```text
ครั้งแรก:
sandbox ว่าง

หลัง Write:
มี solution.py

หลังแก้ไข:
solution.py มีเนื้อหาใหม่
```

ถ้า CrewAI Cache ผลเก่า:

```text
List Sandbox Files
→ อาจยังตอบว่า Sandbox ว่าง

Read Sandbox File
→ อาจคืนโค้ดเวอร์ชันก่อนแก้ไข
```

CrewAI รองรับ `cache_function` เพื่อควบคุมว่า Tool Result ใดควรถูกเก็บ Cache และโปรเจกต์นี้เลือกไม่ Cache ทุก Tool เพื่อป้องกัน Stale State. ([GitHub][7])

หลักสำคัญ:

```text
Caching เหมาะกับ Pure Function

Caching อันตรายกับ Mutable State
```

---

# 9. ผลลัพธ์ตัวอย่าง

Agent สร้าง:

```text
sandbox/solution.py
```

โปรแกรมวนลูป 1,000,000 พจน์ของอนุกรม แล้วคืนค่า:

```text
3.1415916535897743
```

จากนั้น Agent บันทึกรายงานที่ `output/solution.md` โดยอธิบายว่าได้เขียนและรันโปรแกรมใน Sandbox แล้ว. ([GitHub][8])

Repository ยังมี Artifact เก่าชื่อ `output/code_and_output.txt` ซึ่ง Agent ระบุว่าไม่มี Execution Tool และทำได้เพียงคาดการณ์ Output ขณะที่ Artifact รุ่นใหม่ใช้ Sandbox Tools และรายงานว่ารันจริงแล้ว สิ่งนี้แสดงว่า Output Artifacts ภายใน Repository อาจมาจากคนละ Iteration ของโปรเจกต์และไม่ควรถูกมองว่าเป็น Source of Truth เดียวกัน. ([GitHub][9])

---

# 10. การรันผ่าน Docker ให้อะไร

ข้อดีของการรันใน Container:

```text
Python Process ไม่รันตรงใน Crew Process
Environment เริ่มใหม่ทุกครั้ง
Container ถูกลบหลังจบด้วย --rm
มีเฉพาะ Sandbox Directory ที่ถูก Mount เข้าไป
ใช้ Python Version คงที่คือ 3.13-slim
```

แต่คำว่า “Docker Sandbox” ไม่ได้หมายความว่าปลอดภัยสมบูรณ์

ในคำสั่งปัจจุบัน Sandbox Directory ถูก Bind Mount แบบ Read–Write เนื่องจากไม่ได้ใช้ `:ro` ดังนั้นโค้ดใน Container สามารถสร้าง แก้ไข หรือลบไฟล์ภายใน Host Sandbox Directory ได้ Docker ระบุว่า Bind Mount ให้สิทธิ์เขียนไฟล์ Host โดย Default. ([GitHub][7])

---

# 11. ข้อจำกัดของ Run Tool

## คืนเฉพาะ stdout

Tool คืน:

```python
result.stdout
```

แต่ไม่คืน:

```text
stderr
return code
execution duration
timeout status
```

หากโปรแกรมเกิด:

```python
raise ValueError("failed")
```

Error จะอยู่ใน `stderr` แต่ Agent อาจได้รับเพียง String ว่าง ทำให้เข้าใจผิดว่าโปรแกรมไม่มี Output แทนที่จะรู้ว่าโปรแกรม Crash. ([GitHub][7])

Structured Result ที่เหมาะกว่าควรเป็น:

```python
class ExecutionResult(BaseModel):
    success: bool
    exit_code: int
    stdout: str
    stderr: str
    timed_out: bool
```

---

## Timeout ไม่เท่ากับ Resource Limit

Tool มี:

```python
timeout=60
```

จึงจำกัดเวลาที่ `subprocess.run()` รอ แต่คำสั่ง Docker ไม่มี:

```text
--memory
--cpus
--pids-limit
```

Docker ระบุว่า Container ไม่มี Resource Constraints โดย Default และสามารถใช้ทรัพยากรได้เท่าที่ Kernel Scheduler อนุญาต. ([GitHub][7])

โค้ดอันตราย เช่น:

```python
data = []
while True:
    data.append("x" * 10_000_000)
```

อาจใช้ Memory จำนวนมากก่อนถึง Timeout

---

## Network ยังไม่ได้ปิด

คำสั่งไม่มี:

```bash
--network none
```

จึงเชื่อมต่อ Default Bridge Network และมี External Network Access ตามการตั้งค่า Docker Host

Docker ระบุว่า Container ที่ไม่ได้กำหนด `--network` จะถูกเชื่อมกับ Default Bridge และ Bridge ใช้ Masquerading เพื่อให้ Container เข้าถึงเครือข่ายภายนอกได้. ([GitHub][7])

ดังนั้นโค้ดที่ Agent สร้างสามารถพยายาม:

```text
ส่ง HTTP Request
ดาวน์โหลดข้อมูล
ส่งข้อมูลออกภายนอก
สแกน Service ใน Docker Network
```

---

# 12. ช่องโหว่ Path Traversal

จากโค้ดปัจจุบัน การสร้าง Path ใช้แนวทาง:

```python
path = SANDBOX_DIR / filename
```

แต่ไม่ได้ตรวจว่า Path ที่ Resolve แล้วยังอยู่ภายใน `SANDBOX_DIR`

จึงอนุมานได้ว่า Input เช่น:

```text
../.env
../../pyproject.toml
/path/to/another/file
```

อาจทำให้ `Read Sandbox File` หรือ `Write Sandbox File` เข้าถึงไฟล์นอก Sandbox ได้ เพราะ Tools สองตัวนี้ทำงานใน Python Process ฝั่ง Host ไม่ได้ทำงานภายใน Docker Container. ([GitHub][7])

นี่เป็นความเสี่ยงร้ายแรงกว่า Code Container เพราะ Agent อาจ:

```text
อ่าน .env
อ่าน API Keys
เขียนทับ Source Code
แก้ Configuration
```

วิธีป้องกัน:

```python
def safe_path(filename: str) -> Path:
    root = SANDBOX_DIR.resolve()
    candidate = (root / filename).resolve()

    if candidate != root and root not in candidate.parents:
        raise ValueError("Path escapes sandbox")

    return candidate
```

ยังควรปฏิเสธ:

```text
Absolute paths
Symlinks ที่ชี้ออกนอก Sandbox
Directories ที่ไม่อนุญาต
File extensions ที่ไม่รองรับ
```

---

# 13. “รันผ่าน” ไม่เท่ากับ “ถูกต้อง”

โปรแกรมสามารถ:

```text
Exit code = 0
Print output
ไม่เกิด Exception
```

แต่ยังผิดเชิง Logic ได้

ตัวอย่าง:

```python
print(3.14159)
```

โปรแกรมนี้ให้ผลลัพธ์ใกล้เคียงโจทย์ แต่ไม่ได้คำนวณอนุกรมเลย

ดังนั้น:

```text
Executed Successfully
≠
Requirement Satisfied
```

ระบบต้องมี Tests ที่ตรวจทั้ง:

```text
Behavior
Algorithm
Edge cases
Performance
```

---

# 14. Expected Output ยังไม่ใช่ Quality Gate

Task ขอเพียง:

```text
สรุปสิ่งที่ทำและผลลัพธ์สุดท้าย
```

แต่ไม่มี Guardrail ตรวจว่า:

```text
มีไฟล์ Python จริงหรือไม่
Tool Run ถูกเรียกหรือไม่
Exit Code เป็น 0 หรือไม่
ผลลัพธ์ตรงกับ Expected Value หรือไม่
มี Tests หรือไม่
```

CrewAI รองรับ Task Guardrails ทั้งแบบ Function-based และ LLM-based รวมถึงกำหนดจำนวน Retry เมื่อ Validation ไม่ผ่าน แต่ Lab นี้ยังไม่ได้ใช้. ([CrewAI Documentation][10])

ตัวอย่าง Gate:

```python
def validate_solution(task_output):
    solution = SANDBOX_DIR / "solution.py"

    if not solution.exists():
        return False, "solution.py was not created"

    execution = run_validated_solution()

    if execution.exit_code != 0:
        return False, execution.stderr

    return True, task_output.raw
```

---

# 15. Safer Execution Command

สำหรับ Assignment ที่ไม่ต้องใช้อินเทอร์เน็ต คำสั่งอาจถูก Harden เป็น:

```bash
docker run --rm \
  --network none \
  --memory 256m \
  --cpus 1 \
  --pids-limit 64 \
  --read-only \
  --cap-drop ALL \
  --security-opt no-new-privileges \
  --tmpfs /tmp:size=64m \
  -v <sandbox>:/workspace:rw \
  -w /workspace \
  python:3.13-slim \
  python solution.py
```

Sandbox ยังต้อง Writable เพราะ Agent ต้องสร้างหรือแก้ไฟล์ แต่ควร Mount เฉพาะ Directory ที่จำเป็นและตรวจ Path ทุกครั้ง

สำหรับระบบที่รับ Assignment จากผู้ใช้ภายนอกหรือเป็น Multi-tenant การใช้ Local Docker Container เพียงอย่างเดียวอาจยังไม่เพียงพอ ควรใช้ Dedicated Isolated Sandbox ที่มี Network Policy, Credential Isolation, Filesystem Boundary และ Resource Quotas ตามคำแนะนำปัจจุบันของ CrewAI. ([CrewAI Documentation][6])

---

# 16. Safer Coding Pipeline

```text
Assignment
    ↓
Input Validation
    ↓
Planning
    ↓
Write Code
    ↓
Static Checks
├── Parse AST
├── Block dangerous imports
└── Check file path
    ↓
Isolated Execution
├── No network
├── CPU limit
├── Memory limit
├── Process limit
└── Timeout
    ↓
Capture ExecutionResult
├── stdout
├── stderr
├── exit code
└── duration
    ↓
Deterministic Tests
    ↓
Security Scan
    ↓
Human Review
    ↓
Publish Artifact
```

---

# 17. วิธีรัน Lab

ต้องมี:

```text
Docker Desktop หรือ Docker Engine
OPENAI_API_KEY
CrewAI CLI
```

จาก Project Root:

```powershell
cd 3_crewai\reference\coder
crewai install
docker version
crewai run
```

เนื่องจาก Tool เรียกคำสั่ง `docker run` โดยตรง Docker Daemon ต้องพร้อมใช้งานในเครื่อง. ([GitHub][7])

หลังจบให้ตรวจ:

```text
sandbox/solution.py
output/solution.md
```

---

# 18. สิ่งที่ควรสังเกตใน Trace

เมื่อรัน Lab ให้ตรวจลำดับ Tool Calls:

```text
List Sandbox Files
→ Write Sandbox File
→ Run Sandbox Python File
→ Read Sandbox File
→ Run again
```

คำถามที่ควรถาม:

```text
Agent เขียนไฟล์ก่อนรันหรือไม่
Agent ตรวจ stdout หรือไม่
เมื่อเกิด Error Agent แก้ไขหรือไม่
Agent เรียก Tool ซ้ำกี่ครั้ง
Agent สรุปตรงกับ Tool Result หรือไม่
```

Tracing ถูกเปิดไว้ที่ Crew Level และ `verbose=True` ถูกเปิดทั้ง Agent และ Crew. ([GitHub][2])

---

# 19. แบบฝึกหัดแนะนำ

## Exercise 1 — ทำให้โปรแกรม Error

เปลี่ยน Assignment ให้ Agent สร้าง Code ที่มีโอกาสผิด เช่น:

```text
Read a CSV file named data.csv and calculate
the average of the amount column.
```

อย่าใส่ `data.csv` แล้วดูว่า Agent:

```text
ตรวจไฟล์ก่อนหรือไม่
จัดการ FileNotFoundError หรือไม่
สร้างข้อมูลปลอมหรือไม่
```

---

## Exercise 2 — เพิ่ม Tests

ให้ Agent ต้องสร้าง:

```text
solution.py
test_solution.py
```

แล้วรัน:

```bash
python -m unittest test_solution.py
```

ตรวจว่า Agent แยก Implementation ออกจาก Verification ได้หรือไม่

---

## Exercise 3 — Capture stderr

แก้ Run Tool ให้คืน:

```text
stdout
stderr
returncode
```

จากนั้นทดลอง Code ที่เกิด Syntax Error แล้วเปรียบเทียบความสามารถในการแก้ปัญหา

---

## Exercise 4 — ปิด Network

เพิ่ม:

```bash
--network none
```

แล้วให้ Assignment พยายามดาวน์โหลดหน้าเว็บ ตรวจว่า Execution ถูก Block หรือไม่

---

## Exercise 5 — ทดสอบ Path Traversal

ส่งชื่อไฟล์:

```text
../test_escape.txt
```

ตรวจว่า Tool ปัจจุบันสามารถเขียนออกนอก Sandbox หรือไม่ จากนั้นเพิ่ม `safe_path()` และยืนยันว่าถูกปฏิเสธ

---

## Exercise 6 — เพิ่ม Task Guardrail

ให้ Task ผ่านได้เมื่อ:

```text
solution.py มีอยู่
Exit code = 0
Tests ผ่าน
Output ตรงกับ Expected Value
```

CrewAI รองรับ Guardrail ที่คืน `(True, result)` หรือ `(False, feedback)` และสามารถส่ง Feedback กลับให้ Agent แก้ไขงานได้. ([CrewAI Documentation][10])

---

# 20. Misconceptions ที่ต้องแก้

## “Agent เขียน Code ได้ หมายความว่า Agent รัน Code ได้”

ไม่จริง

การสร้าง Text กับการ Execute เป็น Capability คนละประเภท Lab นี้ต้องเพิ่ม Custom Tools เพื่อให้เกิด File และ Runtime Actions

## “Docker เท่ากับปลอดภัย”

ไม่จริง

Container ปัจจุบันยังมี Writable Bind Mount, Network Access และไม่มี CPU หรือ Memory Limits

## “Timeout ป้องกันทุกอย่าง”

ไม่จริง

ก่อนครบ 60 วินาที Code อาจใช้ Memory, CPU, Process หรือ Network จำนวนมาก

## “ถ้าโปรแกรมไม่ Error แปลว่าคำตอบถูก”

ไม่จริง

โปรแกรมสามารถรันสำเร็จแต่ใช้ Algorithm ผิดหรือ Hard-code ผลลัพธ์

## “Expected Output คือ Test”

ไม่จริง

Expected Output เป็น Prompt Guidance ไม่ใช่ Deterministic Assertion

## “Sandbox Tools เข้าถึงได้เฉพาะ Sandbox เสมอ”

ไม่จริงใน Implementation ปัจจุบัน เพราะ Filename ยังไม่ได้รับการ Validate หลัง Resolve Path

## “นี่คือ Multi-Agent Coding Team”

ไม่ใช่

Lab นี้มี Coder Agent เพียงหนึ่งตัวและ Coding Task หนึ่งรายการ

---

# 21. Checklist ก่อนจบ Lab

### Agent มี Tools อะไร

```text
List Sandbox Files
Read Sandbox File
Write Sandbox File
Run Sandbox Python File
```

### Code ถูก Execute ที่ไหน

ภายใน Docker Container ที่ใช้ Image `python:3.13-slim`

### Sandbox Directory อยู่ที่ไหน

```text
coder/sandbox/
```

### Final Summary อยู่ที่ไหน

```text
output/solution.md
```

### ใช้ Built-in `allow_code_execution` หรือไม่

ไม่ ใช้ Custom Sandbox Tools

### ทำไม Tool Cache ถูกปิด

เพราะ Filesystem State เปลี่ยนตลอด และ Cache อาจคืนข้อมูลเก่า

### Run Tool คืน Error Details ครบหรือไม่

ไม่ คืนเพียง `stdout`

### มี Network Isolation หรือไม่

ยังไม่มี เพราะไม่ได้กำหนด `--network none`

### มี CPU และ Memory Limits หรือไม่

ยังไม่มี

### มี Path Boundary Validation หรือไม่

ยังไม่มี

### การรันสำเร็จพิสูจน์ Correctness หรือไม่

ไม่ ต้องมี Tests และ Deterministic Validation

---

# แก่นของ Lab 4

```text
LLM
= สร้างแผนและตัดสินใจ

Tools
= อ่าน เขียน และรันโค้ด

Sandbox Files
= External Working State

Docker
= Execution Boundary

stdout
= Feedback จาก Environment

Agent Loop
= Write → Run → Inspect → Fix
```

บทเรียนสำคัญที่สุดคือ:

> **Coding Agent จะกลายเป็น Agent จริงก็ต่อเมื่อมันได้รับ Feedback จาก Environment ไม่ใช่เพียงพิมพ์ Code ออกมา การเขียนไฟล์ รันโปรแกรม อ่าน Error และแก้ไขใหม่คือวงจรที่เปลี่ยน Code Generator ให้เป็น Coding Agent**

แต่ต้องจำอีกด้านหนึ่งว่า:

> **เมื่อ Agent สามารถรัน Code ได้ เราไม่ได้เพียงเพิ่มความสามารถ แต่กำลังเปิด Execution Surface ใหม่ทั้งหมด Sandbox, Path Validation, Network Policy, Resource Limits, Tests และ Human Review จึงเป็นส่วนหนึ่งของ Product Architecture ไม่ใช่เพียงรายละเอียดด้าน Security ที่ค่อยเพิ่มภายหลัง**

[1]: https://github.com/ed-donner/agents/tree/main/3_crewai/reference/coder "agents/3_crewai/reference/coder at main · ed-donner/agents · GitHub"
[2]: https://raw.githubusercontent.com/ed-donner/agents/main/3_crewai/reference/coder/src/coder/crew.py "raw.githubusercontent.com"
[3]: https://raw.githubusercontent.com/ed-donner/agents/main/3_crewai/reference/coder/src/coder/config/agents.yaml "raw.githubusercontent.com"
[4]: https://raw.githubusercontent.com/ed-donner/agents/main/3_crewai/reference/coder/src/coder/config/tasks.yaml "raw.githubusercontent.com"
[5]: https://raw.githubusercontent.com/ed-donner/agents/main/3_crewai/reference/coder/src/coder/main.py "raw.githubusercontent.com"
[6]: https://docs.crewai.com/en/concepts/agents "Agents - CrewAI"
[7]: https://raw.githubusercontent.com/ed-donner/agents/main/3_crewai/reference/coder/src/coder/tools/sandbox_tools.py "raw.githubusercontent.com"
[8]: https://github.com/ed-donner/agents/blob/main/3_crewai/reference/coder/output/solution.md "agents/3_crewai/reference/coder/output/solution.md at main · ed-donner/agents · GitHub"
[9]: https://github.com/ed-donner/agents/blob/main/3_crewai/reference/coder/output/code_and_output.txt "agents/3_crewai/reference/coder/output/code_and_output.txt at main · ed-donner/agents · GitHub"
[10]: https://docs.crewai.com/en/concepts/tasks "Tasks - CrewAI"
