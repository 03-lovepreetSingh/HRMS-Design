# Design Document
# Autonomous AI HR Decision Agent

**Version:** 1.0  
**Date:** January 2025  
**Project:** AI-Powered HRMS  
**Team:** INVICTUS  
**Based on:** Requirements Document v1.0

---

## Table of Contents

1. Overview
2. Architecture
3. Components and Interfaces
4. Data Models
5. Correctness Properties
6. Error Handling
7. Testing Strategy

---

## 1. Overview

### 1.1 Design Philosophy

The Autonomous AI HR Decision Agent is designed around three core principles:

1. **AI-First Architecture**: Every user interaction flows through the AI layer, enabling natural language processing and intelligent decision-making
2. **Policy-Driven Decisions**: All AI decisions are grounded in company policies retrieved through semantic search, ensuring consistency and compliance
3. **Human-in-the-Loop**: Low-confidence decisions are escalated to humans, maintaining trust while maximizing automation

### 1.2 Key Design Decisions

**Decision 1: Monorepo with Turborepo**
- **Rationale**: Enables code sharing between frontend and backend, simplifies dependency management, and provides efficient build caching
- **Trade-offs**: Slightly more complex initial setup, but significant long-term maintainability benefits

**Decision 2: Next.js 14 with App Router**
- **Rationale**: Server Components reduce client-side JavaScript, improve performance, and enable server-side AI processing
- **Trade-offs**: Newer API requires team learning, but provides better performance and developer experience

**Decision 3: PostgreSQL with pgvector**
- **Rationale**: Combines relational data management with vector similarity search in a single database, reducing complexity
- **Trade-offs**: Requires PostgreSQL 14+, but eliminates need for separate vector database

**Decision 4: Claude Sonnet 4.5 as Primary LLM**
- **Rationale**: Superior reasoning capabilities for complex HR decisions, better context handling
- **Trade-offs**: Higher cost than smaller models, but critical for decision accuracy

**Decision 5: Confidence-Based Routing**
- **Rationale**: Balances automation with accuracy by escalating uncertain decisions
- **Trade-offs**: Reduces automation rate slightly, but maintains trust and accuracy

### 1.3 System Context

```
┌─────────────────────────────────────────────────────────────┐
│                    External Systems                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │   Claude     │  │    GPT-4     │  │   AWS SES    │     │
│  │   API        │  │     API      │  │   (Email)    │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│              Autonomous AI HR Decision Agent                 │
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │              Frontend (Next.js)                     │    │
│  │  - Chat Interface  - Dashboards  - Admin Panel     │    │
│  └────────────────────────────────────────────────────┘    │
│                            │                                 │
│  ┌────────────────────────────────────────────────────┐    │
│  │           Backend Services (Node.js)                │    │
│  │  - AI Decision Engine  - Leave Service              │    │
│  │  - Policy Service  - Escalation Service             │    │
│  └────────────────────────────────────────────────────┘    │
│                            │                                 │
│  ┌────────────────────────────────────────────────────┐    │
│  │         Data Layer (PostgreSQL + pgvector)          │    │
│  │  - User Data  - Policies  - Audit Logs             │    │
│  └────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                         Users                                │
│  Employee  │  Manager  │  HR Manager  │  Admin              │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Architecture

### 2.1 High-Level Architecture

The system follows a layered architecture with clear separation of concerns:

```
┌─────────────────────────────────────────────────────────────┐
│                    Presentation Layer                        │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Next.js Frontend (React Server Components)          │  │
│  │  - Chat UI  - Dashboards  - Forms  - Admin Panel    │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                            │
                    ┌───────┴───────┐
                    │  REST API     │
                    │  WebSocket    │
                    └───────┬───────┘
                            │
┌─────────────────────────────────────────────────────────────┐
│                   Application Layer                          │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              API Routes (Next.js)                     │  │
│  │  - Authentication  - Authorization  - Validation     │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                            │
┌─────────────────────────────────────────────────────────────┐
│                   Business Logic Layer                       │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │
│  │  AI Chat     │  │    Leave     │  │   Policy     │    │
│  │  Service     │  │   Service    │  │   Service    │    │
│  └──────────────┘  └──────────────┘  └──────────────┘    │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │
│  │ Escalation   │  │  Attendance  │  │    Task      │    │
│  │  Service     │  │   Service    │  │   Service    │    │
│  └──────────────┘  └──────────────┘  └──────────────┘    │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │
│  │ Notification │  │   Employee   │  │    Audit     │    │
│  │  Service     │  │   Service    │  │   Service    │    │
│  └──────────────┘  └──────────────┘  └──────────────┘    │
└─────────────────────────────────────────────────────────────┘
                            │
┌─────────────────────────────────────────────────────────────┐
│                   AI Decision Layer                          │
│  ┌──────────────────────────────────────────────────────┐  │
│  │           AI Decision Engine                          │  │
│  │  ┌────────────────┐  ┌────────────────┐             │  │
│  │  │   LangChain    │  │    Context     │             │  │
│  │  │  Orchestrator  │  │   Assembly     │             │  │
│  │  └────────────────┘  └────────────────┘             │  │
│  │  ┌────────────────┐  ┌────────────────┐             │  │
│  │  │   Confidence   │  │   Validation   │             │  │
│  │  │    Scoring     │  │     Rules      │             │  │
│  │  └────────────────┘  └────────────────┘             │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                            │
                    ┌───────┴───────┐
                    │               │
            ┌───────▼─────┐  ┌─────▼──────┐
            │   Claude    │  │   GPT-4    │
            │  Sonnet 4.5 │  │ (Fallback) │
            └─────────────┘  └────────────┘
                            │
┌─────────────────────────────────────────────────────────────┐
│                      Data Layer                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              Drizzle ORM                              │  │
│  └──────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │         PostgreSQL with pgvector                      │  │
│  │  - Relational Tables  - Vector Embeddings            │  │
│  └──────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              AWS S3 (Policy Documents)                │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 Monorepo Structure

```
hrms/
├── apps/
│   ├── web/                    # Next.js frontend application
│   │   ├── app/               # App Router pages
│   │   │   ├── (auth)/        # Authentication pages
│   │   │   ├── (dashboard)/   # Dashboard pages
│   │   │   ├── api/           # API routes
│   │   │   └── layout.tsx     # Root layout
│   │   ├── components/        # React components
│   │   │   ├── chat/          # Chat interface components
│   │   │   ├── dashboard/     # Dashboard components
│   │   │   ├── ui/            # shadcn/ui components
│   │   │   └── shared/        # Shared components
│   │   └── lib/               # Utility functions
│   │
│   └── api/                    # Backend API (if separate)
│       ├── src/
│       │   ├── services/      # Business logic services
│       │   ├── routes/        # API routes
│       │   └── middleware/    # Express middleware
│       └── package.json
│
├── packages/
│   ├── database/              # Database schema and migrations
│   │   ├── schema/            # Drizzle schema definitions
│   │   ├── migrations/        # Database migrations
│   │   └── seed/              # Seed data
│   │
│   ├── ai/                    # AI/LLM integration
│   │   ├── decision-engine/   # Decision engine logic
│   │   ├── langchain/         # LangChain setup
│   │   └── prompts/           # Prompt templates
│   │
│   ├── types/                 # Shared TypeScript types
│   │   └── index.ts
│   │
│   ├── config/                # Shared configuration
│   │   └── index.ts
│   │
│   └── ui/                    # Shared UI components
│       └── components/
│
├── turbo.json                 # Turborepo configuration
├── package.json               # Root package.json
└── tsconfig.json              # Root TypeScript config
```

### 2.3 Data Flow

#### 2.3.1 Leave Request Flow

