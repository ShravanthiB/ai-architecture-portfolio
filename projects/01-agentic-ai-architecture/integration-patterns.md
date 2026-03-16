# Integration Patterns

**Project**: Enterprise Agentic AI Architecture
**Purpose**: Define integration patterns between agents and enterprise systems

---

## Overview

This document defines the integration architecture that connects autonomous AI agents with enterprise systems. The patterns leverage MuleSoft Anypoint Platform for enterprise-grade integration, governance, and observability.

**Key Principle**: Agents never directly access enterprise systems. All integration flows through MuleSoft for control, security, and monitoring.

---

## Pattern 1: Event-Driven Agent Triggering

### Description

Customer actions across multiple channels trigger events that are normalized, queued, and routed to appropriate agents for processing.

### Architecture

```
┌─────────────────────────────────────────────────────┐
│           Customer Touchpoints                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐         │
│  │ Web Form │  │  Email   │  │   Chat   │  ...    │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘         │
└───────┼─────────────┼─────────────┼────────────────┘
        │             │             │
        ▼             ▼             ▼
┌─────────────────────────────────────────────────────┐
│        MuleSoft Event Listeners                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐         │
│  │  HTTP    │  │  SMTP    │  │ WebSocket│         │
│  │ Listener │  │ Listener │  │ Listener │         │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘         │
│       │             │             │                 │
│       └─────────────┼─────────────┘                 │
│                     ▼                                │
│          ┌─────────────────────┐                    │
│          │ Event Normalizer    │                    │
│          │  - Extract metadata  │                   │
│          │  - Standardize format│                   │
│          │  - Enrich with context                   │
│          └──────────┬──────────┘                    │
└─────────────────────┼────────────────────────────────┘
                      ▼
┌─────────────────────────────────────────────────────┐
│            Apache Kafka (Message Queue)              │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────┐   │
│  │ Inbound     │  │  Actions    │  │  Audit   │   │
│  │ Events Topic│  │  Topic      │  │  Topic   │   │
│  └──────┬──────┘  └──────┬──────┘  └──────────┘   │
└─────────┼────────────────┼──────────────────────────┘
          │                │
          ▼                │
┌─────────────────────┐    │
│ Agent Orchestrator  │    │
│  - Read from queue  │    │
│  - Route to agent   │    │
│  - Track state      │    │
└─────────┬───────────┘    │
          │                │
          ▼                │
┌─────────────────────┐    │
│  Specialized Agent  │    │
│  - Process request  │    │
│  - Generate response│────┘ (Publishes actions)
│  - Take actions     │
└─────────────────────┘
```

### Event Schema

```json
{
  "event_id": "uuid",
  "event_type": "customer_inquiry",
  "timestamp": "2026-03-16T10:30:00Z",
  "channel": "web_form",
  "customer": {
    "email": "customer@example.com",
    "name": "John Doe",
    "customer_id": "SF_001",
    "account_id": "ACC_123"
  },
  "content": {
    "subject": "Question about pricing",
    "message": "I'm interested in your enterprise plan...",
    "metadata": {
      "page_url": "https://example.com/pricing",
      "utm_source": "google"
    }
  },
  "context": {
    "conversation_id": "conv_456",
    "previous_interactions": 2,
    "customer_segment": "enterprise"
  }
}
```

### Benefits

**Decoupling**:
- Agents don't need to know about channel specifics
- Easy to add new channels without changing agents
- Channels can evolve independently

**Scalability**:
- Message queue buffers traffic spikes
- Agents can scale independently
- Horizontal scaling of event processors

**Reliability**:
- Messages persisted in queue (durability)
- Automatic retry on failures
- Dead letter queues for problematic messages

**Observability**:
- All events centrally logged
- End-to-end tracing with correlation IDs
- Performance monitoring at each stage

### Implementation Notes

**MuleSoft Configuration**:
- Event listeners deployed as separate flows
- Shared normalization library for consistency
- Policy enforcement at ingestion point
- Rate limiting to prevent abuse

