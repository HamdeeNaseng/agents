# Episodic Learning Artifact

## Week 2 — Lab 2: Multi-Agent Orchestration

**หลักสูตร:** AI Engineer Agentic Track
**ผู้สอน:** Ed Donner
**Repository:** `ed-donner/agents`
**ไฟล์เรียน:** `2_openai/2_lab2.ipynb`
**หัวข้อหลัก:** Code Orchestration, Parallel Agents, Agent-as-a-Tool, Handoffs และ Multi-Agent Control
**สถานะ:** เรียนและสรุปแนวคิดแล้ว

---

# 1. Context

Week 2 Lab 1 แนะนำ OpenAI Agents SDK ผ่าน Primitive หลัก:

```text
Agent
Runner
Function Tool
Tracing
Streaming
Session
```

Lab 2 ขยายจาก Agent ตัวเดียวไปสู่ระบบที่มีหลาย Agents ทำงานร่วมกัน

คำถามหลักของ Lab นี้ไม่ใช่เพียง:

```text
จะสร้าง Agent หลายตัวอย่างไร
```

แต่คือ:

```text
ใครควรควบคุม Workflow
ใครควรเลือก Agent ถัดไป
ใครควรถือ Context
ใครควรสร้าง Final Answer
```

Lab แสดงแนวทาง Orchestration สามแบบ:

```text
1. Code Orchestration
2. Agent-as-a-Tool
3. Handoff
```

---

# 2. Learning Objectives

หลังจบ Lab 2 สามารถอธิบายได้ว่า:

1. Agent Orchestration คืออะไร
2. Code-driven และ LLM-driven orchestration แตกต่างกันอย่างไร
3. `asyncio.gather()` ใช้รัน Agents แบบ Concurrent อย่างไร
4. Fan-out/Fan-in Pattern ทำงานอย่างไร
5. Writer, Picker และ Sender Agents มีบทบาทต่างกันอย่างไร
6. Agent-as-a-Tool ทำงานอย่างไร
7. Nested Agent Run คืออะไร
8. Handoff แตกต่างจาก Agent-as-a-Tool อย่างไร
9. ใครเป็นเจ้าของ Final Answer ในแต่ละ Pattern
10. `tool_choice="required"` รับประกันและไม่รับประกันอะไร
11. Agent Graph และ Trace ให้ข้อมูลคนละแบบอย่างไร
12. ทำไม Multi-Agent ไม่ได้ดีกว่า Single Agent เสมอไป

---

# 3. Orchestration

Orchestration คือการควบคุมความสัมพันธ์ของงานและ Agents:

```text
Agent ใดทำงาน
Agent ทำงานเมื่อใด
งานใดทำพร้อมกันได้
งานใดต้องรอผลก่อนหน้า
Output ถูกส่งต่อไปที่ใด
ใครตัดสินใจขั้นตอนถัดไป
ใครตอบผู้ใช้เป็นคนสุดท้าย
```

ระบบ Multi-Agent ที่ไม่มี Orchestration ชัดเจนอาจมี Agents หลายตัว แต่ไม่สามารถรับประกันได้ว่างานจะเดินไปในทิศทางที่ถูกต้อง

---

# 4. Project Situation

Lab สร้างทีม Sales Agents สำหรับเขียนและเลือก Cold Email

บทบาทหลัก:

```text
Professional Sales Agent
→ เขียนแบบมืออาชีพและน่าเชื่อถือ

Humorous Sales Agent
→ เขียนแบบเป็นกันเองและมีอารมณ์ขัน

Executive Sales Agent
→ เขียนแบบกระชับและตรงประเด็น

Sales Picker
→ เลือก Email ที่ดีที่สุด

Sales Sender
→ เตรียมและส่ง Email
```

Agents สามตัวแรกได้รับเป้าหมายเดียวกัน แต่มี Instructions แตกต่างกัน

นี่คือ **Role Specialization**

```text
Shared Goal
+
Different Roles
=
Diverse Candidate Outputs
```

---

# 5. External Side Effect

Lab รองรับการส่งข้อความผ่าน:

```text
SMTP Email
หรือ
Pushover Notification
```

Function กลาง:

