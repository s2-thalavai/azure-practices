# Azure Functions

Azure Functions is a serverless compute service that lets you run event-driven code without managing infrastructure. 
It's ideal for lightweight, modular tasks triggered by events like 

- HTTP requests,
- timers, or 
- cloud service changes.

It’s designed for 

- event-driven, 
- stateless, and
- short-lived tasks,

making it ideal for microservices, automation, and reactive workflows.

## What Is Azure Functions?

Azure Functions is part of **Microsoft Azure’s serverless offerings**. 
It allows developers to write small pieces of code ("functions") **that execute in response to events**. 
These functions **automatically scale** and only **consume resources while running**, making them cost-effective and efficient.

### Why Use Azure Functions?

- **No server management:** You don’t need to provision or maintain servers.

- **Event-driven execution:** Functions respond to triggers like HTTP requests, queue messages, or file uploads.

- **Cost efficiency:** You pay only for the time your code runs.

- **Rapid development:** Focus on business logic without worrying about infrastructure.

### Common Use Cases

- **API endpoints:** Lightweight HTTP-triggered functions for microservices or REST APIs.

- **Data processing:** **React to changes** in databases, blob storage, or queues.

- **Scheduled tasks:** Cron-like jobs for cleanup, reporting, or notifications.

- **IoT and telemetry:** Handle device messages or sensor data.

- **Workflow automation:** Trigger actions based on business events (e.g., invoice received → send email).

| **Category**       | **Example**                                                   |
|---------------------|---------------------------------------------------------------|
| **API endpoints**   | Lightweight REST APIs, webhook handlers                       |
| **Data processing** | Blob ingestion, queue-based ETL, telemetry pipelines          |
| **Automation**      | Invoice validation, email dispatch, cleanup jobs              |
| **IoT**             | Sensor data ingestion, device event handling                  |
| **DevOps**          | CI/CD hooks, alerting, provisioning scripts                   |
| **AI orchestration**| Trigger RAG workflows, semantic extraction, validation        |

### Pros

- **Scalability:** Automatically scales based on demand.

- **Language flexibility:** Supports C#, JavaScript, Python, Java, PowerShell, and more.

-  **Integration:** Built-in bindings for Azure services like Cosmos DB, Event Grid, and Service Bus.

-  **Fast deployment:** Ideal for rapid prototyping and agile workflows.

-  **Durable Functions:** Enables stateful workflows with checkpoints.



### Cons

- **Cold start latency:** Latency on first invocation (esp. in Consumption plan).

- **Limited execution time:** Consumption plan functions have a timeout (default 5 minutes).

- **Complex debugging:** Distributed, event-driven architecture can complicate tracing.

- **Vendor lock-in:** Deep integration with Azure services may reduce portability.

- **Limited local state:** Stateless by design; use external storage for persistence.


## When to Use vs. Avoid

### Use Azure Functions when:

- You need lightweight, short-lived tasks.

- Your workload is event-driven and sporadic.

- You want to minimize infrastructure overhead.

### Avoid Azure Functions when:

- You need long-running processes or persistent state.

- You require fine-grained control over runtime and environment.

- Cold start latency is unacceptable for your use case.

---

## Strategic Fit for Your Architecture

- Given your expertise in modular document intelligence workflows, Azure Functions can be a powerful tool for:
 
- Semantic extraction triggers: Fire on blob upload or queue message.

- Contextual validation: Modular validators per vendor/invoice type.

- Audit-friendly dispatch: Email workflows with retry metadata and summary metrics.

- RAG orchestration: Trigger LangChain flows from Service Bus or Event Grid.

---

# Use Case: Invoice Validation with Audit Logging

- **Trigger:** Blob upload or Service Bus message Functionality:

- Parse invoice metadata (vendor, amount, date)

- Validate against business rules

- Log audit trail (status, timestamp, retry count)

- Emit result to downstream queue or storage

## Folder Structure

```Code
InvoiceValidatorFunction/
├── function.json
├── index.js (or main.py / run.csx)
├── auditLogger.js
├── validator.js
├── bindings/
│   └── blobTrigger.json
├── utils/
│   └── retryTracker.js
```

### Sample: index.js (Node.js)

```javascript
const { validateInvoice } = require('./validator');
const { logAudit } = require('./auditLogger');
const { trackRetry } = require('./utils/retryTracker');

module.exports = async function (context, invoiceBlob) {
  const invoice = JSON.parse(invoiceBlob);
  const retryInfo = trackRetry(context);

  const result = validateInvoice(invoice);
  await logAudit({
    invoiceId: invoice.id,
    vendor: invoice.vendor,
    status: result.status,
    timestamp: new Date().toISOString(),
    retryCount: retryInfo.count,
    functionInvocationId: context.invocationId
  });

  context.log(`Invoice ${invoice.id} processed: ${result.status}`);
  context.bindings.outputQueue = JSON.stringify(result);
};

```

