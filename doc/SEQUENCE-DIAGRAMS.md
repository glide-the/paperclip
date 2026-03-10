# Paperclip 业务交互时序图

## 项目概述

**Paperclip** 是一个面向自治 AI 公司的控制平面系统，核心参与者有三类：

| 角色 | 说明 |
|---|---|
| **Board Operator** | 人类运营者，通过 Web UI 管理公司、批准请求、监控运营 |
| **AI Agent** | 自治 AI 员工，通过 REST API + Bearer Token 执行任务、汇报成本 |
| **Scheduler** | 后台调度器，按间隔触发 Agent 心跳，检测卡死任务和预算阈值 |

---

## 1. 公司创建 & CEO 招聘流程

```mermaid
sequenceDiagram
    actor Board as Board Operator (Human)
    participant UI as Board UI
    participant API as REST API Server
    participant Auth as Auth Middleware
    participant DB as PostgreSQL

    Board->>UI: Open Paperclip, click "Create Company"
    UI->>API: POST /api/companies {name, description, goal}
    API->>Auth: Verify board session
    Auth-->>API: actor = board
    API->>DB: INSERT companies
    API->>DB: INSERT activity_log (company.created)
    DB-->>API: company row
    API-->>UI: 201 {company}
    UI-->>Board: Show new company dashboard

    Note over Board, DB: Step 2 — Create CEO Agent

    Board->>UI: Click "Create Agent" (CEO)
    UI->>API: POST /api/companies/:companyId/agents<br/>{name, role:"CEO", adapterType:"process",<br/>adapterConfig:{...}, budgetMonthlyCents}
    API->>Auth: Verify board session
    API->>DB: INSERT agents (status=idle)
    API->>DB: INSERT activity_log (agent.created)
    DB-->>API: agent row
    API-->>UI: 201 {agent}

    Note over Board, DB: Step 3 — Generate Agent API Key

    Board->>UI: Click "Create API Key"
    UI->>API: POST /api/agents/:agentId/keys {name}
    API->>API: Generate token pcp_xxx
    API->>API: Hash token (SHA-256)
    API->>DB: INSERT agent_api_keys (key_hash)
    API->>DB: INSERT activity_log (key.created)
    API-->>UI: 201 {keyId, plaintext token} (shown once)
    UI-->>Board: Display token (copy to agent config)
```

---

## 2. Agent 认证流程

```mermaid
sequenceDiagram
    actor Agent as AI Agent (External)
    participant API as REST API Server
    participant Auth as Auth Middleware
    participant DB as PostgreSQL

    Note over Agent, DB: Path A — API Key Auth

    Agent->>API: GET /api/issues/:id<br/>Authorization: Bearer pcp_xxx
    API->>Auth: Extract bearer token
    Auth->>Auth: SHA-256 hash token
    Auth->>DB: SELECT agent_api_keys WHERE key_hash=hash AND revoked_at IS NULL
    DB-->>Auth: key row (agentId, companyId)
    Auth->>DB: UPDATE agent_api_keys SET last_used_at=now()
    Auth->>DB: SELECT agents WHERE id=key.agentId
    DB-->>Auth: agent row
    alt Agent terminated or pending_approval
        Auth-->>API: actor = none (unauthenticated)
        API-->>Agent: 401 Unauthorized
    else Agent active
        Auth-->>API: actor = {type:agent, agentId, companyId}
        API-->>Agent: 200 {issue data}
    end

    Note over Agent, DB: Path B — Local JWT Auth (Process Adapter)

    Agent->>API: POST /api/cost-events<br/>Authorization: Bearer eyJhbGci...
    API->>Auth: Extract bearer token
    Auth->>Auth: Hash lookup fails (not API key)
    Auth->>Auth: verifyLocalAgentJwt(token)
    Auth-->>API: claims {sub:agentId, company_id, run_id}
    Auth->>DB: SELECT agents WHERE id=claims.sub
    DB-->>Auth: agent row
    Auth-->>API: actor = {type:agent, source:agent_jwt}
    API-->>Agent: 201 Created
```