**Kafka Configuration**:
- Partitioning by customer_id for ordering
- Retention: 7 days
- Replication factor: 3 (high availability)
- Consumer groups for agent types

**Error Handling**:
- Malformed events → Dead letter queue
- Processing failures → Retry with backoff
- Max retries exceeded → Alert + manual review

---

## Pattern 2: API Gateway for Agent Actions

### Description

Agents take actions (create lead, send email, update case) through a centralized API gateway that provides authentication, authorization, governance, and observability.

### Architecture

```
┌─────────────────────────────────────────────────────┐
│              AI Agent                                │
│  Wants to: Create Lead in Salesforce                │
└─────────────────────┬───────────────────────────────┘
                      │
                      │ POST /api/crm/leads
                      │ Authorization: Bearer <token>
                      │ X-Agent-ID: lead_qual_agent_01
                      │
                      ▼
┌─────────────────────────────────────────────────────┐
│           MuleSoft API Gateway                       │
│  ┌───────────────────────────────────────────────┐  │
│  │  1. Authentication                             │  │
│  │     - Verify agent token                       │  │
│  │     - Check agent identity                     │  │
│  └───────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────┐  │
│  │  2. Authorization                              │  │
│  │     - Check agent permissions                  │  │
│  │     - Validate action allowed                  │  │
│  └───────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────┐  │
│  │  3. Policy Enforcement                         │  │
│  │     - Rate limiting (100 req/min per agent)    │  │
│  │     - Cost tracking (log LLM costs)            │  │
│  │     - Compliance checks                        │  │
│  └───────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────┐  │
│  │  4. Request Transformation                     │  │
│  │     - Agent format → Salesforce format         │  │
│  │     - Field mapping                            │  │
│  │     - Data validation                          │  │
│  └───────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────┐  │
│  │  5. Routing & Invocation                       │  │
│  │     - Route to appropriate system              │  │
│  │     - Handle authentication to target          │  │
│  │     - Circuit breaker pattern                  │  │
│  └────────────────────┬──────────────────────────┘  │
└─────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────┐
│           Target System (Salesforce)                 │
│  - Create Lead object                                │
│  - Apply business rules                              │
│  - Trigger workflows                                 │
└─────────────────────┬───────────────────────────────┘
                      │
                      │ Response: Lead created, ID=12345
                      │
                      ▼
┌─────────────────────────────────────────────────────┐
│           MuleSoft API Gateway                       │
│  ┌───────────────────────────────────────────────┐  │
│  │  6. Response Transformation                    │  │
│  │     - Salesforce format → Agent format         │  │
│  │     - Error normalization                      │  │
│  └────────────────────┬──────────────────────────┘  │
│  ┌───────────────────────────────────────────────┐  │
│  │  7. Audit Logging                              │  │
│  │     - Log action taken                         │  │
│  │     - Record agent, timestamp, result          │  │
│  └───────────────────────────────────────────────┘  │
└──────────────────────┬────────────────────────────┘
                       │
                       │ Response: {"lead_id": "12345", "status": "success"}
                       │
                       ▼
┌─────────────────────────────────────────────────────┐
│              AI Agent                                │
│  Lead created successfully!                          │
└─────────────────────────────────────────────────────┘
```

### Agent Action API Specification

**Base URL**: `https://api.company.com/v1/agent-actions`

**Authentication**: Bearer token (agent-specific)

**Common Headers**:
- `Authorization: Bearer <token>`
- `X-Agent-ID: <agent_identifier>`
- `X-Conversation-ID: <conversation_id>` (for tracing)
- `X-Request-ID: <uuid>` (idempotency)

#### CRM Actions

**Create Lead**:
```
POST /crm/leads
{
  "email": "prospect@example.com",
  "first_name": "Jane",
  "last_name": "Smith",
  "company": "Acme Corp",
  "phone": "+1-555-0100",
  "qualification_score": 85,
  "notes": "Interested in enterprise plan...",
  "source": "web_form"
}
```

**Update Lead**:
```
PUT /crm/leads/{lead_id}
{
  "status": "qualified",
  "qualification_notes": "Budget: $50K, Authority: VP, Need: High, Timeline: Q2"
}
```

