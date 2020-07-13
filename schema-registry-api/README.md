# Schema Registry REST API
> This README describes the *schema-registry-rest-api* component. Please refer
> to the top level [README](../README.md) for an overview of all components
>  documented in this repository.

The REST API offers four main capabilities:

1. Creating, reading, and deleting schemas.
2. Creating, reading, and deleting individual versions of a schema.
3. Reading and updating the global compatibility rule for the registry.
4. Creating, reading, updating, and deleting compatibility rules that apply to individual schemas.

### Authentication
As described above, you can authenticate to the schema registry using an API key. This is supplied as the password portion of a HTTP basic authentication header. Set the username portion of this header to the word “token”. It is also possible to grant a bearer token for a system ID or user and supply this as a credential. To do this specify an HTTP header in the format: “Authorization: Bearer $TOKEN” (where $TOKEN is the bearer token).
For example:

```
curl –H "Authorization: Bearer $TOKEN" ...
```

### Errors
If an error condition is encountered, the schema registry will return a non-2XX range HTTP status code. The body of the response will contain a JSON object in the following form:

```
{
    "error_code":404,
    "message":"No artifact with id 'my-schema' could be found."
}
```

The properties of the error JSON object are as follows:

Property name | Description
--- | ---
error_code | The HTTP status code of the response.
message | A description of the cause of the problem.
incident | This field is only included if the error is a result of a problem with the schema registry. This value can be used by IBM service to correlate a request to diagnostic information captured by the registry.

### Create a schema
This endpoint is used to store a schema in the registry. The schema data is sent as the body of the POST request. An ID for the schema can be included using the ‘X-Registry-ArtifactId' request header. If this header is not present in the request, an ID will be generated. The content type header must be set to “application/json”.

Example curl request:

```
curl -u token:$APIKEY -H 'Content-Type: application/json' -H 'X-Registry-ArtifactId: my-schema' $URL/artifacts -d '{"type":"record","name":"Citizen","fields":[{"name": "firstName","type":"string"},{"name":"lastName","type":"string"},{"name":"age","type":"int"},{"name":"phoneNumber","type":"string"}]}'
```

Example response:

```
{"id":"my-schema","type":"AVRO","version":1,"createdBy":"","createdOn":1579267788258,"modifiedBy":"","modifiedOn":1579267788258,"globalId":75}
```

Creating a schema requires at least both:

- Reader role access to the Event Streams cluster resource type.
- Manager role access to the schema resource that matches the schema being created.

### List schemas
You can generate a list of the IDs of all of the schemas stored in the registry by making a GET request to the /artifacts endpoint.

Example curl request:

```
curl -u token:$APIKEY $URL/artifacts
```

Example response:

```
["my-schema"]
```

Listing schemas requires at least:

- Reader role access to the Event Streams cluster resource type.

### Delete a schema
Schemas are deleted from the registry by issuing a DELETE request to the /artifacts/{schema-id} endpoint (where {schema-id} is the ID of the schema). If successful, an empty response and a status code of 204 (no content) is returned.

Example curl request:

```
curl -u token:$APIKEY –X DELETE $URL/artifacts/my-schema
```

Deleting a schema requires at least both:

- Reader role access to the Event Streams cluster resource type.
- Manager role access to the schema resource that matches the schema being deleted.

### Create a new version of a schema
To create a new version of a schema, make a POST request to the /artifacts/{schema-id}/versions endpoint, (where {schema-id} is the ID of the schema). The body of the request must contain the new version of the schema.

If the request is successful, the new schema version is created as the new latest version of the schema, with an appropriate version number, and a response with status code 200 (OK) and a payload containing metadata describing the new version (including the version number), is returned.

Example curl request:

```
curl -u token:$APIKEY -H 'Content-Type: application/json' $URL/artifacts/my-schema/versions -d '{"type":"record","name":"Citizen","fields":[{"name": "firstName","type":"string"},{"name":"lastName","type":"string"},{"name":"age","type":"int"},{"name":"phoneNumber","type":"string"}]}'
```

Example response:

```
{"id":"my-schema","type":"AVRO","version":2,"createdBy":"","createdOn": 1579267978382,"modifiedBy":"","modifiedOn":1579267978382,"globalId":83}
```

Creating a new version of a schema requires at least both:

- Reader role access to the Event Streams cluster resource type.
- Writer role access to the schema resource that matches the schema getting a new version.

### Get the latest version of a schema
To retrieve the latest version of a particular schema, make a GET request to the /artifacts/{schema-id} endpoint, (where {schema-id} is the ID of the schema). If successful, the latest version of the schema is returned in the payload of the response.

Example curl request:

```
curl -u token:$APIKEY $URL/artifacts/my-schema
```

