---
title: Google BigQuery
permalink: /config/databases/google-bigquery
---

## Prerequisites

<!-- prettier-ignore-start -->
[[info | ]]
| In order to connect Google BigQuery to Cube.js, you need to provide service
| account credentials. Cube.js requires the service account to have **BigQuery
| Data Viewer** and **BigQuery Job User** roles enabled. You can learn more
| about acquiring Google BigQuery credentials [here][bq-docs-getting-started].
<!-- prettier-ignore-end -->

- [The Google Cloud Project ID][google-cloud-docs-projects] for the
  [BigQuery][bq] project
- [A set of Google Cloud service credentials][google-support-create-svc-account]
  [which allow access][bq-docs-getting-started] to the [BigQuery][bq] project
- [The Google Cloud region][bq-docs-regional-locations] for the [BigQuery][bq]
  project

## Setup

### Manual

Add the following to a `.env` file in your Cube.js project:

```bash
CUBEJS_DB_TYPE=bigquery
CUBEJS_DB_BQ_PROJECT_ID=my-bigquery-project-12345
CUBEJS_DB_BQ_KEY_FILE=/path/to/my/keyfile.json
CUBEJS_DB_BQ_LOCATION=us-central1
```

You could also encode the key file using Base64 and set the result to
`CUBEJS_DB_BQ_CREDENTIALS`:

```dotenv
CUBEJS_DB_BQ_CREDENTIALS=$(cat /path/to/my/keyfile.json | base64)
```

## Environment Variables

| Environment Variable           | Description                                                      | Possible Values                                                         | Required |
| ------------------------------ | ---------------------------------------------------------------- | ----------------------------------------------------------------------- | :------: |
| `CUBEJS_DB_BQ_PROJECT_ID`      | The Google BigQuery project ID to connect to                     | A valid Google BigQuery Project ID                                      |    ✅    |
| `CUBEJS_DB_BQ_KEY_FILE`        | The path to a JSON key file for connecting to Google BigQuery    | A valid Google BigQuery JSON key file                                   |    ✅    |
| `CUBEJS_DB_BQ_CREDENTIALS`     | A Base64 encoded JSON key file for connecting to Google BigQuery | A valid Google BigQuery JSON key file encoded as a Base64 string        |    ❌    |
| `CUBEJS_DB_BQ_LOCATION`        | The Google BigQuery dataset location to connect to               | [A valid Google BigQuery regional location][bq-docs-regional-locations] |    ✅    |
| `CUBEJS_DB_EXPORT_BUCKET`      | The name of a bucket in cloud storage                            | A valid bucket name from cloud storage                                  |    ❌    |
| `CUBEJS_DB_EXPORT_BUCKET_TYPE` | The cloud provider where the bucket is hosted                    | `gcs`                                                                   |    ❌    |

## SSL

Cube.js does not require any additional configuration to enable SSL as Google
BigQuery connections are made over HTTPS.

## Export bucket

### Google Cloud Storage

<!-- prettier-ignore-start -->
[[warning |]]
| BigQuery only supports using Google Cloud Storage for export buckets.
<!-- prettier-ignore-end -->

For [improved pre-aggregation performance with large
datasets][ref-caching-large-preaggs], enable export bucket functionality by
configuring Cube.js with the following environment variables:

<!-- prettier-ignore-start -->
[[info |]]
| When using an export bucket, remember to assign the **Storage Object Admin**
| role to your BigQuery credentials (`CUBEJS_DB_EXPORT_GCS_CREDENTIALS`).
<!-- prettier-ignore-end -->

```dotenv
CUBEJS_DB_EXPORT_BUCKET=export_data_58148478376
CUBEJS_DB_EXPORT_BUCKET_TYPE=gcp
CUBEJS_DB_EXPORT_GCS_CREDENTIALS=<BASE64_ENCODED_SERVICE_CREDENTIALS_JSON>
```

[bq]: https://cloud.google.com/bigquery
[bq-docs-getting-started]:
  https://cloud.google.com/docs/authentication/getting-started
[bq-docs-credentials]:
  https://console.cloud.google.com/apis/credentials/serviceaccountkey
[bq-docs-regional-locations]:
  https://cloud.google.com/bigquery/docs/locations#regional-locations
[google-cloud-docs-projects]:
  https://cloud.google.com/resource-manager/docs/creating-managing-projects#before_you_begin
[google-support-create-svc-account]:
  https://support.google.com/a/answer/7378726?hl=en
[ref-caching-large-preaggs]: /using-pre-aggregations#large-pre-aggregations
[ref-env-var]: /reference/environment-variables#database-connection
