# Azure SQL (Operational / Write model) to Azure Cosmos DB (Read Model) in a CQRS + Event-Driven architecture.

This is widely used in enterprise systems for building low-latency query models, GraphQL BFFs, and portal dashboards.

# ✅ **How to Post Data from Azure SQL → Azure Cosmos (Read Model)**

You DO **NOT** replicate SQL tables directly.  
You **project events** into a **denormalized Cosmos read model**.

✅ **Source of truth:** Azure SQL (write model with RLS)  
✅ **Projection:** Cosmos DB (materialized view)

----------

# ✅ **Recommended Architecture (Enterprise Standard)**

### **1. Domain services write to Azure SQL**

-   Vendor API, Workflow, Rule Engine update SQL normally
    
-   SQL is your _authoritative_ state
    

Example:

`UPDATE Vendor SET status='VALIDATED'  WHERE vendor_id=123` 

----------

### **2. Services publish domain events to Azure Service Bus**

Whenever SQL changes:

✅ Write Model Update  
✅ Domain Event Emitted

Example events when SQL changes:

-   `vendor.submitted`
    
-   `vendor.updated`
    
-   `document.ocr.completed`
    
-   `invoice.validated`
    
-   `workflow.state.changed`
    

This is handled using **Outbox Pattern** or **transactional event emitters**.

----------

# ✅ **3. GraphQL BFF / Read Model Projector consumes ASB events**

A dedicated **Read Model Projector** (Go or Node.js) consumes events:

### Projector responsibilities:

✅ Read event payload  
✅ Fetch missing details from Azure SQL (if needed)  
✅ Build denormalized document  
✅ Upsert into Cosmos DB

This builds a **query-optimized** model.

----------

# ✅ **4. Projector writes into Cosmos (Materialized View)**

Use **idempotent upsert**:

`{  "id":  "vendor-123",  "tenantId":  "contoso",  "vendorName":  "ABC Traders",  "status":  "VALIDATED",  "documents":  [...],  "lastUpdatedAt":  "2025-11-10T10:20:00Z"  }` 

Cosmos containers typically follow:  
✅ `/tenantId` partition key  
✅ `/id` document identifier

----------

# ✅ **5. GraphQL/API reads directly from Cosmos**

Ultra-fast reads for UI portals:

-   Search
    
-   Dashboard
    
-   Vendor 360
    
-   Realtime workflow view
    
-   Approval queues
    

Cosmos handles:  
✅ High read throughput  
✅ Low latency  
✅ Multi-tenant scaling  
✅ Automatic indexing

----------

# ✅ End-to-End Flow (Simple View)

`Azure SQL → Domain Event  →  ASB Topic  
            → Read Model Projector  
            →  Cosmos DB (materialized  view)  
            →  GraphQL / Portal UI` 

----------

# ✅ **Correct Pattern: Outbox + Projector**

### **Outbox Pattern (SQL → ASB)**

-   SQL transaction inserts into an `Outbox` table
    
-   Background worker publishes outbox records → ASB
    
-   Ensures **exactly-once** event publishing
    
-   Eliminates race conditions
    

### **Read Model Projector (ASB → Cosmos)**

-   Reads each event
    
-   Ensures **idempotency**
    
-   Upserts into Cosmos
    
-   Maintains denormalized read model
    

----------

# ✅ Sample Projector Pseudocode (Go)

`for event := range serviceBusConsumer { switch event.Type { case  "vendor.updated":
        vendor := sqlRepo.GetVendor(event.VendorId)
        model := BuildVendorReadModel(vendor)
        cosmos.Upsert(model) case  "document.ocr.completed":
        doc := sqlRepo.GetDocument(event.DocId)
        cosmos.UpsertDocumentProjection(doc)
    }
}` 

----------

# ✅ Patterns You MUST Use (Best Practices)

### ✅ 1. **Idempotent Upsert**

Avoid duplicate writes. Use event version or timestamp.

### ✅ 2. **Versioned Read Model**

Include `eventVersion` or `lastUpdatedAt`.

### ✅ 3. **Multi-tenant Partitioning**

Partition by `/tenantId`.

### ✅ 4. **Schema Evolution**

Cosmos supports evolving documents without migrations.

### ✅ 5. **Dead Letter Queue (DLQ)**

Read model projector must handle failures gracefully.

----------

# ✅ Good Naming Conventions

| Event                  | Cosmos Document   |
| ---------------------- | ----------------- |
| vendor.updated         | VendorReadModel   |
| document.ocr.completed | DocumentReadModel |
| workflow.state.changed | ApprovalReadModel |
| invoice.validated      | InvoiceReadModel  |