```python
def send_message(subject, text_body, html_body):
    if USE_EMAIL:
        send_email(subject, text_body, html_body)
    else:
        push(f"Subject: {subject}\n\n{text_body}")
```

Agent เห็นเพียง Business Capability:

```text
send_message
```

แต่ไม่จำเป็นต้องรู้รายละเอียด Infrastructure ว่าใช้ SMTP หรือ Pushover

---

# 6. Business Capability กับ Infrastructure

Mental Model:

```text
Agent Intent
→ Send Message
→ Infrastructure Adapter
   ├── SMTP
   └── Pushover
```

ข้อดี:

```text
เปลี่ยน Provider ได้
Test ง่ายขึ้น
Agent Instructions ไม่ผูกกับ Infrastructure
แยก Business Logic ออกจาก Integration Logic
```

---

# 7. Side-Effect Safety

การส่ง Email หรือ Notification เป็น Action ที่เกิดขึ้นจริง

ระหว่าง Development ควรเปลี่ยน Tool ให้:

```text
พิมพ์ข้อความ
บันทึก Draft
หรือใช้ Mock
```

แทนการส่งจริง

เหตุผล:

```text
Agent อาจเรียก Tool ซ้ำ
Run อาจถูกทดลองหลายครั้ง
เกิด Duplicate Message
ข้อความอาจยังไม่ได้ตรวจ
ข้อมูลอาจถูกส่งออกโดยไม่ตั้งใจ
```

---

# 8. Code Orchestration

Code Orchestration คือการที่ Python เป็นผู้กำหนด Workflow

```text
Python Code
→ เลือก Agent ที่ต้องรัน
→ กำหนด Parallelism
→ รวบรวม Outputs
→ ส่งให้ Agent ถัดไป
→ ควบคุมจุดส่ง Email
```

ตัวอย่าง:

```python
results = await asyncio.gather(
    Runner.run(sales_agent1, message),
    Runner.run(sales_agent2, message),
    Runner.run(sales_agent3, message),
)
```

---

# 9. `asyncio.gather()`

`asyncio.gather()` เริ่ม Async Tasks หลายตัวโดยไม่ต้องรอ Task แรกเสร็จก่อนเริ่มตัวถัดไป

Flow:

```text
                    ┌─ Writer 1 ─┐
User Request ───────├─ Writer 2 ─┼─ Gather Results
                    └─ Writer 3 ─┘
```

เหมาะเมื่อ Tasks:

```text
ไม่พึ่งพากัน
ใช้ Input เดียวกัน
เริ่มพร้อมกันได้
รวมผลภายหลังได้
```

ไม่เหมาะเมื่อ:

```text
Task B ต้องใช้ผล Task A
Task C ต้องรอ Approval
งานมี Dependency ตามลำดับ
```

---

# 10. Concurrent ไม่เท่ากับ Guaranteed Parallel

`asyncio.gather()` ทำให้ Network-bound Agent Runs เริ่มและรอร่วมกันแบบ Concurrent

ไม่ได้หมายความว่า Python CPU จะ Execute ทุกคำสั่งพร้อมกันแบบ Hardware Parallel เสมอไป

ในบริบท API Calls ประโยชน์หลักคือ:

```text
ไม่เสียเวลารอ Model A
ก่อนเริ่ม Model B และ C
```

---

# 11. Fan-out/Fan-in

Lab นำ Pattern จาก Week 1 กลับมาใช้

## Fan-out

```text
หนึ่ง Request
→ หลาย Writer Agents
```

## Fan-in

```text
หลาย Candidate Outputs
→ Picker Agent
```

ภาพรวม:

```text
Writer 1 ─┐
Writer 2 ─┼─► Picker ─► Selected Email
Writer 3 ─┘
```

---

# 12. Candidate Generation

Agents หลายตัวสร้าง Candidate ที่มี Style ต่างกัน

ข้อดี:

```text
เพิ่มความหลากหลาย
ลดการพึ่ง Output เดียว
มีทางเลือกให้ Evaluator
ทดลอง Prompt Strategies หลายแบบ
```

ข้อเสีย:

```text
Model Calls เพิ่ม
Token Cost เพิ่ม
Latency เพิ่ม
ต้องมีขั้นตอนเลือก
Failure Points เพิ่ม
```

---

# 13. Picker Agent

Picker รับ Emails ทั้งหมดและเลือก Candidate ที่ดีที่สุด

```text
Candidate 1
Candidate 2
Candidate 3
      ↓
Picker LLM
      ↓
Selected Candidate
```

Picker เป็นตัวอย่างของ:

```text
LLM-as-a-Judge
```

ซึ่งต่อยอดจาก Week 1 Lab 2

---

# 14. Picker ไม่ใช่ Ground Truth

Picker อาจได้รับผลกระทบจาก:

```text
Position Bias
Style Bias
Length Bias
Self-preference
Prompt Sensitivity
```

Email ที่ถูกเลือกอาจเขียนดี แต่ไม่ได้หมายความว่า:

```text
ข้อมูลถูกต้อง
ไม่อวดอ้างเกินจริง
ตรง Brand Policy
ถูกกฎหมาย
ได้ Conversion สูงสุด
ไม่เข้าข่าย Spam
```

---

# 15. Evaluation Improvements

ระบบจริงควรใช้ Rubric เช่น:

```text
Factual Accuracy
Value Proposition
Clarity
Tone
Brand Compliance
Claim Safety
Legal Compliance
Call-to-Action Quality
Spam Risk
```

และเสริม:

```text
Deterministic Checks
Human Review
A/B Testing
Historical Performance
```

---

# 16. Sender Function Tool

ตัวอย่าง:

```python
@function_tool
def send_email_tool(
    subject: str,
    text_body: str,
    html_body: str
) -> str:
    """Send an email with the given subject and body."""
    send_message(subject, text_body, html_body)
    return "Email sent successfully"
```

SDK ใช้:

```text
Function Name
Docstring
Type Annotations
```

เพื่อสร้าง Tool Schema

---

# 17. `tool_choice="required"`

ตัวอย่าง:

```python
require_tool = ModelSettings(
    tool_choice="required"
)
```

ความหมาย:

```text
auto
→ Model เลือกว่าจะเรียก Tool หรือไม่

required
→ Model ต้องสร้าง Tool Call
```

ใช้เมื่อ Workflow ต้องการให้ Agent ทำ Action ผ่าน Tool ไม่ใช่เพียงอธิบายว่าจะทำ

---

# 18. สิ่งที่ `required` รับประกัน

```text
ต้องเกิด Tool Call
```

แต่ไม่รับประกันว่า:

```text
เลือก Tool ถูก
Arguments ถูกต้อง
Action เหมาะสม
Tool Execute สำเร็จ
ไม่มี Side Effect ซ้ำ
ควรทำ Action ตอนนั้นจริง
```

ดังนั้น Tool Choice ไม่ใช่ Business Validation

---

# 19. Code-Orchestrated Workflow

Flow เต็ม:

```text
1. Code เริ่ม Writer Agents สามตัว
2. Writers ทำงานแบบ Concurrent
3. Code รวบรวม Drafts
4. Code ส่ง Drafts ให้ Picker/Sender
5. Agent เลือก Draft
6. Agent เรียก Send Tool
7. SDK Execute Tool
8. Final Response ถูกสร้าง
```

Application เป็นผู้ควบคุม Architecture หลัก

---

# 20. จุดแข็งของ Code Orchestration

```text
Predictable
Control Flow ชัดเจน
Test Step แยกได้
บังคับ Parallelism ได้
เพิ่ม Validation Gate ง่าย
ควบคุม Cost และจำนวน Calls ได้
Debug ง่ายกว่า
```

เหมาะกับ Workflow ที่:

```text
มีลำดับชัดเจน
มีข้อกำหนดเข้ม
ต้องตรวจสอบย้อนหลัง
ต้องควบคุม Side Effects
```

---

# 21. ข้อจำกัดของ Code Orchestration

```text
ต้องเขียน Routing Logic เอง
Workflow เปลี่ยนตามบริบทได้ยากกว่า
Branches จำนวนมากทำให้ Code ซับซ้อน
Programmer ต้องคาดการณ์เส้นทางล่วงหน้า
```

---