```
Employee → Chat UI → API Route → AI Chat Service
                                      ↓
                            Extract Intent & Entities
                                      ↓
                            AI Decision Engine
                                      ↓
                    ┌─────────────────┴─────────────────┐
                    │                                    │
            Context Assembly                    Policy Retrieval
                    │                                    │
        ┌───────────┼───────────┐                      │
        │           │           │                       │
    User Profile  Leave Balance  Historical      Vector Search
                                 Decisions        (pgvector)
                    │           │           │           │
                    └───────────┴───────────┴───────────┘
                                      ↓
                            LLM Processing (Claude)
                                      ↓
                        ┌─────────────┴─────────────┐
                        │                           │
                Confidence >= 85%          Confidence < 85%
                        │                           │
                  Auto-Approve                  Escalate
                        │                           │
                  Update Records            Escalation Queue
                        │                           │
                  Send Notifications        Notify Approver
                        │                           │
                  Audit Log                   Audit Log
                        │                           │
                        └───────────┬───────────────┘
                                    ↓
                            Response to Employee
```

#### 2.3.2 Policy Search Flow

```
User Query → AI Chat Service
                  ↓
        Generate Query Embedding
        (OpenAI text-embedding-3-small)
                  ↓
        Vector Similarity Search
        (pgvector cosine similarity)
                  ↓
        Retrieve Top 5 Policy Chunks
        (with similarity scores)
                  ↓
        Filter by Effective Date
                  ↓
        Return Relevant Context
                  ↓
        LLM Synthesizes Response
```

---

## 3. Components and Interfaces

### 3.1 Frontend Components

#### 3.1.1 Chat Interface Component

**Purpose**: Provide conversational interface for all HR interactions

**Component Structure**:
```typescript
// components/chat/ChatInterface.tsx
interface ChatInterfaceProps {
  userId: string;
  conversationId?: string;
}

interface Message {
  id: string;
  role: 'user' | 'assistant' | 'system';
  content: string;
  timestamp: Date;
  metadata?: {
    confidence?: number;
    policyReferences?: string[];
    actionButtons?: ActionButton[];
  };
}

interface ActionButton {
  label: string;
  action: string;
  variant: 'primary' | 'secondary';
}

// Main component
export function ChatInterface({ userId, conversationId }: ChatInterfaceProps) {
  const [messages, setMessages] = useState<Message[]>([]);
  const [input, setInput] = useState('');
  const [isTyping, setIsTyping] = useState(false);
  
  // WebSocket connection for real-time updates
  const ws = useWebSocket(`/api/chat/ws?userId=${userId}`);
  
  // Send message handler
  const sendMessage = async (content: string) => {
    // Implementation
  };
  
  return (
    <div className="chat-container">
      <MessageList messages={messages} />
      <TypingIndicator visible={isTyping} />
      <MessageInput value={input} onChange={setInput} onSend={sendMessage} />
    </div>
  );
}
```

**Key Features**:
- Real-time message streaming via WebSocket
- Markdown rendering for formatted responses
- Action buttons for quick actions (approve, reject, view details)
- Policy reference links
- Conversation history with infinite scroll
- Mobile-responsive design

---

#### 3.1.2 Dashboard Components

**Employee Dashboard**:
```typescript
// components/dashboard/EmployeeDashboard.tsx
interface DashboardData {
  leaveBalance: LeaveBalance[];
  upcomingLeaves: Leave[];
  pendingRequests: Request[];
  recentAttendance: Attendance[];
  pendingTasks: Task[];
}

export function EmployeeDashboard({ userId }: { userId: string }) {
  const { data, isLoading } = useDashboardData(userId);
  
  return (
    <div className="dashboard-grid">
      <LeaveBalanceCard balances={data.leaveBalance} />
      <UpcomingLeavesCard leaves={data.upcomingLeaves} />
      <PendingRequestsCard requests={data.pendingRequests} />
      <AttendanceCard attendance={data.recentAttendance} />
      <TasksCard tasks={data.pendingTasks} />
      <QuickActionsCard />
    </div>
  );
}
```

**Manager Dashboard**:
```typescript
// components/dashboard/ManagerDashboard.tsx
interface ManagerDashboardData {
  teamSize: number;
  pendingApprovals: number;
  teamLeaveCalendar: CalendarEvent[];
  attendanceSummary: AttendanceSummary;
  taskCompletionRate: number;
  escalations: Escalation[];
}

export function ManagerDashboard({ managerId }: { managerId: string }) {
  const { data } = useManagerDashboard(managerId);
  
  return (
    <div className="dashboard-grid">
      <TeamOverviewCard data={data} />
      <PendingApprovalsCard count={data.pendingApprovals} />
      <TeamLeaveCalendar events={data.teamLeaveCalendar} />
      <AttendanceSummaryCard summary={data.attendanceSummary} />
      <EscalationsCard escalations={data.escalations} />
    </div>
  );
}
```

---

### 3.2 Backend Services

#### 3.2.1 AI Chat Service

**Purpose**: Process natural language queries and coordinate with AI Decision Engine

**Interface**:
```typescript
// services/ai-chat/AIChatService.ts
interface ChatMessage {
  conversationId: string;
  userId: string;
  content: string;
  role: 'user' | 'assistant';
}

interface ChatResponse {
  messageId: string;
  content: string;
  metadata: {
    intent: string;
    entities: Record<string, any>;
    confidence: number;
    policyReferences?: string[];
    actionRequired?: boolean;
  };
}

class AIChatService {
  /**
   * Process user message and generate AI response
   */
  async processMessage(message: ChatMessage): Promise<ChatResponse> {
    // 1. Extract intent and entities
    const { intent, entities } = await this.extractIntent(message.content);
    
    // 2. Route to appropriate handler
    switch (intent) {
      case 'leave_request':
        return await this.handleLeaveRequest(message, entities);
      case 'policy_query':
        return await this.handlePolicyQuery(message, entities);
      case 'leave_balance_query':
        return await this.handleLeaveBalanceQuery(message, entities);
      case 'attendance_regularization':
        return await this.handleAttendanceRegularization(message, entities);
      default:
        return await this.handleGeneralQuery(message);
    }
  }
  
  /**
   * Extract intent and entities from user message
   */
  private async extractIntent(content: string): Promise<{
    intent: string;
    entities: Record<string, any>;
  }> {
    const prompt = `
      Extract the intent and entities from this HR query:
      "${content}"
      
      Return JSON with:
      - intent: one of [leave_request, policy_query, leave_balance_query, attendance_regularization, general_query]
      - entities: relevant extracted information
    `;
    
    const response = await this.llmClient.complete(prompt);
    return JSON.parse(response);
  }
  
  /**
   * Handle leave request
   */
  private async handleLeaveRequest(
    message: ChatMessage,
    entities: any
  ): Promise<ChatResponse> {
    // Validate entities
    const { leaveType, startDate, endDate, reason } = entities;
    
    // Call AI Decision Engine
    const decision = await this.decisionEngine.evaluateLeaveRequest({
      userId: message.userId,
      leaveType,
      startDate,
      endDate,
      reason
    });
    
    // Generate response
    return this.formatLeaveResponse(decision);
  }
}
```

---

#### 3.2.2 AI Decision Engine

**Purpose**: Core decision-making logic using LLM and policy context