**Create Case**:
```
POST /crm/cases
{
  "account_id": "ACC_123",
  "contact_email": "customer@example.com",
  "subject": "Login issue",
  "description": "Customer unable to login...",
  "priority": "medium",
  "category": "technical"
}
```

#### Communication Actions

**Send Email**:
```
POST /communications/email
{
  "to": "customer@example.com",
  "template_id": "follow_up_template",
  "variables": {
    "customer_name": "John",
    "next_steps": "..."
  },
  "track_opens": true
}
```

**Schedule Meeting**:
```
POST /calendar/meetings
{
  "attendees": ["sales_rep@company.com", "prospect@example.com"],
  "duration_minutes": 30,
  "proposed_times": ["2026-03-20T14:00:00Z", "2026-03-20T15:00:00Z"],
  "subject": "Discovery Call",
  "description": "..."
}
```

#### Data Enrichment Actions

**Enrich Company**:
```
POST /enrichment/company
{
  "domain": "example.com"
}

Response:
{
  "company_name": "Example Corp",
  "industry": "Software",
  "employee_count": 500,
  "revenue": "$50M",
  "tech_stack": ["Salesforce", "AWS", "Slack"],
  "confidence": 0.95
}
```

### Authorization Model

**Permission Matrix**:

| Agent | CRM Read | CRM Write | Send Email | Schedule Meeting | Refund | Account Modify |
|-------|----------|-----------|------------|------------------|--------|----------------|
| Lead Qualification | ✅ | ✅ (Leads only) | ✅ | ✅ (< 1 week) | ❌ | ❌ |
| Customer Support | ✅ | ✅ (Cases only) | ✅ | ❌ | ✅ (< $500) | ❌ |
| Sales Assistant | ✅ | ✅ (Opportunities) | ✅ | ✅ | ❌ | ❌ |
| Data Enrichment | ✅ | ✅ (enrichment fields) | ❌ | ❌ | ❌ | ❌ |
| Supervisor | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |

**Policy Enforcement**:
- Agents can only perform actions they're authorized for
- Sensitive actions (refunds, account changes) have additional checks
- All actions logged with agent identity
- Rate limits prevent runaway costs

### Benefits

**Security**:
- Centralized authentication and authorization
- Agents never have direct system credentials
- Fine-grained permission control
- Audit trail for compliance

**Abstraction**:
- Agents don't need to know Salesforce API details
- Easy to change underlying systems
- Consistent API across all enterprise systems

**Governance**:
- Rate limiting prevents abuse
- Cost tracking for LLM and API usage
- Compliance policy enforcement
- Quality gates (validation, business rules)

**Observability**:
- All actions logged centrally
- Performance metrics per agent
- Error rates and patterns
- End-to-end tracing

---

## Pattern 3: Human-in-the-Loop Workflow

### Description

For critical decisions, agents request human approval through an integrated workflow system, maintaining conversation context and providing seamless handoff.

### Architecture

```
┌─────────────────────────────────────────────────────┐
│              AI Agent                                │
│  Processing conversation...                          │
│  Identifies: Needs human approval                    │
│    - Customer requests 20% discount                  │
│    - Agent policy: Max 10% without approval          │
└─────────────────────┬───────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────┐
│           Approval Request Service                   │
│  ┌───────────────────────────────────────────────┐  │
│  │  Create Approval Request                       │  │
│  │   - Capture full context                       │  │
│  │   - Agent recommendation                       │  │
│  │   - Customer history                           │  │
│  │   - Business impact                            │  │
│  └────────────────────┬──────────────────────────┘  │
└─────────────────────────────────────────────────────┘
                         │
                         ├────────────────────┬─────────────────┐
                         ▼                    ▼                 ▼
┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
│  Slack Channel   │ │     Email        │ │ Salesforce Task  │
│  #agent-approvals│ │  manager@co.com  │ │  Assigned to mgr │
└────────┬─────────┘ └────────┬─────────┘ └────────┬─────────┘
         │                    │                     │
         │  Human manager reviews request            │
         │                    │                     │
         └────────────────────┴─────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────┐
│           Approval Decision Interface                │
│  ┌───────────────────────────────────────────────┐  │
│  │  Context:                                      │  │
│  │   - Customer: Acme Corp (Enterprise)           │  │
│  │   - Request: 20% discount on annual plan       │  │
│  │   - Value: $40K → $32K                         │  │
│  │   - Agent recommendation: Approve (high value) │  │
│  │   - Customer history: 2 years, no issues       │  │
│  │                                                │  │
│  │  [Approve] [Reject] [Counteroffer]            │  │
│  └───────────────────────────────────────────────┘  │
└──────────────────────┬────────────────────────────┘
                       │
                       │ Decision: Approved
                       │
                       ▼
┌─────────────────────────────────────────────────────┐
│              AI Agent                                │
│  - Receives approval                                 │
│  - Resumes conversation                              │
│  - "Great news! I've been authorized to offer..."    │
│  - Continues to close deal                           │
└─────────────────────────────────────────────────────┘
```

