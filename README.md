# Microsoft Azure Practices

## Do I need Service Mesh

built a layered perimeter that handles:

    North-south traffic control (public APIM → private NGINX Ingress → private AKS)
    
    Network isolation (private cluster, private ingress)
    
    Centralized API exposure (APIM as the public gateway)
    
    TLS termination and routing (at APIM)
    
### Might Not Need a Service Mesh If:
    
    not doing complex service-to-service routing (e.g., canary, A/B, failover).
    
    You’ve already implemented mTLS or secure payload encryption between services.
    
    managing observability and tracing via Azure Monitor, App Insights, or custom logging.
    
    You want to minimize sidecar overhead and keep your cluster lean.
    
    You’ve got tight ingress control and don’t expose internal services directly.

### Might Consider a Service Mesh If:

    You want fine-grained control over pod-to-pod communication, especially for internal APIs.
    
    You need automatic mTLS between services without manual cert rotation.
    
    You’re scaling to multi-team microservices and want centralized policy enforcement.
    
    You want L7 authorization (e.g., allow only certain services to call /api/secure).
    
    You’re troubleshooting intermittent service issues and need distributed tracing.


### Current Setup vs Service Mesh
    
    Capability	            Your Setup (APIM + NGINX + AKS)	            Service Mesh (e.g., Istio, Linkerd)
    
    Public                  API Gateway	                                ✅ APIM	Optional
    Ingress Control	        ✅ NGINX	                                  ✅ Ingress Gateway
    Pod-to-Pod Security	    🔶 NetworkPolicies/manual mTLS	            ✅ Automatic mTLS
    API-Level AuthZ	        🔶 App-level or APIM policies	              ✅ Istio AuthorizationPolicy
    Observability	          ✅ Azure Monitor/App Insights	              ✅ Prometheus + Jaeger
    Traffic Shaping	        🔶 NGINX-level	                            ✅ Per-service routing rules

Given your focus on secure payload flows, cost optimization, and modular design, 
you might not need a full mesh unless:

    You’re planning to scale internal APIs across teams or environments.
    
    You want centralized enforcement of internal access policies.
    
    You’re hitting visibility or security gaps in east-west traffic.

If you're just looking to restrict pod-to-pod access for specific APIs, 
you could start with Kubernetes NetworkPolicies and app-level auth, and only adopt a mesh if those controls become too brittle or hard to manage.


### How would a service mesh impact my current setup?


## AZ Monitor:
  
  To monitor your AKS, Azure Functions, Static Web Apps, and Azure SQL DB using Azure Monitor, 
  the monthly cost depends primarily on **log ingestion volume, retention, and alerting frequency**.

### Estimated Monthly Cost Breakdown (Azure Monitor + Workbooks)
    
    Resource	                  Log Volume (Est.)	            Pricing Tier	          Cost/GB	            Monthly Cost
    
    AKS (Container Insights)      20 GB	                        Analytics Logs	          $2.30	              ~$46.00
    Azure Functions               10 GB	                        Basic Logs	              $0.50	              ~$5.00
    Static Web Apps               05 GB	                        Basic Logs	              $0.50	              ~$2.50
    Azure SQL DB                  10 GB	                        Analytics Logs	          $2.30	              ~$23.00
    Workbook Queries	           —                           Included	                —	                   $0.00
    
    🔹 **Total Estimated Cost: ~$76.50/month**

This assumes moderate production usage with optimized diagnostic settings and 30-day retention. 
Costs may vary based on verbosity, sampling, and query frequency.


### Azure Monitor Log Pricing Tiers
    
    Tier	            Price per GB	        Retention Included
    
    Basic Logs	        $0.50/GB	            30 days
    Analytics Logs	    $2.30/GB	            31–90 days
    Auxiliary Logs	    $0.05/GB	            30 days

mix tiers per table to optimize cost. 

  example: 
  
   Basic Logs         for infrequent queries and 
   Analytics Logs     for alerting and dashboards.


