# System Architecture Diagrams

**Project**: Enterprise Agentic AI Architecture
**Purpose**: Visual representations of system architecture

---

## High-Level System Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          Customer Interaction Layer                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐    │
│  │   Web    │  │  Email   │  │   Chat   │  │  Phone   │  │  Mobile  │    │
│  │   Form   │  │          │  │  Widget  │  │  (Voice) │  │   App    │    │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘    │
└───────┼─────────────┼─────────────┼─────────────┼─────────────┼───────────┘
        │             │             │             │             │
        └─────────────┴─────────────┴─────────────┴─────────────┘
                                    │
        ╔═══════════════════════════╧═════════════════════════════╗
        ║             MuleSoft Integration Layer                   ║
        ║  ┌────────────────┐  ┌────────────────┐                ║
        ║  │ Event Listeners │  │  API Gateway   │                ║
        ║  │  - Normalize    │  │  - Auth/Authz  │                ║
        ║  │  - Enrich       │  │  - Policies    │                ║
        ║  │  - Route        │  │  - Rate Limit  │                ║
        ║  └────────┬────────┘  └────────┬───────┘                ║
        ╚═══════════╧══════════════════════╧══════════════════════╝
                    │                      │
        ┌───────────▼────────┐  ┌─────────▼──────────┐
        │  Kafka Message     │  │  MuleSoft Anypoint │
        │  Queue             │  │  Platform APIs     │
        └───────────┬────────┘  └─────────┬──────────┘
                    │                      │
        ┌───────────▼──────────────────────▼───────────────────────┐
        │              Agent Orchestration Platform                 │
        │  ┌────────────────────────────────────────────────────┐  │
        │  │         Supervisor Agent (Coordinator)             │  │
        │  │  - Route conversations                             │  │
        │  │  - Monitor performance                             │  │
        │  │  - Manage escalations                              │  │
        │  │  - Coordinate multi-agent workflows                │  │
        │  └──────┬────────┬────────┬────────┬──────────────┬──┘  │
        │         │        │        │        │              │      │
        │  ┌──────▼──┐ ┌──▼───┐ ┌──▼───┐ ┌──▼────┐  ┌─────▼───┐ │
        │  │  Lead   │ │Support│ │Sales │ │ Data  │  │ Future  │ │
        │  │  Qual   │ │ Agent │ │Assist│ │Enrich │  │ Agents  │ │
        │  │  Agent  │ │       │ │Agent │ │Agent  │  │         │ │
        │  └──────┬──┘ └──┬───┘ └──┬───┘ └──┬────┘  └─────────┘ │
        │         │       │        │        │                     │
        │  ┌──────▼───────▼────────▼────────▼──────────────────┐ │
        │  │          LLM Gateway (Multi-Model Router)          │ │
        │  │  - Select appropriate model                        │ │
        │  │  - Cost optimization                               │ │
        │  │  - Failover handling                               │ │
        │  └──────┬────────────────────────────────────┬────────┘ │
        └─────────┼────────────────────────────────────┼──────────┘
                  │                                     │
        ┌─────────▼───────┐  ┌─────────┐  ┌──────────▼─────────┐
        │   OpenAI GPT-4  │  │ Claude  │  │  Llama 3 / Custom  │
        └─────────────────┘  │ Sonnet  │  └────────────────────┘
                             └─────────┘
                                  │
        ┌─────────────────────────┴────────────────────────────┐
        │                   Data Layer                          │
        │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
        │  │  PostgreSQL  │  │   Pinecone   │  │    Redis     │
        │  │  (Structured)│  │   (Vectors)  │  │   (Cache)    │
        │  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘
        └─────────┼──────────────────┼──────────────────┼────────┘
                  │                  │                  │
        ┌─────────▼──────────────────▼──────────────────▼────────┐
        │           Salesforce (System of Record)                 │
        │  - Leads, Contacts, Accounts                            │
        │  - Opportunities, Cases                                 │
        │  - Business Rules, Workflows                            │
        └─────────────────────────────────────────────────────────┘

                    External Systems
        ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
        │   Email     │ │ Calendar    │ │ Knowledge   │
        │   Service   │ │ Systems     │ │    Base     │
        └─────────────┘ └─────────────┘ └─────────────┘
