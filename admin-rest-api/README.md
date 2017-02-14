# admin-rest-api

> This README describes the *admin-rest-api* component. Please refer
> to the top level [README](../README.md) for an overview of all components
>  documented in this repository.

# Message Hub Administration REST API

This REST API allows users of the
[IBM Message Hub service](http://www-03.ibm.com/software/products/en/ibm-message-hub)
to administer
[Kafka topics](#using-the-rest-api-to-administer-kafka-topics)
and [bridges](#using-the-rest-api-to-administer-bridges) associated with an
instance of the service. You can use this API to perform the following
operations:
  - [Create a Kafka topic](#creating-a-kafka-topic)
  - [List Kafka topics](#listing-kafka-topics)
  - [Delete a Kafka topic](#deleting-a-kafka-topic)
  - [Create a bridge](#creating-a--bridge)
  - [Get information about a bridge](#getting-information-about-a-bridge)
  - [List bridges](#listing-bridges)
  - [Update a bridge](#updating-a-bridge)
  - [Delete a bridge](#deleting-a-bridge)
  - [Pause a bridge](#pausing-a-bridge)
  - [Resume a bridge](#resuming-a-bridge)

The Admin REST API is also [documented using swagger](./admin-rest-api.yaml).

## Access control

Access to the API is protected using an API key. An API key must be specified
on each HTTP request to the API, using the `X-Auth-Token` HTTP header. API keys
can be obtained in a number of ways:
  - From the `VCAP_SERVICES` environment variable of an application bound to
    an instance of the Message Hub service. The API key is associated with a
    property named `api_key`
  - Selecting an instance of the Message Hub service from within the Bluemix
    User Interface and choosing the "Service Credentials" tab.
  - Using the `bluemix` command line tool and specifying the
    `service service-key` command.

In all cases the API key will be a 48 character string consisting of a random
mixture of upper-case letters, lower-case letters, and numbers.

If you do not specify an API key when making a request to the API, or you
specify an API key which is not valid then the request will fail with a HTTP
status code of 401 returned in the response.

## Locating the correct Administration API endpoint

The REST API is specific to the Bluemix region within which you are using the
Message Hub service. You can find the appropriate URL to use by examining the
`kafka_admin_url` property of the `VCAP_SERVICES` environment variable set when
you bind an application to an instance of the Message Hub service.

## Using the REST API to administer Kafka topics

### Creating a Kafka topic

You can create a Kafka topic by issuing a POST request to the `/admin/topics`
path. The body of the request must contain a JSON document, for example:

```
{
  "name": "ExampleTopic",
  "partitions": 2
}
```

Or a JSON document containing a `configs` object that specifies additional
properties:

```
{
  "name": "ExampleTopic2",
  "partitions": 2,
  "configs": {
    "retentionMs": 86400000
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

If the request to create a Kafka topic succeeds then HTTP status code 202
(Accepted) is returned. If the operation fails then a HTTP status code of
422 (Unprocessable Entity) is returned, and a
[JSON object](#information-returned-when-a-request-fails) containing
additional information about the failure is returned as the body of the
response.

#### Example

The REST endpoint for creating a Kafka topic can be exercised using the
following snippet of curl. You will need to supply your own API key and
specify the correct endpoint for the Bluemix region that you are using.

```
curl -v -H 'Content-Type: application/json' -H 'Accept: */*' \
    -H 'X-Auth-Token: yourapikeyhere' \
    -d '{ "name": "ExampleTopic", "partitions": 2 }' \
    https://admin-endpoint-goes-here/admin/topics
```

### Listing Kafka topics

You can list all of your Kafka topics by issuing a GET request to the
`/admin/topics` path. The response will contain a list of topics in
the following format:

```
[
  {
    "name": "topic1",
    "markedForDeletion": false,
    "partitions": 1,
    "retentionMs": 86400000
  },
  { "name": "topic2",
    "markedForDeletion": true,
    "partitions": 2,
    "retentionMs": 86400000
  }
]
```

A successful response will have HTTP status code 200 (OK) and contain an
array of JSON objects, where each object represents a Kafka topic and has the
following properties:

| Property name     | Description                                             |
|-------------------|---------------------------------------------------------|
| name              | The name of the Kafka topic.                            |
| markedForDeletion | `true` if the topic is in the process of being deleted. |
| partitions        | The number of partitions assigned to the Kafka topic.   |
| retentionsMs      | The retention period for messages on the topic (in ms). |

#### Example

The following curl command can be used to list all of your Kafka topics. Note
that you will need to specify your own API key as well as the correct endpoint
for the Bluemix region you are using:

```
curl -v -H -H 'Accept: application/json' \
    -H 'X-Auth-Token: yourapikeyhere' \
    https://admin-endpoint-goes-here/admin/topics/
```

### Deleting a Kafka topic

To delete a Kafka topic, issue a DELETE request to the `/admin/topics/TOPICNAME`
path (where TOPICNAME is the name of the Kafka topic that you want to delete).

A 202 (Accepted) status code is returned if the REST API accepts the delete
request or status code 422 (Unprocessable Entity) if the delete request is
rejected. If a delete request is rejected then the body of the HTTP response
will contain a [JSON object](#information-returned-when-a-request-fails) which
provides additional information about why the request was rejected.

Kafka deletes topics asynchronously. Deleted topics may still appear in the
response to a [list topics request](#listing-kafka-topics) for a short period
of time after the completion of a REST request to delete the topic.

#### Example

The following curl command deletes a topic called `MYTOPIC`. You will need to
substitute your own API key as well as the endpoint for the Bluemix region you
are using:

```
curl -v -X DELETE -H 'Content-Type: application/json' -H 'Accept: */*' \
    -H 'X-Auth-Token: yourapikeyhere' \
    https://admin-endpoint-goes-here/admin/topics/MYTOPIC
```

### Information returned when a request fails

If a request to the API fails then the response body that is returned will
contain a JSON object with more details about why the request failed. For
example:

```
{
  "errorCode": 403,
  "errorMessage": "Unauthorized"
}
```

The `error_code` property of this object contains a numeric error code that
can be used by programs that use the API and want to react differently
depending on the nature of the particular error condition. The `message`
property contains a English language description of the error condition.

## Using the REST API to administer bridges

:construction: The REST API methods for administering bridges are under active
development and may be subject to change.

### Creating a bridge

You can create a bridge by issuing a POST request to the `/admin/bridges`
path. The request must contain a JSON document in its body. The basic structure
of this JSON document is common across all of the types of bridge that you can
create, and is as follows:

```
{
  "name": "mybridge"
  "topic": "aKafkaTopic",
  "type": ...,
  "configuration": {
    ...
  }
}
```
The `name` property specifies a name to associate with the bridge. Many of the
other REST calls for working with bridges require you to refer to the bridge
by this name. Bridges can only be named using the following restricted set of
characters:
  - lowercase alphabet (a-z)
  - numbers (0-9)

Each bridge reads data from, or writes data to a Kafka topic. The `topic`
property is used to specify the name of the Kafka topic that the bridge will
transfer data to or from.

The value of the `type` property determines what kind of bridge is created.
The structure for the value of the `configuration` property is dependent on the
type of bridge being created.  The following types of bridge can be created:

| Value of `type` property | Bridge type           |
|--------------------------|-----------------------|
| `objectStorageOut`       | Object Storage bridge |

#### Creating an Object Storage bridge

The Object Storage bridge allows you to store data from Kafka messages in an
instance of the
[Object Storage service](https://console.ng.bluemix.net/catalog/services/object-storage/).
You can use the Object Storage bridge to:
  - Copy data from Kafka into long term storage so that the data is retained
    for a longer period of time than the Kafka topic's retention period.
  - Take periodic "snapshots" of the data in Kafka so that it can be analysed
    by an offline process.

To create an instance of the Object Storage bridge: specify a value of
`objectStorageOut` as the `type` property in the JSON document POSTed to
`/admin/bridges`. The value of the `configuration` property used for creating an
Object Storage bridge contains both properties that determine how the bridge
will use the Object Storage service.  Here's an example of a complete JSON
document for creating an Object Storage bridge:

```
{
  "name": "osbridge",
  "topic": "osbridgetopic",
  "type": "objectStorageOut",
  "configuration" : {
    "credentials" : {
      "authUrl" : "https://identity.open.softlayer.com",
      "region" : "dallas",
      "password" : "xxxxxxxxxxxxxxxx",
      "projectId" : "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
      "userId" : "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
    },
    "container" : "oscontainer",
    "uploadDurationThresholdSeconds" : 600,
    "uploadSizeThresholdKB" : 1024,
    "partitioning" : [ {
        "type" : "kafkaOffset"
      }
    ]
  }
}
```

The value of the `credentials` property is a JSON object which contains all of
the properties needed to connect to an instance of the Object Storage service.
You can obtain credentials from the Object Storage service in a number of
ways:
  - From the `VCAP_SERVICES` environment variable of an application bound to
    an instance of the Object Storage service.
  - Selecting an instance of the Object Storage service from within the Bluemix
    User Interface and choosing the "Service Credentials" tab.
  - Using the `bluemix` command line tool and specifying the
    `service key-create` command.

In most cases there is a 1:1 mapping between the names of credentials provided
by the Object Storage Service and the property names in the `credentials`
JSON object. There are however a couple of properties that may catch you out:
  - The Object Storage service `auth_url` property is converted to camel case
    (`authUrl`) when specified in the `credentials` object.
  - The Object Storage service credentials contain two properties with similar
    names: `userName` and `userId`. The `userId` property from the Object
    Storage service credentials should be used for the identically named
    property in the bridge's `credentials` object. It is easy to pick up the
    value from the Object Storage service's `userName` property by mistake - but
    if you do, the bridge will not be able to authenticate with the Object
    Storage service.

The `container` property of the bridge's `configuration` JSON object specifies
the Object Storage service container into which the bridge will write objects.
If this container does not already exist, the bridge will create it at the point
it is ready to write its first object.

You can use the `uploadDurationThresholdSeconds` and `uploadSizeThresholdKB`
properties to control how frequently the bridge uploads data to the Object
Storage Service. The `uploadDurationThresholdSeconds` configures an approximate
threshold (in seconds) after which any data read from Kafka will be uploaded to
the Object Storage service. `uploadSizeThresholdKB` operates in a similar way
but is based on the amount of data read from Kafka reaching a certain threshold.

When the Object Storage bridge uploads data to the Object Storage service it
will create one or more objects containing the data. The decision of how to
partition messages and name the resulting objects depends on how the bridge has
been configured. Currently the bridge supports two ways in which Kafka messages
can be partitioned into Object Storage objects:
  - By Kafka message offset.
  - By ISO 8601 date present in each Kafka message (note: this requires the
    Kafka messages to contain a valid JSON format object).

To create a bridge that partitions data based on Kafka message offset, specify
a `partitioning` array that contains an object with a `type` property set to
the value `kafkaOffset`. For example:

```
"partitioning" : [ {
    "type" : "kafkaOffset"
  }
]
```

Note that the example complete JSON document for creating a bridge, above,
creates a bridge which uses Kafka message offset-based partitioning.

To create a bridge which partitions based on an ISO 8601 data present in the
(JSON format) Kafka message data:
  - Specify an `inputFormat` property with the value `json` 
  - Specify the following:
    - a `partitioning` array that contains an object with a `type`
      property set to the value `dateIso8601`
    - a `propertyName` property set to the name of the JSON property,
      present in the Kafka messages, that
      contains the date information to use for partitioning.

For example the following JSON document will create a bridge which expects to
receive JSON format data from Kafka and will partition the data into Object
Storage objects. The partitioning is based on an ISO 8601 format date, which the
bridge is configured to look for as the value for the `timestamp` property
present in the JSON format Kafka message payloads:

```
{
  "name": "osbridge",
  "topic": "osbridgetopic",
  "type": "objectStorageOut",
  "configuration" : {
    "credentials" : {
      "authUrl" : "https://identity.open.softlayer.com",
      "region" : "dallas",
      "password" : "xxxxxxxxxxxxxxxx",
      "projectId" : "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
      "userId" : "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
    },
    "container" : "oscontainer",
    "inputFormat" : "json",
    "uploadDurationThresholdSeconds" : 600,
    "uploadSizeThresholdKB" : 1024,
    "partitioning" : [ {
        "propertyName" : "timestamp",
        "type" : "dateIso8601"
      }
    ]
  }
}
```

A HTTP status code 201 (Created) is returned when a request to create a bridge
completes successfully. If the operation fails then a HTTP status code of 422
(Unprocessable Entity) is returned, and a
[JSON object](#information-returned-when-a-request-fails) containing additional
information about the failure is returned as the body of the response.

##### Examples

The following curl command creates a bridge called `osbridge` which uses Kafka
offsets to partition data into Object Storage Service objects. Note that you
will need to substitute in your own API key and Admin REST endpoint:

```
curl -X POST -v -H 'Content-Type: application/json' -H 'Accept: */*' \
    -H 'X-Auth-Token: yourapikeyhere' \
    https://admin-endpoint-goes-here/admin/bridges \
    -d @- << REQUEST
{
  "name": "osbridge",
  "topic": "osbridgetopic",
  "type": "objectStorageOut",
  "configuration" : {
    "credentials" : {
      "authUrl" : "https://identity.open.softlayer.com",
      "region" : "dallas",
      "password" : "xxxxxxxxxxxxxxxx",
      "projectId" : "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
      "userId" : "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
    },
    "container" : "oscontainer",
    "uploadDurationThresholdSeconds" : 600,
    "uploadSizeThresholdKB" : 1024,
    "partitioning" : [ {
        "type" : "kafkaOffset"
      }
    ]
  }
}
REQUEST
```

### Getting information about a bridge

To get information about a specific instance of a bridge, issue a GET request
to the `/admin/bridges/BRIDGENAME` path (where BRIDGENAME is the name of the
bridge).

A 200 (OK) status code is returned if the REST API accepts the request or a
status code of 404 (Not Found) is returned if no bridge for the specified bridge
name.

If the request succeeds then the body of the response will contain a JSON
object in the following format:

```
{
  "configuration":{
    ...
  },
  "state":{
    "message":"The bridge is running.",
    "value":"running"
  }
}
```

The value of the `configuration` property returns the same JSON object as was
used to create the bridge (see [Creating a bridge](#creating-a-bridge)). However
the password supplied for the Object Storage Service will be replaced by a fixed
number of asterisk ('\*') characters.

The `status` property of the object is itself a JSON object which contains a
`message` property describing the status of the bridge and a `value` property.
`value` can have the following values:
  - paused - the bridge is paused (see the
    [API for pausing bridges](#pausing-a-bridge) section for more information).
  - running - the bridge is running.
  - unavailable - the state of the bridge cannot be determined at this time.

#### Example

The following curl command requests information about a bridge named `bridge1`.
You will need to substitute your own API key and Admin REST endpoint:

```
curl -v -H 'Content-Type: application/json' -H 'Accept: */*' \
     -H 'X-Auth-Token: yourapikeyhere' \
     https://admin-endpoint-goes-here/admin/bridges/bridge1 \
```

### Listing bridges

You can retrieve a list of all of your bridges by issuing a GET request to the
`/admin/bridges` path. The response will contain a JSON array of objects in
the same format as the response returned when [getting information about a
specific instance of a bridge](#getting-information-about-a-bridge).

#### Example

The following curl command requests information about all of the bridges that
you have currently defined. As before you will need to substitute your own
API key and Admin REST endpoint:

```
curl -v -H 'Content-Type: application/json' -H 'Accept: */*' \
     -H 'X-Auth-Token: yourapikeyhere' \
     https://admin-endpoint-goes-here/admin/bridges \
```

### Updating a bridge

To update a bridge's configuration issue a PUT request to the
`/admin/bridges/BRIDGENAME` path (where BRIDGENAME is the name of the bridge
that you want to update). The body of the update request should be a JSON
document that contains the new configuration for the bridge - in an identical
format to that used when [creating a bridge](#creating-a-bridge).

If the update request succeeds then the HTTP response will have 200 (OK) status
code. A status code of 404 (Not Found) will be returned if the bridge does not
exist, and a status code of 422 (Unprocessable Entity) for other errors
processing the request. If the update request fails then the body of the HTTP
response will contain a
[JSON object](#information-returned-when-a-request-fails) which provides
additional information about why the request was rejected.

It is not possible to change the values for the following properties, from those
specified when the bridge is created:
  - `name`
  - `topic`
  - `type`

#### Example

The following curl command updates a bridge named `osbridge`. Note that for
this request to succeed the name, topic, and type specified in the JSON
request body must match the values specified when the bridge was created.

```
curl -X PUT -v -H 'Content-Type: application/json' -H 'Accept: */*' \
    -H 'X-Auth-Token: yourapikeyhere' \
    https://admin-endpoint-goes-here/admin/bridges/osbridge \
    -d @- << REQUEST
{
  "name": "osbridge",
  "topic": "osbridgetopic",
  "type": "objectStorageOut",
  "configuration" : {
    "credentials" : {
      "authUrl" : "https://identity.open.softlayer.com",
      "region" : "dallas",
      "password" : "xxxxxxxxxxxxxxxx",
      "projectId" : "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
      "userId" : "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
    },
    "container" : "oscontainer",
    "uploadDurationThresholdSeconds" : 600,
    "uploadSizeThresholdKB" : 1024,
    "partitioning" : [ {
        "type" : "kafkaOffset"
      }
    ]
  }
}
REQUEST
```

### Deleting a bridge

You can delete a bridge by issuing a DELETE request to the
`/admin/bridges/BRIDGENAME` path (where BRIDGENAME is the name of the bridge
that you want to delete). The request is not expected to have a body.

The status code of the response will be 204 (No Content) if the bridge has been
deleted or 404 (Not Found) if there is no bridge that matches the bridge name
specified in the URL of the request.

#### Example

The following curl command deletes a bridge named `bridge1`. Substitute your
own API key and REST API endpoint:

```
curl -X DELETE -v -H 'Content-Type: application/json' \
     -H 'X-Auth-Token: yourapikeyhere' \
     https://admin-endpoint-goes-here/admin/bridges/bridge1
```

### Pausing a bridge

Bridges can be paused to suspend their processing. To pause a bridge issue a
POST to the `/admin/bridges/BRIDGENAME/pause` path (where BRIDGENAME is the
name of the bridge that you want to pause). The request is not expected to have
a body.

The status code of the response will be 204 (No Content) if the bridge has been
paused or 404 (Not Found) if there is no bridge that matches the bridge name
specified in the URL of the request. Pausing a bridge which is already paused
returns the successful 204 (No Content) status code.

#### Example

The following curl command will pause a bridge named `bridge1`. Remember to
substitute your own API key and REST API endpoint:

```
curl -X POST -v -H 'Content-Type: application/json' \
     -H 'X-Auth-Token: yourapikeyhere' \
     https://admin-endpoint-goes-here/admin/bridges/bridge1/pause
```

### Resuming a bridge

A bridge which has been paused can subsequently be resumed. Doing so will
cause the bridge to resume its processing. To resume a bridge issue a POST
request to the `/admin/bridges/BRIDGENAME/resume` path (where BRIDGENAME is the
name of the bridge that you want to resume). The request is not expected to have
a body.

The status code of the response will be 204 (No Content) if the bridge has been
resumed or 404 (Not Found) if there is no bridge that matches the bridge name
specified in the URL of the request. Resuming a bridge which is already running
returns the successful 204 (No Content) status code.

#### Example

The following curl command will resume a bridge named `bridge1`. You will need
to substitute your own API key and REST API endpoint.

```
curl -X POST -v -H 'Content-Type: application/json' \
     -H 'X-Auth-Token: yourapikeyhere' \
     https://admin-endpoint-goes-here/admin/bridges/bridge1/resume
```

## License
The project is licensed under the Eclipse Public License - v 1.0 (see the
[LICENSE](../LICENSE) file in the root directory of the repository).
