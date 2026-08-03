# Episodic Learning Artifact

## Week 5 — Lab 4: Mastra with TypeScript

**หลักสูตร:** AI Engineer Agentic Track
**ผู้สอน:** Ed Donner
**Repository:** `ed-donner/agents`
**โฟลเดอร์:** `5_agent_frameworks/4_mastra`
**เอกสารหลัก:** `lab.md`
**ไฟล์ทดลอง:** `step1.ts`–`step5.ts`
**ไฟล์ประกอบ:** `worker.ts`, `board.ts`, `tools.ts`, `src/mastra/index.ts`, `scripts/dev.ts`, `SWAP_AI.md`
**หัวข้อหลัก:** Mastra, TypeScript Agent Loop, Zod Tool Schema, MCP Client, SQLite Board, Studio และ Model Portability
**สถานะ:** เรียนและสรุป Lab 4 แล้ว

---

# 1. Context

Week 5 ใช้ Business Contract เดิมกับ Agent Framework หลายตัว เพื่อเปรียบเทียบ Framework โดยไม่เปลี่ยนโจทย์

Framework ที่เรียนก่อนหน้า:

```text
Lab 1
Google ADK

Lab 2
AWS Strands
Pydantic AI

Lab 3
Microsoft Agent Framework
Agno
```

Lab 4 เปลี่ยนจาก Python ไปเป็น:

```text
TypeScript
+
Mastra
```

Contract กลางยังเหมือนเดิม:

```text
อ่าน Goal จาก SQLite Board
        ↓
สร้าง Steps
        ↓
อ่าน notes.txt
        ↓
แปลเนื้อหาเป็นภาษาสเปน
        ↓
เขียน spanish.txt
        ↓
ปิด Steps
        ↓
ปิด Goal
```

สิ่งที่คงเดิม:

```text
Goal
Todo schema
Board state
Workspace
notes.txt
spanish.txt
Filesystem MCP
Agent instructions
Definition of work
```

สิ่งที่เปลี่ยน:

```text
Programming language
Agent API
Tool declaration
Runtime validation
MCP lifecycle
Development UI
Model adapter
```

---

# 2. Learning Objectives

หลังจบ Lab 4 สามารถอธิบายได้ว่า:

1. Mastra สร้าง Agent ด้วย `id`, `name`, `instructions` และ `model` อย่างไร
2. Model routing string เช่น `openai/gpt-5.4-mini` ทำงานอย่างไร
3. `agent.generate()` เริ่ม Model–Tool Loop อย่างไร
4. Agent ที่ไม่มี Tools แตกต่างจาก Tool-using Agent อย่างไร
5. Mastra Tools สร้างด้วย `createTool()` อย่างไร
6. Zod ใช้สร้าง Runtime Input Schema อย่างไร
7. Tool variable name และ Tool ID ต่างกันอย่างไร
8. TypeScript Board ใช้ `node:sqlite` อย่างไร
9. SQLite Board แตกต่างจาก Agent State อย่างไร
10. `MCPClient` เชื่อม Filesystem MCP Server อย่างไร
11. `listTools()` ดึง Tools จาก MCP Server อย่างไร
12. `disconnect()` ปิด MCP Lifecycle อย่างไร
13. Function Tools และ MCP Tools ถูกรวมใน Agent อย่างไร
14. `maxSteps` ทำหน้าที่เป็น Execution Budget อย่างไร
15. `onStepFinish` ใช้สังเกต Agent Loop อย่างไร
16. Terminal Worker และ Studio Worker แตกต่างกันอย่างไร
17. Mastra Studio ช่วย Debug และ Trace Agent อย่างไร
18. Vercel AI SDK ช่วย Model Portability อย่างไร
19. Zod Validation ต่างจาก Business Validation อย่างไร
20. TypeScript เปลี่ยน Syntax แต่ไม่เปลี่ยน Agent Pattern อย่างไร

---

# 3. Prerequisites

ควรเข้าใจแนวคิดจาก Labs ก่อนหน้า:

```text
Model
System instructions
Tool schema
Tool call
Tool result
Agent loop
MCP
SQLite board
Goal and steps
External artifact
Shared worker contract
```

Environment:

```text
Node.js 24 LTS
npm
npx
TypeScript
OpenAI API key
```

Environment Variable:

```env
OPENAI_API_KEY=...
```

เข้าสู่โฟลเดอร์:

```powershell
cd 5_agent_frameworks/4_mastra
```

ติดตั้ง Dependencies:

```powershell
npm install
```

ตรวจ Environment:

```powershell
node --version
npm --version
npx --version
```

Dependencies หลัก:

```text
@mastra/core
@mastra/mcp
@ai-sdk/openai
zod
tsx
typescript
dotenv
```

---

# 4. Lab Structure

Lab นี้ไม่ได้ใช้ Jupyter Notebook

แต่ใช้ TypeScript Programs ห้าตัว:

```text
step1.ts
step2.ts
step3.ts
step4.ts
step5.ts
```

Mapping:

```text
Step 1
Create the agent

Step 2
Run it

Step 3
Add tools

Step 4
Add MCP

Step 5
Put it in a loop with a goal
```

รัน:

```powershell
npm run step1
npm run step2
npm run step3
npm run step4
npm run step5
```

---

# 5. Mastra Mental Model

```text
Mastra Agent
=
Identity
+ Instructions
+ Model
+ Tools
+ Model-driven loop
```

Flow:

```text
User assignment
        ↓
Mastra Agent
        ↓
Model decides
        ├── Return text
        ├── Call tool
        ├── Read tool result
        └── Continue
```

