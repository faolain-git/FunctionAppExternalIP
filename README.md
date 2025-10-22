# Azure Function App – Outbound IP Test

This project creates a simple **Azure Function App** (Windows, .NET 8, Isolated Worker) that determines and returns its **external outbound IP address**.  
It is primarily intended for testing network egress configuration in Azure environments.

---

## Prerequisites

Before you begin, ensure you have the following installed:

- [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli)
- [Azure Functions Core Tools (v4)](https://learn.microsoft.com/en-us/azure/azure-functions/functions-run-local)
- [.NET SDK 8.0](https://dotnet.microsoft.com/en-us/download)
- [Visual Studio Code](https://code.visualstudio.com/)
- Azure subscription access (with permissions to deploy Function Apps)

---

## 1. Initialize a Local Azure Function Project

```bash
func init MyFunctionApp --worker-runtime dotnet-isolated --target-framework net8.0
cd MyFunctionApp
```
## 2. Create the Function
```bash
func new --name GetOutboundIP --template "HTTP trigger" --authlevel "anonymous"
```

Replace the contents of GetOutboundIP.cs with the following:
```csharp
using System.Net.Http;
using System.Threading.Tasks;
using Microsoft.Azure.Functions.Worker;
using Microsoft.Azure.Functions.Worker.Http;
using Microsoft.Extensions.Logging;

namespace MyFunctionApp
{
    public class GetOutboundIP
    {
        private readonly HttpClient _client = new HttpClient();
        private readonly ILogger<GetOutboundIP> _logger;

        public GetOutboundIP(ILogger<GetOutboundIP> logger)
        {
            _logger = logger;
        }

        [Function("GetOutboundIP")]
        public async Task<HttpResponseData> Run(
            [HttpTrigger(AuthorizationLevel.Anonymous, "get", "post")] HttpRequestData req)
        {
            _logger.LogInformation("Processing request to get outbound IP.");

            var ip = await _client.GetStringAsync("https://api.ipify.org");
            var response = req.CreateResponse(System.Net.HttpStatusCode.OK);
            await response.WriteStringAsync($"Outbound IP: {ip}");
            return response;
        }
    }
}
```
## 3. Test Locally

Start the function locally:
```bash
func start
```

Then open the URL shown in the output (for example):

http://localhost:7071/api/GetOutboundIP

You should see the function return your local outbound IP (e.g. Outbound IP: 20.13.235.22).
Use Ctrl + C to stop the local function.

## 4. Publish to Azure

Login and set your subscription:
```ps
az login
az account set --subscription "<YOUR_SUBSCRIPTION_NAME>"
```
Deploy the function:
```ps
func azure functionapp publish <YOUR_FUNCTION_APP_NAME>
```
Example:
```ps
func azure functionapp publish itcnofrixion
```
If your Function App has restricted public access, ensure your client IP is allowed in:
Azure Portal → Function App → Networking → Access Restrictions.

## 4.1 Alternative Publish Method
Use the Publish Profile for ZIP Deployment

Publish your function locally:
```bash
dotnet publish -c Release -o ./publish
```

Zip the published files:

```ps
cd publish
Compress-Archive -Path * -DestinationPath ../MyFunctionApp.zip
cd ..
```

Deploy using the publish profile with az functionapp deployment:
```ps
$Profile = "C:\Path\To\itcnofrixion.PublishSettings"
az functionapp deployment source config-zip `
    --resource-group ITC-FNAPP-NEU-RG `
    --name itcnofrixion `
    --src MyFunctionApp.zip `
    --subscription <YourSubscriptionID>
```

## 5. Test in Azure

After successful deployment, open:

https://<YOUR_FUNCTION_APP_NAME>.azurewebsites.net/api/GetOutboundIP

It should return the outbound public IP of your Function App as seen on the internet.