**Interface**:
```typescript
// services/ai-decision/DecisionEngine.ts
interface DecisionContext {
  userProfile: UserProfile;
  policies: PolicyChunk[];
  historicalDecisions: Decision[];
  teamContext: TeamContext;
}

interface DecisionRequest {
  type: 'leave' | 'attendance' | 'policy_exception';
  userId: string;
  data: Record<string, any>;
}

interface DecisionResult {
  decision: 'approve' | 'reject' | 'escalate';
  confidence: number;
  rationale: string;
  policyReferences: string[];
  validationErrors?: string[];
}

class DecisionEngine {
  /**
   * Evaluate a decision request
   */
  async evaluate(request: DecisionRequest): Promise<DecisionResult> {
    // 1. Assemble context
    const context = await this.assembleContext(request);
    
    // 2. Validate against business rules
    const validation = await this.validateRules(request, context);
    if (!validation.isValid) {
      return {
        decision: 'reject',
        confidence: 100,
        rationale: validation.errors.join(', '),
        policyReferences: [],
        validationErrors: validation.errors
      };
    }
    
    // 3. Get LLM decision
    const llmDecision = await this.getLLMDecision(request, context);
    
    // 4. Apply confidence-based routing
    if (llmDecision.confidence >= 85) {
      return llmDecision;
    } else if (llmDecision.confidence >= 70) {
      // Execute but flag for review
      return { ...llmDecision, decision: llmDecision.decision };
    } else {
      // Escalate
      return { ...llmDecision, decision: 'escalate' };
    }
  }
  
  /**
   * Assemble decision context
   */
  private async assembleContext(
    request: DecisionRequest
  ): Promise<DecisionContext> {
    const [userProfile, policies, historicalDecisions, teamContext] = 
      await Promise.all([
        this.getUserProfile(request.userId),
        this.getRelevantPolicies(request),
        this.getHistoricalDecisions(request),
        this.getTeamContext(request.userId)
      ]);
    
    return { userProfile, policies, historicalDecisions, teamContext };
  }
  
  /**
   * Get LLM decision
   */
  private async getLLMDecision(
    request: DecisionRequest,
    context: DecisionContext
  ): Promise<DecisionResult> {
    const prompt = this.buildDecisionPrompt(request, context);
    
    try {
      // Try Claude first
      const response = await this.claudeClient.complete(prompt);
      return this.parseDecisionResponse(response);
    } catch (error) {
      // Fallback to GPT-4
      console.warn('Claude failed, falling back to GPT-4', error);
      const response = await this.gpt4Client.complete(prompt);
      return this.parseDecisionResponse(response);
    }
  }
  
  /**
   * Build decision prompt
   */
  private buildDecisionPrompt(
    request: DecisionRequest,
    context: DecisionContext
  ): string {
    return `
You are an AI HR assistant making a decision on a ${request.type} request.

USER PROFILE:
${JSON.stringify(context.userProfile, null, 2)}

RELEVANT POLICIES:
${context.policies.map(p => p.content).join('\n\n')}

HISTORICAL SIMILAR DECISIONS:
${context.historicalDecisions.map(d => 
  `Request: ${d.request}\nDecision: ${d.decision}\nRationale: ${d.rationale}`
).join('\n\n')}

TEAM CONTEXT:
${JSON.stringify(context.teamContext, null, 2)}

CURRENT REQUEST:
${JSON.stringify(request.data, null, 2)}

Based on the above information, make a decision and provide:
1. Decision: approve, reject, or escalate
2. Confidence: 0-100 (how confident are you in this decision?)
3. Rationale: Clear explanation referencing specific policies
4. Policy References: IDs of policies used

Return JSON format:
{
  "decision": "approve|reject|escalate",
  "confidence": 85,
  "rationale": "...",
  "policyReferences": ["policy-id-1", "policy-id-2"]
}
`;
  }
  
  /**
   * Validate business rules
   */
  private async validateRules(
    request: DecisionRequest,
    context: DecisionContext
  ): Promise<{ isValid: boolean; errors: string[] }> {
    const errors: string[] = [];
    
    if (request.type === 'leave') {
      const { leaveType, startDate, endDate } = request.data;
      const balance = context.userProfile.leaveBalance[leaveType];
      const days = this.calculateLeaveDays(startDate, endDate);
      
      // Check leave balance
      if (balance < days) {
        errors.push(`Insufficient ${leaveType} leave balance. Available: ${balance}, Requested: ${days}`);
      }
      
      // Check overlapping leaves
      const hasOverlap = await this.checkOverlappingLeaves(
        request.userId,
        startDate,
        endDate
      );
      if (hasOverlap) {
        errors.push('Leave dates overlap with existing approved leave');
      }
      
      // Check advance notice
      const advanceNoticeDays = this.calculateAdvanceNotice(startDate);
      if (advanceNoticeDays < 0) {
        errors.push('Cannot request leave for past dates');
      }
    }
    
    return {
      isValid: errors.length === 0,
      errors
    };
  }
}
```

---

#### 3.2.3 Policy Service

**Purpose**: Manage policy documents and semantic search

**Interface**:
```typescript
// services/policy/PolicyService.ts
interface Policy {
  id: string;
  title: string;
  category: string;
  content: string;
  version: number;
  status: 'draft' | 'active' | 'archived';
  effectiveDate: Date;
}

interface PolicyChunk {
  id: string;
  policyId: string;
  content: string;
  embedding: number[];
  chunkIndex: number;
}

interface SearchResult {
  chunk: PolicyChunk;
  similarity: number;
  policy: Policy;
}

class PolicyService {
  /**
   * Upload and process policy document
   */
  async uploadPolicy(file: File, metadata: Partial<Policy>): Promise<Policy> {
    // 1. Extract text from file
    const text = await this.extractText(file);
    
    // 2. Create policy record
    const policy = await this.db.insert(policies).values({
      ...metadata,
      content: text,
      status: 'draft',
      version: 1
    });
    
    // 3. Chunk text
    const chunks = this.chunkText(text, 500); // 500 tokens per chunk
    
    // 4. Generate embeddings
    const embeddings = await this.generateEmbeddings(chunks);
    
    // 5. Store embeddings
    await this.storeEmbeddings(policy.id, chunks, embeddings);
    
    return policy;
  }
  
  /**
   * Semantic search for relevant policies
   */
  async searchPolicies(
    query: string,
    limit: number = 5
  ): Promise<SearchResult[]> {
    // 1. Generate query embedding
    const queryEmbedding = await this.generateEmbedding(query);
    
    // 2. Vector similarity search using pgvector
    const results = await this.db.execute(sql`
      SELECT 
        pe.id,
        pe.policy_id,
        pe.chunk_text,
        pe.chunk_index,
        p.title,
        p.category,
        p.effective_date,
        1 - (pe.embedding <=> ${queryEmbedding}::vector) as similarity
      FROM policy_embeddings pe
      JOIN policies p ON pe.policy_id = p.id
      WHERE p.status = 'active'
        AND p.effective_date <= NOW()
      ORDER BY pe.embedding <=> ${queryEmbedding}::vector
      LIMIT ${limit}
    `);
    
    return results.map(r => ({
      chunk: {
        id: r.id,
        policyId: r.policy_id,
        content: r.chunk_text,
        embedding: [],
        chunkIndex: r.chunk_index
      },
      similarity: r.similarity,
      policy: {
        id: r.policy_id,
        title: r.title,
        category: r.category,
        effectiveDate: r.effective_date
      }
    }));
  }
  
  /**
   * Generate embeddings using OpenAI
   */
  private async generateEmbeddings(texts: string[]): Promise<number[][]> {
    const response = await this.openai.embeddings.create({
      model: 'text-embedding-3-small',
      input: texts
    });
    
    return response.data.map(d => d.embedding);
  }
  
  /**
   * Chunk text into smaller pieces
   */
  private chunkText(text: string, maxTokens: number): string[] {
    // Simple chunking by paragraphs with token limit
    const paragraphs = text.split('\n\n');
    const chunks: string[] = [];
    let currentChunk = '';
    let currentTokens = 0;
    
    for (const para of paragraphs) {
      const paraTokens = this.estimateTokens(para);
      
      if (currentTokens + paraTokens > maxTokens && currentChunk) {
        chunks.push(currentChunk.trim());
        currentChunk = para;
        currentTokens = paraTokens;
      } else {
        currentChunk += '\n\n' + para;
        currentTokens += paraTokens;
      }
    }
    
    if (currentChunk) {
      chunks.push(currentChunk.trim());
    }
    
    return chunks;
  }
}
```

---

#### 3.2.4 Leave Service

**Purpose**: Manage leave requests, balances, and approvals