Agent Pattern ยังเหมือน Python Frameworks:

```text
Model
→ Tool request
→ Tool execution
→ Observation
→ Next model decision
→ Completion
```

---

# 6. Step 1 — Create the Agent

```typescript
import "./env.ts";
import { Agent } from "@mastra/core/agent";

const agent = new Agent({
  id: "assistant",
  name: "Assistant",
  instructions:
    "You are a concise, friendly assistant. " +
    "Reply in a single short sentence.",
  model: "openai/gpt-5.4-mini",
});
```

องค์ประกอบ:

```text
id
→ Stable identifier

name
→ Human-readable name

instructions
→ System-level guidance

model
→ Provider/model routing
```

ขั้นนี้ยังไม่มี Model Call

```text
Agent created
≠ Agent executed
```

---

# 7. `id` vs `name`

```typescript
id: "assistant",
name: "Assistant",
```

ความต่าง:

```text
id
= Programmatic identity

name
= Display identity
```

`id` อาจถูกใช้ใน:

```text
Agent registry
Studio
Tracing
API endpoint
A2A discovery
```

`name` ใช้เพื่อให้มนุษย์อ่านง่าย

---

# 8. Model Routing String

```typescript
model: "openai/gpt-5.4-mini"
```

รูปแบบ:

```text
provider/model
```

ดังนั้น:

```text
openai
→ Provider

gpt-5.4-mini
→ Model identifier
```

Mastra ใช้ Vercel AI SDK เป็น Model Layer

ข้อดี:

```text
ตั้งค่าง่าย
เปลี่ยน Model ง่าย
Provider abstraction ชัด
ใช้ Model ecosystem ของ Vercel AI SDK
```

แต่:

```text
API portability
≠ Behavioral portability
```

Model ต่างตัวอาจเลือก Tools หรือวางแผนต่างกัน

---

# 9. Step 2 — Run the Agent

```typescript
const reply = await agent.generate(
  "Say hello in Spanish.",
);

console.log(reply.text);
```

Final Text:

```typescript
reply.text
```

ตอนยังไม่มี Tools:

```text
User
→ Model
→ Text
```

จึงยังเป็น LLM Call ที่ถูกห่อด้วย Agent API

---

# 10. `generate()` and the Agent Loop

เมื่อไม่มี Tools:

```text
generate()
→ Model response
```

เมื่อมี Tools:

```text
generate()
→ Model call
→ Tool request
→ Tool execution
→ Tool result
→ Next model call
→ Final response
```

Method เดียวกันรองรับทั้ง:

```text
Simple LLM response
และ
Multi-step tool loop
```

---

# 11. `process.exit(0)`

Step Scripts ใช้:

```typescript
process.exit(0);
```

เพราะ Mastra อาจยังคงเปิด Model Connection Pool หลังงานเสร็จ

เหมาะกับ:

```text
One-shot terminal script
```

ไม่เหมาะกับ:

```text
Long-running server
Shared worker process
Request-serving application
```

เพราะจะปิด Process ทั้งหมดทันที

---

# 12. TypeScript SQLite Board

`board.ts` ใช้:

```typescript
import { DatabaseSync } from "node:sqlite";
```

Schema:

```sql
CREATE TABLE todos (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    parent_id INTEGER,
    task TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending',
    result TEXT NOT NULL DEFAULT ''
)
```

Todo Types:

```text
Goal
→ parent_id = null

Step
→ parent_id = goal.id
```

Status:

```text
pending
→ in_progress
→ done
```

---

# 13. Board Functions

```text
resetBoard()
→ สร้าง Board ใหม่

addGoal()
→ เพิ่ม Goal

addStep()
→ เพิ่ม Step

listTodos()
→ อ่าน Board

claimTodo()
→ เปลี่ยน Status เป็น in_progress

completeTodo()
→ เปลี่ยน Status เป็น done

showBoard()
→ แสดง Board ให้มนุษย์อ่าน
```

Mastra Agent ใช้ Board ผ่าน Tool Layer

ไม่ได้เข้าถึง Board Functions ทุกตัวโดยตรง

---

# 14. `BOARD_PATH`

```typescript
export const BOARD_PATH =
  process.env.BOARD_PATH ??
  join(
    dirname(fileURLToPath(import.meta.url)),
    "board.sqlite",
  );
```

Standalone Mode:

```text
BOARD_PATH ไม่ถูกตั้ง
→ ใช้ board.sqlite ภายใน Lab
```

Shared-board Mode:

```text
Orchestrator ตั้ง BOARD_PATH
→ Worker เปิด Shared Board
```

ค่าอ่านตอน Module ถูก Import

ดังนั้น:

```text
Environment configuration
ต้องเกิดก่อน import board.ts
```

---

# 15. `node:sqlite`

ข้อดี:

```text
มากับ Node
ไม่ต้องติดตั้ง Database package เพิ่ม
Code สั้น
เหมาะกับ Demo
```

ข้อจำกัด:

```text
ใช้ Synchronous API
Block Event Loop ขณะ Query
ทุก Function เปิดและปิด Connection ใหม่
ไม่เหมาะกับ High-throughput server
```

ใน Lab งานมีขนาดเล็ก จึงยังใช้งานได้เหมาะสม

---

# 16. WAL and Busy Timeout

Board ใช้:

```sql
PRAGMA journal_mode=WAL
PRAGMA busy_timeout=5000
```

ช่วย:

```text
ลด database locked errors
รองรับ Readers หลายตัว
ปรับปรุง concurrent access
```

แต่:

