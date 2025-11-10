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