**Interface**:
```typescript
// services/leave/LeaveService.ts
interface LeaveRequest {
  userId: string;
  leaveType: 'casual' | 'sick' | 'earned' | 'unpaid';
  startDate: Date;
  endDate: Date;
  reason: string;
}

interface LeaveBalance {
  userId: string;
  year: number;
  leaveType: string;
  allocated: number;
  used: number;
  pending: number;
  available: number;
}

class LeaveService {
  /**
   * Submit leave request
   */
  async submitLeaveRequest(request: LeaveRequest): Promise<Leave> {
    // 1. Calculate leave days
    const days = this.calculateLeaveDays(request.startDate, request.endDate);
    
    // 2. Create leave record
    const leave = await this.db.insert(leaves).values({
      ...request,
      daysCount: days,
      status: 'pending',
      decisionType: 'pending'
    });
    
    // 3. Update pending balance
    await this.updatePendingBalance(request.userId, request.leaveType, days);
    
    // 4. Trigger AI decision
    const decision = await this.decisionEngine.evaluate({
      type: 'leave',
      userId: request.userId,
      data: request
    });
    
    // 5. Process decision
    if (decision.decision === 'approve') {
      await this.approveLeave(leave.id, 'AI', decision);
    } else if (decision.decision === 'escalate') {
      await this.escalateLeave(leave.id, decision);
    } else {
      await this.rejectLeave(leave.id, 'AI', decision);
    }
    
    return leave;
  }
  
  /**
   * Approve leave
   */
  async approveLeave(
    leaveId: string,
    approvedBy: string,
    decision: DecisionResult
  ): Promise<void> {
    // 1. Update leave status
    await this.db.update(leaves)
      .set({
        status: 'approved',
        approvedBy,
        approvedAt: new Date(),
        decisionType: approvedBy === 'AI' ? 'ai_auto' : 'human',
        aiConfidence: decision.confidence,
        aiRationale: decision.rationale
      })
      .where(eq(leaves.id, leaveId));
    
    // 2. Update leave balance
    const leave = await this.getLeave(leaveId);
    await this.updateLeaveBalance(
      leave.userId,
      leave.leaveType,
      -leave.daysCount,
      'used'
    );
    await this.updateLeaveBalance(
      leave.userId,
      leave.leaveType,
      -leave.daysCount,
      'pending'
    );
    
    // 3. Send notifications
    await this.notificationService.sendLeaveApproval(leave);
    
    // 4. Audit log
    await this.auditService.log({
      action: 'leave_approved',
      entityType: 'leave',
      entityId: leaveId,
      userId: leave.userId,
      metadata: { decision, approvedBy }
    });
  }
  
  /**
   * Calculate leave days excluding weekends and holidays
   */
  private calculateLeaveDays(startDate: Date, endDate: Date): number {
    let days = 0;
    const current = new Date(startDate);
    
    while (current <= endDate) {
      // Skip weekends (Saturday = 6, Sunday = 0)
      if (current.getDay() !== 0 && current.getDay() !== 6) {
        // Check if not a holiday
        if (!this.isHoliday(current)) {
          days++;
        }
      }
      current.setDate(current.getDate() + 1);
    }
    
    return days;
  }
  
  /**
   * Get leave balance for user
   */
  async getLeaveBalance(userId: string, year: number): Promise<LeaveBalance[]> {
    return await this.db.select()
      .from(leaveBalances)
      .where(
        and(
          eq(leaveBalances.userId, userId),
          eq(leaveBalances.year, year)
        )
      );
  }
}
```

---

#### 3.2.5 Escalation Service

**Purpose**: Manage escalation routing and queue

**Interface**:
```typescript
// services/escalation/EscalationService.ts
interface EscalationRequest {
  requestType: 'leave' | 'attendance' | 'policy_exception' | 'other';
  requestId: string;
  requesterId: string;
  aiRecommendation: string;
  aiConfidence: number;
  priority: 'low' | 'medium' | 'high' | 'critical';
}

interface Escalation {
  id: string;
  requestType: string;
  requestId: string;
  requesterId: string;
  assignedTo: string;
  status: 'pending' | 'in_review' | 'approved' | 'rejected';
  priority: string;
  aiRecommendation: string;
  aiConfidence: number;
  decisionRationale?: string;
  createdAt: Date;
  resolvedAt?: Date;
  slaBreachAt: Date;
}

class EscalationService {
  /**
   * Create escalation
   */
  async createEscalation(request: EscalationRequest): Promise<Escalation> {
    // 1. Determine approver
    const approver = await this.determineApprover(request);
    
    // 2. Calculate SLA breach time
    const slaHours = this.getSLAHours(approver.role);
    const slaBreachAt = new Date(Date.now() + slaHours * 60 * 60 * 1000);
    
    // 3. Create escalation
    const escalation = await this.db.insert(escalations).values({
      ...request,
      assignedTo: approver.id,
      status: 'pending',
      slaBreachAt
    });
    
    // 4. Send notification
    await this.notificationService.sendEscalationNotification(
      approver.id,
      escalation
    );
    
    // 5. Audit log
    await this.auditService.log({
      action: 'escalation_created',
      entityType: 'escalation',
      entityId: escalation.id,
      userId: request.requesterId,
      metadata: { request, approver: approver.id }
    });
    
    return escalation;
  }
  
  /**
   * Determine appropriate approver
   */
  private async determineApprover(
    request: EscalationRequest
  ): Promise<User> {
    const requester = await this.userService.getUser(request.requesterId);
    
    // Routing logic
    if (request.requestType === 'leave') {
      const leave = await this.leaveService.getLeave(request.requestId);
      
      // Extended leave (> 10 days) → HR
      if (leave.daysCount > 10) {
        return await this.getHRManager();
      }
      
      // Short notice (< 2 days) or > 3 days → Manager
      const advanceNotice = this.calculateAdvanceNotice(leave.startDate);
      if (advanceNotice < 2 || leave.daysCount > 3) {
        return await this.userService.getUser(requester.managerId);
      }
    }
    
    if (request.requestType === 'policy_exception') {
      return await this.getHRManager();
    }
    
    // Default to manager
    return await this.userService.getUser(requester.managerId);
  }
  
  /**
   * Get SLA hours based on approver role
   */
  private getSLAHours(role: string): number {
    const slaMap = {
      'manager': 4,
      'hr_manager': 8,
      'admin': 24
    };
    return slaMap[role] || 24;
  }
  
  /**
   * Resolve escalation
   */
  async resolveEscalation(
    escalationId: string,
    decision: 'approve' | 'reject',
    rationale: string,
    resolvedBy: string
  ): Promise<void> {
    // 1. Update escalation
    await this.db.update(escalations)
      .set({
        status: decision === 'approve' ? 'approved' : 'rejected',
        decisionRationale: rationale,
        resolvedAt: new Date()
      })
      .where(eq(escalations.id, escalationId));
    
    // 2. Process original request
    const escalation = await this.getEscalation(escalationId);
    
    if (escalation.requestType === 'leave') {
      if (decision === 'approve') {
        await this.leaveService.approveLeave(
          escalation.requestId,
          resolvedBy,
          { decision: 'approve', confidence: 100, rationale, policyReferences: [] }
        );
      } else {
        await this.leaveService.rejectLeave(
          escalation.requestId,
          resolvedBy,
          { decision: 'reject', confidence: 100, rationale, policyReferences: [] }
        );
      }
    }
    
    // 3. Send notifications
    await this.notificationService.sendEscalationResolution(escalation, decision);
    
    // 4. Audit log
    await this.auditService.log({
      action: 'escalation_resolved',
      entityType: 'escalation',
      entityId: escalationId,
      userId: resolvedBy,
      metadata: { decision, rationale }
    });
  }
}
```

---

### 3.3 LLM Integration

#### 3.3.1 LangChain Setup

**Purpose**: Orchestrate LLM calls with fallback and retry logic

