# Triggering Azure DevOps Build Pipelines on ACR Base Image Updates

## Overview

When a base image in Azure Container Registry (ACR) is updated, you may want to automatically trigger Azure DevOps (ADO) build pipelines to rebuild dependent application images. This guide covers several approaches to achieve this automation.

## Approaches to Trigger ADO Pipelines from ACR Updates

### 1. ACR Tasks with Base Image Triggers (Recommended)

Azure Container Registry Tasks can automatically detect when base images are updated and trigger builds. This is the most integrated approach within the Azure ecosystem.

#### Setup Steps:

1. **Create an ACR Task that tracks base image dependencies:**

```bash
# Create a task that builds your application image and tracks base image dependencies
az acr task create \
    --registry myregistry \
    --name myapp-task \
    --image myapp:{{.Run.ID}} \
    --context https://github.com/myorg/myapp.git#main \
    --file Dockerfile \
    --git-access-token $GIT_PAT \
    --commit-trigger-enabled false  # Disable source code triggers if only wanting base image triggers
```

2. **The task will automatically trigger when base images update:**
   - ACR Tasks monitors the base image specified in your Dockerfile
   - When the base image is pushed to the same registry, it triggers a rebuild
   - You can track both local ACR base images and external registry images

3. **To integrate with Azure DevOps, use ACR Task webhooks:**

```bash
# Create a webhook that calls your ADO pipeline
az acr webhook create \
    --name myapp-webhook \
    --registry myregistry \
    --uri https://dev.azure.com/myorg/_apis/public/distributedtask/webhooks/mywebhook?api-version=6.0-preview \
    --actions push \
    --scope myapp:*
```

### 2. ACR Webhooks + Azure Logic Apps + ADO (Alternative Approach)

This approach uses ACR webhooks to trigger a Logic App that then calls Azure DevOps.

#### Setup Steps:

1. **Create an ACR Webhook:**

```bash
az acr webhook create \
    --name base-image-webhook \
    --registry myregistry \
    --uri https://your-logic-app-url \
    --actions push \
    --scope baseimages/*
```

2. **Create a Logic App with HTTP trigger:**
   - HTTP Request trigger to receive webhook calls from ACR
   - Azure DevOps action to "Queue a new build"
   - Configure with your ADO organization, project, and pipeline ID

3. **Logic App Workflow:**
   ```json
   {
       "trigger": "When HTTP request is received",
       "action": "Azure DevOps - Queue a new build",
       "configuration": {
           "organization": "myorg",
           "project": "myproject", 
           "buildDefinitionId": 123
       }
   }
   ```

### 3. ACR Webhooks + Event Grid + Azure Functions + ADO

For more complex scenarios requiring custom logic or filtering.

#### Setup Steps:

1. **Configure ACR to send events to Event Grid:**

```bash
# Subscribe to ACR events via Event Grid
az eventgrid event-subscription create \
    --name acr-events \
    --source-resource-id /subscriptions/.../resourceGroups/myRG/providers/Microsoft.ContainerRegistry/registries/myregistry \
    --endpoint https://myfunction.azurewebsites.net/api/HttpTrigger \
    --event-delivery-schema eventgridschema
```

2. **Create Azure Function to process events:**
   - HTTP trigger function to receive Event Grid events
   - Filter for base image push events
   - Call Azure DevOps REST API to trigger builds

