# Google Cloud Logging to BigQuery via Pub/Sub (with SMT)

This guide details the end-to-end setup for reliably exporting Google Cloud Logging data to a BigQuery table using Pub/Sub. It includes configuring a **Single Message Transform (SMT) User-Defined Function (UDF)** to handle schema incompatibilities (e.g., deeply nested JSON payloads) and setting up a **Dead Letter Queue (DLQ)** for production-grade error handling.

---

## 🏗 Architecture Overview

1.  **Cloud Logging**: Captures system/application logs.
2.  **Log Router Sink**: Publishes matching logs to a **Pub/Sub Topic**.
3.  **Pub/Sub BigQuery Subscription**: Consumes the messages.
4.  **Inline SMT UDF**: Intercepts the message, stringifying nested JSON fields to prevent BigQuery schema mismatch errors.
5.  **BigQuery Table**: Successfully transformed messages are streamed directly into storage.
6.  **Dead Letter Topic**: Messages that fail transformation or ingestion are routed here for inspection.

---

## 🚀 Step 1: Create BigQuery Resources

Create a dataset in BigQuery using `bq` command in Cloud Shell to host the table for your logs.
```bash
bq --location=asia-southeast3 mk --dataset your_project:cloud_logging
```

Then create a table with appropriate schema. The following example uses the native `JSON` data type to handle the varied structure of Cloud Logging payloads flexibly.
```bash

# 1. Create the schema file
cat <<EOF > schema.json
[
  {"name": "subscription_name", "type": "STRING"},
  {"name": "message_id", "type": "STRING"},
  {"name": "publish_time", "type": "TIMESTAMP"},
  {"name": "attributes", "type": "JSON"},
  {"name": "insertId", "type": "STRING"},
  {"name": "logName", "type": "STRING"},
  {"name": "timestamp", "type": "TIMESTAMP"},
  {"name": "receiveTimestamp", "type": "TIMESTAMP"},
  {"name": "severity", "type": "STRING"},
  {"name": "textPayload", "type": "STRING"},
  {"name": "jsonPayload", "type": "JSON"},
  {"name": "protoPayload", "type": "JSON"},
  {"name": "resource", "type": "RECORD", "fields": [
    {"name": "type", "type": "STRING"},
    {"name": "labels", "type": "JSON"}
  ]},
  {"name": "labels", "type": "JSON"},
  {"name": "httpRequest", "type": "RECORD", "fields": [
    {"name": "requestMethod", "type": "STRING"},
    {"name": "requestUrl", "type": "STRING"},
    {"name": "requestSize", "type": "INTEGER"},
    {"name": "status", "type": "INTEGER"},
    {"name": "responseSize", "type": "INTEGER"},
    {"name": "userAgent", "type": "STRING"},
    {"name": "remoteIp", "type": "STRING"},
    {"name": "serverIp", "type": "STRING"},
    {"name": "referer", "type": "STRING"},
    {"name": "latency", "type": "STRING"},
    {"name": "cacheLookup", "type": "BOOLEAN"},
    {"name": "cacheHit", "type": "BOOLEAN"},
    {"name": "cacheValidatedWithOriginServer", "type": "BOOLEAN"},
    {"name": "cacheFillBytes", "type": "INTEGER"},
    {"name": "protocol", "type": "STRING"}
  ]},
  {"name": "operation", "type": "RECORD", "fields": [
    {"name": "id", "type": "STRING"},
    {"name": "producer", "type": "STRING"},
    {"name": "first", "type": "BOOLEAN"},
    {"name": "last", "type": "BOOLEAN"}
  ]},
  {"name": "trace", "type": "STRING"},
  {"name": "spanId", "type": "STRING"},
  {"name": "traceSampled", "type": "BOOLEAN"},
  {"name": "sourceLocation", "type": "RECORD", "fields": [
    {"name": "file", "type": "STRING"},
    {"name": "line", "type": "INTEGER"},
    {"name": "function", "type": "STRING"}
  ]}
]
EOF

# 2. Run the bq mk command using the file
bq mk \
--table \
--schema ./schema.json \
--time_partitioning_field timestamp \   ## adjust as needed
--time_partitioning_type DAY \          ## adjust as needed
--clustering_fields logName,severity \  ## adjust as needed
your_project:cloud_logging.unified_cloud_logs
```