```text
Database concurrency
≠ Task ownership
```

WAL ไม่ได้ป้องกัน Workers หลายตัว Claim Task เดียวกัน

---

# 17. Step 3 — Create Mastra Tools

Mastra Tool สร้างด้วย:

```typescript
createTool()
```

Imports:

```typescript
import { createTool } from "@mastra/core/tools";
import { z } from "zod";
```

Tool Structure:

```text
id
description
inputSchema
execute
```

Mental Model:

```text
Tool metadata
+
Runtime schema
+
Implementation
```

---

# 18. `show_todos`

```typescript
export const showTodos = createTool({
  id: "show_todos",

  description:
    "List every todo on the board.",

  inputSchema:
    z.object({}),

  execute: async () => ({
    todos: listTodos(),
  }),
});
```

ไม่มี Input Arguments:

```typescript
z.object({})
```

ผลลัพธ์:

```text
todos
→ List of board records
```

---

# 19. `plan_steps`

```typescript
export const planSteps = createTool({
  id: "plan_steps",

  description:
    "Break a goal into an ordered checklist.",

  inputSchema: z.object({
    goalId: z.number(),
    steps: z.array(
      z.string(),
    ),
  }),

  execute: async ({
    goalId,
    steps,
  }) => ({
    goalId,

    stepIds:
      steps.map(
        (step) =>
          addStep(
            goalId,
            step,
          ),
      ),
  }),
});
```

Zod ตรวจ:

```text
goalId
→ number

steps
→ array of strings
```

---

# 20. `complete_task`

```typescript
export const completeTask = createTool({
  id: "complete_task",

  description:
    "Mark a todo done and record a result.",

  inputSchema: z.object({
    taskId: z.number(),
    result: z.string(),
  }),

  execute: async ({
    taskId,
    result,
  }) => {
    completeTodo(
      taskId,
      result,
    );

    return {
      taskId,
      status: "done",
    };
  },
});
```

Zod ช่วยให้ Tool ไม่รับ Input ที่ผิด Shape

แต่ไม่ได้ยืนยันว่าการปิด Task ถูกต้องตาม Business Rule

---

# 21. Zod Schema

Zod เป็น Runtime Validation Library

แตกต่างจาก TypeScript Type:

```text
TypeScript type
→ ตรวจตอนพัฒนาและ Compile

Zod schema
→ ตรวจข้อมูลตอน Runtime
```

Model Tool Arguments มาจากข้อมูล Runtime

จึงต้องใช้ Runtime Schema เช่น Zod

---

# 22. Zod Is Not Business Validation

Zod ตรวจว่า:

```text
taskId เป็น number
result เป็น string
steps เป็น array
```

แต่ไม่ตรวจว่า:

```text
Task ID มีอยู่จริง
Task เป็นของ Worker นี้
Goal มี Steps ค้างหรือไม่
spanish.txt ถูกสร้างหรือไม่
Translation ถูกต้องหรือไม่
```

ดังนั้น:

```text
Schema-valid
≠ Business-valid
```

---

# 23. Tool Map

Mastra ส่ง Tools เป็น Object:

```typescript
tools: {
  showTodos,
  completeTask,
}
```

ต่างจากหลาย Python Frameworks ที่ใช้:

```python
tools=[
    show_todos,
    complete_task,
]
```

Tool Map ทำให้:

```text
TypeScript variable names
ผูกกับ
Tool objects
```

และสามารถรวม Tools ด้วย Object Spread ได้

---

# 24. Tool Variable Name vs Tool ID

```typescript
showTodos
```

คือ TypeScript Variable Name

```typescript
id: "show_todos"
```

คือ Agent-facing Tool ID

Mental Model:

```text
showTodos
= Developer-facing identifier

show_todos
= Model-facing capability name
```

ช่วยให้ Code ใช้ `camelCase` แต่ Tool Protocol ใช้ชื่อรูปแบบอื่นได้

---

# 25. Add Tools to Agent

```typescript
const boardAgent = new Agent({
  id: "board-agent",
  name: "Board Agent",

  instructions:
    "You help manage a shared todo board.",

  model:
    "openai/gpt-5.4-mini",

  tools: {
    showTodos,
    completeTask,
  },
});
```

เมื่อถาม:

```text
What is on the board?
```

Agent อาจ:

```text
Recognize need for current state
→ Call show_todos
→ Read board result
→ Answer
```

นี่คือ Agent Loop รอบแรก

---

# 26. Step 4 — MCP Client

Mastra ใช้:

```typescript
import { MCPClient } from "@mastra/mcp";
```

สร้าง Filesystem MCP:

```typescript
export function makeFilesystem(
  dir = WORKSPACE,
): MCPClient {

  return new MCPClient({
    servers: {
      filesystem: {
        command: "npx",

        args: [
          "-y",
          "@modelcontextprotocol/" +
          "server-filesystem",
          dir,
        ],

        stderr: "ignore",
        cwd: dir,
      },
    },
  });
}
```

---

# 27. MCP Architecture

```text
Mastra Agent
    ↓
MCPClient
    ↓ stdio
Node Filesystem MCP Server
    ↓
Workspace
```

MCP Server ให้ Tools เช่น:

```text
List files
Read file
Write file
Edit file
Create directory
```

Tool Names จริงขึ้นกับ MCP Server Version

---

# 28. `cwd`

```typescript
cwd: dir
```

ทำให้ Relative File Paths ถูก Resolve ภายใน Workspace

ตัวอย่าง:

```text
notes.txt
→ <workspace>/notes.txt

spanish.txt
→ <workspace>/spanish.txt
```

