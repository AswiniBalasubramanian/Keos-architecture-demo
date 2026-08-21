# KEOS Memory & Validation Specification

**Layer:** 04 — KEOS Context (Contextual Memory)
**Position:** Between KEOS Orbis (06A, agent runtime) and KEOS Nexus (03, knowledge hub)
**Status:** Draft v1

---

## 1. Purpose

KEOS Context is the layer responsible for giving an agent **continuity** — the ability to know who it is talking to, what has already happened in this interaction, what happened in prior interactions, and what role/project constraints apply, without re-deriving all of that from scratch on every turn.

Without this layer, every agent invocation starts cold: no user history, no role context, no memory of a decision made two messages ago. This is the single most common cause of agents that feel "dumb" in production despite a strong model underneath — the model was never given continuity to reason with.

**One-line definition:** *Maintains persistent user, role, project, and interaction context for intelligent execution.*

---

## 2. Memory taxonomy

KEOS Context is not one store — it is three distinct memory concerns, each with different latency, durability, and consistency requirements. Conflating them (e.g., using a session cache as the system of record for long-term user memory) is the most common architecture mistake this layer sees, and is called out explicitly in §5.

| Concern | Question it answers | Latency need | Durability | Component group |
|---|---|---|---|---|
| **Agent memory** | "What do I know about this user/agent relationship over time?" | Sub-second read, async write acceptable | Long-term, survives restarts | Agent Memory |
| **Feature / context store** | "What computed signals (features, embeddings, scores) apply right now?" | Sub-second read, batch or streaming write | Medium-term, versioned | Feature / Context Store |
| **Session / KV state** | "What is the state of *this* conversation, right now?" | Millisecond read/write | Short-term, TTL'd, disposable | Session / KV Store |

A healthy KEOS Context implementation uses **at least the Session / KV layer**, and adds Agent Memory once continuity across sessions matters. The Feature / Context Store is optional — needed when the agent consumes precomputed signals (e.g., a user's propensity score, a project's risk tier) rather than deriving everything from raw memory.

---

## 3. Component groups (from the reference catalog)

### 3.1 Agent Memory
Long-term, cross-session memory purpose-built for agent/LLM workloads — typically fact extraction, summarization, and semantic recall rather than raw key-value storage.

| Component | Notes |
|---|---|
| Mem0 | Persistent multi-level memory layer for agents |
| Zep / Zep Cloud | Long-term memory service with fact extraction and recall |
| Letta (MemGPT) | Stateful agent framework with tiered memory management |
| Cognee | Builds a semantic graph from agent interactions |
| LangMem | LangChain memory abstractions for agent state |
| Bedrock AgentCore Memory | AWS-managed short and long-term agent memory |
| Vertex AI Memory Bank | Google-managed agent memory store |
| Graphiti, MemoryScope, Motorhead, Papr Memory | Additional vendor/OSS options |

### 3.2 Feature / Context Store
Serves precomputed, versioned signals consistently to both training and inference paths.

| Component | Notes |
|---|---|
| Feast | Open-source feature store, online/offline consistency |
| Tecton | Managed feature platform with streaming transforms |
| SageMaker / Vertex AI / Azure ML Feature Store | Cloud-native equivalents |
| Databricks Feature Store | Lakehouse-integrated |
| Hopsworks, Featureform, Chalk, Fennel | Additional vendor/OSS options |

### 3.3 Session / KV Store
Millisecond-latency state for the current interaction — conversation turns, in-flight tool call state, rate-limit counters.

| Component | Notes |
|---|---|
| Redis / Valkey | In-memory store, cache/session/vector workloads |
| Amazon DynamoDB / ElastiCache | AWS-native |
| Firestore / Memorystore for Redis | GCP-native |
| Azure Cosmos DB / Cache for Redis | Azure-native |
| etcd, Hazelcast, Aerospike, ScyllaDB, Cloudflare KV | Additional options |

---

## 4. Data flow