# 22. LLM Orchestration

LLM Orchestration ให้ Agent ตัดสินใจว่า:

```text
ควรเรียก Specialist ใด
ควรเรียกกี่ครั้ง
ควรเรียกตามลำดับใด
ควร Handoff เมื่อใด
```

Lab แสดงสองรูปแบบ:

```text
Agent-as-a-Tool
Handoff
```

---

# 23. Agent-as-a-Tool

Specialist Agent ถูกแปลงให้เป็น Tool ของ Manager:

```python
writer_tool = writer_agent.as_tool(
    tool_name="professional_email_writer",
    tool_description="Write a professional sales email"
)
```

Manager เห็น Writer Agent เหมือน Tool หนึ่งตัว

แต่ภายใน Tool มี Nested Agent Run

---

# 24. Agent Tool กับ Function Tool

## Function Tool

```text
Python Function
→ Execute Code
→ Return Result
```

## Agent Tool

```text
Nested Agent
→ Call LLM
→ อาจใช้ Tools
→ Return Final Output
```

ทั้งสองปรากฏเป็น Tool ต่อ Manager แต่ต่างกันด้าน:

```text
Cost
Latency
Failure Mode
Context
Complexity
```

---

# 25. Agent-as-a-Tool Flow

```text
Manager Agent
      ↓
Calls Writer Agent as Tool
      ↓
Writer performs Nested Run
      ↓
Draft returned to Manager
      ↓
Manager continues reasoning
      ↓
Manager gives Final Answer
```

ย่อเป็น:

```text
Agent A
→ Agent B
→ Agent A
```

---

# 26. Nested Agent Run

ตัวอย่าง Trace:

```text
Parent Run: Sales Manager
├── Agent Tool Call
│   └── Nested Run: Writer Agent
└── Parent Agent Continues
```

Specialist ทำงานภายใต้ Parent Run และคืน Output กลับไปยัง Manager

---

# 27. Ownership ใน Agent-as-a-Tool

Manager ยังคงเป็น:

```text
Conversation Owner
Workflow Owner
Final Answer Owner
```

Specialist เป็นเพียงผู้ทำ Subtask

Metaphor:

```text
Manager ขอรายงานจากผู้เชี่ยวชาญ
ผู้เชี่ยวชาญส่งรายงานกลับ
Manager เป็นคนตัดสินใจและคุยกับลูกค้าต่อ
```

---

# 28. จุดแข็งของ Agent-as-a-Tool

```text
Manager รวมหลาย Specialist Outputs ได้
Manager ควบคุม Final Answer
แบ่ง Instructions ตามบทบาทได้
เหมาะกับงานย่อยที่มีขอบเขตชัด
สามารถรวม Policy ไว้ส่วนกลาง
```

---

# 29. ข้อจำกัดของ Agent-as-a-Tool

```text
Manager อาจไม่เรียก Specialist ครบ
อาจเรียกซ้ำ
Nested Calls เพิ่ม Cost
Manager Context ใหญ่ขึ้น
Manager อาจสรุป Specialist ผิด
Tool Selection ไม่ Deterministic
```

Instructions ที่บอกว่า “เรียกทุก Tool” ไม่ได้เป็น Guarantee

---

# 30. Handoff

Handoff คือการโอน Active Agent และความรับผิดชอบไปยัง Specialist

```text
Agent A
→ Transfer
→ Agent B
```

หลัง Handoff Agent B รับช่วงการสนทนาและเป็นผู้สร้าง Final Output ต่อไป

---

# 31. Handoff Flow

```text
Sales Manager
      ↓
Handoff
      ↓
Sales Sender
      ↓
Select Email
      ↓
Send Email
      ↓
Final Response
```

โดยทั่วไปไม่ได้ย้อนกลับไปหา Manager ภายใน Flow เดิม

---

# 32. Agent-as-a-Tool กับ Handoff

## Agent-as-a-Tool

```text
A → B → A
```

B ทำ Subtask และคืนผลให้ A

## Handoff

```text
A → B
```

B รับช่วงความรับผิดชอบต่อ

---

# 33. Metaphor

## Agent-as-a-Tool

