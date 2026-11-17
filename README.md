# Knowledge-Engine-as-a-Service (KaaS) Platform

> A standalone, multi-tenant AI retrieval gateway that transforms isolated, application-specific search into a universal Knowledge-Engine-as-a-Service platform.

---

## 1. Executive Summary

Instead of forcing individual software applications to build custom document loaders, chunking logic, and vector stores, the **Knowledge-Engine-as-a-Service (KaaS)** platform acts as a universal, private AI retrieval gateway—comparable to a self-hosted, enterprise-controlled alternative to Custom GPTs or Dify.

Any authorized application can stream raw documents into the platform via standard REST APIs and instantly query them using secure, context-aware semantic search.

---

## 2. The Problem vs. The Solution

| Challenge with Isolated RAG | The KaaS Solution |
| :--- | :--- |
| **Siloed Development:** Every application writes duplicate document parsing, chunking, and embedding logic. | **Single Gateway:** A unified API handles file parsing, vectorization, and retrieval across all applications. |
| **Domain-Locked Data Models:** Schemas hardcoded to specific entities (e.g., tennis courts, legal files) fail when scaled across verticals. | **Abstract Payloads:** Generic metadata envelopes allow any business vertical to reuse the same backend engine. |
| **Zero Data Mobility:** Strict multi-tenancy isolates data completely; systems cannot share or monetize valuable documentation. | **Access Ledger Pattern:** Granular permission rules support private tenant isolation, cross-organization sharing, and commercial knowledge licensing. |
| **Uncontrolled Token Usage:** Applications lack centralized budget caps, leading to unpredictable LLM and vector compute costs. | **Centralized Budgeting:** API key validation enforces hard rate limits and monthly usage caps per tenant. |

---

## 3. System Architecture & Workflow

External applications interact exclusively with the **KaaS Gateway** through authenticated API keys. The gateway validates permissions and monthly token budgets, routes raw source files to encrypted object storage, processes embeddings asynchronously, and executes multi-tenant semantic vector queries in PostgreSQL.

```text
+-------------------------------------------------------------------+
|                   External Calling Applications                   |
|        (e.g., Sports Tech, Legal Tech, Enterprise SaaS)           |
+---------------------------------+---------------------------------+
                                  |
                  HTTPS / TLS 1.3 + X-KaaS-Api-Key
                                  v
+-------------------------------------------------------------------+
|                        KaaS API Gateway                           |
|  * Redis Auth & Monthly Token Budget Middleware                   |
|  * Domain-Agnostic Ingestion & Semantic Search Controller         |
+--------+------------------------+------------------------+--------+
         |                        |                        |
         v                        v                        v
+-----------------+    +---------------------+    +-----------------+
|  Backblaze B2   |    | PostgreSQL +        |    |  React Admin    |
|  Object Storage |    | pgvector & EF Core  |    |  Console        |
|  (AES-256)      |    | (RLS + Access       |    |  * Doc Ledger   |
|                 |    |  Ledger Filtering)  |    |  * RAG Sandbox  |
+-----------------+    +---------------------+    +-----------------+
```

---

## 4. Key Architectural Pillars

### Pillar 1: The Multi-Tenant Access Ledger

Traditional multi-tenant systems enforce strict isolation by matching a single tenant identifier:

```sql
WHERE TenantId == @CurrentTenantId
```

While secure, absolute isolation prevents organizations from collaborating or licensing their operational knowledge. KaaS introduces a dual-layer **Access Ledger Pattern**:

* **Private by Default:** Documents remain restricted to the owner tenant unless explicitly shared.
* **Granular Sharing:** An `IsPubliclyShared` flag exposes public knowledge libraries, while a `TenantAccessGrant` lookup table grants controlled read access to specific third-party tenants.
* **Commercialization:** Premium operators can license specialized standard operating procedures (SOPs), guidelines, or training content to partner organizations on a subscription basis without relinquishing custody of their underlying files.

```csharp
// Universal Multi-Tenant Access Filter
var accessibleRecords = await _context.KnowledgeRecords
    .Where(k => k.TenantId == userTenantId
             || k.IsPubliclyShared
             || _context.TenantAccessGrants.Any(g => g.RecordId == k.Id && g.GranteeTenantId == userTenantId))
    .OrderBy(k => k.Embedding.CosineDistance(queryVector))
    .Take(topK)
    .ToListAsync();
```