3. **Example Function Code (C#):**

```csharp
[FunctionName("ProcessACREvents")]
public static async Task<IActionResult> Run(
    [HttpTrigger(AuthorizationLevel.Function, "post")] HttpRequest req,
    ILogger log)
{
    string requestBody = await new StreamReader(req.Body).ReadToEndAsync();
    var events = JsonSerializer.Deserialize<EventGridEvent[]>(requestBody);
    
    foreach (var gridEvent in events)
    {
        if (gridEvent.EventType == "Microsoft.ContainerRegistry.ImagePushed")
        {
            var eventData = JsonSerializer.Deserialize<ContainerRegistryImagePushedEventData>(gridEvent.Data.ToString());
            
            // Check if this is a base image
            if (eventData.Target.Repository.StartsWith("baseimages/"))
            {
                // Trigger ADO pipeline
                await TriggerADOPipeline(eventData.Target.Repository);
            }
        }
    }
    
    return new OkResult();
}
```

### 4. Direct ADO Incoming Webhooks

Azure DevOps supports incoming webhooks that can be called directly from ACR.

#### Setup Steps:

1. **Create Incoming Webhook Service Connection in ADO:**
   - Go to Project Settings → Service Connections
   - Create "Incoming WebHook" connection
   - Define webhook name and secret for security

2. **Configure Pipeline:**

```yaml
trigger: none
pr: none

resources:
  webhooks:
    - webhook: ACRWebHook
      connection: MyACRWebhookConnection

stages:
- stage: Build
  jobs:
  - job: BuildJob
    steps:
    - bash: |
        echo "Image updated: ${{ parameters.ACRWebHook.resource.target.repository }}"
        echo "Tag: ${{ parameters.ACRWebHook.resource.target.tag }}"
        # Your build logic here
```

3. **Create ACR Webhook pointing to ADO:**

```bash
az acr webhook create \
    --name ado-trigger \
    --registry myregistry \
    --uri https://dev.azure.com/myorg/_apis/public/distributedtask/webhooks/ACRWebHook?api-version=6.0-preview \
    --actions push \
    --scope baseimages/*
```

## Best Practices

### Security Considerations

1. **Use webhook secrets for authentication:**
   - Configure secrets in ACR webhooks
   - Verify payload signatures in receiving endpoints

2. **Limit webhook scope:**
   - Use specific repository/tag patterns
   - Avoid triggering on all registry events

3. **Use managed identities:**
   - For Azure Functions and Logic Apps calling ADO
   - Avoid storing credentials in configuration

### Monitoring and Reliability

1. **Implement retry logic:**
   - Handle transient failures in webhook calls
   - Use exponential backoff for retries

2. **Add logging and monitoring:**
   - Track webhook delivery success/failure
   - Monitor pipeline trigger rates

3. **Test webhook endpoints:**
   - Use ACR webhook ping functionality
   - Validate payload formats

### Performance Optimization

1. **Filter events appropriately:**
   - Only trigger on relevant base image updates
   - Use repository and tag filters

2. **Batch processing:**
   - Consider batching multiple related updates
   - Implement debouncing for rapid successive updates

## Example Scenarios

### Scenario 1: Multi-Stage Application

You have a base Python image that multiple application images depend on:

```dockerfile
# Base image Dockerfile
FROM python:3.11-slim
RUN pip install common-dependencies
```

```dockerfile
# Application Dockerfile  
FROM myregistry.azurecr.io/baseimages/python-base:latest
COPY app/ /app
```

**Solution:** Use ACR Tasks with base image tracking to automatically rebuild all dependent application images.

### Scenario 2: Security Patching

You regularly update base images with security patches and need to ensure all applications are rebuilt:

**Solution:** Combine ACR Tasks with scheduled triggers and webhook notifications to ADO for additional testing pipelines.

### Scenario 3: Multi-Environment Deployment

You want different environments to be updated at different cadences:

**Solution:** Use Event Grid with Azure Functions to implement custom routing logic based on image tags and environment requirements.

## Troubleshooting

### Common Issues

1. **Webhook delivery failures:**
   - Check endpoint availability and authentication
   - Verify webhook URL and payload format
   - Review ACR webhook event logs

2. **Pipeline not triggering:**
   - Confirm webhook scope matches updated images
   - Verify ADO webhook configuration
   - Check service connection permissions

3. **Authentication errors:**
   - Validate webhook secrets and signatures
   - Ensure proper service principal permissions
   - Check token expiration dates

### Debugging Steps

1. **Test webhook manually:**
   ```bash
   az acr webhook ping --name mywebhook --registry myregistry
   ```

2. **Review webhook events:**
   ```bash
   az acr webhook list-events --name mywebhook --registry myregistry
   ```

3. **Monitor pipeline runs:**
   - Check ADO pipeline history
   - Review run logs for trigger source

## Conclusion

The recommended approach is to use **ACR Tasks with base image triggers** combined with **ACR webhooks** to call Azure DevOps pipelines. This provides:

- Native integration within Azure ecosystem
- Automatic base image dependency tracking
- Reliable webhook delivery
- Built-in retry and monitoring capabilities

For more complex scenarios requiring custom logic, consider using Event Grid with Azure Functions or Logic Apps as intermediary processors.