As an alternative to the `bq` command on Cloud Shell, you may also run the following Data Definition Language (DDL) statement in BigQuery Studio to achieve the same result.
```sql
CREATE SCHEMA `your_project.cloud_logging`
  OPTIONS (
    location = 'asia-southeast3'
    );
```
```sql
CREATE OR REPLACE TABLE `your_project.cloud_logging.unified_cloud_logs` (
  -- ==========================================
 -- Pub/Sub Message Metadata
 -- ==========================================
 subscription_name STRING,
 message_id STRING,
 publish_time TIMESTAMP,
 attributes JSON, 
  -- ==========================================
 -- Cloud Logging: Core Identity & Routing
 -- ==========================================
 insertId STRING,
 logName STRING,
 timestamp TIMESTAMP,
 receiveTimestamp TIMESTAMP,
 severity STRING,
  -- ==========================================
 -- Cloud Logging: Flexible Payloads
 -- ==========================================
 textPayload STRING,
 jsonPayload JSON,
 protoPayload JSON,
  -- ==========================================
 -- Cloud Logging: Monitored Resource
 -- ==========================================
 resource STRUCT<
   type STRING,
   labels JSON
 >,
  -- ==========================================
 -- Cloud Logging: User/System defined labels
 -- ==========================================
 labels JSON,
  -- ==========================================
 -- Cloud Logging: HTTP Request Context
 -- ==========================================
 httpRequest STRUCT<
   requestMethod STRING,
   requestUrl STRING,
   requestSize INT64,
   status INT64,
   responseSize INT64,
   userAgent STRING,
   remoteIp STRING,
   serverIp STRING,
   referer STRING,
   latency STRING,
   cacheLookup BOOLEAN,
   cacheHit BOOLEAN,
   cacheValidatedWithOriginServer BOOLEAN,
   cacheFillBytes INT64,
   protocol STRING
 >,
  -- ==========================================
 -- Cloud Logging: Grouping/Trace Information
 -- ==========================================
 operation STRUCT<
   id STRING,
   producer STRING,
   first BOOLEAN,
   last BOOLEAN
 >,
 trace STRING,
 spanId STRING,
 traceSampled BOOLEAN,
  -- ==========================================
 -- Cloud Logging: Source Code Context
 -- ==========================================
 sourceLocation STRUCT<
   file STRING,
   line INT64,
   function STRING
 >
)
PARTITION BY DATE(timestamp)  --adjust as needed
CLUSTER BY logName, severity; --adjust as needed
```

---

## 📩 Step 2: Create Pub/Sub Resources

### 2.1 Create Topics and DLQ
Create the main ingestion topic, the dead-letter topic, and a recovery subscription using the `gcloud` CLI:

```bash
# Create the main log topic
gcloud pubsub topics create cloud-logs-ingestion-topic

# Create the dead-letter topic
gcloud pubsub topics create cloud-logs-dlq-topic

# Create a pull subscription on the DLQ topic for manual inspection
gcloud pubsub subscriptions create cloud-logs-dlq-sub \
    --topic=cloud-logs-dlq-topic
```

### 2.2 Configure the BigQuery Subscription & UDF

Grant BigQuery Data Editor on the dataset
```bash
# 1. Get the Project Number and Project ID
PROJECT_NUMBER=$(gcloud projects describe $(gcloud config get-value project) --format="value(projectNumber)")
PROJECT_ID=$(gcloud config get-value project)

# 2. Define the Pub/Sub Service Account name
PUBSUB_SERVICE_ACCOUNT="service-${PROJECT_NUMBER}@gcp-sa-pubsub.iam.gserviceaccount.com"

echo "Granting permissions for: $PUBSUB_SERVICE_ACCOUNT"
# 3. Grant BigQuery Data Editor to the service account
gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:$PUBSUB_SERVICE_ACCOUNT" \
    --role="roles/bigquery.dataEditor"
```

Navigate to **Pub/Sub > Subscriptions** in the Google Cloud Console and configure the following:

| Setting | Value |
| :--- | :--- |
| **Subscription ID** | `cloud-logs-bq-sub` |
| **Topic** | `cloud-logs-ingestion-topic` |
| **Delivery Type** | Write to BigQuery |
| **Target Table** | `unified_cloud_logs` |
| **Use Table Schema** | **Check** (Use Table Schema) |
| **Metadata** | **Check** "Write metadata" |
| **Fields** | **Uncheck** "Drop unknown fields" |

#### Payload Transformation (Inline UDF)
Expand **Payload transformation**, select **Inline**, set the function name to `processCloudLogs`, and paste the following:

```javascript
function processCloudLogs(message, metadata) {
  try {
    const data = JSON.parse(message.data);

    // Helper function to safely stringify objects to avoid BQ schema drift
    function stringifyIfObject(parentObj, fieldName) {
      if (parentObj && parentObj[fieldName] && typeof parentObj[fieldName] === 'object') {
        parentObj[fieldName] = JSON.stringify(parentObj[fieldName]);
      }
    }

    stringifyIfObject(data, 'jsonPayload');
    stringifyIfObject(data, 'protoPayload');
    stringifyIfObject(data, 'labels');

    if (data.resource && typeof data.resource === 'object') {
      stringifyIfObject(data.resource, 'labels');
    }

    message.data = JSON.stringify(data);
    return message;

  } catch (error) {
    message.attributes = message.attributes || {};
    message.attributes["udf_error"] = error.toString();
    return message;
  }
}
```

> **Note:** Under **Dead lettering**, select `cloud-logs-dlq-topic` and set **Maximum delivery attempts** to 5.

---

## 🪵 Step 3: Create the Log Router Sink

Configure Cloud Logging to push relevant logs to your Pub/Sub topic.

```bash
gcloud logging sinks create cloud-logging-pubsub-log-sink \
    pubsub.googleapis.com/projects/cloud-logging-bq-demo/topics/cloud-logs-ingestion-topic \
    --log-filter="severity >= INFO" \
    --description="Routes INFO and above logs to Pub/Sub for BQ ingestion"
```

---

## 🔐 Step 4: Production-Ready IAM Configuration

Ensure the following permissions are granted for the pipeline to flow securely:

* **Log Router → Pub/Sub**: Grant the Log Sink Service Account the `Pub/Sub Publisher` role on the ingestion topic.
* **Pub/Sub → BigQuery**: Grant the Pub/Sub Service Account (`service-PROJECT_NUMBER@gcp-sa-pubsub.iam.gserviceaccount.com`) the `BigQuery Data Editor` role on the dataset.
* **Pub/Sub → DLQ**: Grant the Pub/Sub Service Account `Pub/Sub Publisher` on the DLQ topic and `Pub/Sub Subscriber` on the BQ subscription.

```bash
# 1. Get the Service Account name from the sink you just created
SINK_SERVICE_ACCOUNT=$(gcloud logging sinks describe cloud-logging-pubsub-log-sink \
    --format="value(writerIdentity)")

# 2. Grant that Service Account the Publisher role on your topic
gcloud pubsub topics add-iam-policy-binding cloud-logs-ingestion-topic \
    --member=$SINK_SERVICE_ACCOUNT \
    --role="roles/pubsub.publisher"
```

```bash
# 1. Get the Project Number and Project ID
PROJECT_NUMBER=$(gcloud projects describe $(gcloud config get-value project) --format="value(projectNumber)")
PROJECT_ID=$(gcloud config get-value project)

# 2. Define the Pub/Sub Service Account name
PUBSUB_SERVICE_ACCOUNT="service-${PROJECT_NUMBER}@gcp-sa-pubsub.iam.gserviceaccount.com"

echo "Granting permissions for: $PUBSUB_SERVICE_ACCOUNT"


# 4. Grant Pub/Sub Publisher on the Dead Letter Topic
gcloud pubsub topics add-iam-policy-binding cloud-logs-dlq-topic \
    --member="serviceAccount:$PUBSUB_SERVICE_ACCOUNT" \
    --role="roles/pubsub.publisher"

# 5. Grant Pub/Sub Subscriber on the BigQuery Subscription
gcloud pubsub subscriptions add-iam-policy-binding cloud-logs-bq-sub \
    --member="serviceAccount:$PUBSUB_SERVICE_ACCOUNT" \
    --role="roles/pubsub.subscriber"
```

To test this, you may manually insert log entries into Google Cloud Logging from the command line interface using Cloud Shell
```bash
gcloud logging write test-log '{"test_field": "trigger", "status": 1.0}' \
    --payload-type=json \
    --severity=INFO
```
---

## 📈 Step 5: Monitoring and Maintenance

1.  **Set up Alerts on the DLQ**: Create a Cloud Monitoring alert for the metric `pubsub.googleapis.com/subscription/num_undelivered_messages` for your `cloud-logs-dlq-sub`.
2.  **Monitor UDF Latency**: Track the `subscription/message_transform_latencies` metric to ensure your inline JavaScript executes under 500ms.
3.  **Review BigQuery Partitioning**: The DDL includes `PARTITION BY DATE(timestamp)`. Ensure your queries leverage this field in the `WHERE` clause to minimize costs.