หากไม่มี `cwd` Agent และ MCP Server อาจตีความ Relative Path ไม่ตรงกัน

---

# 29. `stderr: "ignore"`

```typescript
stderr: "ignore"
```

ช่วย:

```text
ซ่อน Startup Banner
ลด Console Noise
เลี่ยง Child-process stderr issue
```

ข้อดีของ Mastra คือ Option นี้อยู่ใน Public Configuration

ไม่ต้อง:

```text
Subclass
Monkeypatch
แก้ Internal API
```

ข้อเสีย:

```text
MCP startup errors อาจถูกซ่อน
Debugging ยากขึ้น
```

---

# 30. `listTools()`

```typescript
const filesystem =
  makeFilesystem();

const tools =
  await filesystem.listTools();
```

Conceptual Flow:

```text
Connect MCP server
→ Discover tool catalog
→ Convert MCP schemas
→ Return Mastra Tool Map
```

จากนั้น:

```typescript
tools:
  await filesystem.listTools()
```

ทำให้ Agent ใช้ MCP Tools ได้

---

# 31. `disconnect()`

หลังใช้งาน:

```typescript
await filesystem.disconnect();
```

Lifecycle:

```text
Create client
→ Connect/list tools
→ Run agent
→ Disconnect
```

ถ้าไม่ Disconnect:

```text
MCP child process อาจยังทำงาน
Node process อาจไม่จบ
Resource อาจรั่ว
Run ต่อไปอาจชน Process เดิม
```

Mastra จึงให้ Developer จัด Lifecycle อย่างชัดเจน

---

# 32. Function Tools and MCP Tools

Board Tools:

```typescript
export const boardTools = {
  showTodos,
  planSteps,
  completeTask,
};
```

MCP Tools:

```typescript
await filesystem.listTools()
```

รวม:

```typescript
tools: {
  ...boardTools,
  ...(await filesystem.listTools()),
}
```

Agent จึงเห็น Capability Surface เดียว:

```text
Board operations
+
Filesystem operations
```

---

# 33. Step 5 — Goal-driven Agent

```typescript
const worker = new Agent({
  id: "worker",
  name: "Worker",

  instructions:
    INSTRUCTIONS,

  model:
    "openai/gpt-5.4-mini",

  tools: {
    ...boardTools,
    ...(await filesystem.listTools()),
  },
});
```

รัน:

```typescript
await worker.generate(
  "Please work the pending goal on the board.",
  {
    maxSteps: 25,
  },
);
```

---

# 34. Expected Agent Loop

```text
show_todos
    ↓
Find active goal
    ↓
plan_steps
    ↓
Read notes.txt
    ↓
Translate content
    ↓
Write spanish.txt
    ↓
complete_task(step)
    ↓
Repeat
    ↓
complete_task(goal)
```

สรุป:

```text
Read
→ Plan
→ Act
→ Check off
→ Repeat
```

---

# 35. `maxSteps`

```typescript
maxSteps: 25
```

ทำหน้าที่เป็น Execution Budget

ช่วยป้องกัน:

```text
Infinite loop
Excessive tool calls
Unbounded cost
Excessive latency
```

แต่:

```text
Stopped within budget
≠ Task succeeded
```

หาก Agent หยุดเพราะถึง Limit งานอาจยังไม่เสร็จ

---

# 36. Application Lifecycle

ก่อนรัน Worker:

```typescript
mkdirSync(
  WORKSPACE,
  { recursive: true },
);

rmSync(
  join(
    WORKSPACE,
    "spanish.txt",
  ),
  { force: true },
);

resetBoard();

const goalId =
  addGoal(GOAL);

claimTodo(goalId);
```

Application เป็นผู้:

```text
สร้าง Workspace
ลบ Output เดิม
Reset Board
Seed Goal
Claim Goal
Start Agent
```

Agent เป็นผู้:

```text
อ่าน Goal
สร้าง Steps
ใช้ Tools
สร้าง Artifact
ปิด Tasks
```

หลัก:

```text
Application
= Lifecycle authority

Agent
= Flexible executor
```

---

# 37. `worker.ts`

`worker.ts` คือ Terminal Worker สำหรับ:

```text
Standalone Lab
และ
Day 5 Shared Orchestration
```

Standalone:

```powershell
npm run worker
```

ทำงาน:

```text
Reset board
→ Seed goal
→ Claim goal
→ Run agent
→ Print tool calls
→ Show board
→ Show spanish.txt
```

---

# 38. `onStepFinish`

```typescript
await worker.generate(
  message,
  {
    maxSteps: 25,

    onStepFinish: (
      step,
    ) => {
      for (
        const call
        of step.toolCalls ?? []
      ) {
        console.log(
          call.payload.toolName,
          call.payload.args,
        );
      }
    },
  },
);
```

ช่วยสังเกต:

```text
Tool name
Tool arguments
Tool order
Number of steps
```

นี่คือ Runtime Observability

แต่:

```text
Tool call observed
≠ Tool result correct
```

---

# 39. Standalone Mode

เมื่อไม่มี Task ID:

```text
Worker owns:
Board reset
Goal creation
Workspace cleanup
Execution
Output display
```

Message:

```text
Please work the pending goal on the board.
```

---

# 40. Shared-board Mode

Worker รับ:

```text
Task ID
Board path
Environment variables
```

Worker จะ:

```text
ใช้ Shared Board
ใช้ Shared Workspace
Claim Task ที่ระบุ
ทำเฉพาะ Task นั้น
ปิด Task แล้วหยุด
```