### Approval Request Schema

```json
{
  "approval_id": "APR_789",
  "requested_at": "2026-03-16T10:35:00Z",
  "requester": {
    "agent_id": "lead_qual_agent_01",
    "agent_type": "Lead Qualification",
    "conversation_id": "conv_456"
  },
  "request_type": "discount_approval",
  "details": {
    "customer": {
      "name": "Acme Corp",
      "account_id": "ACC_123",
      "tier": "enterprise",
      "lifetime_value": "$250K",
      "health_score": 95
    },
    "action": {
      "type": "apply_discount",
      "discount_percentage": 20,
      "original_amount": "$40K",
      "discounted_amount": "$32K",
      "plan": "Enterprise Annual"
    },
    "context": {
      "reason": "Customer has competing offer from competitor",
      "urgency": "high",
      "decision_timeline": "Today",
      "agent_recommendation": "approve",
      "agent_confidence": 0.85
    }
  },
  "approvers": [
    {
      "role": "sales_manager",
      "email": "sales_manager@company.com",
      "required": true
    }
  ],
  "sla": {
    "response_required_by": "2026-03-16T11:35:00Z",
    "escalation_if_no_response": "regional_vp"
  }
}
```

### Workflow States

```
┌─────────────┐
│   Pending   │  (Waiting for human decision)
└──────┬──────┘
       │
       ├─────────────────┬─────────────────┬──────────────┐
       │                 │                 │              │
       ▼                 ▼                 ▼              ▼
┌──────────┐      ┌──────────┐     ┌──────────┐  ┌──────────┐
│ Approved │      │ Rejected │     │Counteroffer│ │ Expired  │
└────┬─────┘      └────┬─────┘     └─────┬──────┘ └────┬─────┘
     │                 │                 │              │
     └─────────────────┴─────────────────┴──────────────┘
                            │
                            ▼
                    ┌──────────────┐
                    │   Completed  │
                    └──────────────┘
```

### SLA Management

**Response Time SLAs**:
- High priority: 1 hour
- Medium priority: 4 hours
- Low priority: 24 hours

**Escalation Path**:
```
Primary Approver (No response after SLA/2)
    ↓
Send reminder
    ↓
Primary Approver (No response after SLA)
    ↓
Escalate to Secondary Approver
    ↓
Secondary Approver (No response after 2x SLA)
    ↓
Auto-reject or emergency escalation
```

**Agent Behavior During Approval**:
```python
# Pseudocode
agent.request_approval(action="apply_discount", details={...})

# Agent pauses conversation
agent.send_message_to_customer(
    "Let me check on that for you. "
    "I'll have an answer for you shortly."
)

# Agent polls for approval (or receives callback)
while approval.status == "pending":
    if time_elapsed > timeout:
        agent.send_message_to_customer(
            "I'm still working on getting you an answer. "
            "It should just be a few more minutes."
        )
    wait(30_seconds)

if approval.status == "approved":
    agent.resume_conversation(approved_action)
elif approval.status == "rejected":
    agent.resume_conversation(alternative_offer)
else:  # expired
    agent.escalate_to_human_agent()
```