**Implementation**:
```typescript
// packages/ai/langchain/LLMClient.ts
import { ChatAnthropic } from '@langchain/anthropic';
import { ChatOpenAI } from '@langchain/openai';

interface LLMConfig {
  claudeApiKey: string;
  openaiApiKey: string;
  temperature: number;
  maxTokens: number;
}

class LLMClient {
  private claude: ChatAnthropic;
  private gpt4: ChatOpenAI;
  
  constructor(config: LLMConfig) {
    this.claude = new ChatAnthropic({
      apiKey: config.claudeApiKey,
      model: 'claude-sonnet-4.5',
      temperature: config.temperature,
      maxTokens: config.maxTokens
    });
    
    this.gpt4 = new ChatOpenAI({
      apiKey: config.openaiApiKey,
      model: 'gpt-4',
      temperature: config.temperature,
      maxTokens: config.maxTokens
    });
  }
  
  /**
   * Complete prompt with automatic fallback
   */
  async complete(
    prompt: string,
    options?: { useFallback?: boolean }
  ): Promise<string> {
    try {
      // Try Claude first
      const response = await this.claude.invoke([
        { role: 'user', content: prompt }
      ]);
      return response.content as string;
    } catch (error) {
      console.error('Claude API error:', error);
      
      if (options?.useFallback !== false) {
        // Fallback to GPT-4
        console.log('Falling back to GPT-4');
        try {
          const response = await this.gpt4.invoke([
            { role: 'user', content: prompt }
          ]);
          return response.content as string;
        } catch (fallbackError) {
          console.error('GPT-4 API error:', fallbackError);
          throw new Error('Both LLM providers failed');
        }
      }
      
      throw error;
    }
  }
  
  /**
   * Complete with retry logic
   */
  async completeWithRetry(
    prompt: string,
    maxRetries: number = 3
  ): Promise<string> {
    let lastError: Error;
    
    for (let i = 0; i < maxRetries; i++) {
      try {
        return await this.complete(prompt);
      } catch (error) {
        lastError = error as Error;
        console.log(`Retry ${i + 1}/${maxRetries} after error:`, error);
        
        // Exponential backoff
        await this.sleep(Math.pow(2, i) * 1000);
      }
    }
    
    throw lastError!;
  }
  
  private sleep(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}
```

---

### 3.4 Database Schema (Drizzle ORM)

**Key Tables**:
```typescript
// packages/database/schema/users.ts
import { pgTable, uuid, varchar, timestamp, pgEnum } from 'drizzle-orm/pg-core';

export const roleEnum = pgEnum('role', ['admin', 'hr_manager', 'manager', 'employee']);
export const statusEnum = pgEnum('status', ['active', 'inactive', 'on_leave']);

export const users = pgTable('users', {
  id: uuid('id').primaryKey().defaultRandom(),
  employeeId: varchar('employee_id', { length: 50 }).unique().notNull(),
  email: varchar('email', { length: 255 }).unique().notNull(),
  passwordHash: varchar('password_hash', { length: 255 }).notNull(),
  name: varchar('name', { length: 255 }).notNull(),
  phone: varchar('phone', { length: 20 }),
  role: roleEnum('role').notNull(),
  department: varchar('department', { length: 100 }),
  designation: varchar('designation', { length: 100 }),
  managerId: uuid('manager_id').references(() => users.id),
  joinDate: timestamp('join_date').notNull(),
  status: statusEnum('status').default('active'),
  createdAt: timestamp('created_at').defaultNow(),
  updatedAt: timestamp('updated_at').defaultNow()
});
```

```typescript
// packages/database/schema/leaves.ts
import { pgTable, uuid, varchar, date, integer, timestamp, pgEnum } from 'drizzle-orm/pg-core';

export const leaveTypeEnum = pgEnum('leave_type', ['casual', 'sick', 'earned', 'unpaid']);
export const leaveStatusEnum = pgEnum('leave_status', ['pending', 'approved', 'rejected', 'cancelled']);
export const decisionTypeEnum = pgEnum('decision_type', ['ai_auto', 'ai_recommended', 'human']);

export const leaves = pgTable('leaves', {
  id: uuid('id').primaryKey().defaultRandom(),
  userId: uuid('user_id').references(() => users.id).notNull(),
  leaveType: leaveTypeEnum('leave_type').notNull(),
  startDate: date('start_date').notNull(),
  endDate: date('end_date').notNull(),
  daysCount: integer('days_count').notNull(),
  reason: varchar('reason', { length: 500 }),
  status: leaveStatusEnum('status').default('pending'),
  approvedBy: uuid('approved_by').references(() => users.id),
  approvedAt: timestamp('approved_at'),
  decisionType: decisionTypeEnum('decision_type'),
  aiConfidence: integer('ai_confidence'),
  aiRationale: varchar('ai_rationale', { length: 1000 }),
  createdAt: timestamp('created_at').defaultNow(),
  updatedAt: timestamp('updated_at').defaultNow()
});
```

```typescript
// packages/database/schema/policies.ts
import { pgTable, uuid, varchar, text, integer, timestamp, pgEnum } from 'drizzle-orm/pg-core';
import { vector } from 'pgvector/drizzle-orm';

export const policyStatusEnum = pgEnum('policy_status', ['draft', 'active', 'archived']);

export const policies = pgTable('policies', {
  id: uuid('id').primaryKey().defaultRandom(),
  title: varchar('title', { length: 255 }).notNull(),
  category: varchar('category', { length: 100 }).notNull(),
  content: text('content').notNull(),
  version: integer('version').default(1),
  status: policyStatusEnum('status').default('draft'),
  effectiveDate: timestamp('effective_date'),
  createdBy: uuid('created_by').references(() => users.id),
  approvedBy: uuid('approved_by').references(() => users.id),
  createdAt: timestamp('created_at').defaultNow(),
  updatedAt: timestamp('updated_at').defaultNow()
});

export const policyEmbeddings = pgTable('policy_embeddings', {
  id: uuid('id').primaryKey().defaultRandom(),
  policyId: uuid('policy_id').references(() => policies.id).notNull(),
  chunkText: text('chunk_text').notNull(),
  embedding: vector('embedding', { dimensions: 1536 }),
  chunkIndex: integer('chunk_index').notNull(),
  createdAt: timestamp('created_at').defaultNow()
});
```

---

## 4. Data Models

### 4.1 Core Domain Models

**User Model**:
```typescript
interface User {
  id: string;
  employeeId: string;
  email: string;
  name: string;
  role: 'admin' | 'hr_manager' | 'manager' | 'employee';
  department: string;
  designation: string;
  managerId?: string;
  joinDate: Date;
  status: 'active' | 'inactive' | 'on_leave';
}
```

**Leave Model**:
```typescript
interface Leave {
  id: string;
  userId: string;
  leaveType: 'casual' | 'sick' | 'earned' | 'unpaid';
  startDate: Date;
  endDate: Date;
  daysCount: number;
  reason: string;
  status: 'pending' | 'approved' | 'rejected' | 'cancelled';
  approvedBy?: string;
  approvedAt?: Date;
  decisionType: 'ai_auto' | 'ai_recommended' | 'human';
  aiConfidence?: number;
  aiRationale?: string;
}
```

**Policy Model**:
```typescript
interface Policy {
  id: string;
  title: string;
  category: string;
  content: string;
  version: number;
  status: 'draft' | 'active' | 'archived';
  effectiveDate: Date;
  createdBy: string;
  approvedBy?: string;
}
```

**Decision Context Model**:
```typescript
interface DecisionContext {
  userProfile: User;
  leaveBalance: LeaveBalance[];
  policies: PolicyChunk[];
  historicalDecisions: Decision[];
  teamContext: {
    managerId: string;
    teamSize: number;
    teamOnLeave: number;
    teamCapacity: number;
  };
}
```

### 4.2 Data Relationships

