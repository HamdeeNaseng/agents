# Week 3 — Lab 3: CrewAI Stock Picker

โปรเจกต์:

```text
3_crewai/reference/stock_picker/
```

Lab นี้ต่อยอดจาก Financial Researcher โดยเพิ่ม 4 เรื่องสำคัญ:

```text
1. Structured Outputs ด้วย Pydantic
2. Hierarchical Process พร้อม Manager Agent
3. Crew/Agent Memory
4. Custom Tool ที่สร้าง Side Effect จริง
```

โปรเจกต์ปัจจุบันใช้ Python `>=3.10,<3.14` และล็อก `crewai[tools]==1.14.4` จึงควรยึดพฤติกรรมของเวอร์ชันในหลักสูตรก่อนนำเอกสาร CrewAI รุ่นใหม่กว่ามาผสม. ([GitHub][1])

---

## Learning Objectives

หลังจบ Lab นี้ควรอธิบายได้ว่า:

1. Trending Finder, Researcher, Stock Picker และ Manager แบ่งงานกันอย่างไร
2. Hierarchical Process ต่างจาก Sequential Process อย่างไร
3. `manager_agent` และ `allow_delegation=True` ทำอะไร
4. Structured Task Output ผ่าน `output_pydantic` ทำงานอย่างไร
5. Task Context ส่ง Candidate จากขั้นหนึ่งไปยังอีกขั้นอย่างไร
6. Memory ช่วยให้ Crew เรียกข้อมูลจากรอบก่อนอย่างไร
7. ทำไม Memory ไม่สามารถรับประกันว่า “ห้ามเลือกซ้ำ” ได้
8. Custom Push Tool สร้าง Side Effect อะไร
9. ทำไมการเลือกหุ้นจากข่าวไม่เท่ากับ Investment Analysis ที่ตรวจสอบแล้ว
10. จุดใดควรถูกควบคุมด้วย Code แทน Prompt

---

# 1. Architecture Overview

```text
Sector + Current Date
        ↓
Manager Agent
        ↓
Trending Company Finder
        ↓
Find Trending Companies
        ↓
Structured TrendingCompanyList
        ↓
Financial Researcher
        ↓
Research Trending Companies
        ↓
Structured TrendingCompanyResearchList
        ↓
Stock Picker
        ↓
Pick Best Company
        ├── Push Notification
        └── Detailed Decision Report
```

Tasks หลักมีสามรายการ:

```text
find_trending_companies
→ research_trending_companies
→ pick_best_company
```

ผลลัพธ์ถูกเขียนลง:

```text
output/trending_companies.json
output/research_report.json
output/decision.md
```

Task ที่สองใช้ผลจาก Task แรกเป็น Context ส่วน Task สุดท้ายใช้ผลวิจัยจาก Task ที่สอง. ([GitHub][2])

---

# 2. Agents ในระบบ

ระบบกำหนด Agent Configurations สี่บทบาท:

```text
trending_company_finder
financial_researcher
stock_picker
manager
```

สามตัวแรกเป็น Worker Agents ส่วน `manager` ถูกสร้างแยกเป็น `manager_agent` ภายใน Crew. Agents ทั้งหมดในไฟล์ Configuration ปัจจุบันใช้ `openai/gpt-5.4-mini`. ([GitHub][3])

---

## Trending Company Finder

หน้าที่:

```text
อ่านข่าวล่าสุดใน Sector
→ หา 2–3 บริษัทที่กำลังเป็นกระแส
→ อธิบายว่าทำไมแต่ละบริษัทกำลังได้รับความสนใจ
```

Agent ถูกสั่งว่า:

```text
Always pick new companies.
Don't pick the same company twice.
```

และได้รับ `SerperDevTool` เพื่อค้นเว็บ. ([GitHub][3])

Mental Model:

```text
Trending Finder
= Candidate Discovery
```

มันยังไม่ได้ตัดสินว่าบริษัทใดดีที่สุด แต่สร้าง Candidate Set ให้ Researcher ตรวจต่อ

---

## Financial Researcher

หน้าที่:

```text
รับรายชื่อ Trending Companies
→ ค้นข้อมูลออนไลน์
→ วิเคราะห์แต่ละบริษัทเชิงลึก
```

Agent มี `SerperDevTool` เช่นเดียวกับ Finder และคืนผลผ่าน Structured Output. ([GitHub][3])