### Benefits

**Safety**:
- Critical decisions reviewed by humans
- Prevents agent overreach
- Maintains customer trust

**Context Preservation**:
- Full conversation history available
- Agent provides recommendation
- Customer data readily accessible

**Efficiency**:
- Asynchronous (doesn't block other work)
- Mobile-friendly approval interface
- Smart routing to right approver

**Auditability**:
- All approvals logged
- Decision rationale captured
- Compliance trail maintained

---

## Pattern 4: RAG Integration for Knowledge Base

### Description

Support agent retrieves relevant information from knowledge base using Retrieval-Augmented Generation (RAG) to answer customer questions accurately.

### Architecture

```
┌─────────────────────────────────────────────────────┐
│        Customer Support Agent                        │
│  Customer asks: "How do I reset my password?"        │
└─────────────────────┬───────────────────────────────┘
                      │
                      │ 1. Generate embedding
                      │    for question
                      │
                      ▼
┌─────────────────────────────────────────────────────┐
│           Embedding Service                          │
│  Convert question to vector (1536 dimensions)        │
└─────────────────────┬───────────────────────────────┘
                      │
                      │ 2. Query vector DB
                      │
                      ▼
┌─────────────────────────────────────────────────────┐
│        Pinecone Vector Database                      │
│  ┌───────────────────────────────────────────────┐  │
│  │  Semantic Search                               │  │
│  │   - Compare query embedding to all docs        │  │
│  │   - Return top 5 most similar chunks           │  │
│  │   - Ranked by cosine similarity                │  │
│  └────────────────────┬──────────────────────────┘  │
└─────────────────────────────────────────────────────┘
                         │
                         │ 3. Retrieved chunks
                         │
                         ▼
┌─────────────────────────────────────────────────────┐
│           Re-Ranking Service                         │
│  Cross-encoder model re-ranks for accuracy           │
│  Final top 3 chunks selected                         │
└─────────────────────┬───────────────────────────────┘
                      │
                      │ 4. Augmented context
                      │
                      ▼
┌─────────────────────────────────────────────────────┐
│        Customer Support Agent (LLM)                  │
│  ┌───────────────────────────────────────────────┐  │
│  │  System Prompt                                 │  │
│  │    You are a helpful support agent...          │  │
│  │                                                │  │
│  │  Context (from knowledge base):                │  │
│  │    [Doc 1: Password reset instructions...]     │  │
│  │    [Doc 2: Security settings...]               │  │
│  │    [Doc 3: Account recovery...]                │  │
│  │                                                │  │
│  │  Customer Question:                            │  │
│  │    "How do I reset my password?"               │  │
│  │                                                │  │
│  │  Generate accurate response based on context   │  │
│  └───────────────────────────────────────────────┘  │
└─────────────────────┬───────────────────────────────┘
                      │
                      │ 5. Response with citations
                      │
                      ▼
┌─────────────────────────────────────────────────────┐
│              Customer                                │
│  Receives: "Here's how to reset your password:      │
│    1. Go to login page...                            │
│    2. Click 'Forgot Password'...                     │
│                                                      │
│  [Source: Password Reset Guide, updated Mar 2026]"  │
└─────────────────────────────────────────────────────┘
```

### Knowledge Base Preparation

**Document Ingestion Pipeline**:
```
Documentation Source (Wiki, Docs, PDFs)
    ↓
1. Extract text content
    ↓
2. Clean and normalize
    ↓
3. Chunk into passages (512 tokens)
    ↓
4. Generate metadata (title, section, date, category)
    ↓
5. Create embeddings (OpenAI ada-002)
    ↓
6. Store in Pinecone with metadata
```

**Metadata Schema**:
```json
{
  "chunk_id": "doc_123_chunk_5",
  "text": "To reset your password, navigate to...",
  "source_document": "password_reset_guide.md",
  "section": "Account Security",
  "last_updated": "2026-03-01",
  "category": "authentication",
  "product": "core_platform",
  "confidence": 0.98
}
```

### Retrieval Strategy

**Hybrid Search**:
1. **Semantic search** (vector similarity) - finds conceptually similar content
2. **Keyword search** (BM25) - finds exact term matches
3. **Combine results** - merge and re-rank

**Filtering**:
- Only return docs updated in last 6 months
- Filter by product/category if known
- Exclude deprecated content

**Re-Ranking**:
- Use cross-encoder to score query-document relevance
- Select top 3 for context window
- Balance relevance vs diversity

### Response Generation

**Prompt Template**:
```
You are a customer support agent for [Company]. Answer the customer's
question accurately using ONLY the provided context. If the answer is
not in the context, say so clearly.

CONTEXT:
{retrieved_chunks}

CUSTOMER QUESTION:
{customer_question}

INSTRUCTIONS:
- Provide clear, step-by-step instructions if applicable
- Cite your sources using [Source: Document Name]
- If uncertain, admit it and offer to escalate
- Be friendly and empathetic
```

### Benefits

**Accuracy**:
- Responses grounded in actual documentation
- Reduces hallucinations
- Always up-to-date with latest docs

**Transparency**:
- Citations provide source verification
- Customers can verify information
- Easier to identify doc gaps

**Maintainability**:
- Update docs, not AI model
- No retraining required
- Easy to add new content

---

## Integration Observability

### Metrics to Track

**Per Pattern**:
- Request volume
- Response time (p50, p95, p99)
- Error rate
- Retry rate

**Agent Actions**:
- Actions per agent per hour
- Success/failure rate by action type
- Cost per action
- Authorization denials

**Approvals**:
- Approval request rate
- Approval time (SLA compliance)
- Approval/rejection ratio
- Escalation rate

**RAG**:
- Retrieval latency
- Chunk relevance scores
- Citation usage rate
- Knowledge gaps identified

### Distributed Tracing

**Trace Spans**:
```
Request (e.g., "Create Lead")
├─ Authentication (50ms)
├─ Authorization (30ms)
├─ Rate Limit Check (10ms)
├─ Transform Request (20ms)
├─ Call Salesforce API (300ms)
├─ Transform Response (15ms)
└─ Audit Log (25ms)

Total: 450ms
```

**Correlation IDs**:
- `X-Request-ID`: Unique per request
- `X-Conversation-ID`: Tracks full conversation
- `X-Agent-ID`: Identifies which agent
- `X-Trace-ID`: Distributed tracing

---

## Error Handling Patterns

### Retry with Exponential Backoff

```python
max_retries = 3
base_delay = 1  # second

for attempt in range(max_retries):
    try:
        response = call_api()
        return response
    except TransientError as e:
        if attempt == max_retries - 1:
            raise
        delay = base_delay * (2 ** attempt)  # 1s, 2s, 4s
        sleep(delay)
```

### Circuit Breaker

```
CLOSED (Normal operation)
    │
    │ (Failures exceed threshold)
    ▼
OPEN (Reject all requests immediately)
    │
    │ (After timeout)
    ▼
HALF-OPEN (Allow limited requests)
    │
    ├─ Success → CLOSED
    └─ Failure → OPEN
```

### Graceful Degradation

If Salesforce unavailable:
- Cache last known data
- Queue write operations for later
- Notify user of delayed processing
- Offer alternative actions

---

## Security Patterns

### Agent Authentication

**JWT Token**:
```json
{
  "agent_id": "lead_qual_agent_01",
  "agent_type": "lead_qualification",
  "permissions": ["crm.lead.write", "email.send"],
  "issued_at": "2026-03-16T10:00:00Z",
  "expires_at": "2026-03-16T11:00:00Z"
}
```

**Token Rotation**:
- Tokens expire every hour
- Automatic refresh before expiration
- Revocation on security events

### Data Privacy

**PII Handling**:
- Encrypt PII in transit and at rest
- Mask sensitive data in logs
- Implement right-to-be-forgotten
- GDPR/CCPA compliance

**Data Minimization**:
- Only retrieve data needed for task
- Don't store data longer than required
- Anonymize analytics data

---

*Last Updated: March 2026*
*Version: 1.0*
