# Azure Log Analytics Data Export

## Summary

This Azure Function App using FastAPI enables the export of big data (10M+ records per hour) from Azure Log Analytics to Blob Storage via Python SDKs. In testing, 50M records with 10 columns were successfully exported in approximately 1 hour using a Consumption (Serverless) hosting plan.

This work expands upon: [How to use logic apps to handle large amounts of data from log analtyics](https://techcommunity.microsoft.com/t5/azure-integration-services-blog/how-to-use-logic-apps-to-handle-large-amount-of-data-from-log/ba-p/2797466) and [FastAPI on Azure Functions](https://blog.pamelafox.org/2022/11/fastapi-on-azure-functions-with-azure.html)

<b>Inputs and Outputs</b>:
- <b>Input</b>: log analytics workspace table(s), columns, and date range
- <b>Output</b>: JSON (list format, line delimited), CSV, or PARQUET files

<b>Azure HTTP Functions</b>:
1. <b>azure_ingest_test_data()</b>: creates and ingests test data (optional)
2. <b>azure_submit_query()</b>: submits single query that is split into smaller queries/jobs and sends to queue
3. <b>azure_submit_queries()</b>: breaks up initial query and submits multiple queries in parallel
4. <b>azure_get_status()</b>: gives high-level status of query (number of sub-queries, successes, failures, row counts, file sizes)

<b>Azure Queue Functions</b>:
1. <b>azure_queue_query()</b>: processes split queries
2. <b>azure_queue_process()</b>: processes subqueries and saves output to storage blobs 
3. <b>azure_queue_query_poison()</b>: processes invalid messages in query queue and saves to table log
4. <b>azure_queue_process_poison()</b>: processes invalid message in process queue and saves to table log

![image](https://github.com/dtagler/azure-log-analytics-data-export/assets/108005114/a018ec7c-c252-462c-8330-9b240feccb9f)
  
## Files

- <b>azure-log-analytics-data-export.ipynb</b>: python notebook for development, testing, or interactive use
- <b>function_app.py</b>: Azure Function App python source code
- <b>host.json</b>: Azure Function App settings
- <b>requirements.txt</b>: python package requirements file

## Setup Notes

<b>Create the Following Azure Resources</b>:
1. Log Analytics Workspace (data source)
2. Storage Account
- Container (data output destination)
- Queues (temp storage for split query messages/jobs)
- Tables (logging for status checks)
3. Azure Function App (Python 3.11+, consumption or premium plan)
4. Azure API Management

<b>Authentication Method (Managed Identity or Service Principal) Requirements</b>:
- Setup via Azure Portal -> Function App -> Identity -> System Assigned -> On -> Azure Role Assignments
1. <b>Monitoring Metrics Publisher</b>: Ingest to Log Analytics (optional)
2. <b>Log Analytics Contributor</b>: Query Log Analytics
3. <b>Storage Queue Data Contributor</b>: Storage Queue Send/Get/Delete
4. <b>Storage Queue Data Message Processor</b>: Storage Queue Trigger for Azure Function
5. <b>Storage Blob Data Contributor</b>: Upload to Blob Storage
6. <b>Storage Table Data Contributor</b>: Logging

<b>Required Environment Variables for Queue Triggers via Managed Identity</b>: 
- Setup via Azure Portal -> Function App -> Settings -> Configuration -> Environment Variables
1. <b>storageAccountConnectionString__queueServiceUri</b> -> https://<STORAGE_ACCOUNT>.queue.core.windows.net/
2. <b>storageAccountConnectionString__credential</b> -> managedidentity
3. <b>QueueQueryName</b> -> <STORAGE_QUEUE_NAME_FOR_QUERIES>
4. <b>QueueProcessName</b> -> <STORAGE_QUEUE_NAME_FOR_PROCESSING>

<b>Data Collection Endpoint and Rule Setup for Log Analytics Ingest</b>:
1. Azure Portal -> Monitor -> Create Data Collection Endpoint
2. Azure Portal -> Log Analytics -> Table -> Create New Custom Table
3. Reference: [Tutorial: Send data to Azure Monitor Logs with Logs ingestion API (Azure portal)
](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/tutorial-logs-ingestion-portal)

<b>Azure Storage Setup</b>:
1. Create 1 container for data output files
   - <STORAGE_CONTAINER_NAME>
3. Create 4 queues for messages/jobs
   - <STORAGE_QUEUE_NAME_FOR_QUERIES>
   - <STORAGE_QUEUE_NAME_FOR_PROCESSING>
   - <STORAGE_QUEUE_NAME_FOR_QUERIES>-poison for failed messages
   - <STORAGE_QUEUE_NAME_FOR_PROCESSING>-poison for failed messages
5. Create 3 tables for logging (i.e. ingestlog, querylog, and processlog)
   - <STORAGE_TABLE_INGEST_LOG_NAME>
   - <STORAGE_TABLE_QUERY_LOG_NAME>
   - <STORAGE_TABLE_PROCESS_LOG_NAME>

<b>Queue Trigger Setup:</b>:
- To fix message encoding errors (default is base64), add "extensions": {"queues": {"messageEncoding": "none"}} to host.json
- Note: Failed messages/jobs are sent to <QUEUE_NAME>-poison

<b>API Management Setup:</b>
- Note: API management is used for interactive Swagger documenation
1. Create API Management Service -> Consumption Pricing Tier
2. Add API -> Function App
   - Function App: <YOUR_FUNCTION>
   - Display Name: Protected API Calls
   - Name: protected-api-calls
   - Suffix: api
3. Remove all operations besides POST
   - Edit POST operation 
      - Display name: azure_ingest_test_data
      - URL: POST /azure_ingest_test_data
   - Clone and Edit new POST operation 
      - Display name: azure_ingest_test_data
      - URL: POST /azure_ingest_test_data
   - Clone and Edit new POST operation 
      - Display name: azure_ingest_test_data
      - URL: POST /azure_ingest_test_data
   - Clone and Edit new POST operation 
      - Display name: azure_ingest_test_data
      - URL: POST /azure_ingest_test_data
   - Edit OpenAPI spec json operation ids to match above
4.. Add API -> Function App
   - Function App: <YOUR_FUNCTION>
   - Display Name: Public Docs
   - Name: public-docs
   - Suffix: public
5. Remove all operations besides GET
   - Settings -> uncheck 'subscription required'
   - Edit GET operation
      - Display name: Documentation
      - URL: GET /docs
   - Clone and Edit new GET operation
      - Display name: OpenAPI Schema
      - URL: GET /openapi.json
   - Edit OpenAPI spec json operation ids to match above
   - Test at https://<APIM_NAME>.azure-api.net/public/docs

<b>Optional Environment Variables (reduces number of params in requests)</b>:
- Setup via Azure Portal -> Function App -> Settings -> Configuration -> Environment Variables
1. <b>QueueURL</b> -> <STORAGE_QUEUE_URL>
2. <b>TableURL</b> -> <STORAGE_TABLE_URL>
3. <b>TableIngestName</b> -> <STORAGE_TABLE_INGEST_LOG_NAME>
4. <b>TableQueryName</b> -> <STORAGE_TABLE_QUERY_LOG_NAME>
5. <b>TableProcessName</b> -> <STORAGE_TABLE_PROCESS_LOG_NAME>

## Usage

<b>1. Execute HTTP trigger <b>azure_submit_queries()</b> or <b>azure_submit_query()</b> with query and connection parameters:</b>

- HTTP POST Request Body Example:

```json
{
    "subscription_id" : "XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX",
    "resource_group_name" : "XXXXXXXXXXXXXXXXXXXXXXX",
    "log_analytics_worksapce_name" : "XXXXXXXXXXXXXXXX",
    "log_analytics_workspace_id" : "XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX",
    "storage_blob_url" : "https://XXXXXXXXXXXXXXXXXXXXX.blob.core.windows.net/",
    "storage_blob_container_name" : "XXXXXXXXXXXXX",
    "table_names_and_columns" : { "XXXXXXXXXXXXXXX_CL": ["TimeGenerated","DataColumn1","DataColumn2","DataColumn3","DataColumn4","DataColumn5","DataColumn6","DataColumn7","DataColumn8","DataColumn9"]},
    "start_datetime" : "2024-03-19 00:00:00",
    "end_datetime" : "2024-03-20 00:00:00"
}
```

- HTTP Response Examples:
    - azure_submit_queries()

```json
{
    "query_uuid": "XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX",
    "split_status": "Success",
    "table_names": "XXXXXXXXXXX_CL",
    "start_datetime": "2024-04-04 00:00:00.000000",
    "end_datetime": "2024-04-10 00:00:00.000000",
    "number_of_messages_generated": 6,
    "number_of_messages_sent": 6,
    "total_row_count": 2010000,
    "runtime_seconds": 0.9,
    "split_datetime": "2024-04-12 14:06:41.688752"
}
```

- HTTP Response Examples:
    - azure_submit_query()

```json
{
    "query_uuid": "XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX",
    "submit_status": "Success",
    "table_names": "XXXXXXXXXXX_CL",
    "start_datetime": "2024-03-19 00:00:00.000000",
    "end_datetime": "2024-03-20 00:00:00.000000",
    "total_row_count": 23000000,
    "subqueries_generated": 95,
    "subqueries_sent_to_queue": 95,
    "runtime_seconds": 92.1,
    "submit_datetime": "2024-03-26 16:24:38.771336"
}
```

This query will be split into sub-queries and saved as messages in a queue, which will be automatically processed in parallel and sent to a storage account container. 

<b>2. Execute HTTP trigger <b>azure_get_status()</b> with query uuid:</b>

- HTTP POST Request Body Example:
  
```json
{
    "query_uuid" : "XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX"
}
```

- HTTP Response Example:

```json
{
    "query_uuid": "XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX",
    "query_partitions" : 1,
    "submit_status": "Success",
    "processing_status": "Partial",
    "percent_complete": 29.5,
    "runtime_since_submit_seconds": 463.6
    "estimated_time_remaining_seconds": 1107.9,
    "number_of_subqueries": 95,
    "number_of_subqueries_success": 28,
    "number_of_subqueries_failed": 0,
    "query_row_count": 23000000,
    "output_row_count": 6972002,
    "output_file_size": 2.05,
    "output_file_units" : "GB"
}
```

## Issues

1. Azure Function App stops processing sub-queries, queue trigger not processing messages in queue:
   - Manually restart Azure Function App in Azure Portal
   - Use Premium or Dedicated Plan

2. Submit Query function exceeds 10 min limit and fails
   - Use azure_submit_queries() function 
   - Reduce the datetime range of the query (recommend less than 100M records)
   - Decrease break_up_query_freq value in azure_submit_query()
   - Decrease parallel_process_break_up_query_freq value in azure_submit_queries()
   - Use Premium or Dedicated Plan with no time limit

## Changelog

2.0.0:
- changed to FastAPI in order to use Swager interactive docs

1.5.0:
- added azure_submit_queries() function for larger datetime ranges and parallel processing

1.4.0:
- refactored code and made pylint edits
- changed logging to % formatting from f-strings

1.3.1:
- Fixed UTC time zone bug
- Added estimated time remaining to get_status() response
- Added option to put storage queue and table params in env variables

1.3.0:
- Added pydantic input validation
- Added Open API yaml file for Azure API Management

1.2.0:
- Added get_status() azure function

1.1.0:
- Added logging to Azure Table Storage
- Added row count checks

1.0.0:
- Initial release