Mental Model:

```text
Finder
= ใครน่าสนใจ

Researcher
= แต่ละบริษัทมีสถานะอย่างไร
```

---

## Stock Picker

หน้าที่:

```text
รับผลการวิจัย
→ เปรียบเทียบบริษัท
→ เลือกหนึ่งบริษัท
→ อธิบายว่าทำไมเลือก
→ อธิบายว่าทำไมไม่เลือกบริษัทอื่น
→ ส่ง Push Notification
```

Stock Picker ไม่มี Search Tool แต่ได้รับ Custom Tool ชื่อ `send_push_notification`. ([GitHub][3])

นี่เป็นการแบ่งสิทธิ์ตามบทบาท:

```text
Finder และ Researcher
→ เข้าถึง Web Search

Stock Picker
→ เข้าถึง Notification Side Effect
```

---

## Manager Agent

Manager มีเป้าหมายจัดการโครงการและมอบหมายงานให้ Agent ที่เหมาะสม โดยถูกสร้างด้วย:

```python
manager = Agent(
    config=self.agents_config["manager"],
    allow_delegation=True
)
```

จากนั้นส่งเข้า Crew ผ่าน:

```python
manager_agent=manager
process=Process.hierarchical
```

([GitHub][3])

Mental Model:

```text
Manager
= หัวหน้าทีม

Workers
= ผู้เชี่ยวชาญแต่ละฝ่าย

Tasks
= งานที่ต้องส่งมอบ
```

---

# 3. Hierarchical Process

Lab ก่อนหน้าใช้:

```text
Process.sequential
```

หมายถึง Framework เดิน Task ตามลำดับที่กำหนดโดยตรง

Lab นี้ใช้:

```python
Process.hierarchical
```

Hierarchical Process มี Manager คอยประสานงาน มอบหมาย Task และตรวจผลลัพธ์ของ Worker Agents. CrewAI รองรับการใช้ Manager ที่ระบบสร้างให้หรือ Custom Manager Agent ที่ผู้พัฒนากำหนดเอง ซึ่ง Lab นี้เลือกแบบ Custom Manager. ([CrewAI Documentation][4])

Flow เชิงแนวคิด:

```text
Task Request
    ↓
Manager
    ↓
เลือกหรือควบคุม Worker
    ↓
Worker Executes
    ↓
Manager Reviews
    ↓
Next Task
```

---

## Hierarchical ไม่ได้แปลว่าทุกอย่างเป็นอิสระ

Tasks ใน YAML ยังคงกำหนด Agent ไว้:

```yaml
find_trending_companies:
  agent: trending_company_finder
```

```yaml
research_trending_companies:
  agent: financial_researcher
```

```yaml
pick_best_company:
  agent: stock_picker
```

ดังนั้น Lab นี้ไม่ได้ปล่อยให้ Manager สร้าง Workflow ใหม่ทั้งหมด แต่ Manager ทำหน้าที่กำกับ Crew ที่มี Tasks และ Worker Roles กำหนดไว้แล้ว. ([GitHub][2])

---

## `allow_delegation=True`

Manager สามารถมอบหมายงานให้ Agent อื่นภายใต้ Hierarchical Process

แต่ Delegation ไม่รับประกันว่า:

```text
เลือก Agent ถูกเสมอ
ไม่มอบหมายซ้ำ
ไม่เรียก Agent เกินจำเป็น
ตรวจงานได้แม่นยำ
```

CrewAI จึงมีตัวเลือกควบคุม เช่น Maximum Iterations และ Requests per Minute สำหรับ Workflow Hierarchical. ([CrewAI Documentation][4])

---

# 4. Structured Outputs

Lab กำหนด Pydantic Models สี่ชุด

## Trending Company

```python
class TrendingCompany(BaseModel):
    name: str
    ticker: str
    reason: str
```

## Trending Company List

```python
class TrendingCompanyList(BaseModel):
    companies: list[TrendingCompany]
```

## Company Research

```python
class TrendingCompanyResearch(BaseModel):
    name: str
    market_position: str
    future_outlook: str
    investment_potential: str
```

## Research List

```python
class TrendingCompanyResearchList(BaseModel):
    research_list: list[TrendingCompanyResearch]
```

([GitHub][5])

---

# 5. `output_pydantic`

Task แรก:

```python
Task(
    config=...,
    output_pydantic=TrendingCompanyList
)
```