```

---

## Detailed Agent Orchestration Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                    Incoming Customer Event                            │
│     { customer_id, message, channel, context }                       │
└────────────────────────────┬─────────────────────────────────────────┘
                             │
                             ▼
              ┌──────────────────────────────┐
              │   Supervisor Agent           │
              │                              │
              │  1. Analyze Intent           │
              │     - NLU on message         │
              │     - Check customer context │
              │     - Determine agent type   │
              │                              │
              │  2. Select Agent             │
              │     - Route based on intent  │
              │     - Check agent capacity   │
              │     - Load balance           │
              │                              │
              │  3. Initialize Context       │
              │     - Customer history       │
              │     - Conversation state     │
              │     - Agent instructions     │
              └────────┬────────┬────────────┘
                       │        │
        ┌──────────────┘        └──────────────┐
        │                                       │
        ▼                                       ▼
┌──────────────────┐                  ┌──────────────────┐
│ Lead Qual Agent  │                  │ Support Agent    │
│                  │                  │                  │
│ ┌──────────────┐ │                  │ ┌──────────────┐ │
│ │ Conversation │ │                  │ │ RAG Search   │ │
│ │   Manager    │ │                  │ │   Engine     │ │
│ └──────┬───────┘ │                  │ └──────┬───────┘ │
│        │         │                  │        │         │
│ ┌──────▼───────┐ │                  │ ┌──────▼───────┐ │
│ │  LLM Call    │ │                  │ │  LLM Call    │ │
│ │  (GPT-4)     │ │                  │ │  (Claude)    │ │
│ └──────┬───────┘ │                  │ └──────┬───────┘ │
│        │         │                  │        │         │
│ ┌──────▼───────┐ │                  │ ┌──────▼───────┐ │
│ │  Action      │ │                  │ │  Action      │ │
│ │  Executor    │ │                  │ │  Executor    │ │
│ └──────┬───────┘ │                  │ └──────┬───────┘ │
└────────┼─────────┘                  └────────┼─────────┘
         │                                     │
         └─────────┬───────────────────────────┘
                   │
                   ▼
        ┌────────────────────────┐
        │  MuleSoft API Gateway  │
        │  - Validate action     │
        │  - Check permissions   │
        │  - Execute via APIs    │
        └────────────┬───────────┘
                     │
        ┌────────────┴────────────┐
        │                         │
        ▼                         ▼
┌──────────────┐         ┌──────────────┐
│  Salesforce  │         │  Email Svc   │
│  API Call    │         │  API Call    │
└──────┬───────┘         └──────┬───────┘
       │                        │
       └───────────┬────────────┘
                   │
                   ▼
            ┌────────────┐
            │  Response  │
            │  to Agent  │
            └─────┬──────┘
                  │
                  ▼
       ┌────────────────────┐
       │  Agent Processes   │
       │  Response &        │
       │  Continues         │
       │  Conversation      │
       └────────────────────┘
```

---

## Data Flow: Event-Driven Architecture

