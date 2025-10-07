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


## 2. Add Retry Logic for Transient SMTP Failures

Wrap sendMail in a retry loop with exponential backoff:

```js
const sendWithRetry = async (mailOptions, retries = 3) => {
    for (let attempt = 0; attempt <= retries; attempt++) {
        try {
            return await transporter.sendMail(mailOptions);
        } catch (error) {
            if (attempt === retries) throw error;
            await new Promise(res => setTimeout(res, 1000 * Math.pow(2, attempt)));
        }
    }
};
```

---

### Replace:

```js
const info = await transporter.sendMail(mailOptions);
```

### With:

```js
const info = await sendWithRetry(mailOptions);
```
---

### 3. Audit-Friendly Logging

Include appName, recipient count, and status summary in final logs:

```js
context.log(`${appName} dispatch complete. Total: ${emailResults.length}, Sent: ${emailResults.filter(e => e.status === 'sent').length}, Failed: ${emailResults.filter(e => e.status === 'failed').length}`);
```

### 4. Optional: Modularize Dispatcher

Extract the email dispatch logic into a reusable function for onboarding clarity:

```js
async function dispatchEmail(recipient, appName, transporter, semaphore, context, emailResults) {
    const toAddresses = parseAddresses(recipient.toEmail);
    const ccAddresses = parseAddresses(recipient.ccEmail);
    const bccAddresses = parseAddresses(recipient.bccEmail);
    const subject = recipient.emailSubject;
    const html = recipient.emailTemplate;

    if (!toAddresses.length || !subject || !html) {
        context.log.warn(`Skipped email: Missing fields for ${JSON.stringify(toAddresses)}`);
        return;
    }

    const mailOptions = {
        from: `"Azure Function" <${process.env.SMTP_USER}>`,
        to: toAddresses,
        cc: ccAddresses.length ? ccAddresses : undefined,
        bcc: bccAddresses.length ? bccAddresses : undefined,
        subject,
        html
    };

    try {
        await semaphore.use(async () => {
            const info = await sendWithRetry(mailOptions);
            context.log(`Email sent to ${toAddresses.join(', ')} | Message ID: ${info.messageId}`);
            emailResults.push({ to: toAddresses, status: 'sent', messageId: info.messageId });
        });
    } catch (error) {
        context.log.error(`Failed to send email to ${toAddresses.join(', ')}`, error.message);
        emailResults.push({ to: toAddresses, status: 'failed', error: error.message });
    }
}
```
---