Message:

```text
Work only task #<id>
Build and check its result
Mark that task done
Then stop
```

---

# 41. Shared-board Configuration

`BOARD_PATH` ถูกอ่านใน `board.ts` ตอน Import

ดังนั้น Orchestrator ต้องตั้ง:

```text
BOARD_PATH
```

ก่อนเริ่ม `worker.ts`

Manual Example:

```powershell
$env:BOARD_PATH =
  "C:\shared\board.sqlite"

npx tsx worker.ts `
  3 `
  "C:\shared\board.sqlite"
```

Argument ที่สองมีไว้เป็น Worker Contract แต่ Code พึ่ง Environment Variable เป็นหลักสำหรับ Board Module

---

# 42. `WORKER_MODEL`

```typescript
model:
  "openai/" +
  (
    process.env.WORKER_MODEL ??
    "gpt-5.4-mini"
  )
```

ช่วยให้ Orchestrator เปลี่ยน Model ID ได้

ตัวอย่าง:

```powershell
$env:WORKER_MODEL =
  "gpt-5.4-mini"
```

แต่ Provider ยังคงเป็น:

```text
openai
```

ถ้าจะเปลี่ยน Endpoint หรือ Provider ต้องสร้าง Model Provider Object เพิ่ม

---

# 43. Model Portability

```typescript
import {
  createOpenAI,
} from "@ai-sdk/openai";

const provider =
  createOpenAI({
    baseURL:
      process.env.OPENAI_BASE_URL,

    apiKey:
      process.env.OPENAI_API_KEY,
  });

const worker =
  new Agent({
    id: "worker",
    name: "Worker",
    instructions: INSTRUCTIONS,

    model:
      provider(
        "gpt-5.4-mini",
      ),

    tools: {
      ...boardTools,
      ...(await filesystem.listTools()),
    },
  });
```

สิ่งที่ไม่เปลี่ยน:

```text
Board
Tool schemas
MCP server
Instructions
Worker contract
```

---

# 44. Model Portability Is Not Behavior Portability

Model ใหม่อาจ:

```text
สร้าง Steps ต่างกัน
เรียก Tools ต่างลำดับ
ใช้ Calls มากกว่า
ปิด Goal เร็วกว่า
สร้าง Translation ต่างกัน
```

ดังนั้นหลัง Model Swap ต้องทดสอบ:

```text
Tool selection
Planning
Budget
Artifacts
Completion behavior
```

---

# 45. Mastra Studio

เริ่ม:

```powershell
npm run dev
```

เปิด:

```text
http://localhost:4111
```

Studio แสดง:

```text
Agent interaction
Model steps
Tool calls
Tool results
Execution traces
Model settings
```

จุดเด่น:

```text
Visual development
Live tracing
Agent discovery
Easy experimentation
```

---

# 46. Agent Registration for Studio

```typescript
export const worker =
  new Agent({
    id: "worker",
    name: "Worker",
    instructions: "...",
    model:
      "openai/gpt-5.4-mini",
    tools:
      boardTools,
  });

export const mastra =
  new Mastra({
    agents: {
      worker,
    },
  });
```

`Mastra` ทำหน้าที่เป็น Agent Registry

Studio ใช้ Registry นี้เพื่อค้นพบและแสดง Agent

---

# 47. Terminal Worker vs Studio Worker

## Terminal Worker

มี:

```text
Board tools
Filesystem MCP tools
notes.txt
spanish.txt
```

## Studio Worker

มี:

```text
Board tools
```

ไม่มี Filesystem MCP

เหตุผล:

```text
Studio bundle รันจาก .mastra/output
และไม่สามารถ spawn npx ได้ตาม Setup ปัจจุบัน
```

Studio Goal จึงใส่ข้อความต้นฉบับไว้ใน Board โดยตรง

ดังนั้น:

```text
Studio demonstration
≠ Full terminal MCP pipeline
```

---

# 48. Studio Board Reset

เมื่อ Studio Module ถูก Load:

```typescript
resetBoard();
claimTodo(
  addGoal(...),
);
```

ผล:

```text
Startup หรือ reload
→ Board reset
→ New goal seeded
```

เหมาะกับ Demo

ไม่เหมาะกับ Durable Application เพราะ Development Reload อาจลบ State เดิม

---

# 49. Native A2A Preview

Lab ระบุว่า Agent ที่ Register ใน Mastra สามารถถูก Expose ผ่าน Native A2A Support พร้อม:

```text
Agent Card
JSON-RPC endpoint
```

แต่ Lab นี้ไม่ได้ทดสอบ A2A Pipeline จริง

จึงควรถือว่าเป็น:

```text
Framework capability preview
```

ไม่ใช่ส่วนหนึ่งของ Acceptance Test ของ Worker

---

# 50. Shared Weakness — `complete_task`

```typescript
execute: async ({
  taskId,
  result,
}) => {
  completeTodo(
    taskId,
    result,
  );

  return {
    taskId,
    status: "done",
  };
}
```

ไม่ได้ตรวจ:

```text
Task ID มีอยู่
Task เป็นของ Worker นี้
Steps ทั้งหมดเสร็จ
Artifact มีอยู่
Artifact ไม่ว่าง
Translation ถูกต้อง
```

Zod Validation จึงไม่เพียงพอ

---

# 51. Shared Weakness — Task Claim

```typescript
UPDATE todos
SET status = 'in_progress'
WHERE id = ?
```

Workers หลายตัวสามารถ Claim Task เดียวกันได้

Safer Query:

```sql
UPDATE todos
SET status = 'in_progress'
WHERE id = ?
  AND status = 'pending'
```