```
┌──────────────────────────────────────────────────────────────┐
│  Step 1: Customer Action                                      │
│  Customer submits web form                                    │
└────────────────┬─────────────────────────────────────────────┘
                 │
                 │ HTTP POST
                 │
                 ▼
┌──────────────────────────────────────────────────────────────┐
│  Step 2: MuleSoft Event Listener                             │
│  - Receives POST request                                     │
│  - Normalizes data format                                    │
│  - Enriches with metadata (timestamp, channel, etc.)         │
└────────────────┬─────────────────────────────────────────────┘
                 │
                 │ Publish Event
                 │
                 ▼
┌──────────────────────────────────────────────────────────────┐
│  Step 3: Kafka Topic (inbound_events)                        │
│  - Event persisted                                           │
│  - Partitioned by customer_id                                │
│  - Available for consumption                                 │
└────────────────┬─────────────────────────────────────────────┘
                 │
                 │ Consumer reads
                 │
                 ▼
┌──────────────────────────────────────────────────────────────┐
│  Step 4: Agent Orchestrator                                  │
│  - Reads event from queue                                    │
│  - Retrieves customer context from DB                        │
│  - Routes to Supervisor Agent                                │
└────────────────┬─────────────────────────────────────────────┘
                 │
                 │ Invoke
                 │
                 ▼
┌──────────────────────────────────────────────────────────────┐
│  Step 5: Supervisor Agent                                    │
│  - Analyzes message intent                                   │
│  - Selects appropriate specialized agent                     │
│  - Passes context to selected agent                          │
└────────────────┬─────────────────────────────────────────────┘
                 │
                 │ Delegate
                 │
                 ▼
┌──────────────────────────────────────────────────────────────┐
│  Step 6: Specialized Agent (e.g., Lead Qual)                 │
│  - Processes customer message                                │
│  - Calls LLM with context                                    │
│  - Determines action needed                                  │
└────────────────┬─────────────────────────────────────────────┘
                 │
                 │ Action Request
                 │
                 ▼
┌──────────────────────────────────────────────────────────────┐
│  Step 7: MuleSoft API Gateway                                │
│  - Authenticates agent                                       │
│  - Authorizes action                                         │
│  - Enforces policies (rate limit, etc.)                      │
│  - Routes to target system                                   │
└────────────────┬─────────────────────────────────────────────┘
                 │
                 │ API Call
                 │
                 ▼
┌──────────────────────────────────────────────────────────────┐
│  Step 8: Salesforce                                          │
│  - Creates Lead record                                       │
│  - Executes business rules                                   │
│  - Returns success response                                  │
└────────────────┬─────────────────────────────────────────────┘
                 │
                 │ Response
                 │
                 ▼
┌──────────────────────────────────────────────────────────────┐
│  Step 9: Agent Receives Confirmation                         │
│  - Lead created successfully                                 │
│  - Generates customer-facing response                        │
│  - Publishes response event                                  │
└────────────────┬─────────────────────────────────────────────┘
                 │
                 │ Publish
                 │
                 ▼
┌──────────────────────────────────────────────────────────────┐
│  Step 10: Response Delivery                                  │
│  - MuleSoft routes response to customer's channel            │
│  - Email, chat, etc. updated                                 │
│  - Customer receives response                                │
└──────────────────────────────────────────────────────────────┘

Total Time: ~2-5 seconds end-to-end
```

---

## Agent Decision Flow

```
                    Customer Message
                           │
                           ▼
                 ┌──────────────────┐
                 │  Extract Intent  │
                 └────────┬─────────┘
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
        ▼                 ▼                 ▼
   ┌────────┐        ┌────────┐       ┌────────┐
   │ Sales? │        │Support?│       │Other?  │
   └───┬────┘        └───┬────┘       └───┬────┘
       │ Yes             │ Yes            │ Yes
       │                 │                │
       ▼                 ▼                ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ Lead Qual    │  │ Support      │  │ Analyze      │
│ Agent        │  │ Agent        │  │ Further      │
└──────┬───────┘  └──────┬───────┘  └──────┬───────┘
       │                 │                 │
       │                 │                 │
       ▼                 ▼                 ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ Qualify      │  │ Search KB    │  │ Multi-agent  │
│ Conversation │  │ (RAG)        │  │ Consensus    │
└──────┬───────┘  └──────┬───────┘  └──────┬───────┘
       │                 │                 │
       │                 │                 │
       ▼                 ▼                 ▼
   ┌────────────┐   ┌────────────┐   ┌────────────┐
   │ Confidence │   │ Confidence │   │ Confidence │
   │   Check    │   │   Check    │   │   Check    │
   └─────┬──────┘   └─────┬──────┘   └─────┬──────┘
         │                │                │
    ┌────┼────┐      ┌────┼────┐      ┌────┼────┐
    │    │    │      │    │    │      │    │    │
    ▼    ▼    ▼      ▼    ▼    ▼      ▼    ▼    ▼
  High  Med  Low   High  Med  Low   High  Med  Low
    │    │    │      │    │    │      │    │    │
    │    │    └──────┼────┼────┴──────┘    │    │
    │    │           │    │                 │    │
    │    │           ▼    ▼                 │    │
    │    │      ┌─────────────┐             │    │
    │    └─────→│  Supervisor │←────────────┘    │
    │           │  Review     │                  │
    │           └──────┬──────┘                  │
    │                  │                         │
    │             ┌────┼────┐                    │
    │             │         │                    │
    │             ▼         ▼                    │
    │         ┌────────┐ ┌────────┐             │
    │         │Approve │ │Escalate│             │
    │         └───┬────┘ └───┬────┘             │
    │             │          │                   │
    └─────────────┘          │                   │
                             │                   │
                             ▼                   │
                       ┌──────────┐              │
                       │  Human   │              │
                       │  Agent   │              │
                       └─────┬────┘              │
                             │                   │
                             └───────────────────┘
                                     │
                                     ▼
                              ┌─────────────┐
                              │  Execute    │
                              │  Action     │
                              └─────┬───────┘
                                    │
                                    ▼
                              ┌─────────────┐
                              │  Customer   │
                              │  Response   │
                              └─────────────┘
```