Task ที่สอง:

```python
Task(
    config=...,
    output_pydantic=TrendingCompanyResearchList
)
```

ทำให้ Task Output ต้องถูกแปลงเป็น Pydantic Model แทนการส่ง Plain Text ที่ไม่มีโครงสร้าง. CrewAI จะใส่ Structured Result ไว้ใน `TaskOutput.pydantic` เมื่อ Task ถูกกำหนดด้วย `output_pydantic`. ([GitHub][5])

Flow:

```text
Natural-language research
        ↓
Pydantic validation
        ↓
Typed candidate list
        ↓
Next task context
```

---

## ประโยชน์

Structured Output ช่วยให้:

```text
รายชื่อบริษัทมี Field แน่นอน
Ticker ถูกแยกจากชื่อบริษัท
Reason ถูกแยกจากตัว Candidate
Research แต่ละบริษัทมีรูปแบบเดียวกัน
Task ถัดไปอ่านข้อมูลได้ง่ายขึ้น
```

แต่ยังต้องจำ:

```text
Schema Valid
≠
Company Correct
≠
Ticker Correct
≠
Financial Analysis Correct
```

Pydantic ตรวจว่ามี `ticker: str` แต่ไม่ตรวจว่า Ticker นั้นมีอยู่จริงหรือเป็นหลักทรัพย์ของบริษัทที่กล่าวถึง

---

# 6. ทำไม Task สุดท้ายไม่ใช้ Pydantic

`pick_best_company` ไม่มี `output_pydantic`

ผลลัพธ์จึงเป็นรายงานข้อความใน:

```text
output/decision.md
```

([GitHub][2])

ข้อจำกัดคือ Application ไม่สามารถอ่านผลอย่างแน่นอนได้ว่า:

```text
selected_company คืออะไร
ticker คืออะไร
score เท่าไร
ปัจจัยเสี่ยงคืออะไร
บริษัทใดถูกตัดออก
```

Schema ที่แข็งแรงกว่าควรเป็น:

```python
class StockDecision(BaseModel):
    selected_company: str
    ticker: str
    rationale: str
    risks: list[str]
    rejected_companies: list[str]
    uncertainty_notes: list[str]
```

แล้วค่อยแปลง Structured Decision เป็น Markdown ภายหลัง

---

# 7. Task Context

ใน YAML:

```yaml
research_trending_companies:
  context:
    - find_trending_companies
```

และ:

```yaml
pick_best_company:
  context:
    - research_trending_companies
```

`context` คือรายการ Tasks ที่ Output จะถูกใช้เป็นข้อมูลสำหรับ Task ปัจจุบัน. ([GitHub][2])

Flow:

```text
TrendingCompanyList
        ↓
Research Task

TrendingCompanyResearchList
        ↓
Stock Picking Task
```

นี่คือการร่วมมือผ่าน Artifact Transfer เช่นเดียวกับ Financial Researcher Lab

---

# 8. Memory

Lab ตั้ง:

```python
memory=True
```

ให้กับ:

```text
Trending Finder
Financial Researcher
Stock Picker
Crew
```

([GitHub][5])

เป้าหมายของ Prompt คือให้ระบบจำบริษัทที่เคยพบหรือเคยเลือก และพยายามไม่เลือกบริษัทเดิมซ้ำ

---

## Memory ทำงานอย่างไรใน CrewAI รุ่นปัจจุบัน

เอกสาร CrewAI ปัจจุบันอธิบายว่า เมื่อ Crew ใช้ `memory=True` ระบบจะสร้าง Shared Memory เก็บข้อเท็จจริงที่สกัดจาก Task Output หลังแต่ละ Task และ Recall ข้อมูลที่เกี่ยวข้องก่อน Task ถัดไป โดย Agents ใช้ Shared Crew Memory เว้นแต่ถูกกำหนด Agent-level Memory แยก. ([CrewAI Documentation][6])

อย่างไรก็ตาม โปรเจกต์หลักสูตรล็อก CrewAI `1.14.4` ขณะที่เอกสารนี้เป็น `1.15.5` ดังนั้นรายละเอียด Storage และ Unified Memory API อาจต่างกัน ควรตรวจ Trace และ Directory Storage จริงขณะรัน Lab. ([GitHub][1])

---

# 9. Memory ไม่ได้รับประกันว่าไม่เลือกซ้ำ