หัวหน้าเรียกผู้เชี่ยวชาญมาช่วยวิเคราะห์ แล้วหัวหน้าเป็นผู้สื่อสารกับลูกค้าต่อ

## Handoff

พนักงานฝ่ายแรกโอนเคสไปยังฝ่ายเฉพาะทาง ฝ่ายใหม่รับผิดชอบลูกค้าต่อจากจุดนั้น

---

# 34. การเลือกใช้ Pattern

ใช้ Agent-as-a-Tool เมื่อ:

```text
Specialist ทำงานย่อย
Manager ต้องรวมหลายผลลัพธ์
Manager ต้องถือ Final Answer
ต้องใช้ Policy ส่วนกลาง
```

ใช้ Handoff เมื่อ:

```text
Specialist ควรรับช่วงบทสนทนา
งานเข้าสู่ Domain ใหม่
Agent เดิมไม่ควรควบคุมต่อ
Ownership ต้องเปลี่ยนจริง
```

---

# 35. Hybrid Pattern

Lab ใช้ Pattern ผสม:

```text
Sales Manager
├── Writer Agent 1 as Tool
├── Writer Agent 2 as Tool
├── Writer Agent 3 as Tool
└── Handoff to Sales Sender
```

เหตุผล:

```text
Writers
→ ทำ Subtasks แล้วคืน Draft

Sender
→ รับช่วง Stage สุดท้ายทั้งหมด
```

---

# 36. Control Ownership

คำถามหลักของการออกแบบ Multi-Agent:

```text
ใครถือ Control ตอนนี้
ใครเห็น Context ใด
ใครตัดสินใจต่อ
ใครรับผิดชอบ Final Output
```

การมี Agent หลายตัวโดยไม่ตอบคำถามเหล่านี้ทำให้ระบบ Debug และ Audit ยาก

---

# 37. Agent Graph

```python
draw_graph(sales_manager)
```

Graph แสดง Static Architecture:

```text
Agents
Function Tools
Agent Tools
Handoff Relationships
```

ตอบว่า:

```text
ระบบสามารถเดินไปทางใดได้บ้าง
```

---

# 38. Trace

Trace แสดง Runtime Execution:

```text
Agent ใดถูกเรียกจริง
Tool ใดถูกใช้
Nested Runs ใดเกิดขึ้น
Handoff เกิดเมื่อใด
ใช้เวลาเท่าไร
```

ตอบว่า:

```text
Run นี้เดินทางไปทางใดจริง
```

---

# 39. Graph กับ Trace

```text
Graph
= แผนที่เส้นทางที่เป็นไปได้

Trace
= ประวัติการเดินทางจริง
```

ทั้งสองจำเป็นสำหรับ Multi-Agent Observability

---

# 40. Pattern Comparison

| ประเด็น          | Code Orchestration | Agent-as-a-Tool  | Handoff                       |
| ---------------- | ------------------ | ---------------- | ----------------------------- |
| ผู้ควบคุม Flow   | Python             | Manager LLM      | Routing Agent แล้ว Specialist |
| ความคาดเดาได้    | สูง                | ปานกลาง          | ต่ำกว่า                       |
| Final Answer     | Code เลือก Agent   | Manager          | Specialist                    |
| Specialist คืนผล | ให้ Code           | ให้ Manager      | รับช่วงต่อ                    |
| เหมาะกับ         | Fixed Workflow     | Bounded Subtasks | Ownership Transfer            |
| Parallelism      | Code บังคับได้     | ขึ้นกับ Manager  | ไม่ใช่จุดหลัก                 |
| Debug            | ง่ายกว่า           | Nested Trace     | Handoff Trace                 |
| Cost Control     | ง่ายกว่า           | ยากขึ้น          | ยากขึ้น                       |
| Flexibility      | ปานกลาง            | สูง              | สูง                           |

---

# 41. Multi-Agent ไม่ได้ดีกว่าเสมอไป

Multi-Agent เพิ่ม:

```text
Model Calls
Prompts
Context Transfers
Latency
Cost
Failure Points
Tracing Complexity
```

ควรใช้เมื่อ:

```text
บทบาทแตกต่างกันจริง
Agent ต้องใช้ Tools หรือ Policies ต่างกัน
Context ควรถูกแยก
Ownership ต้องเปลี่ยน
Parallelism ให้ประโยชน์ชัดเจน
```

หาก Agent เดียวทำงานได้ดี Multi-Agent อาจเป็น Complexity ที่ไม่จำเป็น

---

# 42. Parallel Agents ไม่เท่ากับ Autonomous Agents

Code:

```python
asyncio.gather(...)
```

เป็น Multi-Agent Execution แต่ Code เป็นผู้ตัดสินใจทั้งหมดว่า Agents ใดต้องทำงาน

ดังนั้น:

```text
Concurrent Multi-Agent
≠
Autonomous Orchestration
```

---

# 43. LLM Instructions ไม่ใช่ Control Guarantee

แม้ Prompt ระบุ:

```text
Call all three writers
Wait for all responses
Then hand off
```

LLM อาจ:

```text
ข้าม Agent
เรียก Agent ซ้ำ
Handoff เร็วเกินไป
เลือกก่อนครบ
ไม่ส่ง Email
```

เมื่อ Step ใดเป็นข้อบังคับทางธุรกิจ ควรใช้ Code หรือ Deterministic Gate บังคับแทนการพึ่ง Prompt เพียงอย่างเดียว

---

# 44. Picker Output Limitation

Picker คืนข้อความธรรมดา จึงตรวจสอบได้ยากว่า:

```text
เลือก Candidate หมายเลขใด
เหตุผลคืออะไร
คะแนนตาม Rubric เท่าไร
```

ควรใช้ Structured Output:

```json
{
  "selected_candidate": 2,
  "reason": "Clear and concise",
  "scores": {
    "clarity": 5,
    "tone": 4,
    "accuracy": 5
  },
  "subject": "...",
  "text_body": "...",
  "html_body": "..."
}
```

---

# 45. Missing Validation Gate

Workflow ใน Lab:

```text
Generate
→ Pick
→ Send
```

ระบบจริงควรเป็น:

```text
Generate
→ Pick
→ Validate
→ Approve
→ Send
```

Validation อาจประกอบด้วย:

```text
Fact Check
Brand Policy
Spam Detection
Recipient Validation
Legal Compliance
Human Approval
```

---

# 46. Idempotency

Side-effect Tool เช่น Email Sender ต้องป้องกันการส่งซ้ำ

ควรมี:

```text
Idempotency Key
Message ID
Run ID
Duplicate Check
Delivery Status
```

เพราะ Tool Call อาจเกิดซ้ำจาก:

```text
Retry
Network Timeout
Agent Loop
User กดซ้ำ
Run ใหม่
```

---

# 47. Tool Result Accuracy

Function ไม่ควรคืน:

```text
Email sent successfully
```

หากยังไม่ได้ตรวจ SMTP หรือ Provider Response

ควรคืน Structured Result:

```json
{
  "success": true,
  "provider": "smtp",
  "message_id": "abc123"
}
```

หรือ:

```json
{
  "success": false,
  "error": "Authentication failed"
}
```

---

# 48. Safer Architecture

```text
User Request
      ↓
Input Validation
      ↓
Writer Agents in Parallel
      ↓
Structured Drafts
      ↓
Picker Agent
      ↓
Deterministic Policy Checks
      ↓
Human Approval
      ↓
Sender Tool
      ↓
Delivery Verification
      ↓
Audit Log
```

---

# 49. What Code Should Control

```text
จำนวน Candidates
Required Agents
Parallelism
Validation Gates
Approval
Retry Limits
Budget
Side-effect Permissions
Termination Conditions
```

---

# 50. What LLM Should Control

```text
Writing Style
Content Organization
Contextual Adaptation
Draft Generation
Qualitative Comparison
Natural-language Reasoning
```

หลักการ:

```text
Code controls invariants
LLM handles ambiguity
```

---

# 51. Misconceptions Corrected

## ความเข้าใจคลาดเคลื่อนที่ 1

> Multi-Agent ดีกว่า Single Agent เสมอ

**ข้อเท็จจริง:**
Multi-Agent เหมาะเมื่อการแบ่งบทบาทให้ประโยชน์มากกว่าความซับซ้อนที่เพิ่มขึ้น