แล้วตรวจ:

```text
changes == 1
```

สรุป:

```text
SQLite locking
≠ Task ownership
```

---

# 52. Shared Weakness — Filesystem

MCP จำกัด Root อยู่ใน Workspace

แต่ Agent ยังสามารถ:

```text
เขียนทับ notes.txt
ลบ Artifact
สร้างไฟล์ผิด
อ่านคำสั่งอันตรายในไฟล์
```

ดังนั้น:

```text
Workspace boundary
≠ Full security sandbox
```

---

# 53. MCP Version Drift

Command:

```text
npx -y @modelcontextprotocol/server-filesystem
```

ไม่ได้ Pin Version

Run ใหม่อาจได้ Version ใหม่ที่:

```text
เปลี่ยน Tool names
เปลี่ยน Schemas
เปลี่ยน Behavior
เพิ่ม Breaking Changes
```

Production ควร Pin Dependency Version ที่ทดสอบแล้ว

---

# 54. Hidden MCP Errors

```typescript
stderr: "ignore"
```

อาจซ่อน:

```text
Package installation error
Permission failure
Invalid directory
Protocol warning
Server crash
```

Production ควรใช้:

```text
Structured logs
Health checks
Exit-code monitoring
Startup timeout
```

---

# 55. `maxSteps` Is Not Definition of Done

`maxSteps` กำหนดว่า Agent ทำได้กี่รอบ

ไม่ได้กำหนดว่า:

```text
Board ถูกต้อง
ทุก Step สำเร็จ
Goal ควรปิด
Artifact ถูกต้อง
```

Safer Architecture:

```text
Agent run
    ↓
Board validator
    ↓
Artifact validator
    ↓
Goal completion gate
```

---

# 56. Deterministic Validator Example

```typescript
function validateResult(): void {
  const todos =
    listTodos();

  const unfinished =
    todos.filter(
      (todo) =>
        todo.status !== "done",
    );

  if (
    unfinished.length > 0
  ) {
    throw new Error(
      "Unfinished todos remain",
    );
  }

  const output =
    join(
      WORKSPACE,
      "spanish.txt",
    );

  if (
    !existsSync(output)
  ) {
    throw new Error(
      "spanish.txt is missing",
    );
  }

  const content =
    readFileSync(
      output,
      "utf-8",
    ).trim();

  if (!content) {
    throw new Error(
      "spanish.txt is empty",
    );
  }
}
```

---

# 57. Mastra vs Previous Frameworks

| Framework   | Tool declaration     | Run method       | MCP lifecycle                  |
| ----------- | -------------------- | ---------------- | ------------------------------ |
| Google ADK  | Typed functions      | Runner           | `McpToolset`                   |
| Strands     | `@tool`              | `invoke_async()` | `MCPClient`                    |
| Pydantic AI | Plain functions      | `run()`          | `async with agent`             |
| MAF         | Plain functions      | `run()`          | `async with MCPStdioTool`      |
| Agno        | Plain functions      | `arun()`         | `async with MCPTools`          |
| Mastra      | `createTool()` + Zod | `generate()`     | `listTools()` + `disconnect()` |

Core Loop:

```text
Model
→ Tool
→ Observation
→ Model
→ Completion
```

ยังเหมือนกันทั้งหมด

---

# 58. Major Mastra Strengths

```text
TypeScript-native
Zod runtime schemas
Vercel AI SDK integration
Clean MCP options
Mastra Studio
Web ecosystem compatibility
Agent registry
Native A2A direction
```

เหมาะกับ:

```text
TypeScript teams
Web applications
Node backends
Teams using Zod
Vercel AI SDK users
Agent UI and tracing workflows
```

---

# 59. Major Caveats

```text
Framework evolves quickly
Node version matters
MCP disconnect is manual
Studio and terminal tools differ
Studio reload resets board
Zod does not enforce business rules
Synchronous SQLite blocks Event Loop
MCP server version is unpinned
One-shot scripts force process exit
```

---

# 60. When to Choose Mastra

เหมาะเมื่อ:

```text
ทีมหลักใช้ TypeScript
ต้องการ Web-native agent framework
ใช้ Zod อยู่แล้ว
ต้องการ Vercel AI SDK
ต้องการ Studio สำหรับ development
ต้องการ agent features ใน Node application
```

ไม่ใช่เหตุผลเพียงพอที่จะเลือก:

```text
Code สั้นกว่าเล็กน้อย
Studio ดูสวย
ใช้ TypeScript แล้วต้องปลอดภัยเสมอ
```

ควรพิจารณาเพิ่ม:

```text
Persistence
Deployment
Security
MCP lifecycle
Validation
Observability
Framework stability
```

---

# 61. Misconceptions Corrected

## ความเข้าใจคลาดเคลื่อนที่ 1

> เปลี่ยนเป็น TypeScript แล้ว Agent Pattern เปลี่ยน

**ข้อเท็จจริง:**
ยังเป็น Model–Tool–Observation Loop เหมือนเดิม

---

## ความเข้าใจคลาดเคลื่อนที่ 2

> TypeScript Types ตรวจ Tool Arguments ตอน Runtime

**ข้อเท็จจริง:**
ต้องใช้ Runtime Schema เช่น Zod

---

## ความเข้าใจคลาดเคลื่อนที่ 3

> Zod ทำให้ Tool Operation ถูกต้อง

**ข้อเท็จจริง:**
Zod ตรวจ Data Shape ไม่ได้ตรวจ Business Invariants

---

