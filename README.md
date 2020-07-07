# Distributed tracing Using Functions and Log Analytics

This document represents a proof of concept. All the "packages" transit needs to be traceable.
A producer generates data that is stored in a storage account. The storage account has a queue container.
The queue container triggers a function that pushes the content to API Management.
The API Management then posts the data to a backend Rest API server and writes the operations logs to an Azure Event Hubs. A function is triggered by the Azure Event Hubs' event and posts the log data to Azure Log Analytics as a custom log.
From Azure Log Analytics, it is possible to query the traces, using Kusto Query Language.

Requirements:

- The message posted to the Rest API needs to be easily traceble end to end.
- The architecture must be cost-effective
- The achitecture needs to scale fast if needed

![simple_view](media/10000fts_view.png)

## Function to process Queue

The Function consumes queue data adds a custom currelation ID and then posts to Azure API Management.

<https://github.com/thiagofborn/queueTriggerToSendPost>

## Function to read from Azure Event Hubs

The Function reads from Azure Event Hubs and sends the information formated to a specific Azure Log Analytics's workspace customized.

<https://github.com/thiagofborn/eventHubsTriggerLogAnalytics>

## The Azure API Management custom Policy

The **"REST_API_BACKEND"**, is runnimg on Azure as Web APP.

```xml
<policies>
    <inbound>
        <set-variable name="message-id" value="@(Guid.NewGuid())" />
        <set-backend-service base-url="https://REST_API_BACKEND.azurewebsites.net" />
    </inbound>
    <backend>
        <forward-request follow-redirects="true" />
    </backend>
    <outbound>
        <log-to-eventhub logger-id="apimcustomlogseh">@{
          var statusLine = string.Format("HTTP/1.1 {0} {1}\r\n",
                                              context.Response.StatusCode,
                                              context.Response.StatusReason);

          var body = context.Response.Body?.As<string>(true);
          if (body != null && body.Length > 1024)
          {
              body = body.Substring(0, 1024);
          }

          var headers = context.Request.Headers
                               .Where(h => h.Key == "CustomCorrelationId")
                               .Select(h => string.Format("{0}: {1}", h.Key, String.Join(", ", h.Value)));

          var CustcurrelantionId = (headers.Any()) ? string.Join(string.Empty, headers) : string.Empty;

          return string.Join(",", DateTime.UtcNow, context.Deployment.ServiceName, context.RequestId, context.Request.IpAddress, context.Operation.Name, CustcurrelantionId);
     }</log-to-eventhub>
    </outbound>
    <on-error>
        <base />
    </on-error>
</policies>
```

## Backend Server Rest API

To keep simple, I used an Azure Web APP written in .Net Core to run a Rest API application.  To be convenient during the Azure API Management integration, I used OpenAPI definitions on the application.

<https://github.com/thiagofborn/restapimocktests>
