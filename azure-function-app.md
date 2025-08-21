## Azure Function app Http Trigger using JavaScript with Node Runtime

### 1. Create a Local Function App

        func init MyFunctionApp --javascript
        cd MyFunctionApp
        func new --name HttpTrigger --template "HTTP trigger" --authlevel "anonymous"

This creates a folder with your function and a host.json file.

### 2. Function Code

    module.exports = async function (context, req) {
    
      context.log('JavaScript HTTP trigger function started.');
    
        try {
            const name = req.query.name || (req.body && req.body.name);
    
            const responseMessage = name
                ? `Hello, ${name}. This HTTP triggered function executed successfully.`
                : `This HTTP triggered function executed successfully. 
                Pass a name in the query string or in the request body for a personalized response.`;
    
            context.res = {
                status: 200,
                body: responseMessage
            };
        } catch (error) {
            context.log.error('Error processing request:', error);
    
            context.res = {
                status: 500,
                body: 'An unexpected error occurred while processing your request.'
            };
        }
    };

### 3. Run Locally

    func start
    
**output**:

    HttpTrigger: [GET,POST] http://localhost:7071/api/HttpTrigger

### test on local:

    curl http://localhost:7071/api/HttpTrigger?name=Siva

    Or 
    
    use Postman to send a POST request with a JSON body:

    {
      "name": "Sivasankar Thalavai"
    }

### test on Azure portal:

### Input:

<img width="565" height="596" alt="image" src="https://github.com/user-attachments/assets/1360c007-6241-49ad-874a-2af5404df57a" />

### Output:

<img width="844" height="589" alt="image" src="https://github.com/user-attachments/assets/6affe5b4-d65a-443d-b00d-efc92dec45d2" />

## Code to Send Mail using SMTP
        
        require('dotenv').config();
        const nodemailer = require('nodemailer');
        
        module.exports = async function (context, req) {
            context.log('SMTP Email Function Triggered');
        
            try {
                const { to, subject, text, html } = req.body || {};
        
                // Validate required fields
                if (!to || !subject || (!text && !html)) {
                    context.res = {
                        status: 400,
                        body: "Missing required fields: 'to', 'subject', and either 'text' or 'html'."
                    };
                    return;
                }
        
                // Create transporter
                const transporter = nodemailer.createTransport({
                    host: process.env.SMTP_HOST || 'smtp.example.com',
                    port: parseInt(process.env.SMTP_PORT) || 587,
                    secure: parseInt(process.env.SMTP_PORT) === 465, // true for 465
                    auth: {
                        user: process.env.SMTP_USER,
                        pass: process.env.SMTP_PASS
                    }
                });
        
                // Mail options
                const mailOptions = {
                    from: `"Azure Function" <${process.env.SMTP_USER}>`,
                    to,
                    subject,
                    text,
                    html
                };
        
                // Send mail
                const info = await transporter.sendMail(mailOptions);
                context.log('Email sent: ${info.messageId}');
        
                context.res = {
                    status: 200,
                    body: {
                        message: "Email sent successfully",
                        messageId: info.messageId,
                        to
                    }
                };
            } catch (error) {
                context.log.error('Error sending email:', error);
        
                context.res = {
                    status: 500,
                    body: {
                        message: "Failed to send email",
                        error: error.message,
                        timestamp: new Date().toISOString()
                    }
                };
            }
        };

### local.settings.json

        {
          "IsEncrypted": false,
          "Values": {  
            "SMTP_HOST": "smtp.office365.com",
            "SMTP_PORT": "587",
            "SMTP_USER": "noreply@gmail.com",
            "SMTP_PASS": "Password"
          }
        }

### input

        {
          "to": "recipient@example.com",
          "subject": "Test Email",
          "text": "This is a plain text email.",
          "html": "<p>This is an <strong>HTML</strong> email.</p>"
        }

### output:

        {
          "message": "Email sent successfully",
          "messageId": "<5943cf71-1ca4-ee6d-dce2-dc45ae87d403@marlabs.com>",
          "to": "recipient@example.com"
        }
## Azure Function app Service Bus Trigger using JavaScript with Node Runtime

### Code Sample

        module.exports = async function (context, mySbMsg) {
        
            context.log('Service Bus Triggered');
        
            try {
                context.log('Raw message:', mySbMsg);
        
                const parsedMessage = typeof mySbMsg === 'string' ? JSON.parse(mySbMsg) : mySbMsg;
                context.log('Parsed message:', parsedMessage);
        
                const { recipientEmails, fromEmail } = parsedMessage;
        
                if (!Array.isArray(recipientEmails) || !fromEmail) {
                    context.log.warn('Missing required fields: recipientEmails or fromEmail');
                    return;
                }
        
                context.log('Validation passed. Proceeding with email dispatch...');
                // Continue with transporter and email logic...
            } catch (error) {
                context.log.error('Fatal error during function execution:', error.message);
                context.log.error('Stack trace:', error.stack);
            }
        };

### output:
        
        2025-08-21T07:10:24Z   [Information]   Parsed message: {
          appName: 'TIMESHEET',
          fromEmail: 'DoNotReply@gmail.com',
          recipientEmails: [
            {
              toEmail: [Array],
              ccEmail: [Array],
              bccEmail: [Array],
              emailSubject: 'Timesheet Reminder',
              emailTemplate: '<p>Hello, please submit your timesheet by EOD.</p>'
            }
          ]
        }
        2025-08-21T07:10:24Z   [Information]   ✅ Validation passed. Proceeding with email dispatch...
        2025-08-21T07:10:24Z   [Information]   Executed 'Functions.cnf-mail-sbt-trigger' (Succeeded, Id=07f6b51e-0a4d-41a5-88dc-af000e017fed, Duration=4ms)

        
