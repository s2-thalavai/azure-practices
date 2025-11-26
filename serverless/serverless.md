# azure-serverless

# 1. Core Compute Serverless

## 🔹 Azure Functions

**What it is:** Event-driven, serverless code (what you’re already using).  
**Triggers from:** HTTP, Service Bus, Event Grid, Timers, Storage queues/blobs, etc.  
**Use cases:**

-   Background jobs (email sending, file processing)
    
-   APIs / webhooks
    
-   Glue logic between systems (integrations, transformations)
    

You only pay for execution time and number of executions (on Consumption / Elastic Premium plans).

----------

## 🔹 Azure Logic Apps

**What it is:** Low-code serverless workflow engine.  
**Think:** “Power Automate, but in Azure for pro/integration scenarios.”

**Use cases:**

-   Orchestration of systems (Salesforce, SAP, Office 365, Service Bus, HTTP, etc.)
    
-   Approval flows, data sync, integration workflows
    
-   When the workflow is more important than custom code
    

Often:

> Logic App orchestrates → Azure Functions do custom compute → Service Bus/Event Grid glue them.

----------

## 🔹 Azure Container Apps (serverless containers)

**What it is:** Run containers with scale-to-zero and KEDA-based autoscaling (e.g., on HTTP or Service Bus).  
**Why it’s “serverless-like”:**

-   No VM management.
    
-   Scale based on events/CPU/queue length.
    
-   You can run your Functions runtime _inside_ Container Apps as well.
    

Great when:

-   You outgrow just Functions (more complex runtimes, background workers, message processors).
    
-   You need more control over container environment but still want serverless scaling.
    

----------

# 2. Eventing & Messaging (serverless backbone)

## 🔹 Azure Event Grid

**What it is:** Serverless event router (pub/sub style).  
**Use cases:**

-   React to events from Storage, Resource changes, custom apps, etc.
    
-   “Fire an event → multiple subscribers (Functions, Logic Apps, WebHooks) react.”
    

## 🔹 Azure Service Bus

Not strictly “serverless compute,” but often used with serverless:

-   Enterprise messaging: queues & topics
    
-   Ordered, reliable messaging, sessions, dead-lettering  
    You already use this with Functions.
    

## 🔹 Azure Storage Queues

Cheaper, simpler queue for Functions:

-   Good for simple background workloads, batch jobs.
    

----------

# 3. Data / Database Serverless Options

## 🔹 Azure Cosmos DB (Serverless)

-   Cosmos DB has a **serverless capacity mode** where you pay per request/GB, no fixed RU/s.
    
-   Great for infrequent/variable workloads with JSON docs.
    

## 🔹 Azure SQL Database (Serverless)

-   Compute auto-pauses and resumes.
    
-   You pay for GB + compute used (vCore per second).
    
-   Good for dev / sporadic OLTP workloads.
    

## 🔹 Azure Synapse Analytics – Serverless SQL Pool

-   Query data in data lake with T-SQL, pay per TB of data processed.
    
-   No need to provision a dedicated SQL pool.
    

## 🔹 Azure Data Explorer (Kusto) – Auto-scale / auto-stop

Not purely “serverless” but has very elastic models for log/telemetry workloads.

----------

# 4. Web/API & Frontend Serverless

## 🔹 Azure Static Web Apps

**What it is:** Serverless hosting for frontends (React/Angular/Vue/static) + integrated serverless APIs (Azure Functions) behind `/api`.  
**Includes:**

-   Free SSL
    
-   Global CDN
    
-   Built-in CI/CD integration (GitHub/Azure DevOps)
    

Great for:

-   SPAs + Functions backend
    
-   Static sites with light server-side logic
    

## 🔹 App Service with Consumption-like behavior

App Service itself isn’t fully serverless, but:

-   Can host Functions under the hood.
    
-   Often used in “almost serverless” patterns.
    

----------

# 5. API & Integration Serverless Adjacent

## 🔹 Azure API Management (Consumption Tier)

-   Fully managed API gateway layer.
    
-   **Consumption tier** scales elastically and is billed per call.
    
-   Good to front serverless backends (Functions, Logic Apps, Container Apps).
    

## 🔹 Power Automate / Power Apps (if you include Power Platform)

-   Not strictly Azure, but often integrated in serverless architectures for business workflows and simple apps.
    

----------

# 6. How these typically fit together

Given what you’re already doing (Functions + Service Bus + email + blob storage), an architecture might look like:

-   **Frontend**: Static Web Apps / SPA hosted on Static Web Apps
    
-   **API & Business Logic**: Azure Functions (HTTP-triggered, Service Bus triggered)
    
-   **Background Jobs**: Functions hosted as:
    
    -   Consumption Functions, or
        
    -   Functions in **Container Apps** for containerized scaling
        
-   **Workflows / Integration**: Logic Apps to orchestrate long-running processes
    
-   **Messaging**: Service Bus (topics/queues) + Event Grid for events
    
-   **Data**:
    
    -   Cosmos DB serverless for JSON/state
        
    -   SQL serverless for transactional data
        
-   **Public API Layer**: API Management (consumption) in front of everything


----------