### validator.js

```javascript
function validateInvoice(invoice) {
  const rules = {
    vendor: v => v && v.length > 2,
    amount: a => a > 0,
    date: d => !isNaN(Date.parse(d))
  };

  const status = Object.entries(rules).every(([key, rule]) => rule(invoice[key]))
    ? 'VALID'
    : 'INVALID';

  return { id: invoice.id, status };
}

module.exports = { validateInvoice };
````

### auditLogger.js

```javascript
const { BlobServiceClient } = require('@azure/storage-blob');

async function logAudit(entry) {
  const blobClient = BlobServiceClient.fromConnectionString(process.env.AUDIT_STORAGE);
  const container = blobClient.getContainerClient('audit-logs');
  const blobName = `${entry.invoiceId}-${Date.now()}.json`;
  const blockBlob = container.getBlockBlobClient(blobName);
  await blockBlob.upload(JSON.stringify(entry), Buffer.byteLength(JSON.stringify(entry)));
}

module.exports = { logAudit };
```

### retryTracker.js
```javascript
function trackRetry(context) {
  const count = context.retryContext?.retryCount || 0;
  return { count };
}

module.exports = { trackRetry };
```

### Bindings: function.json

```json
{
  "bindings": [
    {
      "name": "invoiceBlob",
      "type": "blobTrigger",
      "direction": "in",
      "path": "invoices/{name}",
      "connection": "AzureWebJobsStorage"
    },
    {
      "name": "outputQueue",
      "type": "queue",
      "direction": "out",
      "queueName": "validated-invoices",
      "connection": "AzureWebJobsStorage"
    }
  ]
}
```
---

### Step 1: Prerequisites

Make sure you have installed:

1. Node.js (LTS recommended)

2. Azure Functions Core Tools (npm install -g azure-functions-core-tools@4 --unsafe-perm true)

3. Azure CLI (for deployment)

4. Optional: VS Code with Azure Functions extension

Check installation:

  node -v
  func --version
  az --version

### Step 2: Create a New Function App

# Create project folder
mkdir MyFunctionApp
cd MyFunctionApp

# Initialize function app
func init . --worker-runtime node --language javascript

  --worker-runtime node → Node.js runtime
  
  --language javascript → JS function (use --language typescript for TypeScript)

### Step 3: Add a New Function

# Add HTTP-triggered function

   func new


You will see a prompt:

```
Select a template: 
1. HttpTrigger
2. TimerTrigger
...
```

Choose HttpTrigger → give it a name (e.g., HelloFunction) → Authorization level (Anonymous for public access, Function for key-protected).

### Step 4: Function Code Example

Inside fetchTokenHttpTrigger.js:

```
const { app } = require('@azure/functions');

app.http('fetchTokenHttpTrigger', {
    methods: ['GET', 'POST'],
    authLevel: 'anonymous',
    handler: async (request, context) => {
        context.log(`Http function processed request for url "${request.url}"`);

        const name = request.query.get('name') || await request.text() || 'world';

        return { body: `Hello, ${name}!` };
    }
});

```

### Step 5: Run Locally

   func start

Open browser → http://localhost:7071/api/HelloFunction?name=John

You should see:

   Hello, John!

### Step 6: Deploy to Azure

Login to Azure:

  az login

Create a Function App in Azure:

# Variables
RESOURCE_GROUP=myResourceGroup
FUNCTION_APP_NAME=myfuncapp12345
STORAGE_ACCOUNT=mystorageacct12345
LOCATION=CentralIndia

# Create resource group
az group create --name $RESOURCE_GROUP --location $LOCATION

# Create storage account
az storage account create --name $STORAGE_ACCOUNT --location $LOCATION --resource-group $RESOURCE_GROUP --sku Standard_LRS

# Create function app
az functionapp create --resource-group $RESOURCE_GROUP --consumption-plan-location $LOCATION --runtime node --runtime-version 18 --functions-version 4 --name $FUNCTION_APP_NAME --storage-account $STORAGE_ACCOUNT


Deploy your function:

  func azure functionapp publish $FUNCTION_APP_NAME

Get your function URL:

az functionapp function show --function-name HelloFunction --name $FUNCTION_APP_NAME --resource-group $RESOURCE_GROUP


Your HTTP-triggered Azure Function is live!

---

Next steps:

Test the function locally with a POST request, for example using curl or Postman:

```
curl -X POST http://localhost:7071/api/fetchToken \
     -H "Content-Type: application/json" \
     -d '{"code":"<AUTH_CODE>","appName":"app1"}'
```

or with a refresh token:

```
curl -X POST http://localhost:7071/api/fetchToken \
     -H "Content-Type: application/json" \
     -d '{"refreshToken":"<REFRESH_TOKEN>","appName":"app1"}'
```

Check the logs in your terminal to confirm token requests are hitting the function and responses are returned.

Once verified locally, you can deploy to Azure.

---