---

## Multi-Agent Coordination Patterns

### Pattern 1: Sequential Handoff

```
Customer: "I have a question about my account,
           and I'm also interested in upgrading"

        Support Agent
              │
              │ 1. Answers account question
              │ 2. Identifies upsell opportunity
              │ 3. Requests handoff
              │
              ▼
        Supervisor Agent
              │
              │ 1. Validates handoff request
              │ 2. Prepares context
              │ 3. Routes to sales
              │
              ▼
        Sales Assistant Agent
              │
              │ 1. Receives full context
              │ 2. Continues conversation
              │ 3. Discusses upgrade
              │
              ▼
        Customer receives seamless experience
```

### Pattern 2: Parallel Processing

```
New lead submitted via web form
        │
        ▼
    Supervisor
        │
        ├────────────────────────┐
        │                        │
        ▼                        ▼
Lead Qual Agent          Data Enrichment Agent
        │                        │
        │ Qualify lead           │ Enrich company data
        │ Score lead             │ Find contact info
        │ Engage prospect        │ Get tech stack
        │                        │
        └────────┬───────────────┘
                 │
                 ▼
            Supervisor
                 │
                 │ Combine results
                 │
                 ▼
        Create enriched lead
        Route to sales team
```

### Pattern 3: Consensus Decision

```
Complex customer inquiry (ambiguous)
        │
        ▼
    Supervisor routes to multiple agents for analysis
        │
        ├────────────┬────────────┐
        │            │            │
        ▼            ▼            ▼
    Agent A      Agent B      Agent C
        │            │            │
        │ Analyze    │ Analyze    │ Analyze
        │ Recommend  │ Recommend  │ Recommend
        │            │            │
        └────────────┴────────────┘
                     │
                     ▼
              Supervisor
                     │
                     │ Aggregate recommendations
                     │ If consensus → Execute
                     │ If no consensus → Escalate to human
                     │
                     ▼
              Final Decision
```

---

## Deployment Architecture