```
User (1) ──── (N) Leave
  │
  │ (manager)
  │
  └──── (N) User (reports)

User (1) ──── (N) LeaveBalance

Policy (1) ──── (N) PolicyEmbedding

User (1) ──── (N) AuditLog

Leave (1) ──── (0..1) Escalation

User (1) ──── (N) Escalation (assigned_to)
```

---

## 5. Correctness Properties

### 5.1 What Are Correctness Properties?

A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.

Unlike unit tests that verify specific examples, property-based tests verify universal rules that should hold for all inputs. This provides much stronger correctness guarantees and catches edge cases that might be missed by example-based testing.

### 5.2 Authentication and Authorization Properties

**Property 1: Valid Credentials Generate JWT**
*For any* valid user credentials (email and password matching a user in the database), the authentication system should generate a JWT token containing the user's ID and role information.
**Validates: Requirements UM-001.1**

**Property 2: Invalid Credentials Are Rejected**
*For any* invalid credentials (non-existent email, incorrect password, or malformed input), the authentication system should reject the login attempt and create an audit log entry.
**Validates: Requirements UM-001.2**

**Property 3: Expired Tokens Require Re-authentication**
*For any* JWT token with an expiration timestamp in the past, any API request using that token should be rejected with a 401 status and require re-authentication.
**Validates: Requirements UM-001.3**

**Property 4: Password Complexity Enforcement**
*For any* password string, the system should accept it only if it contains at least 8 characters, at least one uppercase letter, at least one lowercase letter, at least one number, and at least one special character.
**Validates: Requirements UM-001.4**

**Property 5: RBAC Permission Enforcement**
*For any* user with a specific role attempting to access any resource, the system should grant access if and only if that role has the required permission for that resource, otherwise returning a 403 error.
**Validates: Requirements UM-002.2, UM-002.3**

**Property 6: Role Changes Update Permissions Immediately**
*For any* user whose role is changed, any subsequent permission check should reflect the new role's permissions without requiring logout or token refresh.
**Validates: Requirements UM-002.5**

### 5.3 AI Chat and NLP Properties

**Property 7: All Queries Are Processed**
*For any* text query submitted by a user, the AI chat interface should process it through the LLM and return a response (either an answer, a clarifying question, or an error message).
**Validates: Requirements AI-001.1**

**Property 8: Ambiguous Queries Trigger Clarification**
*For any* query that the AI cannot parse with sufficient confidence (confidence < 60%), the system should respond with clarifying questions rather than making assumptions.
**Validates: Requirements AI-001.5**

### 5.4 Leave Management Properties

**Property 9: Leave Entity Extraction**
*For any* natural language leave request containing leave type, dates, and reason, the system should correctly extract all four entities (leave type, start date, end date, reason) with at least 90% accuracy.
**Validates: Requirements LM-001.1**

**Property 10: Leave Balance Validation**
*For any* leave request, if the requested days exceed the user's available balance for that leave type, the system should reject the request with a clear error message indicating insufficient balance.
**Validates: Requirements LM-001.2**

**Property 11: Overlapping Leave Detection**
*For any* leave request with date range [start, end], if there exists an approved leave for the same user with overlapping dates, the system should reject the new request.
**Validates: Requirements LM-001.3**

**Property 12: Leave Days Calculation**
*For any* date range [start, end], the calculated leave days should equal the number of weekdays (Monday-Friday) in that range, excluding any dates marked as company holidays.
**Validates: Requirements LM-001.4**

**Property 13: Auto-Approval Based on Criteria**
*For any* leave request where (1) leave balance is sufficient, (2) no team conflicts exist, (3) duration ≤ 3 days, and (4) advance notice ≥ 2 days, the AI decision engine should auto-approve the request with confidence ≥ 85%.
**Validates: Requirements LM-002.1, LM-002.2**

**Property 14: Leave Balance Update on Approval**
*For any* leave request that is approved, the user's leave balance for that leave type should be decremented by the number of leave days within 1 second of approval.
**Validates: Requirements LM-002.4**

### 5.5 Policy Management Properties

**Property 15: Policy Upload Creates Embeddings**
*For any* policy document uploaded in supported format (PDF, DOCX, TXT), the system should extract text, chunk it into sections, generate vector embeddings for each chunk, and store them in the vector database.
**Validates: Requirements PM-001.2**

**Property 16: Policy Version Archival**
*For any* policy that is updated, the previous version should be moved to archived status with its original effective date preserved, and the new version should become active.
**Validates: Requirements PM-001.5**

**Property 17: Semantic Policy Search**
*For any* query string, the policy search should perform vector similarity search and return results ordered by cosine similarity score in descending order.
**Validates: Requirements PM-002.1**

**Property 18: Top-K Search Results**
*For any* policy search query, the system should return at most 5 policy chunks, all with similarity scores above 0.7, ordered by relevance.
**Validates: Requirements PM-002.2**

### 5.6 AI Decision Engine Properties

**Property 19: LLM Fallback on Failure**
*For any* decision request, if the Claude API fails or times out, the system should automatically retry with GPT-4 without user intervention.
**Validates: Requirements DE-002.2**

**Property 20: Decision Response Completeness**
*For any* AI decision, the response should include all required fields: decision (approve/reject/escalate), confidence score (0-100), rationale (non-empty string), and policy references (array of policy IDs).
**Validates: Requirements DE-002.5**

**Property 21: Confidence-Based Routing**
*For any* AI decision with confidence score C:
- If C ≥ 85%, execute autonomously
- If 70 ≤ C < 85%, execute but flag for review
- If C < 70%, escalate to human approver
**Validates: Requirements DE-003.1, DE-003.2, DE-003.3**

**Property 22: Validation Rules Override AI**
*For any* decision request that violates a configured business rule (insufficient balance, invalid dates, policy violation), the system should reject it with confidence 100% regardless of AI recommendation.
**Validates: Requirements DE-004.2, DE-004.3**

### 5.7 Escalation Management Properties

**Property 23: Escalation Routing Logic**
*For any* escalation request of type T:
- If T = 'policy_exception' OR leave duration > 10 days → route to HR
- If T = 'leave' AND (duration > 3 days OR advance notice < 2 days) → route to manager
- If T = 'system_configuration' → route to admin
**Validates: Requirements EM-001.4**

### 5.8 Audit and Compliance Properties

**Property 24: Comprehensive Action Logging**
*For any* user action (login, logout, data access, data modification, decision), an audit log entry should be created containing user ID, action type, timestamp, entity type, entity ID, and IP address.
**Validates: Requirements AC-001.1**

**Property 25: AI Decision Logging**
*For any* AI decision, an audit log entry should be created containing the complete request, assembled context, decision result, rationale, confidence score, and policy references used.
**Validates: Requirements AC-001.2**

**Property 26: Log Immutability**
*For any* audit log entry, once created, it should not be modifiable or deletable through any API endpoint (append-only constraint).
**Validates: Requirements AC-001.4**

**Property 27: Decision Explainability**
*For any* decision (AI or human), when retrieved, the response should include the complete rationale, policy references, and confidence score (if AI decision).
**Validates: Requirements AC-004.1**

### 5.9 Employee Management Properties

**Property 28: Circular Reporting Prevention**
*For any* attempt to set user A's manager to user B, if B reports to A (directly or transitively), the system should reject the update and return an error indicating circular relationship.
**Validates: Requirements EMP-002.5**

### 5.10 Attendance Management Properties

**Property 29: Missing Check-in/Check-out Detection**
*For any* attendance record, if check-in exists but check-out is null and the date is not today, the system should flag it as "missing check-out" in the attendance report.
**Validates: Requirements ATT-001.4**

**Property 30: Regularization Date Validation**
*For any* attendance regularization request, if the date is more than 7 days in the past or in the future, the system should reject the request with an appropriate error message.
**Validates: Requirements ATT-002.2**

---

## 6. Error Handling

### 6.1 Error Categories

The system handles errors in four categories:

1. **Validation Errors**: User input doesn't meet requirements
2. **Business Logic Errors**: Request violates business rules
3. **External Service Errors**: LLM API, database, or external service failures
4. **System Errors**: Unexpected runtime errors

### 6.2 Error Response Format

All API errors follow a consistent format:

```typescript
interface ErrorResponse {
  error: {
    code: string;           // Machine-readable error code
    message: string;        // Human-readable error message
    details?: any;          // Additional error context
    timestamp: string;      // ISO 8601 timestamp
    requestId: string;      // Unique request identifier for tracing
  };
}
```

### 6.3 Error Handling Strategies

#### 6.3.1 Validation Errors (4xx)

**Strategy**: Fail fast with clear feedback

```typescript
// Example: Invalid leave request
{
  error: {
    code: 'VALIDATION_ERROR',
    message: 'Leave request validation failed',
    details: {
      fields: {
        startDate: 'Start date cannot be in the past',
        leaveType: 'Invalid leave type. Must be one of: casual, sick, earned, unpaid'
      }
    },
    timestamp: '2025-01-15T10:30:00Z',
    requestId: 'req_abc123'
  }
}
```

**Handling**:
- Return 400 Bad Request
- Provide specific field-level errors
- Log validation failures for analytics
- Do not retry

#### 6.3.2 Business Logic Errors (4xx)

**Strategy**: Reject with explanation and alternatives

```typescript
// Example: Insufficient leave balance
{
  error: {
    code: 'INSUFFICIENT_LEAVE_BALANCE',
    message: 'You do not have enough casual leave balance',
    details: {
      requested: 5,
      available: 3,
      alternatives: [
        'Request 3 days of casual leave',
        'Request unpaid leave for remaining 2 days',
        'Check your earned leave balance'
      ]
    },
    timestamp: '2025-01-15T10:30:00Z',
    requestId: 'req_abc123'
  }
}
```

**Handling**:
- Return 422 Unprocessable Entity
- Explain why the request was rejected
- Suggest alternatives when possible
- Log for business analytics
- Do not retry

#### 6.3.3 External Service Errors (5xx)

**Strategy**: Retry with exponential backoff and fallback

```typescript
class ExternalServiceError extends Error {
  constructor(
    public service: string,
    public originalError: Error,
    public retryable: boolean
  ) {
    super(`External service error: ${service}`);
  }
}

async function callWithRetry<T>(
  fn: () => Promise<T>,
  maxRetries: number = 3
): Promise<T> {
  let lastError: Error;
  
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error as Error;
      
      // Don't retry non-retryable errors
      if (error instanceof ExternalServiceError && !error.retryable) {
        throw error;
      }
      
      // Exponential backoff: 1s, 2s, 4s
      const delay = Math.pow(2, i) * 1000;
      await sleep(delay);
    }
  }
  
  throw lastError!;
}
```

**LLM API Error Handling**:
```typescript
async function callLLMWithFallback(prompt: string): Promise<string> {
  try {
    // Try Claude first
    return await callWithRetry(() => claudeClient.complete(prompt), 2);
  } catch (claudeError) {
    console.error('Claude failed, falling back to GPT-4', claudeError);
    
    try {
      // Fallback to GPT-4
      return await callWithRetry(() => gpt4Client.complete(prompt), 2);
    } catch (gpt4Error) {
      console.error('Both LLM providers failed', gpt4Error);
      
      // Return graceful degradation response
      throw new ExternalServiceError(
        'LLM',
        gpt4Error as Error,
        false
      );
    }
  }
}
```

**Database Error Handling**:
```typescript
async function executeQuery<T>(query: () => Promise<T>): Promise<T> {
  try {
    return await callWithRetry(query, 3);
  } catch (error) {
    // Log error with context
    logger.error('Database query failed', {
      error,
      query: query.toString(),
      timestamp: new Date()
    });
    
    // Return user-friendly error
    throw new Error('Database temporarily unavailable. Please try again.');
  }
}
```

#### 6.3.4 System Errors (5xx)

**Strategy**: Log, alert, and return generic error

```typescript
// Global error handler
app.use((error: Error, req: Request, res: Response, next: NextFunction) => {
  // Log full error details
  logger.error('Unhandled error', {
    error: error.message,
    stack: error.stack,
    requestId: req.id,
    userId: req.user?.id,
    path: req.path,
    method: req.method
  });
  
  // Alert on-call team for critical errors
  if (isCriticalError(error)) {
    alerting.sendAlert({
      severity: 'critical',
      message: `Unhandled error: ${error.message}`,
      context: { requestId: req.id, path: req.path }
    });
  }
  
  // Return generic error to user (don't leak internals)
  res.status(500).json({
    error: {
      code: 'INTERNAL_SERVER_ERROR',
      message: 'An unexpected error occurred. Our team has been notified.',
      requestId: req.id,
      timestamp: new Date().toISOString()
    }
  });
});
```

### 6.4 Graceful Degradation

When external services fail, the system degrades gracefully:

**LLM Unavailable**:
- Show cached responses for common queries
- Route all requests to human escalation queue
- Display status message: "AI assistant temporarily unavailable. Your request has been forwarded to HR."

**Database Unavailable**:
- Return cached data where possible
- Queue write operations for retry
- Display status message: "System experiencing high load. Some features may be delayed."

**Email Service Unavailable**:
- Queue notifications for retry
- Show in-app notifications as fallback
- Log failed notifications for manual follow-up

### 6.5 Error Monitoring and Alerting

**Error Metrics**:
- Error rate by endpoint
- Error rate by error type
- LLM API failure rate
- Database query failure rate
- Average error response time

**Alerting Thresholds**:
- Error rate > 5% → Warning
- Error rate > 10% → Critical
- LLM API failure rate > 20% → Warning
- Database unavailable → Critical
- Any unhandled exception → Warning

**Error Tracking**:
- All errors logged to CloudWatch
- Critical errors sent to Sentry
- Error dashboards in Grafana
- Weekly error review meetings

---

## 7. Testing Strategy

### 7.1 Testing Pyramid

```
                    ┌─────────────┐
                    │   Manual    │  5%
                    │   Testing   │
                    └─────────────┘
                  ┌─────────────────┐
                  │  Integration    │  15%
                  │     Tests       │
                  └─────────────────┘
              ┌───────────────────────┐
              │   Property-Based      │  40%
              │       Tests           │
              └───────────────────────┘
          ┌───────────────────────────────┐
          │        Unit Tests             │  40%
          └───────────────────────────────┘
```

### 7.2 Unit Testing

**Purpose**: Verify specific examples, edge cases, and error conditions

**Framework**: Jest (TypeScript)

**Coverage Target**: > 80% code coverage

**Focus Areas**:
- Individual function behavior
- Edge cases (empty inputs, boundary values)
- Error conditions
- Mock external dependencies

**Example Unit Tests**:
```typescript
// services/leave/__tests__/LeaveService.test.ts
describe('LeaveService', () => {
  describe('calculateLeaveDays', () => {
    it('should calculate days excluding weekends', () => {
      const service = new LeaveService();
      const days = service.calculateLeaveDays(
        new Date('2025-01-13'), // Monday
        new Date('2025-01-17')  // Friday
      );
      expect(days).toBe(5);
    });
    
    it('should exclude holidays', () => {
      const service = new LeaveService();
      // Assuming Jan 15 is a holiday
      const days = service.calculateLeaveDays(
        new Date('2025-01-13'),
        new Date('2025-01-17')
      );
      expect(days).toBe(4);
    });
    
    it('should return 0 for weekend-only range', () => {
      const service = new LeaveService();
      const days = service.calculateLeaveDays(
        new Date('2025-01-18'), // Saturday
        new Date('2025-01-19')  // Sunday
      );
      expect(days).toBe(0);
    });
  });
  
  describe('validateLeaveBalance', () => {
    it('should reject when balance insufficient', async () => {
      const service = new LeaveService();
      const result = await service.validateLeaveBalance(
        'user-123',
        'casual',
        5 // requesting 5 days
      );
      expect(result.isValid).toBe(false);
      expect(result.error).toContain('Insufficient');
    });
  });
});
```