---

### Pillar 2: Domain-Agnostic Web API & Token Budgeting

To support multiple industries without redesigning database schemas, the ingestion engine decouples from domain-specific entity properties:

* **Generic Payloads:** Rather than binding to fixed properties such as `CourtId` or `CaseFileId`, API endpoints accept an abstract `AssetReferenceKey` alongside flexible key-value metadata tags.
* **Token Budgeting & Security:** Callers authenticate using scoped request headers (`X-KaaS-Api-Key: k_live_...`). Lightweight Redis middleware validates the token, checks real-time consumption against monthly tenant budget thresholds, and throttles excess requests before vector compute operations execute.
* **Asynchronous Processing:** Uploaded documents stream directly to encrypted object storage (Backblaze B2), triggering background workers to extract text, compute vector embeddings, and build searchable indexes.

---

### Pillar 3: Web-Based Management Console

A unified **React** management interface gives non-technical operators and system administrators full visibility and control over their knowledge bases:

* **Document Ledger:** Real-time visibility into uploaded assets, background indexing status (`Queued`, `Processing`, `Indexed`, `Failed`), storage footprint, and visibility scopes (`Private`, `Shared`, `Public`).
* **RAG Playground:** An interactive test sandbox enabling administrators to run sample queries, inspect matched document chunks, and evaluate cosine similarity scores before downstream client applications consume the index.

---

## 5. Security & Isolation Matrix

| Security Layer | Technical Implementation | Operational Safeguard |
| :--- | :--- | :--- |
| **Row-Level Tenancy** | EF Core Global Query Filters + PostgreSQL Row-Level Security (RLS) | Prevents unauthorized data leakage across tenant boundaries. |
| **Storage at Rest** | AES-256 encrypted object storage partitioned by Tenant ID | Isolates raw source documents physically and cryptographically. |
| **Transit Security** | Strict TLS 1.3 across all REST and vector streaming endpoints | Protects API payloads and retrieval streams against interception. |
| **API Control** | Scoped API tokens with instant Redis cache invalidation | Enables immediate revocation of compromised or over-quota keys. |
| **Compliance Logging** | Append-only audit logs for all uploads, queries, and access grants | Supports governance standards for SOC 2, HIPAA, and GDPR auditability. |

---

## 6. Implementation Roadmap

### Phase 1: Foundation (Weeks 1–4)
* Configure PostgreSQL schema with `pgvector` extensions and tenant isolation filters.
* Implement the C# Universal Ingress Controller and asynchronous document streaming pipeline.
* Configure encrypted Backblaze B2 object storage integration.

### Phase 2: Federation & Token Budgeting (Weeks 5–8)
* Deploy Redis-backed rate limiting and monthly tenant token budgeting middleware.
* Implement the Access Ledger authorization system (`IsPubliclyShared` and `TenantAccessGrant`).
* Publish domain-agnostic REST API endpoints and client authentication libraries.

### Phase 3: Administrative Dashboard & Launch (Weeks 9–12)
* Build the React management console for document tracking and API key provisioning.
* Deliver the embedded RAG Playground for live query testing and similarity tuning.
* Complete end-to-end security audits, load testing, and production deployment.

---

## 7. Technology Stack & Tooling

| Category | Technologies |
| :--- | :--- |
| **Backend & API Gateway** | C# / .NET Web API, Entity Framework Core (Global Query Filters) |
| **Database & Vector Search** | PostgreSQL, `pgvector` (Cosine Distance Indexing), Row-Level Security (RLS) |
| **Caching & Rate Limiting** | Redis (API Key Validation, Monthly Token Budgeting, Cache Invalidation) |
| **Object Storage** | Backblaze B2 (AES-256 Encrypted, Tenant-Partitioned Blobs) |
| **Frontend Console** | React (Document Ledger & Interactive RAG Playground) |
| **Security & Compliance** | TLS 1.3, Scoped API Keys (`X-KaaS-Api-Key`), Append-Only Audit Logging |

```