Prompt ระบุ:

```text
Don't pick the same company twice.
```

และเปิด Memory

แต่การ Recall Memory มักอาศัย Semantic Relevance ไม่ใช่การตรวจ Unique Constraint แบบ Database

ดังนั้นอาจเกิด:

```text
Run 1 → เลือก NVIDIA
Run 2 → Memory ไม่ Recall NVIDIA
Run 2 → เลือก NVIDIA อีก
```

หลักสำคัญ:

```text
Memory Recommendation
≠
Deterministic Constraint
```

หาก Requirement จริงคือ “ห้ามเลือกซ้ำ” ควรใช้ Code:

```python
previously_selected = load_selected_tickers()

if candidate.ticker in previously_selected:
    reject_candidate(candidate)
```

Memory ใช้ช่วยให้ Agentมีบริบท ส่วน Code หรือ Database ต้องบังคับกฎที่ห้ามละเมิด

---

# 10. Memory กับ Context ต่างกันอย่างไร

## Context

```text
Output จาก Task ที่ระบุ
→ ส่งให้ Task ถัดไปใน Run ปัจจุบัน
```

## Memory

```text
ข้อมูลที่เก็บจาก Runs/Tasks
→ ค้นคืนตามความเกี่ยวข้องในภายหลัง
```

ตัวอย่าง:

```text
Research Task Output
→ Context ของ Picker ทันที

บริษัทที่เคยเลือกสัปดาห์ก่อน
→ Recall จาก Memory
```

Context เป็น Data Dependency ที่ชัดเจน ส่วน Memory เป็น Retrieval ที่ไม่ควรถือว่า Deterministic

---

# 11. Custom Push Notification Tool

Stock Picker ได้รับ:

```python
tools=[send_push_notification]
```

Tool ถูกสร้างด้วย:

```python
@tool("Send Push Notification")
def send_push_notification(message: str) -> str:
    ...
```

มันอ่าน:

```text
PUSHOVER_USER
PUSHOVER_TOKEN
```

แล้วเรียก Pushover API ผ่าน `requests.post()`. ([GitHub][5])

Environment ที่ต้องมี:

```env
OPENAI_API_KEY=...
SERPER_API_KEY=...
PUSHOVER_USER=...
PUSHOVER_TOKEN=...
```

---

# 12. Push Notification เป็น Side Effect จริง

เมื่อ Stock Picker เรียก Tool จะส่ง Notification ออกนอก Process

```text
LLM Decision
→ Tool Arguments
→ HTTP Request
→ User Notification
```

นี่ไม่ใช่เพียงการสร้างข้อความ แต่เป็น External Action

ระหว่างพัฒนาควรใช้ Mock:

```python
@tool("Preview Push Notification")
def send_push_notification(message: str) -> str:
    print(f"[PREVIEW] {message}")
    return "Notification previewed; nothing was sent."
```

---

## จุดอ่อนของ Tool ปัจจุบัน

โค้ดคืน:

```python
return (
    "Push notification sent with "
    f"API response code: {result}"
)
```

โดย `result` เป็นเพียง HTTP Status Code และไม่มี Timeout, Exception Handling หรือการตรวจว่า Status Code อยู่ในช่วง Success จริงหรือไม่. ([GitHub][7])

ตัวอย่างเช่น API อาจตอบ `401` แต่ข้อความยังขึ้นต้นว่า:

```text
Push notification sent...
```

ควรปรับเป็น:

```python
response = requests.post(
    pushover_url,
    data=payload,
    timeout=10
)
response.raise_for_status()

return {
    "success": True,
    "status_code": response.status_code
}
```

และเพิ่ม Idempotency เพื่อป้องกัน Manager หรือ Agent เรียก Tool ซ้ำ

---

# 13. `main.py`

Runtime Input ปัจจุบันถูก Hard-code:

```python
inputs = {
    "sector": "Technology",
    "current_date": str(datetime.now().date())
}
```

ดังนั้น `crewai run` จะค้นบริษัทใน Sector Technology โดยไม่ได้ถามผู้ใช้ผ่าน Terminal. ([GitHub][8])

Placeholder ที่ใช้ใน YAML คือ:

```text
{sector}
{current_date}
```

ตัวอย่างแก้ให้รับ Input:

```python
def run():
    sector = input("Enter sector: ").strip()

    inputs = {
        "sector": sector or "Technology",
        "current_date": str(datetime.now().date())
    }

    StockPicker().crew().kickoff(inputs=inputs)
```

