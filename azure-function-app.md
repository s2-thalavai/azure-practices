## Azure Function app using JavaScript with Node Runtime

### Code:

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

### Input:

<img width="565" height="596" alt="image" src="https://github.com/user-attachments/assets/1360c007-6241-49ad-874a-2af5404df57a" />

### Output:

<img width="844" height="589" alt="image" src="https://github.com/user-attachments/assets/6affe5b4-d65a-443d-b00d-efc92dec45d2" />