---

## ความเข้าใจคลาดเคลื่อนที่ 2

> `asyncio.gather()` ทำให้ Workflow Autonomous

**ข้อเท็จจริง:**
Python ยังเป็นผู้ควบคุมทุก Agent ที่ต้องทำงาน

---

## ความเข้าใจคลาดเคลื่อนที่ 3

> Agent-as-a-Tool เหมือน Python Function Tool

**ข้อเท็จจริง:**
Agent Tool สร้าง Nested LLM Run จึงมี Cost และ Failure Mode มากกว่า

---

## ความเข้าใจคลาดเคลื่อนที่ 4

> Handoff คือเรียก Specialist แล้วกลับมา Manager

**ข้อเท็จจริง:**
นั่นคือ Agent-as-a-Tool ส่วน Handoff เป็นการโอน Ownership

---

## ความเข้าใจคลาดเคลื่อนที่ 5

> Prompt สามารถรับประกันว่า Manager เรียก Agents ครบ

**ข้อเท็จจริง:**
ข้อบังคับสำคัญควรถูกควบคุมด้วย Code หรือ Gate

---

## ความเข้าใจคลาดเคลื่อนที่ 6

> `tool_choice="required"` รับประกัน Action ที่ถูกต้อง

**ข้อเท็จจริง:**
รับประกันเพียงว่าจะมี Tool Call

---

## ความเข้าใจคลาดเคลื่อนที่ 7

> Graph แสดงสิ่งที่เกิดขึ้นจริง

**ข้อเท็จจริง:**
Graph แสดง Architecture ที่เป็นไปได้ ส่วน Trace แสดง Runtime จริง

---

# 52. Risks Identified

## 52.1 Cost Explosion

Manager อาจเรียก Agent Tools หลายครั้ง

## 52.2 Latency

Nested Agents และหลาย Model Calls เพิ่มเวลาตอบ

## 52.3 Context Growth

Manager ต้องรับ Draft และ Tool Results จำนวนมาก

## 52.4 Wrong Routing

LLM อาจเลือก Specialist หรือ Handoff ผิด

## 52.5 Premature Handoff

Agent อาจส่งงานต่อก่อนข้อมูลครบ

## 52.6 Duplicate Side Effects

Email อาจถูกส่งซ้ำ

## 52.7 Evaluation Bias

Picker อาจเลือกจาก Style แทน Accuracy

## 52.8 Trace Privacy

Draft, Email และ Tool Arguments อาจปรากฏใน Trace

---

# 53. Production Improvements

```text
Structured Agent Outputs
Deterministic Routing Rules
Required-Step Validation
Maximum Agent Turns
Maximum Tool Calls
Timeouts
Retry Policy
Idempotency
Human Approval
Trace Redaction
Cost Budget
Fallback Agent
```

---

# 54. Patterns Learned

## Code-Orchestrated Pipeline

```text
Code
→ Agent A/B/C
→ Aggregate
→ Agent D
```

## Parallel Fan-out/Fan-in

```text
One Task
→ Multiple Agents
→ Aggregate Outputs
```

## Generator–Evaluator

```text
Writers
→ Picker
```

## Agent-as-a-Tool

```text
Manager
→ Nested Specialist
→ Manager
```

## Handoff

```text
Agent A
→ Transfer Ownership
→ Agent B
```

## Hybrid Multi-Agent Pattern

```text
Manager
→ Specialists as Tools
→ Handoff to Final Specialist
```

---

# 55. Connection to Week 2 Lab 1

Lab 1:

```text
Agent
Runner
Function Tools
Trace
```

Lab 2:

```text
หลาย Agents
Nested Runs
Handoffs
Agent Graph
Concurrent Runs
```

Runner ยังคงเป็น Runtime หลัก แต่ Flow มีหลาย Agent Boundaries

---

# 56. Connection to Week 1 Lab 2

Week 1 Lab 2:

```text
หลายโมเดลสร้างคำตอบ
→ Judge เลือก
```

Week 2 Lab 2:

```text
หลาย Agents สร้าง Emails
→ Picker Agent เลือก
```