---

# 14. จุดผิดปกติใน Helper Functions

`run()` ใช้ Keys:

```text
sector
current_date
```

แต่ `train()`, `test()` และ `run_with_trigger()` ยังใช้:

```text
topic
current_year
```

ซึ่งไม่ตรงกับ Placeholders ใน Agent และ Task Configuration. ([GitHub][3])

ดังนั้นสำหรับ Lab นี้ให้ใช้:

```powershell
crewai run
```

ก่อน

หากจะใช้ Train/Test ต้องแก้เป็น:

```python
inputs = {
    "sector": "Technology",
    "current_date": str(datetime.now().date())
}
```

---

# 15. วิธีติดตั้งและรัน

```powershell
cd 3_crewai\reference\stock_picker
crewai install
crewai run
```

Project scripts เชื่อม `crewai run` เข้ากับ `stock_picker.main:run`. ([GitHub][9])

ระหว่างเรียนควรปิด Push จริงก่อน แล้วสังเกต:

```text
Manager มอบหมายงานอย่างไร
Finder สร้าง Search Queries อะไร
ได้บริษัทอะไรบ้าง
Structured Output ถูกต้องหรือไม่
Researcher ใช้ Source ใด
Memory Recall อะไรกลับมา
Stock Picker ใช้เกณฑ์อะไร
Tool ถูกเรียกกี่ครั้ง
```

---

# 16. Financial Risk ของ Architecture นี้

ระบบเลือก “บริษัทที่น่าสนใจจากข่าว” ไม่ใช่เลือกจาก Financial Model ที่ตรวจสอบแล้ว

Pipeline ปัจจุบันยังไม่มี:

```text
ราคาหุ้นปัจจุบันและ Valuation
งบการเงินที่ Normalize แล้ว
Risk-adjusted return
Portfolio constraints
Liquidity analysis
Regulatory filing verification
Backtesting
Human approval
```

ดังนั้นผลของ Lab ควรถูกมองเป็น **Agentic Workflow Exercise** ไม่ใช่คำแนะนำลงทุน

---

# 17. Trending ไม่เท่ากับน่าลงทุน

บริษัทอาจ Trending เพราะ:

```text
ข่าวดี
ข่าวร้าย
คดีความ
การปลดพนักงาน
ราคาหุ้นตก
ข่าวลือ
เหตุการณ์ชั่วคราว
```

ดังนั้น:

```text
Trending
≠
High-quality business

Trending
≠
Undervalued

Trending
≠
Suitable investment
```

ระบบควรแยก:

```text
News attention
Business fundamentals
Valuation
Risk
Investment suitability
```

---

# 18. Pattern ที่เรียนใน Lab นี้

## Candidate Funnel

```text
Many companies
→ Trending candidates
→ Researched candidates
→ One selected company
```

## Hierarchical Orchestration

```text
Manager
→ Delegates
→ Reviews
→ Coordinates
```

## Structured Intermediate Artifacts

```text
TrendingCompanyList
→ TrendingCompanyResearchList
→ Decision
```

## Shared Memory

```text
Past runs
→ Relevant recall
→ Current decision context
```

## Side-effect Tool

```text
Decision
→ Push Notification
```

---

# 19. แบบฝึกหัดแนะนำ

## Exercise 1 — ตรวจบริษัทซ้ำ

รัน Crew หลายครั้งและบันทึก:

```text
บริษัทที่ค้นพบ
บริษัทที่เลือก
Memory ที่ Recall
```

ตรวจว่า Prompt + Memory ห้ามซ้ำได้จริงหรือไม่

## Exercise 2 — เพิ่ม Deterministic Exclusion

เก็บ Ticker ที่เคยเลือกใน JSON หรือ Database แล้วกรอง Candidate ก่อนส่งให้ Picker

## Exercise 3 — Structured Final Decision

เพิ่ม `StockDecision` Pydantic Model ให้ Task สุดท้าย

## Exercise 4 — เพิ่ม Investment Rubric

ให้คะแนน:

```text
Market position
Revenue growth
Profitability
Valuation
Risk
Catalysts
Evidence quality
```

## Exercise 5 — Mock Push Tool

ยืนยันว่า Development Run ไม่ส่ง Notification จริง

## Exercise 6 — เปรียบเทียบ Process