🛠️ Cost Optimization Tips
    
    Use Basic Logs for low-value or infrequent queries (e.g., Static Web Apps).
    
    Filter diagnostic settings: Only collect essential tables (e.g., exclude KubeEvents, ContainerLogV2, FunctionExecutionLogs if not needed).
    
    Use commitment tiers: If your workspace ingests >100 GB/month, commit to a daily volume for lower per-GB rates.
    
    Set retention policies to 30 days: Avoid long-term storage unless needed for compliance.
    
    Use App Insights sampling: For Functions and Web Apps, reduce telemetry volume with adaptive sampling.
    
    Leverage Workbooks for visualization—no extra cost for queries unless using long-term retention or Basic/Auxiliary logs.

## Azure Monitor Workbook

Let’s build a modular Azure Monitor Workbook that tracks cost and ingestion across your key services: 

AKS, Azure Functions, Static Web Apps, and Azure SQL DB. 

This will give you a clean, filterable dashboard with reusable KQL panels.

### Workbook Strategy

   You can build a unified Azure Monitor Workbook with:
      
      Environment filters (Dev, QA, Prod)
      
      Resource type dropdowns (AKS, Functions, SQL, WebApp)
      
      KQL panels for ingestion and cost tracking
      
      Alerting thresholds and SLA summaries
      

**Azure Monitor Workbook: Cost & Ingestion Tracker
**

    Step 1: Create Workbook Shell 
        
        Go to Azure Monitor → Workbooks → New
        
        Title: Cloud Cost & Ingestion Dashboard
        
        Add a drop-down parameter for Environment (e.g., Dev, QA, Prod)
        
        Add another for Resource Type (AKS, Functions, SQL, WebApp)

    
    Step 2: KQL Panel – Log Ingestion by Resource
    
          **kql**
          
          Usage
          | where TimeGenerated > startofday(now())
          | where IsBillable == true
          | summarize GB_Ingested = sum(Quantity) / 1024.0
                    by ResourceId, ResourceType, bin(TimeGenerated, 1d)
          | extend Environment = case(
              ResourceId has "prod", "Prod",
              ResourceId has "dev", "Dev",
              ResourceId has "qa", "QA",
              "Unknown"
          )
          | where Environment == "{Environment}" or "{Environment}" == "All"
          | where ResourceType == "{ResourceType}" or "{ResourceType}" == "All"
        
    Filters by environment and resource type
    
    Shows daily ingestion in GB

  Step 3: KQL Panel – Estimated Cost by Resource

        kql
    
        Usage
        | where TimeGenerated > startofday(now())
        | where IsBillable == true
        | summarize CostUSD = sum(Quantity) * 0.0023  // Analytics Logs rate
                  by ResourceId, ResourceType
        | extend Environment = case(
            ResourceId has "prod", "Prod",
            ResourceId has "dev", "Dev",
            ResourceId has "qa", "QA",
            "Unknown"
        )
        | where Environment == "{Environment}" or "{Environment}" == "All"
        | where ResourceType == "{ResourceType}" or "{ResourceType}" == "All"
    
    Uses $2.30/GB as default Analytics Logs rate

    You can parameterize the rate if needed

  Step 4: Add Time Chart – Ingestion Trend
  
    Visualize GB_Ingested over time
    
    Group by ResourceType or ResourceId
    
    Use stacked area chart for clarity

  Step 5: Add Summary Tiles

  Step 5: Add Summary Tiles
  
      Metric	              KQL Expression
      
      Total GB Ingested	    summarize sum(Quantity)/1024
      Estimated Cost	      summarize sum(Quantity)*0.0023
      Top Resource	        top 1 by sum(Quantity)

** Alerting Panel
** 

  You can add a panel that shows resources exceeding a threshold:
      
      kql
      
      Usage
      | where TimeGenerated > ago(1d)
      | summarize GB = sum(Quantity)/1024 by ResourceId
      | where GB > 5     