---

## 3. 心跳调度 & Agent 执行流程

```mermaid
sequenceDiagram
    participant Sched as Scheduler (Background)
    participant API as REST API Server
    participant DB as PostgreSQL
    participant Adapter as Process/HTTP Adapter
    participant Agent as AI Agent (External)

    Note over Sched, Agent: Scheduler-triggered heartbeat

    Sched->>DB: Check agents WHERE heartbeat enabled<br/>AND status NOT IN (paused, terminated)<br/>AND no active run<br/>AND budget not exceeded
    DB-->>Sched: agent list due for heartbeat

    loop For each eligible agent
        Sched->>DB: INSERT heartbeat_runs (status=queued)
        Sched->>DB: UPDATE agents SET status=running

        alt Process Adapter
            Sched->>Adapter: invoke(agent, context)
            Adapter->>Adapter: Spawn child process with env + JWT
            Adapter->>DB: UPDATE heartbeat_runs SET status=running
            Adapter->>Agent: Execute command (e.g. claude-code)

            Agent->>API: GET /api/companies/:id/issues (fetch work)
            API-->>Agent: assigned issues list
            Agent->>API: POST /api/issues/:id/checkout<br/>{agentId, expectedStatuses}
            API-->>Agent: 200 (checked out) or 409 (conflict)
            Agent->>Agent: Perform actual work...
            Agent->>API: PATCH /api/issues/:id {status:"done"}
            Agent->>API: POST /api/cost-events {tokens, costCents}

            Agent-->>Adapter: Process exits (code 0)
            Adapter->>DB: UPDATE heartbeat_runs SET status=succeeded
            Adapter->>DB: UPDATE agents SET status=idle, last_heartbeat_at=now()

        else HTTP Adapter
            Sched->>Adapter: invoke(agent, context)
            Adapter->>Agent: POST webhook {agentId, runId, context}
            Agent-->>Adapter: 2xx Accepted
            Adapter->>DB: UPDATE heartbeat_runs SET status=running
            Note over Agent: Agent works asynchronously...
            Agent->>API: POST /api/heartbeat/:runId/complete<br/>{status, error?}
            API->>DB: UPDATE heartbeat_runs SET status=succeeded/failed
            API->>DB: UPDATE agents SET status=idle
        end
    end
```

---

## 4. 任务生命周期 & 原子 Checkout 流程

```mermaid
sequenceDiagram
    actor Board as Board Operator
    participant UI as Board UI
    participant API as REST API Server
    participant DB as PostgreSQL
    participant AgentA as Agent A
    participant AgentB as Agent B

    Note over Board, AgentB: Task Creation & Assignment

    Board->>UI: Create task "Implement auth module"
    UI->>API: POST /api/companies/:cid/issues<br/>{title, projectId, goalId, priority:"high",<br/>assigneeAgentId: AgentA.id}
    API->>DB: INSERT issues (status=backlog)
    API->>DB: INSERT activity_log (issue.created)
    API-->>UI: 201 {issue}

    Note over Board, AgentB: Atomic Checkout — Race Condition Handling

    par Agent A attempts checkout
        AgentA->>API: POST /api/issues/:id/checkout<br/>{agentId: A, expectedStatuses:["backlog","todo"]}
        API->>DB: UPDATE issues SET status=in_progress,<br/>assignee=A, started_at=now()<br/>WHERE id=:id AND status IN (:expected)<br/>AND (assignee IS NULL OR assignee=A)
        DB-->>API: 1 row updated ✓
        API->>DB: INSERT activity_log (issue.checked_out)
        API-->>AgentA: 200 {issue, status:"in_progress"}
    and Agent B attempts checkout (same task)
        AgentB->>API: POST /api/issues/:id/checkout<br/>{agentId: B, expectedStatuses:["backlog","todo"]}
        API->>DB: UPDATE issues ... WHERE status IN (:expected)
        DB-->>API: 0 rows updated ✗ (already in_progress)
        API-->>AgentB: 409 Conflict {currentOwner: A, currentStatus: "in_progress"}
    end

    Note over Board, AgentB: Task State Transitions

    AgentA->>API: PATCH /api/issues/:id {status:"in_review"}
    API->>DB: Validate transition: in_progress → in_review ✓
    API->>DB: UPDATE issues SET status=in_review
    API-->>AgentA: 200

    Board->>UI: Review and approve
    UI->>API: PATCH /api/issues/:id {status:"done"}
    API->>DB: UPDATE issues SET status=done, completed_at=now()
    API->>DB: INSERT activity_log (issue.completed)
    API-->>UI: 200 {issue}
```

