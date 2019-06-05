# admin-rest-api

> This README describes the *admin-rest-api* component. Please refer
> to the top level [README](../README.md) for an overview of all components
>  documented in this repository.

# Event Streams Administration REST API

This REST API allows users of the
[IBM Event Streams service](https://cloud.ibm.com/docs/services/EventStreams/index.html)
to administer
[Kafka topics](#using-the-rest-api-to-administer-kafka-topics)
associated with an instance of the service. You can use this API to perform the following
operations:
  - [Create a Kafka topic](#creating-a-kafka-topic)
  - [List Kafka topics](#listing-kafka-topics)
  - [Get a Kafka topic](#getting-a-kafka-topic)
  - [Delete a Kafka topic](#deleting-a-kafka-topic)
  - [Update a Kafka topic configuration](#updating-kafka-topics-configuration)
  
The Admin REST API is also [documented using swagger](./admin-rest-api.yaml).

Note: This Admin REST API works with both Enterprise plan and Standard plan. The Admin REST API that works with Classic plan is located in [here](../admin-rest-api-classic-plan-only). They are compatible in topic management capabilities.

## Access control
All requests support below authorization methods:
 * Basic authorization with user and password. (
  For both standard and enterprise plan, user is 'token', password is the API key from `ibmcloud resource service-keys` for the service instance.)
 * Bearer authorization with bearer token. (This token can be either API key or JWT token obtained from IAM upon login to IBM Cloud. Use `ibmcloud iam oauth-tokens` to retrieve the token after `ibmcloud login`)
 * `X-Auth-Token` header to be set to the API key. This header is deprecated.

##  Administration API endpoint
Administration API endpoint is the `kafka_admin_url` property in the service key for the service instance. This command can be used to retrieve this property.
```bash
$ibmcloud resource service-key "${service_instance_key_name}" --output json > jq -r '.[]|.credentials.kafka_admin_url'
```

In addition, the `Content-type` header has to be set to `application/json`.

Common HTTP status codes:
- 200: Request succeeded.
- 202: Request was accepted.
- 400: Invalid request JSON.
- 401: The authentication header is not set or provided information is not valid.
- 403: Not authorized to perform the operation. Usually it means the API key used is missing a certain role. More details on what role can perform what operation refers to this [document](https://cloud.ibm.com/docs/services/EventStreams?topic=eventstreams-security).
- 404: Unable to find the topic with topic name given by user.
- 422: Semantically invalid request.
- 503: An error occurred handling the request.

Error responses carry a JSON body like the following:
```json
{"error_code":50301,"message":"Unknown Kafka Error", "incident_id": "17afe715-0ff5-4c49-9acc-a4204244a331"}
```
Error codes are of the format `HHHKK` where `HHH` is the HTTP Status Code and `KK` is the Kafka protocol error.  

For E2E debugging purposes, the transaction ID of every request is returned in the HTTP header `X-Global-Transaction-Id`.
If the header is set on the request, it will be honored. If not, it will be generated.
In the event of a non-200 error return code, the transaction ID is also returned in the JSON error response as `incident_id`.


## Using the REST API to administer Kafka topics

### Creating a Kafka topic

You can create a Kafka topic by issuing a POST request to the `/admin/topics`
path. The body of the request must contain a JSON document, for example:

```json
{
    "name": "topicname",
    "partitions": 1,
    "configs": {
        "retentionMs": 86400000,
        "cleanupPolicy": "delete"
    }
}
```

The JSON document must contain a `name` attribute, specifying the name of the
Kafka topic to create. The JSON may also specify the number of partitions to
assign to the topic (using the `partitions` property). If the number of
partitions is not specified then the topic will be created with a single
partition.

You can also specify an optional `configs` object within the request. This
allows the specification of the `retentionMs` property which controls how long
(in milliseconds) Kafka will retain messages published to the topic. After this
time elapses the messages will automatically be deleted to free space. Note
that the value of the `retentionMs` property must be specified in a whole number
of hours (e.g. multiples of 3600000).

Expected HTTP status codes:
- 202: Topic creation request was accepted.
- 400: Invalid request JSON.
- 403: Not authorized to create topic.
- 422: Semantically invalid request.


If the request to create a Kafka topic succeeds then HTTP status code 202
(Accepted) is returned. If the operation fails then a HTTP status code of
422 (Unprocessable Entity) is returned, and a
[JSON object](#information-returned-when-a-request-fails) containing
additional information about the failure is returned as the body of the
response.

#### Example

The REST endpoint for creating a Kafka topic can be exercised using the
following snippet of curl. You will need to supply your own API key or token and specify the correct endpoint for ADMIN API.

```bash
curl -i -X POST -H 'Accept: application/json' -H 'Content-Type: application/json' -H 'Authorization: Bearer ${TOKEN}' --data '{ "name": "newtopic", "partitions": 1}' ${ADMIN_URL}/admin/topics
```

### Listing Kafka topics

You can list all of your Kafka topics by issuing a GET request to the
`/admin/topics` path. 

Expected status codes:
  - 200: the topic list is returned as JSON in the following format:
```json
[
  {
    "name": "topic1",
    "partitions": 1,
    "retentionMs": 86400000,
    "cleanupPolicy": "delete"
  },
  { "name": "topic2",
    "partitions": 2,
    "retentionMs": 86400000,
    "cleanupPolicy": "delete"
  }
]
```

A successful response will have HTTP status code 200 (OK) and contain an
array of JSON objects, where each object represents a Kafka topic and has the
following properties:

| Property name     | Description                                             |
|-------------------|---------------------------------------------------------|
| name              | The name of the Kafka topic.                            |
| partitions        | The number of partitions assigned to the Kafka topic.   |
| retentionsMs      | The retention period for messages on the topic (in ms). |
| cleanupPolicy     | The cleanup policy of the Kafka topic.                  |

#### Example

The following curl command can be used to list all of your Kafka topics.

```bash
curl -i -X GET -H 'Accept: application/json' -H 'Authorization: Bearer ${TOKEN}' ${ADMIN_URL}/admin/topics
```

### Deleting a Kafka topic

To delete a Kafka topic, issue a DELETE request to the `/admin/topics/TOPICNAME`
path (where TOPICNAME is the name of the Kafka topic that you want to delete).

Expected return codes:
- 202: Topic deletion request was accepted.
- 403: Not authorized to delete topic.
- 404: Topic does not exist.
  
A 202 (Accepted) status code is returned if the REST API accepts the delete
request or status code 422 (Unprocessable Entity) if the delete request is
rejected. If a delete request is rejected then the body of the HTTP response
will contain a [JSON object](#information-returned-when-a-request-fails) which
provides additional information about why the request was rejected.

Kafka deletes topics asynchronously. Deleted topics may still appear in the
response to a [list topics request](#listing-kafka-topics) for a short period
of time after the completion of a REST request to delete the topic.

#### Example

The following curl command deletes a topic called `MYTOPIC`
```bash
curl -i -H 'Content-Type: application/json' -X DELETE -H 'Authorization: Bearer ${TOKEN}' ${ADMIN_URL}/admin/topics/MYTOPIC
```

### Getting a Kafka topic

To get a Kafka topic detail information, issue a GET request to the `/admin/topics/TOPICNAME`
path (where TOPICNAME is the name of the Kafka topic that you want to get).  

Expected status codes
  - 200: Retrieve topic details successfully in following format:
```json
{
  "name": "MYTOPIC",
  "partitions": 1,
  "replicationFactor": 3,
  "retentionMs": 86400000,
  "cleanupPolicy": "delete",
  "configs": {
    "cleanup.policy": "delete",
    "min.insync.replicas": "2",
    "retention.bytes": "1073741824",
    "retention.ms": "86400000",
    "segment.bytes": "536870912"
  },
  "replicaAssignments": [
    {
      "id": 0,
      "brokers": {
        "replicas": [
          3,
          2,
          4
        ]
      }
    }
  ]
}
```
#### Example

The following curl command gets a topic called `MYTOPIC`:
```bash
curl -i -X GET -H 'Content-Type: application/json' -H 'Authorization: Bearer ${TOKEN}' ${ADMIN_URL}/admin/topics/MYTOPIC
```

### Updating Kafka topic's configuration

To increase a topic's partition number or to update a topic's configuration, issue an
`PATCH` request to `/admin/topics/{topic}` with the following body:
```json
{
  "new_total_partition_count": 4,
  "configs": [
    {
      "name": "cleanup.policy",
      "value": "compact"
    }
  ]
}
```
Supported configuration keys are 'cleanup.policy', 'retention.ms', 'retention.bytes', 'segment.bytes', 'segment.ms', 'segment.index.bytes'.
And partition number can only be increased, not decreased.

Expected status codes
  - 202: Update topic request was accepted.
  - 400: Invalid request JSON/number of partitions is invalid.
  - 404: Topic specified does not exist.
  - 422: Semantically invalid request.

#### Example
The following curl command updates a topic called `MYTOPIC`, set its `partitions` to 4 and its `cleanup.policy` to be `compact`.
```bash
curl -i -X PATCH -H 'Content-Type: application/json' -H 'Authorization: Bearer ${TOKEN}' --data '{"new_total_partition_count": 4,"configs":[{"name":"cleanup.policy","value":"compact"}]}' ${ADMIN_URL}/admin/topics/MYTOPIC
```