----------

# ✅ Summary

To “post SQL data into Cosmos,” you **do not sync tables**.  
You **project domain events** into Cosmos as **read models**.

✅ ASB → Projector → Cosmos  
✅ CQRS  
✅ Denormalized  
✅ No coupling between SQL schema & Cosmos structure  
✅ Highly scalable and MT-safe

----------


Here is a **clean, enterprise-grade explanation** of:

✅ **Outbox Pattern**  
✅ **Transactional Event Emitters**

…and WHEN/WHY to use each **to reliably publish events from Azure SQL → Azure Service Bus → Cosmos Read Models**.

----------

# ✅ **1. Outbox Pattern (Recommended for Microservices with SQL)**

This is the **gold standard** for reliable event publishing whenever your service uses a relational database (Azure SQL).  
It ensures **exactly-once**, **transactionally consistent** event emission.

----------

## ✅ **How the Outbox Pattern Works**

### **Step 1 — Write SQL change + Outbox record in SAME transaction**

`BEGIN TRANSACTION; UPDATE Vendor SET status =  'VALIDATED'  WHERE id =  123; INSERT  INTO Outbox (id, eventType, payload, createdAt) VALUES ('evt-789', 'vendor.validated', '{...}', GETUTCDATE()); COMMIT;` 

✅ Both SQL update **and** event record succeed or fail together.  
❌ No partial state.  
❌ No dropped events.

----------

### **Step 2 — Background Worker pulls outbox records**

A small worker runs every few seconds:

1.  Reads Outbox rows that are not sent
    
2.  Publishes them to Azure Service Bus
    
3.  Marks them as “sent” or deletes them
    

### Worker Example (Go / Java pseudocode)

`for row := range outboxRepo.FetchUnsent() {
    serviceBus.Publish(row.eventType, row.payload)
    outboxRepo.MarkAsSent(row.id)
}` 

----------

### ✅ Why Outbox Pattern is essential

| Problem Without Outbox                           | Risk                        |
| ------------------------------------------------ | --------------------------- |
| DB updated but event not published               | ❌ Inconsistent read model   |
| Event published but DB transaction failed        | ❌ Phantom events            |
| Microservice crash between DB update and publish | ❌ Lost events               |
| Azure SB transient failures                      | ❌ Partial workflow triggers |

Outbox guarantees:

✅ Exactly-once event emission
✅ No double-publish
✅ No race conditions
✅ Read model stays consistent

----------

# ✅ **2. Transactional Event Emitters (Database + Broker in One Transaction)**

This is used when:

-   The database **and** message broker support the _same_ transaction
    
-   Example: PostgreSQL + Kafka with Kafka Connect
    
-   Or SQL Server with Service Broker
    
-   Or DynamoDB with Streams
    

Azure SQL **does NOT** support distributed messaging transactions with ASB.

👉 **Therefore transactional emitters are NOT compatible with Azure SQL + Azure Service Bus.**

Meaning:

### ✅ **Outbox Pattern is mandatory**

for Azure SQL → ASB → Cosmos architectures.

----------

# ✅ **Why Outbox is superior in Azure-Native EDA**

| Pattern                      | Works with Azure SQL?       | Works with Azure Service Bus? | Reliability        |
| ---------------------------- | --------------------------- | ----------------------------- | ------------------ |
| Outbox                       | ✅ Yes                       | ✅ Yes                         | ✅ Highest          |
| Transactional Emitters       | ❌ No (no XA / 2PC with ASB) | ❌ No                          | ❌ Not supported    |
| Best-effort event publishing | ✅ Yes                       | ✅ Yes                         | ⚠️ Can lose events |


Outbox is the **only** enterprise-safe option in this architecture.

----------

# ✅ Outbox → ASB → Cosmos Read Model (Correct Flow)

```
Azure SQL (Write Model)
        |
   Outbox Table |
 Outbox Worker (Go/Java)
        |
 Azure Service Bus (Events)
        | Read Model Projector
        |
 Azure Cosmos DB (Read Model)
``` 

✅ Strongly consistent workflow  
✅ Event-driven projections  
✅ No schema leaks  
✅ No race conditions

----------

# ✅ Summary: What to Use?

### ✅ You **must** use the Outbox Pattern

Azure SQL + Azure Service Bus do **not** support distributed ACID transactions.

### ✅ Transactional event emitters are **NOT available** in Azure SQL

(no 2-phase commit with ASB).

### ✅ Outbox ensures:

-   Atomic DB write + event record
    
-   Guaranteed delivery
    
-   Idempotent projections
    
-   Robust read model consistency
    

----------