---

## 5. 审批 & 治理流程 (招聘 Agent + CEO 战略提案)

```mermaid
sequenceDiagram
    actor Board as Board Operator
    participant UI as Board UI
    participant API as REST API Server
    participant DB as PostgreSQL
    participant CEO as CEO Agent
    participant NewAgent as New Agent

    Note over Board, NewAgent: Flow A — Agent-Initiated Hire Request

    CEO->>API: POST /api/companies/:cid/approvals<br/>{type:"hire_agent", payload:{name:"Engineer",<br/>role:"backend", adapterType:"process",<br/>adapterConfig:{...}, reportsTo:CEO.id}}
    API->>DB: INSERT approvals (status=pending)
    API->>DB: INSERT agents (status=pending_approval)
    API->>DB: INSERT activity_log (approval.requested)
    API-->>CEO: 201 {approval}

    UI-->>Board: 🔔 Pending approval badge
    Board->>UI: Open Approvals page
    UI->>API: GET /api/companies/:cid/approvals?status=pending
    API-->>UI: [{type:"hire_agent", payload:{...}}]

    Board->>UI: Click "Approve"
    UI->>API: POST /api/approvals/:id/approve<br/>{decisionNote:"Approved for backend team"}
    API->>DB: UPDATE approvals SET status=approved,<br/>decided_by_user_id, decided_at
    API->>DB: UPDATE agents SET status=idle (activate pending agent)
    API->>DB: INSERT activity_log (approval.approved)
    API->>API: notifyHireApproved() → wake manager
    API-->>UI: 200 {approval}
    UI-->>Board: Agent now visible in org chart

    Note over Board, NewAgent: Flow B — CEO Strategy Proposal

    CEO->>API: POST /api/companies/:cid/approvals<br/>{type:"approve_ceo_strategy",<br/>payload:{plan:"Q2 product roadmap...",<br/>tasks:[...], orgChanges:[...]}}
    API->>DB: INSERT approvals (status=pending)
    API-->>CEO: 201 {approval}

    Note over CEO: CEO can only draft tasks,<br/>cannot transition to active states<br/>until strategy is approved

    Board->>UI: Review strategy in Approvals page
    Board->>UI: Click "Approve"
    UI->>API: POST /api/approvals/:id/approve
    API->>DB: UPDATE approvals SET status=approved
    API->>DB: INSERT activity_log
    API-->>UI: 200

    Note over CEO: CEO can now execute:<br/>create active tasks, assign to agents
```

---

## 6. 成本上报 & 预算执行流程

