# Azure Function Deployment Checklist (Node.js + External Modules)

## 1. Project Structure

### Ensure your function folder contains:

  ```Code
  booking-cnf-mail-sbt-trigger/
  ├── index.js
  ├── function.json
  ├── package.json
  ├── node_modules/
  ├── .env
  ```
---
### 2. Install Required Modules

### Run this in your function directory:

```bash
  npm install nodemailer await-semaphore dotenv --save
```

This creates node_modules and updates package.json.
---
### 3. Verify package.json

Ensure dependencies are listed:

```json
"dependencies": {
  "nodemailer": "^6.9.1",
  "await-semaphore": "^0.3.3",
  "dotenv": "^16.3.1"
}
```
---
### 4. Local Test

Run locally to confirm module loading:

```bash
func start
```

---
### 5. Deployment Options

Zip Deploy (Azure CLI / Portal)

```bash
zip -r function.zip .
az functionapp deployment source config-zip \
  --name <FunctionAppName> \
  --resource-group <ResourceGroup> \
  --src function.zip
```

### Include node_modules in the zip

## GitHub Actions / CI/CD
Add a build step:

```yaml
- run: npm install
```

Ensure .funcignore or .gitignore does not exclude node_modules

---
### 6. Environment Variables
Use .env for local dev and configure in Azure Portal:

```env
SMTP_HOST=smtp.office365.com
SMTP_PORT=587
SMTP_USER=your-smtp-user@example.com
SMTP_PASS=your-smtp-password
```

### In Azure:

  Go to Function App → Configuration → Application Settings
  
  Add these as key-value pairs

---

### 7. Clean Build (Optional)

For reproducible installs:

```bash
rm -rf node_modules package-lock.json
npm ci
```
---
### 8. Monitor & Alert

  Enable Application Insights
  
  Track SMTP failures, retries, and throughput
  
  Set alerts on Function errors or Service Bus dead-letter growth

---
