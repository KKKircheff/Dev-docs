# AI-Driven Enterprise Brand Plan Update Tool: Technical Architecture Blueprint

Enterprise brand planning requires systematic annual updates across 45+ page presentations—a process ripe for AI automation with proper guardrails. This research delivers a complete technical architecture for a Human-in-the-Loop (HITL) tool using **Next.js, Supabase, OpenAI models, and open-source libraries** that respects the RED/AMBER/GREEN governance model while enabling intelligent content generation and strategic friction detection.

The recommended stack centers on **LangGraph for multi-agent orchestration** (superior to CrewAI's unreliable hierarchical patterns), **NetworkX/Cytoscape.js for dependency graph management**, **GLiNER for zero-shot guardrail extraction**, and **python-pptx with docling for document processing**. This architecture enables bi-directional ripple effect detection, slide-level preservation across year-over-year updates, and precise content generation that fits slide constraints exactly.

---

## System architecture overview

The architecture follows a three-tier design separating document processing (Python/FastAPI), orchestration (LangGraph), and presentation (Next.js), with Supabase providing unified state management and real-time capabilities.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           PRESENTATION LAYER (Next.js)                       │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │
│  │ Document     │  │ Dependency   │  │ Conflict     │  │ Approval     │    │
│  │ Editor       │  │ Graph View   │  │ Resolution   │  │ Workflow     │    │
│  │ (Tiptap)     │  │ (Cytoscape)  │  │ Modal        │  │ Dashboard    │    │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘    │
│                                    ↕ SSE Streaming                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                           ORCHESTRATION LAYER (LangGraph)                    │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    TOP-LEVEL SUPERVISOR AGENT                        │   │
│  │    • Global Context Management  • Cross-Section Consistency          │   │
│  │    • Task Routing              • Conflict Detection                  │   │
│  └───────────────────────────────┬─────────────────────────────────────┘   │
│              ┌───────────────────┼───────────────────┐                      │
│              ▼                   ▼                   ▼                      │
│  ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐               │
│  │ CONTENT TEAM    │ │ VALIDATION TEAM │ │ RESEARCH TEAM   │               │
│  │ • Section Writer│ │ • Number Check  │ │ • Market Data   │               │
│  │ • Slide Formatter││ • Claims Verify │ │ • Competitor    │               │
│  │ • SWOT Generator││ • CSP Validator │ │ • Trends        │               │
│  └─────────────────┘ └─────────────────┘ └─────────────────┘               │
├─────────────────────────────────────────────────────────────────────────────┤
│                           PROCESSING LAYER (FastAPI + Python)                │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │
│  │ Docling      │  │ python-pptx  │  │ GLiNER NER   │  │ NetworkX     │    │
│  │ Ingestion    │  │ PPTX R/W     │  │ Extraction   │  │ Graph Ops    │    │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘    │
├─────────────────────────────────────────────────────────────────────────────┤
│                           DATA LAYER (Supabase)                              │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │
│  │ PostgreSQL   │  │ LangGraph    │  │ pgvector     │  │ Real-time    │    │
│  │ Core Tables  │  │ Checkpoints  │  │ Embeddings   │  │ Subscriptions│    │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘    │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Dependency graph model for brand plans

A 45-page brand plan maps naturally to a **Directed Acyclic Graph (DAG)** where nodes represent logical content units and edges represent data dependencies. The acyclic property ensures no circular dependencies that would prevent consistent updates.

### Brand plan dependency graph (15 nodes)

| Node ID | Node Name | RAG Color | Type | Description |
|---------|-----------|-----------|------|-------------|
| N1 | Brand Mission | RED | Foundation | Core brand purpose—locked mandate |
| N2 | Target Audience | RED | Foundation | Demographics, psychographics—locked |
| N3 | Market Scope | RED | Foundation | Geographic constraints (EU, China, Brazil) |
| N4 | Brand Positioning | RED | Strategic | Value proposition—locked mandate |
| N5 | Competitive Analysis | AMBER | Research | Market positioning data—human-editable |
| N6 | Brand Personality | AMBER | Strategic | Voice, attributes—requires approval |
| N7 | Regional Goals | AMBER | Strategic | Market-specific objectives |
| N8 | Tone of Voice | GREEN | Tactical | Communication style—AI-generated |
| N9 | Messaging Framework | GREEN | Tactical | Key messages—AI-generated |
| N10 | Channel Strategy | GREEN | Tactical | Distribution channels—AI-generated |
| N11 | Media Plan | GREEN | Execution | Media buying strategy |
| N12 | Budget Allocation | AMBER | Financial | Resource distribution—approval required |
| N13 | Content Calendar | GREEN | Execution | Timing/scheduling |
| N14 | KPI Framework | GREEN | Measurement | Success metrics |
| N15 | Executive Summary | GREEN | Summary | Overview synthesis—generated last |

### Edge definitions (dependency relationships)

```
N1 (Brand Mission) ──────→ N4 (Brand Positioning)
N1 (Brand Mission) ──────→ N6 (Brand Personality)
N2 (Target Audience) ────→ N4 (Brand Positioning)
N2 (Target Audience) ────→ N8 (Tone of Voice)
N2 (Target Audience) ────→ N10 (Channel Strategy)
N3 (Market Scope) ───────→ N7 (Regional Goals)
N3 (Market Scope) ───────→ N10 (Channel Strategy)
N4 (Brand Positioning) ──→ N6 (Brand Personality)
N4 (Brand Positioning) ──→ N9 (Messaging Framework)
N5 (Competitive Analysis)→ N4 (Brand Positioning)
N5 (Competitive Analysis)→ N9 (Messaging Framework)
N6 (Brand Personality) ──→ N8 (Tone of Voice)
N7 (Regional Goals) ─────→ N12 (Budget Allocation)
N8 (Tone of Voice) ──────→ N9 (Messaging Framework)
N10 (Channel Strategy) ──→ N11 (Media Plan)
N10 (Channel Strategy) ──→ N13 (Content Calendar)
N12 (Budget Allocation) ─→ N11 (Media Plan)
N12 (Budget Allocation) ─→ N10 (Channel Strategy)
ALL NODES ───────────────→ N15 (Executive Summary)
```

### Ripple effect detection algorithm

The system detects three types of propagation when any node changes:

**Forward propagation** (downstream impact): When a node changes, all descendant nodes must be re-evaluated. The system uses topological sort to determine update order.

**Backward validation** (upstream review): When a GREEN node changes significantly (e.g., budget tactics), upstream RED/AMBER nodes are flagged for human review to ensure the change aligns with locked mandates.

**Lateral consistency** (sibling nodes): Nodes sharing common ancestors must maintain consistency—for example, Channel Strategy and Media Plan both derive from Budget Allocation and must align.

```typescript
interface RippleEffectAnalysis {
  forwardImpact: string[];       // GREEN nodes needing regeneration
  backwardReview: string[];      // RED/AMBER nodes requiring human review
  lateralCheck: string[];        // Sibling nodes needing consistency check
  updateSequence: string[];      // Topologically sorted update order
  estimatedEffort: 'low' | 'medium' | 'high';
}
```

---

## Table of logical dependencies between brand plan sections

| Source Section | Target Section | Dependency Type | Ripple Effect | Validation Rule |
|----------------|----------------|-----------------|---------------|-----------------|
| Brand Mission | Brand Positioning | `derives_from` | Forward | Positioning must reflect mission values |
| Target Audience | Tone of Voice | `constrains` | Forward | Voice must resonate with audience demographics |
| Target Audience | Channel Strategy | `constrains` | Forward | Channels must reach target segments |
| Market Scope | Regional Goals | `defines_scope` | Forward | Goals limited to approved markets only |
| Market Scope | Budget Allocation | `constrains` | Lateral | Budget only for approved regions |
| Brand Positioning | Messaging Framework | `derives_from` | Forward | Messages must support positioning claims |
| Competitive Analysis | Brand Positioning | `informs` | Backward | Positioning must differentiate from competitors |
| Brand Personality | Creative Guidelines | `defines` | Forward | All creative must match personality |
| Budget Allocation | Channel Strategy | `constrains` | Backward | Channel mix must fit budget constraints |
| Budget Allocation | Media Plan | `constrains` | Forward | Media spend ≤ allocated budget |
| Regional Goals | KPI Framework | `measured_by` | Forward | KPIs must track regional goal progress |
| Tone of Voice | Campaign Concepts | `guides` | Forward | Campaigns must match approved voice |
| All Sections | Executive Summary | `summarizes` | Forward | Summary regenerates when any section changes |

### Constraint Satisfaction Problem (CSP) encoding

RED mandates are encoded as CSP constraints that must hold true for any GREEN content generation:

```python
class BrandPlanCSPValidator:
    def __init__(self, red_mandates: dict):
        self.constraints = {
            'market_scope': lambda content: all(
                market in red_mandates['approved_markets']
                for market in extract_market_mentions(content)
            ),
            'budget_ceiling': lambda content: 
                extract_budget_total(content) <= red_mandates['max_budget'],
            'brand_terms': lambda content: all(
                term in red_mandates['approved_terminology']
                for term in extract_brand_terms(content)
            ),
            'positioning_alignment': lambda content:
                semantic_similarity(content, red_mandates['positioning']) > 0.7
        }
    
    def validate(self, node_id: str, proposed_content: str) -> ValidationResult:
        violations = []
        for constraint_name, check_fn in self.constraints.items():
            if not check_fn(proposed_content):
                violations.append(constraint_name)
        return ValidationResult(valid=len(violations)==0, violations=violations)
```

---

## Recommended open-source libraries for each component

### Graph logic and dependency management

| Library | Language | Purpose | Recommendation |
|---------|----------|---------|----------------|
| **NetworkX** | Python | Backend graph operations, topological sort, traversal | ⭐ Primary choice—mature, excellent docs |
| **python-constraint** | Python | CSP validation, arc consistency, backtracking | ⭐ For RED mandate enforcement |
| **Cytoscape.js** | TypeScript | Frontend visualization, interactive graph | ⭐ Best for UI with dagre layout |
| **@dagrejs/graphlib** | TypeScript | Lightweight graph operations | For pure data operations |

### NLP and strategic friction detection

| Library | Purpose | Key Capability |
|---------|---------|----------------|
| **sentence-transformers** | Semantic embeddings | `all-mpnet-base-v2` for content similarity |
| **cross-encoder/nli-deberta-v3-large** | Contradiction detection | NLI classification (entailment/neutral/contradiction) |
| **MoritzLaurer/DeBERTa-v3-large-mnli** | Document-level NLI | Best HuggingFace NLI model for long texts |
| **spaCy** | Entity extraction, coreference | `en_core_web_trf` for production NER |
| **GLiNER** | Zero-shot NER | Extract brand terms, markets without training |

### PPTX manipulation and document processing

| Library | Language | Capability | Limitation |
|---------|----------|------------|------------|
| **python-pptx** | Python | Full read/write, template support | SmartArt limited |
| **docling** | Python | Multi-format ingestion (PDF, DOCX, PPTX) | — |
| **PptxGenJS** | TypeScript | Generation from scratch | Cannot read existing files |
| **nodejs-pptx** | TypeScript | Read/write existing files | Less mature |

### Multi-agent orchestration

| Framework | Recommendation | Rationale |
|-----------|---------------|-----------|
| **LangGraph** | ⭐ Primary choice | Native TypeScript support, Supabase checkpointing, reliable supervisor pattern |
| **CrewAI** | ⚠️ Avoid for hierarchical | Documented issues with manager-worker pattern in production |
| **AutoGen** | Alternative | Good for .NET environments, requires API wrapper for Next.js |

### UI components for HITL

| Component | Library | Purpose |
|-----------|---------|---------|
| Rich text editor | **Tiptap** | Block-based editing with AI extensions |
| Diff visualization | **react-diff-viewer** | Side-by-side change comparison |
| UI primitives | **shadcn/ui + Radix** | Accessible dialogs, tooltips, badges |
| Real-time sync | **Yjs** or **Liveblocks** | Collaborative editing |

---

## AI guardrail protocols for enterprise brand governance

### Guardrail extraction pipeline

The system extracts constraints from uploaded brand documents through a four-stage pipeline:

**Stage 1 — Document ingestion**: Docling parses PDF/DOCX/PPTX into structured segments with metadata (headers, page numbers, section IDs).

**Stage 2 — Entity extraction**: GLiNER performs zero-shot NER for brand terms, market regions, product names, and numerical constraints without requiring training.

**Stage 3 — Rule classification**: LLM with structured output categorizes extracted rules as factual (static facts), behavioral (required actions), or conditional (IF-THEN logic).

**Stage 4 — Validation and storage**: Schema validation, confidence scoring, and Supabase storage with human verification queue for low-confidence items.

### Guardrail storage schema (Supabase)

```sql
CREATE TABLE guardrails (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID REFERENCES organizations(id),
    constraint_type TEXT CHECK (constraint_type IN (
        'entity', 'rule', 'numerical', 'locked_content'
    )),
    category TEXT, -- 'market', 'brand', 'budget', 'policy'
    name TEXT NOT NULL,
    value JSONB NOT NULL,
    
    -- Provenance tracking
    source_document TEXT NOT NULL,
    source_page INTEGER,
    extracted_text TEXT,
    
    -- Confidence and verification
    confidence_score DECIMAL(3,2),
    extraction_method TEXT, -- 'gliner', 'llm', 'manual'
    status TEXT DEFAULT 'pending' CHECK (status IN (
        'pending', 'verified', 'rejected'
    )),
    verified_by UUID REFERENCES users(id),
    
    version INTEGER DEFAULT 1,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Example value structures:
-- Entity: {"entity_type": "market", "values": ["EU", "China", "Brazil"]}
-- Rule: {"condition": "market = EU", "action": "include_gdpr_notice"}
-- Numerical: {"metric": "budget", "operator": "<=", "value": 50000000}
-- Locked: {"content_type": "tagline", "text": "Innovation Forward"}
```

### Guardrail validation during content generation

```typescript
async function validateAgainstGuardrails(
  content: string,
  sectionType: string,
  projectId: string
): Promise<ValidationResult> {
  // Load active guardrails from Supabase
  const { data: guardrails } = await supabase
    .from('guardrails')
    .select('*')
    .eq('status', 'verified');
  
  const violations: Violation[] = [];
  
  // Check market scope constraints
  const marketGuardrails = guardrails.filter(g => g.category === 'market');
  const mentionedMarkets = await extractMarkets(content);
  for (const market of mentionedMarkets) {
    if (!marketGuardrails.some(g => g.value.values.includes(market))) {
      violations.push({
        type: 'market_scope',
        severity: 'critical',
        message: `Unauthorized market mentioned: ${market}`
      });
    }
  }
  
  // Check numerical constraints
  const budgetGuardrails = guardrails.filter(g => g.constraint_type === 'numerical');
  const extractedNumbers = await extractNumericalClaims(content);
  // ... validate against constraints
  
  return { valid: violations.length === 0, violations };
}
```

---

## HITL workflow patterns and UI design

### Document state machine

```
┌─────────┐    ┌──────────┐    ┌──────────────┐    ┌──────────┐
│  DRAFT  │ →  │  REVIEW  │ →  │   APPROVAL   │ →  │ APPROVED │
│         │    │          │    │   PENDING    │    │          │
└─────────┘    └──────────┘    └──────────────┘    └──────────┘
     ↑              ↓                 ↓                  ↓
     └──── Rejected ←────────────────┴──────────────────┘
                                                   (Rollback)
```

### Approval workflow for multi-stakeholder review

```
Sequential Approval Chain:
Brand Manager → Legal Review → Executive → Published

Parallel Review (for faster turnaround):
     ┌→ Legal ────────────┐
     │                    ↓
Author → Brand Manager → Gate → Executive → Published
     │                    ↑
     └→ Compliance ──────┘
```

### RAG governance visual design

| RAG Status | Visual Treatment | User Interaction |
|------------|------------------|------------------|
| **RED (Locked)** | 🔒 Lock icon + red left border, grayed text | Click shows "Contact Legal to modify" |
| **AMBER (Editable)** | ✏️ Edit icon + orange accent | Click enables editing with approval workflow |
| **GREEN (AI-Generated)** | 🤖 AI badge + green highlight | One-click accept/reject, inline editing |

### Conflict resolution modal wireframe

```
┌────────────────────────────────────────────────────────────────┐
│ ⚠️ Strategic Friction Detected                              [X] │
├────────────────────────────────────────────────────────────────┤
│  Severity: 🔴 Critical                                         │
│                                                                │
│  ┌─Section 2: Strategy──────┐  ┌─Section 8: Budget───────────┐ │
│  │ "Premium positioning     │  │ "40% budget allocated to    │ │
│  │  targeting affluent      │  │  deep discount promotions"  │ │
│  │  consumers"              │  │                             │ │
│  └──────────────────────────┘  └─────────────────────────────┘ │
│                                                                │
│  📊 AI Analysis:                                               │
│  "Premium positioning conflicts with discount-heavy budget.    │
│   Recommend reallocating 25% from discounts to brand-building."│
│                                                                │
│  Confidence: ████████░░ 82%                                    │
│                                                                │
│    [Keep Both]  [Resolve Strategy]  [Resolve Budget]  [Escalate]│
└────────────────────────────────────────────────────────────────┘
```

### Approval dashboard wireframe

```
┌────────────────────────────────────────────────────────────────┐
│ Approval Workflow: Q1 Brand Plan 2026                          │
├────────────────────────────────────────────────────────────────┤
│  ●───────●───────◐───────○───────○                            │
│  Draft   Review  Legal   Exec    Published                     │
│  ✓       ✓       →       ○       ○                            │
│                                                                │
│  Current Stage: Legal Review                                   │
│  Assigned: J. Smith | Due: Jan 30, 2026 (2 days)              │
│                                                                │
│  ┌─Pending Items─────────────────────────────────────────┐    │
│  │ □ Review compliance disclaimers (Section 4)           │    │
│  │ □ Verify trademark usage (Logo guidelines)            │    │
│  │ ☑ Approve market claims (Section 2) ✓                 │    │
│  └───────────────────────────────────────────────────────┘    │
│                                                                │
│  [← Previous Stage]  [Add Comment]  [Approve →]  [Reject]     │
└────────────────────────────────────────────────────────────────┘
```

---

## Code architecture for Next.js + Supabase + OpenAI

### Recommended project structure

```
/brand-plan-generator
├── /src
│   ├── /app                           # Next.js App Router
│   │   ├── /api
│   │   │   ├── /agent
│   │   │   │   ├── /stream/route.ts   # SSE streaming for agent updates
│   │   │   │   └── /start/route.ts    # Initiate generation workflow
│   │   │   ├── /validate/route.ts     # CSP validation endpoint
│   │   │   └── /projects/route.ts     # Project CRUD
│   │   ├── /projects/[id]/page.tsx    # Main editor view
│   │   └── layout.tsx
│   │
│   ├── /lib
│   │   ├── /agents                    # LangGraph agent definitions
│   │   │   ├── supervisor.ts          # Top-level orchestrator
│   │   │   ├── section-writer.ts      # Content generation agent
│   │   │   ├── validator.ts           # Consistency checking agent
│   │   │   └── schemas.ts             # Pydantic/Zod output schemas
│   │   │
│   │   ├── /graph                     # LangGraph workflow
│   │   │   ├── workflow.ts            # Main StateGraph definition
│   │   │   ├── nodes.ts               # Graph node implementations
│   │   │   └── state.ts               # State type definitions
│   │   │
│   │   ├── /db                        # Supabase integration
│   │   │   ├── client.ts              # Supabase client setup
│   │   │   ├── checkpointer.ts        # LangGraph state persistence
│   │   │   └── queries.ts             # Database queries
│   │   │
│   │   ├── /csp                       # Constraint Satisfaction
│   │   │   ├── validator.ts           # CSP validation logic
│   │   │   └── constraints.ts         # Constraint definitions
│   │   │
│   │   └── /pptx                      # PPTX processing (Python API)
│   │       └── client.ts              # FastAPI client
│   │
│   ├── /components
│   │   ├── /editor                    # Tiptap-based document editor
│   │   ├── /graph                     # Cytoscape dependency visualization
│   │   ├── /approval                  # Workflow UI components
│   │   └── /diff                      # Change comparison views
│   │
│   └── /hooks
│       ├── useAgentStream.ts          # SSE subscription hook
│       └── useBrandPlanGraph.ts       # Graph operations hook
│
├── /python-api                        # FastAPI for PPTX/NLP processing
│   ├── /app
│   │   ├── main.py                    # FastAPI app
│   │   ├── /routers
│   │   │   ├── pptx.py               # PPTX manipulation endpoints
│   │   │   ├── extract.py            # Guardrail extraction
│   │   │   └── nlp.py                # Contradiction detection
│   │   └── /services
│   │       ├── docling_service.py    # Document ingestion
│   │       ├── gliner_service.py     # NER extraction
│   │       └── friction_detector.py  # Strategic friction NLP
│   └── requirements.txt
│
├── docker-compose.yml                 # Local Postgres + pgvector
└── package.json
```

### LangGraph workflow implementation

```typescript
// /src/lib/graph/workflow.ts
import { StateGraph, START, END } from '@langchain/langgraph';
import { ChatOpenAI } from '@langchain/openai';
import { SupabaseSaver } from '@skroyc/langgraph-supabase-checkpointer';

interface BrandPlanState {
  projectId: string;
  currentSection: number;
  globalContext: GlobalContext;
  sectionContents: Map<number, SlideContent>;
  validationErrors: ValidationError[];
  messages: BaseMessage[];
}

export function createBrandPlanWorkflow() {
  const model = new ChatOpenAI({ model: 'gpt-4o' });
  
  const workflow = new StateGraph<BrandPlanState>({
    channels: {
      projectId: { value: '' },
      currentSection: { value: 0 },
      globalContext: { value: {} as GlobalContext },
      sectionContents: { value: new Map() },
      validationErrors: { value: [] },
      messages: { reducer: (a, b) => [...a, ...b] }
    }
  });

  // Define nodes
  workflow.addNode('load_context', loadContextNode);
  workflow.addNode('generate_section', generateSectionNode);
  workflow.addNode('validate_csp', validateCSPNode);
  workflow.addNode('detect_friction', detectFrictionNode);
  workflow.addNode('human_review', humanReviewNode);

  // Define edges with conditional routing
  workflow.addEdge(START, 'load_context');
  workflow.addEdge('load_context', 'generate_section');
  workflow.addConditionalEdges('generate_section', routeAfterGeneration);
  workflow.addConditionalEdges('validate_csp', routeAfterValidation);
  workflow.addConditionalEdges('detect_friction', routeAfterFriction);
  workflow.addEdge('human_review', END);

  return workflow;
}

// Conditional routing based on validation results
function routeAfterValidation(state: BrandPlanState): string {
  if (state.validationErrors.length > 0) {
    const critical = state.validationErrors.some(e => e.severity === 'critical');
    return critical ? 'human_review' : 'generate_section'; // Regenerate with feedback
  }
  return 'detect_friction';
}
```

### Structured output for slide-ready content

```typescript
// /src/lib/agents/schemas.ts
import { z } from 'zod';

export const SWOTSlideSchema = z.object({
  title: z.string().max(60).describe("Slide title, max 60 chars"),
  strengths: z.array(z.string().max(100)).length(4),
  weaknesses: z.array(z.string().max(100)).length(4),
  opportunities: z.array(z.string().max(100)).length(4),
  threats: z.array(z.string().max(100)).length(4)
});

export const ExecutiveSummarySchema = z.object({
  title: z.string().max(50).default("Executive Summary"),
  overview: z.string().min(100).max(400),
  key_metrics: z.array(z.object({
    metric: z.string().max(25),
    value: z.string().max(15),
    trend: z.enum(["up", "down", "stable"])
  })).min(3).max(5),
  recommendations: z.array(z.string().max(120)).min(2).max(4)
});

// System prompt for zero-preamble output
export const SLIDE_AGENT_PROMPT = `You generate ONLY slide-ready content in JSON format.

CRITICAL RULES:
- NO preambles ("Here's...", "I'll create...")
- NO explanations or caveats
- EXACTLY the structure specified
- Character limits are HARD LIMITS

Output JSON directly. Nothing else.`;
```

### Supabase database schema

```sql
-- Document projects
CREATE TABLE document_projects (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES auth.users(id),
    name TEXT NOT NULL,
    status TEXT DEFAULT 'draft',
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Dependency graph nodes
CREATE TABLE brand_plan_nodes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID REFERENCES document_projects(id),
    node_key VARCHAR(100) NOT NULL,
    display_name VARCHAR(255) NOT NULL,
    slide_number INTEGER,
    rag_color VARCHAR(10) CHECK (rag_color IN ('RED', 'AMBER', 'GREEN')),
    content JSONB,
    content_hash VARCHAR(64),
    version INTEGER DEFAULT 1,
    UNIQUE(project_id, node_key)
);

-- Dependency graph edges
CREATE TABLE brand_plan_edges (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID REFERENCES document_projects(id),
    source_node_id UUID REFERENCES brand_plan_nodes(id),
    target_node_id UUID REFERENCES brand_plan_nodes(id),
    dependency_type VARCHAR(50),
    UNIQUE(project_id, source_node_id, target_node_id)
);

-- LangGraph checkpoint integration (auto-created by checkpointer)
-- Extended with custom workflow tracking
CREATE TABLE workflow_sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID REFERENCES document_projects(id),
    thread_id TEXT NOT NULL,
    agent_name TEXT NOT NULL,
    status TEXT DEFAULT 'active',
    started_at TIMESTAMPTZ DEFAULT NOW(),
    completed_at TIMESTAMPTZ
);

-- Edit history for rollback
CREATE TABLE edit_history (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    node_id UUID REFERENCES brand_plan_nodes(id),
    previous_content JSONB NOT NULL,
    changed_by TEXT,
    change_type TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

---

## Strategic friction detection implementation

The system detects **8-10 common friction patterns** in brand plans through a hybrid NLP pipeline combining semantic embeddings, NLI classification, and numerical consistency checking.

### Common strategic friction examples

| Friction Type | Claim A | Claim B | Detection Method |
|---------------|---------|---------|------------------|
| Premium vs. Discount | "Premium positioning" | "40% deep discounting" | NLI contradiction + keyword |
| Audience Mismatch | "Youth-focused (18-34)" | "TV targeting 45+" | Demographic entity extraction |
| Sustainability Paradox | "Carbon-neutral by 2025" | "Non-certified suppliers" | Topic + NLI |
| Geographic Conflict | "Expand into APAC" | "15% international cuts" | NER regions + numerical |
| Innovation vs. Cost | "R&D leadership" | "R&D budget -30%" | Topic extraction + numerical |
| Timeline Inconsistency | "Launch Q2 2026" | "Research complete Q4 2026" | Temporal entity + logic |
| KPI Misalignment | "Brand awareness priority" | "All metrics tied to conversion" | Semantic similarity |
| Resource Paradox | "Customer experience first" | "CS headcount reduced" | Topic + directional sentiment |

### Friction detection pipeline

```python
class StrategicFrictionDetector:
    def __init__(self):
        # Semantic embeddings for similarity
        self.embedder = SentenceTransformer('all-mpnet-base-v2')
        # NLI for contradiction classification
        self.nli_model = CrossEncoder('cross-encoder/nli-deberta-v3-large')
    
    def detect_friction(self, document_sections: List[dict]) -> List[Friction]:
        # 1. Extract claims from each section
        all_claims = self._extract_claims(document_sections)
        
        # 2. Find candidate pairs using embedding similarity
        candidates = self._find_similar_pairs(all_claims, threshold=0.4)
        
        # 3. Verify contradictions with NLI
        contradictions = []
        for claim_a, claim_b in candidates:
            scores = self.nli_model.predict([(claim_a.text, claim_b.text)])
            if scores[0].argmax() == 0:  # contradiction label
                contradictions.append(Friction(claim_a, claim_b, scores))
        
        # 4. Check numerical consistency
        numerical_issues = self._check_numerical_consistency(document_sections)
        
        return contradictions + numerical_issues
```

---

## Year-over-year slide mapping algorithm

The system preserves slide structure across annual updates through a multi-layer matching algorithm that identifies which slides are "same topic, needs update" versus "new" versus "deprecated."

### Slide fingerprint extraction

```python
@dataclass
class SlideFingerprint:
    title: str
    title_normalized: str  # lowercase, stripped
    layout_type: str
    placeholder_types: List[str]  # [title, body, chart, image]
    content_hash: str
    text_embedding: np.array  # sentence-transformer embedding
```

### Matching algorithm

```python
def map_slides_yoy(prev_deck: List[Slide], curr_deck: List[Slide]) -> SlideMapping:
    # 1. Extract fingerprints
    prev_fps = [extract_fingerprint(s) for s in prev_deck]
    curr_fps = [extract_fingerprint(s) for s in curr_deck]
    
    # 2. Compute similarity matrix
    similarity_matrix = np.zeros((len(prev_fps), len(curr_fps)))
    for i, fp1 in enumerate(prev_fps):
        for j, fp2 in enumerate(curr_fps):
            similarity_matrix[i,j] = compute_similarity(fp1, fp2)
    
    # 3. Hungarian algorithm for optimal assignment
    row_ind, col_ind = linear_sum_assignment(-similarity_matrix)
    
    # 4. Categorize matches
    matched, new_slides, deprecated = [], [], []
    for i, j in zip(row_ind, col_ind):
        if similarity_matrix[i,j] > 0.6:
            matched.append((i, j, similarity_matrix[i,j]))
        else:
            deprecated.append(i)
            new_slides.append(j)
    
    return SlideMapping(matched=matched, new=new_slides, deprecated=deprecated)

def compute_similarity(fp1: SlideFingerprint, fp2: SlideFingerprint) -> float:
    weights = {'title': 0.35, 'layout': 0.15, 'content': 0.25, 'structure': 0.15, 'position': 0.10}
    
    scores = {
        'title': fuzz.token_sort_ratio(fp1.title_normalized, fp2.title_normalized) / 100,
        'layout': 1.0 if fp1.layout_type == fp2.layout_type else 0.3,
        'content': cosine_similarity(fp1.text_embedding, fp2.text_embedding),
        'structure': jaccard_index(fp1.placeholder_types, fp2.placeholder_types),
        'position': position_overlap_score(fp1, fp2)
    }
    
    return sum(weights[k] * scores[k] for k in weights)
```

---

## Content precision enforcement

The system ensures AI outputs fit slide constraints exactly through structured output schemas, anti-preamble prompting, and validation loops.

### Prompt pattern for exact-fit output

```
SYSTEM:
You are a slide content specialist. Output ONLY JSON matching the schema.

RULES:
1. NO preambles ("Here's...", "I'll create...")
2. EXACTLY 4 bullets per SWOT quadrant
3. Each bullet: 8-12 words maximum
4. Title: max 60 characters
5. Start bullets with action verbs

Output the JSON structure directly. Nothing else.
```

### Validation loop for content precision

```typescript
function validateSlideContent(content: SlideContent): ValidationResult {
  const errors: string[] = [];
  
  // Character limits
  if (content.title.length > 60) {
    errors.push(`Title exceeds 60 chars (${content.title.length})`);
  }
  
  // Exact count enforcement
  if (content.bullets.length !== 4) {
    errors.push(`Expected 4 bullets, got ${content.bullets.length}`);
  }
  
  // Preamble detection
  const preamblePatterns = [/^(here|i'll|let me|sure)/i];
  if (preamblePatterns.some(p => p.test(content.title))) {
    errors.push('Content contains preamble text');
  }
  
  return { valid: errors.length === 0, errors };
}
```

---

## Conclusion: Implementation roadmap

This architecture delivers a production-ready foundation for AI-driven brand plan updates with proper governance, consistency checking, and human oversight.

**Phase 1 (Weeks 1-2)**: Set up LangGraph.js + Next.js + Supabase checkpointing with basic supervisor agent and single section-writer.

**Phase 2 (Weeks 3-4)**: Implement dependency graph with NetworkX backend and Cytoscape.js visualization; add CSP validation for RED mandates.

**Phase 3 (Weeks 5-6)**: Add strategic friction detection using sentence-transformers and DeBERTa NLI; build conflict resolution UI.

**Phase 4 (Weeks 7-8)**: Integrate python-pptx pipeline with docling ingestion; implement YoY slide mapping algorithm.

**Phase 5 (Weeks 9-10)**: Build guardrail extraction with GLiNER; add human verification queue and approval workflows.

**Key technical decisions**: LangGraph over CrewAI for reliable hierarchical orchestration; GLiNER for zero-shot entity extraction without training overhead; python-pptx with in-place updates to preserve formatting; structured outputs with strict JSON schemas for slide-ready content.

The system enables brand teams to update 45-page plans in hours rather than weeks while maintaining strategic coherence, respecting locked mandates, and providing full audit trails for compliance.