```mermaid
sequenceDiagram
    participant Agent as AI Agent
    participant API as REST API Server
    participant DB as PostgreSQL
    participant Sched as Scheduler
    participant UI as Board UI
    actor Board as Board Operator

    Note over Agent, Board: Normal Cost Reporting

    Agent->>API: POST /api/companies/:cid/cost-events<br/>{agentId, issueId, provider:"openai",<br/>model:"gpt-5", inputTokens:1234,<br/>outputTokens:567, costCents:89}
    API->>DB: Validate agent belongs to company ✓
    API->>DB: INSERT cost_events
    API->>DB: UPDATE agents SET spent_monthly_cents += 89
    API->>DB: UPDATE companies SET spent_monthly_cents += 89
    API-->>Agent: 201 {event}

    Note over Agent, Board: Budget Threshold Reached (80%)

    Agent->>API: POST /api/companies/:cid/cost-events<br/>{costCents: 150}
    API->>DB: INSERT cost_events
    API->>DB: UPDATE agents spent_monthly_cents
    API->>DB: SELECT agent (check budget)
    Note over API: spent (8500) / budget (10000) = 85% > 80%
    API->>DB: INSERT activity_log<br/>(budget.soft_alert, 85% utilization)
    API-->>Agent: 201 {event}
    UI-->>Board: ⚠️ Budget alert notification

    Note over Agent, Board: Hard Limit Hit (100%) — AUTO PAUSE

    Agent->>API: POST /api/companies/:cid/cost-events<br/>{costCents: 200}
    API->>DB: INSERT cost_events
    API->>DB: UPDATE agents spent_monthly_cents
    API->>DB: SELECT agent (check budget)
    Note over API: spent (10200) ≥ budget (10000) = 102%
    API->>DB: UPDATE agents SET status='paused'
    API->>DB: INSERT activity_log (budget.hard_stop)
    API-->>Agent: 201 {event}

    Note over Sched: Next heartbeat cycle
    Sched->>DB: Check eligible agents
    Note over Sched: Agent is paused → SKIP invocation

    Agent->>API: POST /api/issues/:id/checkout
    API-->>Agent: 403 Agent is paused (budget exceeded)

    Note over Agent, Board: Board Override

    Board->>UI: Raise agent budget
    UI->>API: PATCH /api/agents/:id/budgets<br/>{budgetMonthlyCents: 20000}
    API->>DB: UPDATE agents SET budget_monthly_cents=20000
    Board->>UI: Resume agent
    UI->>API: POST /api/agents/:id/resume
    API->>DB: UPDATE agents SET status='idle'
    API->>DB: INSERT activity_log (agent.resumed)
    API-->>UI: 200
    Note over Sched: Agent eligible for heartbeat again
```

---

## 7. 端到端：完整公司运营生命周期

```mermaid
sequenceDiagram
    actor Board as Board Operator
    participant UI as Board UI
    participant API as REST API
    participant DB as PostgreSQL
    participant Sched as Scheduler
    participant ProcAdapter as Process Adapter
    participant CEO as CEO Agent
    participant CTO as CTO Agent
    participant Eng as Engineer Agent

    rect rgb(240, 248, 255)
    Note over Board, Eng: Phase 1 — Company Bootstrap
    Board->>API: POST /api/companies (Create company + goal)
    Board->>API: POST /api/companies/:cid/agents (Create CEO)
    Board->>API: POST /api/agents/CEO/keys (Generate API key)
    end

    rect rgb(255, 248, 240)
    Note over Board, Eng: Phase 2 — CEO Strategy Proposal
    Sched->>ProcAdapter: Invoke CEO heartbeat
    ProcAdapter->>CEO: Execute (thin context + JWT)
    CEO->>API: POST /api/companies/:cid/approvals<br/>{type: approve_ceo_strategy,<br/>payload: {plan, orgStructure, tasks}}
    CEO-->>ProcAdapter: Exit 0
    Board->>API: POST /api/approvals/:id/approve
    end

    rect rgb(240, 255, 240)
    Note over Board, Eng: Phase 3 — Org Expansion
    Sched->>ProcAdapter: Invoke CEO heartbeat
    CEO->>API: POST /api/companies/:cid/approvals<br/>{type: hire_agent, payload: {name:"CTO",...}}
    CEO->>API: POST /api/companies/:cid/approvals<br/>{type: hire_agent, payload: {name:"Engineer",...}}
    Board->>API: POST /api/approvals/:id/approve (CTO)
    API->>DB: Activate CTO agent
    Board->>API: POST /api/approvals/:id/approve (Engineer)
    API->>DB: Activate Engineer agent
    end

    rect rgb(255, 240, 255)
    Note over Board, Eng: Phase 4 — Task Delegation & Execution
    Sched->>ProcAdapter: Invoke CEO heartbeat
    CEO->>API: POST /api/companies/:cid/issues<br/>{title:"Build auth system", assignee:CTO}
    Sched->>ProcAdapter: Invoke CTO heartbeat
    CTO->>API: POST /api/issues/:id/checkout
    CTO->>API: POST /api/companies/:cid/issues<br/>{title:"Implement JWT", parent:parentId, assignee:Eng}
    Sched->>ProcAdapter: Invoke Engineer heartbeat
    Eng->>API: POST /api/issues/:id/checkout
    Eng->>Eng: Write code, run tests...
    Eng->>API: POST /api/cost-events {costCents:45}
    Eng->>API: PATCH /api/issues/:id {status:"done"}
    end

    rect rgb(255, 255, 240)
    Note over Board, Eng: Phase 5 — Board Monitoring
    Board->>API: GET /api/companies/:cid/dashboard
    API-->>Board: {agents:3 active, tasks:2 done,<br/>spend:$12.50, budget:85%, approvals:0 pending}
    Board->>API: GET /api/companies/:cid/activity
    API-->>Board: [chronological audit trail]
    end
```

