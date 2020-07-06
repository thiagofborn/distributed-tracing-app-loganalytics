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

Function to consume queue data add a custom currelation ID and post to Azure API Management.

<https://github.com/thiagofborn/queueTriggerToSendPost>

## Function to read from Azure Event Hubs

Function that reads from Azure Event Hubs and sends the information formated to a specific case to Azure Log Analytics.

<https://github.com/thiagofborn/eventHubsTriggerLogAnalytics>