ทดลอง:

```text
Process.sequential
```

เทียบกับ:

```text
Process.hierarchical
```

ตรวจ Model Calls, Cost, Latency และคุณภาพผลลัพธ์

---

# Checklist

### Hierarchical Process ต่างจาก Sequential อย่างไร

Sequential เรียก Tasks ตามลำดับโดยตรง ส่วน Hierarchical ใช้ Manager ประสาน มอบหมาย และตรวจงาน

### `output_pydantic` ทำอะไร

ทำให้ Task Output ถูกแปลงและตรวจตาม Pydantic Model

### Context กับ Memory ต่างกันอย่างไร

Context คือ Dependency ที่ระบุชัดใน Run ปัจจุบัน ส่วน Memory คือข้อมูลที่ถูกค้นคืนตามความเกี่ยวข้อง

### Memory รับประกันไม่เลือกซ้ำหรือไม่

ไม่ ต้องใช้ Code หรือ Persistent Store บังคับ Unique Constraint

### Agent ใดมี Search Tool

Trending Finder และ Financial Researcher

### Agent ใดมี Push Tool

Stock Picker

### Push Tool เป็น Side Effect หรือไม่

เป็น เพราะส่ง HTTP Request ไปยัง Pushover

### Final Decision เป็น Structured Output หรือไม่

ยังไม่เป็น Task สุดท้ายคืนรายงาน Markdown แบบ Plain Text

---

# แก่นของ Lab 3

```text
Finder
→ สร้าง Candidate

Researcher
→ สร้าง Evidence

Picker
→ สร้าง Decision

Manager
→ ควบคุมทีม

Pydantic
→ ควบคุมรูปแบบข้อมูล

Memory
→ เพิ่มบริบทจากอดีต

Code
→ ต้องควบคุมกฎที่ห้ามละเมิด

Tool
→ สร้าง Action ภายนอก
```

บทเรียนสำคัญที่สุดคือ:

> **Memory ช่วยให้ Agent “นึกถึง” สิ่งที่เคยเกิดขึ้น แต่ไม่ควรถูกใช้แทนกฎของระบบ หากบริษัทเดิมห้ามถูกเลือกซ้ำ ต้องบังคับด้วย Code หรือ Database ไม่ใช่หวังว่า Prompt และ Semantic Recall จะทำงานทุกครั้ง**

และ:

> **Hierarchical Multi-Agent System อาจดูใกล้เคียงทีมมนุษย์มากขึ้น แต่ Manager Agent เพิ่ม Model Calls, Cost, Latency และความไม่แน่นอน จึงควรใช้เมื่อการมอบหมายและตรวจงานแบบ Dynamic มีประโยชน์จริง ไม่ใช่เพียงเพราะมี Framework รองรับ**

[1]: https://github.com/ed-donner/agents/blob/main/3_crewai/reference/stock_picker/pyproject.toml "agents/3_crewai/reference/stock_picker/pyproject.toml at main · ed-donner/agents · GitHub"
[2]: https://raw.githubusercontent.com/ed-donner/agents/main/3_crewai/reference/stock_picker/src/stock_picker/config/tasks.yaml "raw.githubusercontent.com"
[3]: https://raw.githubusercontent.com/ed-donner/agents/main/3_crewai/reference/stock_picker/src/stock_picker/config/agents.yaml "raw.githubusercontent.com"
[4]: https://docs.crewai.com/en/learn/hierarchical-process "Hierarchical Process - CrewAI"
[5]: https://raw.githubusercontent.com/ed-donner/agents/main/3_crewai/reference/stock_picker/src/stock_picker/crew.py "raw.githubusercontent.com"
[6]: https://docs.crewai.com/en/concepts/memory "Memory - CrewAI"
[7]: https://github.com/ed-donner/agents/blob/main/3_crewai/reference/stock_picker/src/stock_picker/tools/push_tool.py "agents/3_crewai/reference/stock_picker/src/stock_picker/tools/push_tool.py at main · ed-donner/agents · GitHub"
[8]: https://raw.githubusercontent.com/ed-donner/agents/main/3_crewai/reference/stock_picker/src/stock_picker/main.py "raw.githubusercontent.com"
[9]: https://github.com/ed-donner/agents/tree/main/3_crewai/reference/stock_picker "agents/3_crewai/reference/stock_picker at main · ed-donner/agents · GitHub"