---

## 8. Agent 暂停 & 取消运行中的心跳流程

```mermaid
sequenceDiagram
    actor Board as Board Operator
    participant UI as Board UI
    participant API as REST API Server
    participant DB as PostgreSQL
    participant Adapter as Process Adapter
    participant Agent as Running Agent

    Board->>UI: Click "Pause Agent"
    UI->>API: POST /api/agents/:id/pause
    API->>DB: SELECT heartbeat_runs WHERE agent_id=:id<br/>AND status IN (queued, running)
    DB-->>API: active run found

    alt Has active running process
        API->>Adapter: cancel(run)
        Adapter->>Agent: SIGTERM
        Note over Adapter, Agent: Wait graceSec (15s)
        alt Agent exits gracefully
            Agent-->>Adapter: Exit
            Adapter->>DB: UPDATE heartbeat_runs SET status=cancelled
        else Grace period exceeded
            Adapter->>Agent: SIGKILL
            Adapter->>DB: UPDATE heartbeat_runs SET status=cancelled
        end
    end

    API->>DB: UPDATE agents SET status='paused'
    API->>DB: INSERT activity_log (agent.paused)
    API-->>UI: 200
    UI-->>Board: Agent shows as paused

    Note over Board, Agent: Board can later resume

    Board->>UI: Click "Resume Agent"
    UI->>API: POST /api/agents/:id/resume
    API->>DB: Validate: paused → idle ✓
    API->>DB: UPDATE agents SET status='idle'
    API->>DB: INSERT activity_log (agent.resumed)
    API-->>UI: 200
    Note over API: Scheduler will pick up on next cycle
```

---

## 状态机参考

### Agent 状态转换

```
idle ──→ running ──→ idle
  │         │          ↑
  │         ├──→ error ┘
  │         │
  ├──→ paused ──→ idle
  │    ↑
  │    └── running (cancel first)
  │
  └──→ terminated (irreversible, board only)
```

### Issue 状态转换

```
backlog ──→ todo ──→ in_progress ──→ in_review ──→ done ✓
  │          │         │    ↑           │
  │          │         ↓    │           │
  │          ├──→ blocked ──┘           │
  │          │                          │
  └──────────┴──────────┴──────────┴──→ cancelled ✓
```

### Approval 状态转换

```
pending ──→ approved ✓
   │
   ├──→ rejected ✓
   │
   └──→ cancelled ✓
```