```
06A KEOS Orbis (agent runtime)
        │  reads/writes per turn
        ▼
04 KEOS Context ──────────────┐
   ├─ Session / KV Store       │  turn-by-turn state, TTL'd
   ├─ Agent Memory             │  cross-session facts, summaries
   └─ Feature / Context Store  │  precomputed signals
        │
        │  grounds retrieval against
        ▼
03 KEOS Nexus (knowledge hub — vector/graph)
```

- The **agent runtime (06A)** is the only layer that should read/write memory directly. Downstream layers (Nexus, Intelligence) should never write to Context — they are consulted, not updated, by a memory operation.
- **PII flows through this layer by construction** — user identity, role, and interaction history are exactly the kind of data KEOS Compliance & PII controls exist to govern. Any Contextual Memory design that stores user-identifiable data without a PII control in the architecture (Amazon Macie, MS Presidio, Cloud DLP, Microsoft Purview) is a compliance gap, not just a technical one.

---

## 5. Validation rules

These rules are already implemented in the architecture builder's validation engine (`crossLayerChecks()` and `CRITICAL{}` in the reference application) and are restated here as the normative spec for this layer.

### 5.1 Risk (blocks — architecture cannot work as drawn)

| Rule | Condition | Rationale |
|---|---|---|
| R-MEM-01 | Session / KV Store category is present in the architecture but empty | A category that exists with zero components is a declared-but-unfulfilled requirement, not an absence — treated as critical |
| R-MEM-02 | The same component name backs both Agent Memory and Session / KV Store | Conflates short-term disposable state with long-term durable memory — a retention-policy risk, not just an advisory |

### 5.2 Advisory (safe, but flagged)

| Rule | Condition | Message |
|---|---|---|
| A-MEM-01 | Agents (06A) present, Agent Memory empty | "Agents have no contextual memory — every interaction restarts with no user, role or project continuity." |
| A-MEM-02 | Agent Memory present, Session / KV Store empty | "Agent memory is defined without a session/KV store — short-term state has nowhere fast to live." |
| A-MEM-03 | Any Context category mixes ≥2 cloud vendors (e.g., DynamoDB + Cosmos DB in the same category) | Egress cost and split operational surface |
| A-MEM-04 | Any Context category exceeds 5 components | Consolidation candidate — likely accumulated rather than deliberately chosen |
| A-MEM-05 | Agent Memory or Session/KV Store present, Compliance & PII empty | "KEOS Context stores user-identifiable interaction history (agent memory or session state) with no PII control in the architecture — that history is regulated data." |
| A-MEM-06 | Feature / Context Store present, Data Quality Guards (Layer 02) empty | "A feature/context store is defined with no data quality guards — stale or drifted features will silently degrade agent decisions." |

All six rules above are implemented in `crossLayerChecks()` in the reference application as of this revision.

---

## 6. Reference architectures by cloud profile

| Profile | Agent Memory | Feature / Context Store | Session / KV Store |
|---|---|---|---|
| **AWS** | Bedrock AgentCore Memory, Mem0, Zep | SageMaker Feature Store, Feast | Amazon DynamoDB, Amazon ElastiCache |
| **GCP** | Vertex AI Memory Bank, Mem0, Zep | Vertex AI Feature Store, Feast | Firestore, Memorystore for Redis |
| **Azure** | Mem0, Zep, Letta (MemGPT) | Azure ML Feature Store, Feast | Azure Cosmos DB, Azure Cache for Redis |
| **Composable / OSS** | Mem0, Zep, Letta (MemGPT), Cognee | Feast, Tecton | Redis, Valkey, MongoDB |

---

## 7. Glossary

- **Agent Memory** — cross-session, durable memory purpose-built for agent/LLM recall (facts, summaries, preferences).
- **Session state** — the disposable, TTL'd state of one active conversation or execution.
- **Feature store** — a system that serves versioned, precomputed signals with online/offline consistency.
- **Cross-layer risk** — a validation finding produced by reasoning across two or more layers together, rather than inspecting one category in isolation (see `VALIDATION-TESTS.md` §1C for the full mechanism).