Example response:

```
{"type":"record","name":"Citizen","fields":[{"name": "firstName","type":"string"},{"name":"lastName","type":"string"},{"name":"age","type":"int"},{"name":"phoneNumber","type":"string"}]}
```

Getting the latest version of a schema requires at least both:

- Reader role access to the Event Streams cluster resource type.
- Reader role access to the schema resource that matches the schema being retrieved.

### Getting a specific version of a schema
To retrieve a specific version of a schema, make a GET request to the /artifacts/{schema-id}/versions/{version} endpoint, (where {schema-id} is the ID of the schema, and {version} is the version number of the specific version you need to retrieve). If successful, the specified version of the schema is returned in the payload of the response.

Example curl request

```
curl -u token:$APIKEY $URL/artifacts/my-schema/versions/3
```

Example response:

```
{"type":"record","name":"Citizen","fields":[{"name": "firstName","type":"string"},{"name":"lastName","type":"string"},{"name":"age","type":"int"},{"name":"phoneNumber","type":"string"}]}
```

Getting the latest version of a schema requires at least both:

- Reader role access to the Event Streams cluster resource type.
- Reader role access to the schema resource that matches the schema being retrieved.

### Listing all of the versions of a schema
To list all versions of a schema currently stored in the registry, make a GET request to the /artifacts/{schema-id}/versions endpoint, (where {schema-id} is the ID of the schema). If successful, a list of all current version numbers for the schema is returned in the payload of the response.

Example curl request:

```
curl -u token:$APIKEY $URL/artifacts/my-schema/versions
```

Example response:

```
[1,2,3,5,6,8,100]
```

Getting the list of available versions of a schema requires at least both:

- Reader role access to the Event Streams cluster resource type.
- Reader role access to the schema resource that matches the schema being retrieved.

### Deleting a version of a schema
Schema versions are deleted from the registry by issuing a DELETE request to the /artifacts/{schema-id}/versions/{version} endpoint (where {schema-id} is the ID of the schema, and {version} is the version number of the schema version). If successful, an empty response, and a status code of 204 (no content) is returned. Deleting the only remaining version of a schema will also delete the schema.

Example curl request:

```
curl -u token:$APIKEY –X DELETE $URL/artifacts/my-schema/versions/3
```

Deleting a schema version requires at least both:

- Reader role access to the Event Streams cluster resource type.
- Manager role access to the schema resource that matches the schema being deleted.

### Updating a global rule
Global compatibility rules can be updated by issuing a PUT request to the /rules/{rule-type} endpoint, (where {rule-type} identifies the type of global rule to be updated - currently the only supported type is COMPATIBILITY), with the new rule configuration in the body of the request. If the request is successful, the newly updated rule config is returned in the payload of the response, together with a status code of 200 (OK).

The JSON document sent in the request body must have the following properties:

Property name | Description
--- | ---
type | Must always be set to the value COMPATIBILITY.
config | Must be set to one of the following values: NONE, BACKWARD, BACKWARD_TRANSITIVE, FORWARD, FORWARD_TRANSITIVE, FULL, or FULL_TRANSITIVE (See the section on [compatibility rules](/docs/EventStreams?topic=EventStreams-ES_schema_registry#applying_compatibility_rules) for details on each of these values).

Example curl request:

```
curl -u token:$APIKEY –X PUT $URL/rules/COMPATIBILITY -d '{"type":"COMPATIBILITY","config":"BACKWARD"}'
```

Example response:

```
{"type":"COMPATIBILITY","config":"BACKWARD"}
```

Updating a global rule configuration requires at least:

- Manager role access to the Event Streams cluster resource type.


### Getting the current value of a global rule
The current value of a global rule is retrieved by issuing a GET request to the /rules/{rule-type} endpoint, (where {rule-type} is the type of global rule to be retrieved - currently the only supported type is COMPATIBILITY). If the request is successful, the current rule configuration is returned in the payload of the response, together with a status code of 200 (OK).

Example curl request:

```
curl -u token:$APIKEY $URL/rules/COMPATIBILITY
```

Example response:

```
{"type":"COMPATIBILITY","config":"BACKWARD"}
```

Getting global rule configuration requires at least:

- Reader role access to the Event Streams cluster resource type.

### Creating a per-schema rule
Rules can be applied to a specific schema, overriding any global rules that have been set, by making a POST request to the /artifacts/{schema-id}/rules endpoint, (where {schema-id} is the ID of the schema), with the type and value of the new rule contained in the body of the request, (currently the only supported type is COMPATIBILITY). If successful, an empty response and a status code of 204 (no content) are returned.

Example curl request:

```
curl -u token:$APIKEY $URL/artifacts/my-schema/rules -d '{"type":"COMPATIBILITY","config":"FORWARD"}'
```

Creating per-schema rules requires at least:

- Reader role access to the Event Streams cluster resource type.
- Manager role access to the schema resource for which the rule will apply.

### Getting a per-schema rule
To retrieve the current value of a type of rule being applied to a specific schema, a GET request is made to the /artifacts/{schema-id}/rules/{rule-type} endpoint, (where {schema-id} is the ID of the schema, and {rule-type} is the type of global rule to be retrieved - currently the only supported type is COMPATIBILITY). If the request is successful, the current rule value is returned in the payload of the response, together with a status code of 200 (OK).

Example curl request:

```
curl -u token:$APIKEY $URL/artifacts/my-schema/rules/COMPATIBILITY
```

Example response:

```
{"type":"COMPATIBILITY","config":"FORWARD"}
```

Getting per-schema rules requires at least:

- Reader role access to the Event Streams cluster resource type.
- Reader role access to the schema resource to which the rule applies.

### Updating a per-schema rule
The rules applied to a specific schema are modified by making a PUT request to the /artifacts/{schema-id}/rules/{rule-type} endpoint, (where {schema-id} is the ID of the schema, and {rule-type} is the type of global rule to be retrieved - currently the only supported type is COMPATIBILITY). If the request is successful, the newly updated rule config is returned in the payload of the response, together with a status code of 200 (OK).

Example curl request:

```
curl -u token:$APIKEY –X PUT $URL/artifacts/my-schema/rules/COMPATIBILITY -d '{"type":"COMPATIBILITY","config":"BACKWARD"}'
```

Example response:

```
{"type":"COMPATIBILITY","config":"BACKWARD"}
```

Updating a per-schema rule requires at least:

- Reader role access to the Event Streams cluster resource type.
- Manager role access to the schema resource to which the rule applies.

### Deleting a per-schema rule
The rules applied to a specific schema are deleted by making a DELETE request to the /artifacts/{schema-id}/rules/{rule-type} endpoint,
(where {schema-id} is the ID of the schema, and {rule-type} is the type of global rule to be retrieved - currently the only supported type is COMPATIBILITY). If the request is successful, an empty response is returned, with a status code of 204 (no content).

Example curl request:

```
curl -u token:$APIKEY –X DELETE $URL/artifacts/my-schema/rules/COMPATIBILITY
```

Deleting a per-schema rule requires at least:

- Reader role access to the Event Streams cluster resource type.
- Manager role access to the schema resource to which the rule applies.

## Applying compatibility rules to new versions of schemas
The schema registry supports enforcing compatibility rules when creating a new version of a schema. If a request is made to create a new schema version that does not conform to the required compatibility rule, the registry rejects the request. The following rules are supported:

Compatibility Rule | Tested against | Description
--- | --- | ---
NONE | N/A | No compatibility checking is performed when a new schema version is created.
BACKWARD | Latest version of the schema | A new version of the schema can omit fields that are present in the existing version of the schema.
BACKWARD_TRANSITIVE | All versions of the schema | A new version of the schema can add optional fields that are not present in the existing version of the schema.
FORWARD | Latest version of the schema | A new version of the schema can add fields that are not present in the existing version of the schema.
FORWARD_TRANSITIVE | All versions of the schema | A new version of the schema can omit optional fields that are present in the existing version of the schema.
FULL | Latest version of the schema | A new version of the schema can add optional fields that are not present in the existing version of the schema.
FULL_TRANSITIVE | All versions of the schema | A new version of the schema can omit optional fields that are present in the existing version of the schema.

These rules can be applied at two scopes:

1. At a global scope, which is the default that is used when creating a new schema version.
2. At a per-schema level. If a per-schema level rule is defined, it overrides the global default for the particular schema.

By default, the registry has a global compatibility rule setting of `NONE`. Per-schema level rules must be defined, otherwise the schema will default to using the global setting.

## Using the schema registry with the third party SerDes
Schema registry supports the use of the following third party SerDes:

- Confluent SerDes

To configure the Confluent SerDes to use the schema registry, you need to specify two properties in the configuration of your Kafka client:

Property name | Value
--- | ---
SCHEMA_REGISTRY_URL_CONFIG | Set this to the URL of the schema registry, including your credentials as basic authentication, and with a path of <code>/confluent</code>. For example, if <code>$APIKEY</code> is the API key to use and <code>$HOST</code> is the host from the <code>kafka_http_url</code> field in the service credentials, then the value should be in the form: <code>https://token:$APIKEY@$HOST/confluent</code>
BASIC_AUTH_CREDENTIALS_SOURCE | Set to <code>URL</code>. This instructs the SerDes to use HTTP basic authentication using the credentials supplied in the schema registry URL.