โครงสร้าง Generator–Evaluator เดิมถูกย้ายเข้าสู่ Agents SDK

---

# 57. Connection to Week 1 Lab 4

Week 1 Lab 4 สอนว่า:

```text
LLM เสนอ Action
Application ถือ Authority
```

หลักนี้ยังใช้ใน Multi-Agent:

```text
LLM เสนอ Routing หรือ Handoff
Application ควรกำหนด Policy และ Limits
```

---

# 58. Mental Model

```text
User Goal
   ↓
Orchestrator
   ├── Code
   └── Manager LLM
          ↓
Specialist Agents
          ↓
Candidate Outputs
          ↓
Evaluation or Routing
          ↓
Tool Action or Handoff
          ↓
Final Agent
          ↓
Final Output
```

---

# 59. Final Lessons

## Lesson 1

Orchestration คือการกำหนด Control, Sequence, Context และ Ownership ระหว่าง Agents

## Lesson 2

Code Orchestration คาดเดาง่ายและเหมาะกับข้อบังคับที่ชัดเจน

## Lesson 3

`asyncio.gather()` เหมาะกับ Agent Tasks ที่ไม่พึ่งพากัน

## Lesson 4

Agent-as-a-Tool ให้ Manager รักษา Control และ Final Answer

## Lesson 5

Handoff โอน Control และ Final Answer ไปยัง Specialist

## Lesson 6

Function Tool กับ Agent Tool อาจดูเหมือนกันต่อ Manager แต่มี Runtime Cost ต่างกันมาก

## Lesson 7

Prompt ไม่ควรเป็นกลไกเดียวสำหรับบังคับ Business Invariants

## Lesson 8

Graph แสดง Architecture ส่วน Trace แสดง Execution จริง

## Lesson 9

Multi-Agent เพิ่มความสามารถบางด้าน แต่เพิ่ม Cost, Latency และ Failure Points

## Lesson 10

การเลือก Pattern ควรเริ่มจากคำถามว่าใครควรถือ Control และใครควรเป็นเจ้าของ Final Answer

---

# 60. Memory Summary

```text
Week 2 Lab 2 สอน Multi-Agent Orchestration

สาม Pattern หลัก:

1. Code Orchestration
Python ควบคุม Flow
เหมาะกับ Workflow ที่ชัดเจน
Predictable และ Test ง่าย

2. Agent-as-a-Tool
Manager เรียก Specialist เป็น Tool
Specialist คืนผลให้ Manager
Manager ถือ Final Answer

3. Handoff
Agent เดิมโอน Ownership ให้ Specialist
Specialist รับช่วง Context และ Final Answer

asyncio.gather()
ใช้รัน Agent Tasks ที่เป็นอิสระพร้อมกัน

Fan-out/Fan-in:
หลาย Writers
→ Picker

Picker เป็น LLM-as-a-Judge
ไม่ใช่ Ground Truth

tool_choice="required"
บังคับให้มี Tool Call
แต่ไม่รับประกัน Action ที่ถูกต้อง

Graph:
แสดงเส้นทางที่เป็นไปได้

Trace:
แสดงเส้นทางที่เกิดขึ้นจริง

Code ควรควบคุม:
Required Steps
Limits
Validation
Approval
Side Effects

LLM ควรจัดการ:
Writing
Reasoning
Adaptation
Qualitative Decisions

Multi-Agent ไม่ได้ดีกว่าเสมอไป
ควรใช้เมื่อการแบ่งบทบาท
และ Ownership สร้างประโยชน์จริง
```

---

# 61. Next Episode

หัวข้อถัดไป:

```text
Week 2 — Lab 3

Multiple Model Providers
Structured Outputs
Pydantic Models
Input Guardrails
Output Guardrails
Guardrail Agents
Tripwires
Safety and Validation
```

คำถามสำคัญสำหรับ Lab ถัดไปคือ:

> เมื่อ Agent และ Multi-Agent Workflow สามารถสร้างคำตอบและเรียก Tools ได้แล้ว เราจะบังคับให้ Output มีโครงสร้าง ตรวจ Input ก่อนทำงาน และหยุด Run เมื่อพบความเสี่ยงได้อย่างไร?

```
```