## ความเข้าใจคลาดเคลื่อนที่ 4

> Studio Worker เหมือน Terminal Worker

**ข้อเท็จจริง:**
Studio Worker ไม่มี Filesystem MCP ใน Setup ปัจจุบัน

---

## ความเข้าใจคลาดเคลื่อนที่ 5

> `maxSteps` รับประกันว่างานเสร็จ

**ข้อเท็จจริง:**
เพียงจำกัด Agent Loop

---

## ความเข้าใจคลาดเคลื่อนที่ 6

> `onStepFinish` พิสูจน์ว่า Tool สำเร็จ

**ข้อเท็จจริง:**
แสดง Tool Calls แต่ยังต้องตรวจผลลัพธ์จริง

---

## ความเข้าใจคลาดเคลื่อนที่ 7

> Workspace Scope คือ Full Sandbox

**ข้อเท็จจริง:**
Agent ยังแก้ไฟล์ทั้งหมดภายใน Workspace ได้

---

## ความเข้าใจคลาดเคลื่อนที่ 8

> Model Routing ทำให้ทุก Model มี Behavior เหมือนกัน

**ข้อเท็จจริง:**
API เหมือนกันได้ แต่ Tool-use Behavior อาจต่างกัน

---

# 62. Risks Identified

## 62.1 Premature Completion

Agent ปิด Goal ก่อน Steps ครบ

## 62.2 Missing Artifact

Board Done แต่ไม่มี `spanish.txt`

## 62.3 Invalid Translation

ไฟล์มีอยู่แต่เนื้อหาผิด

## 62.4 Duplicate Claim

Workers ทำ Task เดียวกันพร้อมกัน

## 62.5 MCP Process Leak

ไม่ได้เรียก `disconnect()`

## 62.6 Hidden MCP Error

`stderr` ถูก Ignore

## 62.7 Version Drift

MCP Package หรือ Mastra API เปลี่ยน

## 62.8 Studio/Terminal Divergence

ทดสอบคนละ Tool Pipeline

## 62.9 State Reset

Studio Reload ลบ Board เดิม

## 62.10 Event-loop Blocking

Synchronous SQLite Block Node Event Loop

## 62.11 File Prompt Injection

Agent อ่าน Instructions อันตรายจากไฟล์

## 62.12 Model Behavior Drift

เปลี่ยน Model แล้ว Planning เปลี่ยน

---

# 63. Production Improvements

```text
Atomic task claiming
Goal-completion validator
Artifact validation
Workspace per task
Filesystem permissions
Pinned MCP version
Structured MCP logging
Guaranteed disconnect
Tool-call budget
Model-call budget
Task timeout
Persistent state
Audit trail
Acceptance tests
```

---

# 64. Suggested Exercise — Change the Goal

```typescript
const GOAL =
  "Write a short haiku about Madrid " +
  "into madrid.txt.";
```

ตรวจว่า Agent:

```text
สร้าง Plan เหมาะสม
เลือก File Tools ถูก
สร้าง madrid.txt จริง
ปิด Steps ครบ
ปิด Goal หลังงานเสร็จ
```

---

# 65. Suggested Exercise — Goal Gate

สร้าง:

```typescript
completeGoal
```

ให้ตรวจ:

```text
ทุก Step เป็น done
Required File มีอยู่
File ไม่ว่าง
Task ownership ถูกต้อง
```

ก่อนปิด Goal

---

# 66. Suggested Exercise — Atomic Claim

แก้:

```typescript
claimTodo()
```

ให้ Claim เฉพาะ Task ที่เป็น `pending`

จากนั้นรัน Workers สองตัวกับ Task เดียวกัน

---

# 67. Suggested Exercise — MCP Logging

เปลี่ยน:

```typescript
stderr: "ignore"
```

เป็น Structured Log Destination

จากนั้นทำให้ MCP Server เริ่มไม่สำเร็จและตรวจ Error

---

# 68. Suggested Exercise — Terminal vs Studio

รัน:

```powershell
npm run worker
```

และ:

```powershell
npm run dev
```

เปรียบเทียบ:

```text
Tool set
MCP availability
Board input
File output
Trace visibility
Definition of done
```

---

# 69. Patterns Learned

## TypeScript Agent Pattern

```text
Agent
+ Model routing
+ Tool object map
```

## Runtime Schema Pattern

```text
Tool input
→ Zod validation
→ execute()
```

## MCP Discovery Pattern

```text
MCPClient
→ listTools()
→ Agent tools
```

## Explicit Cleanup Pattern

```text
Create client
→ Use tools
→ disconnect()
```

## Execution Budget Pattern

```text
maxSteps
→ Bound agent loop
```

## Step Observability Pattern

```text
onStepFinish
→ Inspect tool calls
```

## Agent Registry Pattern

```text
Mastra instance
→ Registered agents
→ Studio
```

---

# 70. Connection to Week 5 Lab 1

Google ADK:

```text
Python
LlmAgent
Runner
Function tools
McpToolset
```

Mastra:

```text
TypeScript
Agent
generate()
createTool
MCPClient
```

แม้ API ต่างกัน:

```text
Agent pattern
ยังเหมือนเดิม
```

---

# 71. Connection to Week 5 Lab 2

Pydantic AI เน้น:

```text
Python type contracts
Pydantic validation
```

Mastra เน้น:

```text
TypeScript types
Zod runtime schemas
```

ทั้งสองแสดงว่า:

```text
Typed schema
ช่วย Data Contract

แต่
ไม่แทน Business Validation
```

---

# 72. Connection to Week 5 Lab 3

Agno เน้น:

```text
Lightweight agent
AgentOS
```

Mastra เน้น:

```text
TypeScript-native SDK
Mastra Studio
```

ทั้งสองมี Development/Serving Ecosystem รอบ Agent

แต่ Application Correctness ยังต้องสร้างเอง

---

# 73. Connection to Week 4

## LangChain `create_agent`

คล้าย Mastra Agent:

```text
Model
+ Tools
→ Framework-managed loop
```

## LangGraph

ช่วยอธิบาย Agent Loop ภายใน:

```text
Model node
Tool node
Cycle
Termination
```

## Deep Agents

คล้าย Mastra Worker ในเรื่อง:

```text
Planning
Filesystem
Artifacts
Goal-driven work
```

แต่ Mastra Lab สร้าง Planning State ผ่าน SQLite Board

---

# 74. Lab 4 Mental Model

```text
Application
    ↓
Reset Board
    ↓
Seed and claim Goal
    ↓
Mastra Agent
    ├── Board Tools
    └── Filesystem MCP Tools
            ↓
    Read notes.txt
            ↓
    Translate
            ↓
    Write spanish.txt
            ↓
    Update Steps
            ↓
    Close Goal
```

Studio Variation:

```text
Studio
→ Registered Agent
→ Board Tools only
→ Text embedded in Goal
→ Translation recorded in Goal result
```

---

# 75. Final Lessons

## Lesson 1

การเปลี่ยนจาก Python เป็น TypeScript ไม่ได้เปลี่ยน Agent Loop

## Lesson 2

Mastra Agent ใช้ `id`, `name`, `instructions`, `model` และ `tools`

## Lesson 3

`generate()` รองรับทั้ง Simple Model Call และ Tool Loop

## Lesson 4

Mastra Tools ใช้ `createTool()` และ Zod

## Lesson 5

TypeScript Types ไม่เพียงพอสำหรับ Runtime Tool Arguments

## Lesson 6

Zod ตรวจ Input Shape แต่ไม่ตรวจ Business Correctness

## Lesson 7

`node:sqlite` ทำให้ TypeScript Worker ใช้ Shared Board Contract เดิมได้

## Lesson 8

MCP Tools ถูกดึงผ่าน `listTools()`

## Lesson 9

MCP Client ต้องถูกปิดด้วย `disconnect()`

## Lesson 10

Board Tools และ MCP Tools รวมกันผ่าน Object Spread

## Lesson 11

`maxSteps` เป็น Budget ไม่ใช่ Definition of Done

## Lesson 12

`onStepFinish` ช่วย Observability แต่ไม่พิสูจน์ Tool Success

## Lesson 13

Terminal Worker และ Studio Worker ใช้ Tool Sets ไม่เท่ากัน

## Lesson 14

Mastra Studio เป็น Development Surface ไม่ใช่ Production Validation

## Lesson 15

Application ยังต้องบังคับ Task Ownership, Artifact Validation และ Security

---

# 76. Memory Summary

```text
Week 5 Lab 4 ใช้:
Mastra

Language:
TypeScript

Folder:
5_agent_frameworks/4_mastra

Main files:
lab.md
step1.ts
step2.ts
step3.ts
step4.ts
step5.ts
worker.ts
board.ts
tools.ts

Agent:
new Agent({
  id,
  name,
  instructions,
  model,
  tools
})

Run:
await agent.generate(...)

Final output:
reply.text

Model:
openai/gpt-5.4-mini

Model layer:
Vercel AI SDK

Tools:
createTool()

Tool fields:
id
description
inputSchema
execute

Runtime validation:
Zod

Board:
node:sqlite

Board states:
pending
in_progress
done

Shared contract:
Read Goal
Plan Steps
Read notes.txt
Translate
Write spanish.txt
Complete Steps
Complete Goal

MCP:
MCPClient

Discover tools:
await filesystem.listTools()

Cleanup:
await filesystem.disconnect()

Merge tools:
{
  ...boardTools,
  ...mcpTools
}

Execution budget:
maxSteps: 25

Observability:
onStepFinish

Standalone worker:
npm run worker

Studio:
npm run dev

Studio URL:
http://localhost:4111

Studio worker:
Board tools only

Terminal worker:
Board tools
+ Filesystem MCP

Important difference:
Studio does not test full MCP file pipeline

Model swap:
createOpenAI()
หรือ provider/model routing

Shared weakness:
complete_task ไม่มี Business Validation

Shared concurrency risk:
claimTodo ไม่ atomic

Shared security risk:
Workspace ไม่ใช่ Full Sandbox

Mastra strength:
TypeScript-native
Zod schemas
Clean MCP configuration
Studio
Vercel AI SDK

Application must still enforce:
Task ownership
Artifact validation
Budgets
Permissions
Logging
Acceptance tests
```

---

# 77. Next Episode

Lab ถัดไปจะนำ Workers จากหลาย Framework มาประกอบเข้าด้วยกันผ่าน Shared Board และ Worker Subprocess Contract

สิ่งที่ควรจับตา:

```text
Framework catalog
Task assignment
Subprocess launching
Shared SQLite coordination
Worker selection
Cross-language execution
Result reconciliation
Failure handling
```

คำถามสำคัญสำหรับ Lab ถัดไปคือ:

> เมื่อ Workers จาก Python และ TypeScript สามารถทำงานบน Shared Board เดียวกันได้ Orchestrator จะควบคุม Task Ownership, Failure Recovery และ Definition of Done อย่างไร โดยไม่ต้องรู้รายละเอียดภายในของแต่ละ Framework?