### 7.3 Property-Based Testing

**Purpose**: Verify universal properties across all inputs

**Framework**: fast-check (TypeScript)

**Configuration**: Minimum 100 iterations per test

**Test Tagging**: Each test references its design property

**Example Property Tests**:
```typescript
// services/leave/__tests__/LeaveService.property.test.ts
import fc from 'fast-check';

describe('LeaveService - Property Tests', () => {
  /**
   * Feature: autonomous-ai-hr-agent
   * Property 12: Leave Days Calculation
   * For any date range [start, end], the calculated leave days should equal
   * the number of weekdays (Monday-Friday) in that range, excluding holidays.
   */
  it('Property 12: Leave days calculation excludes weekends and holidays', () => {
    fc.assert(
      fc.property(
        fc.date({ min: new Date('2025-01-01'), max: new Date('2025-12-31') }),
        fc.date({ min: new Date('2025-01-01'), max: new Date('2025-12-31') }),
        (date1, date2) => {
          const [startDate, endDate] = date1 < date2 ? [date1, date2] : [date2, date1];
          
          const service = new LeaveService();
          const calculatedDays = service.calculateLeaveDays(startDate, endDate);
          
          // Manually count weekdays excluding holidays
          let expectedDays = 0;
          const current = new Date(startDate);
          while (current <= endDate) {
            const dayOfWeek = current.getDay();
            if (dayOfWeek !== 0 && dayOfWeek !== 6 && !service.isHoliday(current)) {
              expectedDays++;
            }
            current.setDate(current.getDate() + 1);
          }
          
          expect(calculatedDays).toBe(expectedDays);
        }
      ),
      { numRuns: 100 }
    );
  });
  
  /**
   * Feature: autonomous-ai-hr-agent
   * Property 10: Leave Balance Validation
   * For any leave request, if the requested days exceed the user's available
   * balance for that leave type, the system should reject the request.
   */
  it('Property 10: Insufficient balance is rejected', async () => {
    await fc.assert(
      fc.asyncProperty(
        fc.integer({ min: 0, max: 10 }), // available balance
        fc.integer({ min: 1, max: 20 }), // requested days
        async (availableBalance, requestedDays) => {
          const service = new LeaveService();
          const mockUser = createMockUser({ casualLeaveBalance: availableBalance });
          
          const result = await service.validateLeaveBalance(
            mockUser.id,
            'casual',
            requestedDays
          );
          
          if (requestedDays > availableBalance) {
            expect(result.isValid).toBe(false);
            expect(result.error).toContain('Insufficient');
          } else {
            expect(result.isValid).toBe(true);
          }
        }
      ),
      { numRuns: 100 }
    );
  });
  
  /**
   * Feature: autonomous-ai-hr-agent
   * Property 21: Confidence-Based Routing
   * For any AI decision with confidence score C, routing should follow:
   * C >= 85% → execute autonomously
   * 70% <= C < 85% → execute but flag
   * C < 70% → escalate
   */
  it('Property 21: Confidence-based routing is correct', () => {
    fc.assert(
      fc.property(
        fc.integer({ min: 0, max: 100 }), // confidence score
        (confidence) => {
          const engine = new DecisionEngine();
          const routing = engine.determineRouting(confidence);
          
          if (confidence >= 85) {
            expect(routing.action).toBe('execute');
            expect(routing.flagForReview).toBe(false);
          } else if (confidence >= 70) {
            expect(routing.action).toBe('execute');
            expect(routing.flagForReview).toBe(true);
          } else {
            expect(routing.action).toBe('escalate');
          }
        }
      ),
      { numRuns: 100 }
    );
  });
});
```

### 7.4 Integration Testing

**Purpose**: Verify component interactions and end-to-end flows

**Framework**: Jest + Supertest

**Focus Areas**:
- API endpoint integration
- Database operations
- LLM integration (with mocks)
- WebSocket communication

**Example Integration Test**:
```typescript
// __tests__/integration/leave-request.test.ts
describe('Leave Request Flow', () => {
  it('should complete full leave request flow', async () => {
    // 1. User submits leave request via chat
    const chatResponse = await request(app)
      .post('/api/chat/message')
      .set('Authorization', `Bearer ${userToken}`)
      .send({
        content: 'I need leave from Jan 20 to Jan 22 for personal work'
      });
    
    expect(chatResponse.status).toBe(200);
    expect(chatResponse.body.message).toContain('leave request');
    
    // 2. Verify leave record created
    const leave = await db.query.leaves.findFirst({
      where: eq(leaves.userId, testUser.id),
      orderBy: desc(leaves.createdAt)
    });
    
    expect(leave).toBeDefined();
    expect(leave.status).toBe('approved'); // Auto-approved
    
    // 3. Verify leave balance updated
    const balance = await db.query.leaveBalances.findFirst({
      where: and(
        eq(leaveBalances.userId, testUser.id),
        eq(leaveBalances.leaveType, 'casual')
      )
    });
    
    expect(balance.used).toBe(3);
    expect(balance.available).toBe(7);
    
    // 4. Verify notification sent
    const notification = await db.query.notifications.findFirst({
      where: eq(notifications.userId, testUser.id),
      orderBy: desc(notifications.createdAt)
    });
    
    expect(notification.type).toBe('leave_approved');
    
    // 5. Verify audit log
    const auditLog = await db.query.auditLogs.findFirst({
      where: and(
        eq(auditLogs.entityType, 'leave'),
        eq(auditLogs.entityId, leave.id)
      )
    });
    
    expect(auditLog.action).toBe('leave_approved');
  });
});
```

### 7.5 Manual Testing

**Purpose**: Verify user experience and complex scenarios

**Focus Areas**:
- UI/UX validation
- Cross-browser compatibility
- Accessibility compliance
- Performance under load
- Security testing

**Test Cases**:
1. Complete employee onboarding flow
2. Manager approval workflow
3. HR policy upload and search
4. Dashboard data accuracy
5. Mobile responsiveness
6. Accessibility with screen readers

### 7.6 Test Data Management

**Strategy**: Use factories and fixtures for consistent test data

```typescript
// __tests__/factories/user.factory.ts
export function createMockUser(overrides?: Partial<User>): User {
  return {
    id: faker.string.uuid(),
    employeeId: faker.string.alphanumeric(6),
    email: faker.internet.email(),
    name: faker.person.fullName(),
    role: 'employee',
    department: 'Engineering',
    designation: 'Software Engineer',
    joinDate: faker.date.past(),
    status: 'active',
    ...overrides
  };
}

export function createMockLeaveRequest(overrides?: Partial<Leave>): Leave {
  const startDate = faker.date.future();
  const endDate = new Date(startDate);
  endDate.setDate(endDate.getDate() + 3);
  
  return {
    id: faker.string.uuid(),
    userId: faker.string.uuid(),
    leaveType: 'casual',
    startDate,
    endDate,
    daysCount: 3,
    reason: faker.lorem.sentence(),
    status: 'pending',
    decisionType: 'pending',
    ...overrides
  };
}
```

### 7.7 CI/CD Integration

**Pipeline Stages**:
1. **Lint**: ESLint, Prettier
2. **Type Check**: TypeScript compiler
3. **Unit Tests**: Jest (parallel execution)
4. **Property Tests**: fast-check (100 iterations)
5. **Integration Tests**: API and database tests
6. **Build**: Next.js production build
7. **Deploy**: AWS deployment (staging → production)

**Quality Gates**:
- All tests must pass
- Code coverage > 80%
- No TypeScript errors
- No ESLint errors
- Build succeeds

---

**End of Design Document**

**Document Version**: 1.0  
**Last Updated**: January 2025  
**Next Review Date**: February 2025