```
┌────────────────────────────────────────────────────────────────┐
│                      AWS Cloud (US-EAST-1)                      │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              Kubernetes Cluster (EKS)                     │  │
│  │                                                            │  │
│  │  ┌────────────────┐  ┌────────────────┐  ┌────────────┐ │  │
│  │  │ Supervisor     │  │ Lead Qual      │  │ Support    │ │  │
│  │  │ Agent Pod      │  │ Agent Pod      │  │ Agent Pod  │ │  │
│  │  │ (Replicas: 3)  │  │ (Replicas: 5)  │  │(Replicas:10)│  │
│  │  └────────────────┘  └────────────────┘  └────────────┘ │  │
│  │                                                            │  │
│  │  ┌────────────────┐  ┌────────────────┐                  │  │
│  │  │ Sales Assist   │  │ Data Enrich    │                  │  │
│  │  │ Agent Pod      │  │ Agent Pod      │                  │  │
│  │  │ (Replicas: 3)  │  │ (Replicas: 2)  │                  │  │
│  │  └────────────────┘  └────────────────┘                  │  │
│  │                                                            │  │
│  │  ┌────────────────┐                                       │  │
│  │  │ Orchestrator   │                                       │  │
│  │  │ Service        │                                       │  │
│  │  │ (Replicas: 3)  │                                       │  │
│  │  └────────────────┘                                       │  │
│  │                                                            │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              Message Queue (MSK - Managed Kafka)          │  │
│  │  - 3 brokers                                              │  │
│  │  - Replication factor: 3                                  │  │
│  │  - Topics: inbound_events, agent_actions, audit_log      │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              Data Layer                                   │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │  │
│  │  │ RDS         │  │ ElastiCache │  │ Pinecone    │     │  │
│  │  │ PostgreSQL  │  │ Redis       │  │ (External)  │     │  │
│  │  │ Multi-AZ    │  │ Cluster     │  │             │     │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘     │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              Monitoring & Logging                         │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │  │
│  │  │ Datadog     │  │ CloudWatch  │  │ ELK Stack   │     │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘     │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
└────────────────────────────────────────────────────────────────┘

External Dependencies:
- OpenAI API (GPT-4)
- Anthropic API (Claude)
- MuleSoft Anypoint (Salesforce managed)
- Salesforce (via MuleSoft)
```

---

## Scaling Strategy

```
Normal Load (1000 conversations/hour)
┌──────────────────────────────────┐
│ Supervisor:    3 replicas        │
│ Lead Qual:     5 replicas        │
│ Support:      10 replicas        │
│ Sales Assist:  3 replicas        │
│ Data Enrich:   2 replicas        │
│                                  │
│ Total CPU:  ~30 cores            │
│ Total RAM:  ~60 GB               │
│ Cost:       ~$2000/month         │
└──────────────────────────────────┘

                ↓ (Scale up)

Peak Load (10,000 conversations/hour)
┌──────────────────────────────────┐
│ Supervisor:    10 replicas       │
│ Lead Qual:     20 replicas       │
│ Support:       40 replicas       │
│ Sales Assist:  10 replicas       │
│ Data Enrich:   5 replicas        │
│                                  │
│ Total CPU:  ~100 cores           │
│ Total RAM:  ~200 GB              │
│ Cost:       ~$6000/month         │
└──────────────────────────────────┘

Auto-scaling based on:
- Queue depth (Kafka lag)
- CPU utilization
- Response time SLA
- Error rate
```

---

## Security Architecture

```
┌──────────────────────────────────────────────────────────┐
│  External Actors                                          │
│  - Customers                                             │
│  - Malicious actors                                      │
└─────────────────┬────────────────────────────────────────┘
                  │
                  ▼
         ┌─────────────────┐
         │  WAF & DDoS      │
         │  Protection      │
         └────────┬─────────┘
                  │
                  ▼
         ┌─────────────────┐
         │  Load Balancer   │
         │  (TLS            │
         │  Termination)    │
         └────────┬─────────┘
                  │
                  ▼
         ┌─────────────────┐
         │  MuleSoft        │
         │  API Gateway     │
         │  - Auth/Authz    │
         │  - Rate Limit    │
         │  - Input Valid   │
         └────────┬─────────┘
                  │
         ┌────────┴─────────┐
         │                  │
         ▼                  ▼
┌──────────────┐   ┌──────────────┐
│ Agent        │   │ Enterprise   │
│ Services     │   │ Systems      │
│ (Private     │   │ (Private     │
│  Subnet)     │   │  Network)    │
└──────────────┘   └──────────────┘

Security Layers:
1. Network: VPC, subnets, security groups
2. Auth: JWT tokens, API keys, OAuth
3. Authorization: RBAC, agent permissions
4. Encryption: TLS in transit, encryption at rest
5. Audit: Comprehensive logging
6. Monitoring: Real-time security alerts
```

---

*Last Updated: March 2026*
*Version: 1.0*
