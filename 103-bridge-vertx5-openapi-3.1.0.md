# Move HTTP bridge to OpenAPI 3.1.0 with Vert.x 5

This proposal recommends updating the HTTP bridge in the Strimzi project to adopt OpenAPI 3.1.0, the latest minor release in the 3.x specification series.
The change is prompted by updates introduced in Vert.x as part of the upcoming upgrade from version 4.x to 5.x.

## Current situation

The Strimzi HTTP bridge uses Vert.x 4.x, consistent with all Strimzi operators.
By leveraging the `vertx-web-openapi` component, together with the underlying JSON schema validation dependency, the bridge can validate incoming HTTP requests against the OpenAPI 3.0.0 specification before processing them.

In the existing OpenAPI specification, the HTTP bridge utilizes the `nullable` keyword in some places:

* Allowing a record to be sent with a `null` value, which in Apache Kafka typically represents a "tombstone" message used in compacted topics.
* Permitting the omission of partition count and replication factor parameters when creating a topic through the Admin Client API.

Following the `RecordValue` [definition](https://github.com/strimzi/strimzi-kafka-bridge/blob/main/src/main/resources/openapi.json#L2215) which allows the `value` field o be either a JSON object or null:

```json
"RecordValue": {
  "title": "RecordValue",
  "description": "Value representation for a record. It can be an array, a JSON object or a string",
  "oneOf": [
      {
          "type": "array",
          "items": {}
      },
      {
          "type": "object",
          "nullable": true
      },
      {
          "type": "string"
      }
  ],
  "nullable" : true
}
```

A similar use of `nullable` is found in the `NewTopic` [definition](https://github.com/strimzi/strimzi-kafka-bridge/blob/main/src/main/resources/openapi.json#L2233), which governs topic creation behavior.

## Motivation

The Vert.x team is preparing to release Vert.x 5.x, which introduces several significant changes.
One major change, particularly relevant for OpenAPI v3 support in the bridge, is the transition from the `vertx-web-openapi` component to the new `vertx-web-openapi-router` and `vertx-openapi` components.
This shift also includes an update to the underlying JSON schema validation library.

This new validation library has been refactored to align more closely with the [JSON Schema](https://json-schema.org/) specification.
Such specification defines a way for a JSON object to be null, by using the `type: null` [declaration](https://json-schema.org/understanding-json-schema/reference/null).

In contrast, the `nullable` keyword used in OpenAPI 3.0.0 is not part of the JSON Schema specification.
It is specific to OpenAPI and has no effect when using JSON Schema-compliant validators.

Due to Vert.x 5.x adhering to the JSON Schema standard, the `nullable` flag, used in the current HTTP bridge OpenAPI specification, is no longer supported.
This change causes schema validation to fail when an HTTP client sends a record with a `null` value, even if such input was previously valid.
Below is an example of the validation error returned when attempting to send a record with `value: null`:

```json
{
  "valid" : false,
  "errors" : [ {
    "absoluteKeywordLocation" : "app:///#/properties",
    "keywordLocation" : "#/properties",
    "instanceLocation" : "#/records",
    "error" : "Property \"records\" does not match schema",
    "errorType" : "INVALID_VALUE"
  }, {
    "absoluteKeywordLocation" : "app:///#/properties/records/items",
    "keywordLocation" : "#/properties/records/items",
    "instanceLocation" : "#/records",
    "error" : "Items did not match schema",
    "errorType" : "INVALID_VALUE"
  }, {
    "absoluteKeywordLocation" : "app:///#/properties/records/items/$ref",
    "keywordLocation" : "#/properties/records/items/$ref",
    "instanceLocation" : "#/records/0",
    "error" : "A subschema had errors",
    "errorType" : "INVALID_VALUE"
  }, {
    "absoluteKeywordLocation" : "app:///#/components/schemas/ProducerRecord/properties",
    "keywordLocation" : "#/properties/records/items/$ref/properties",
    "instanceLocation" : "#/records/0/value",
    "error" : "Property \"value\" does not match schema",
    "errorType" : "INVALID_VALUE"
  }, {
    "absoluteKeywordLocation" : "app:///#/components/schemas/ProducerRecord/properties/value/$ref",
    "keywordLocation" : "#/properties/records/items/$ref/properties/value/$ref",
    "instanceLocation" : "#/records/0/value",
    "error" : "A subschema had errors",
    "errorType" : "INVALID_VALUE"
  }, {
    "absoluteKeywordLocation" : "app:///#/components/schemas/RecordValue/oneOf",
    "keywordLocation" : "#/properties/records/items/$ref/properties/value/$ref/oneOf",
    "instanceLocation" : "#/records/0/value",
    "error" : "Instance does not match exactly one subschema (0 matches)",
    "errorType" : "INVALID_VALUE"
  }, {
    "absoluteKeywordLocation" : "app:///#/components/schemas/RecordValue/oneOf/0/type",
    "keywordLocation" : "#/properties/records/items/$ref/properties/value/$ref/oneOf/0/type",
    "instanceLocation" : "#/records/0/value",
    "error" : "Instance type null is invalid. Expected array",
    "errorType" : "INVALID_VALUE"
  }, {
    "absoluteKeywordLocation" : "app:///#/components/schemas/RecordValue/oneOf/1/type",
    "keywordLocation" : "#/properties/records/items/$ref/properties/value/$ref/oneOf/1/type",
    "instanceLocation" : "#/records/0/value",
    "error" : "Instance type null is invalid. Expected object",
    "errorType" : "INVALID_VALUE"
  }, {
    "absoluteKeywordLocation" : "app:///#/components/schemas/RecordValue/oneOf/2/type",
    "keywordLocation" : "#/properties/records/items/$ref/properties/value/$ref/oneOf/2/type",
    "instanceLocation" : "#/records/0/value",
    "error" : "Instance type null is invalid. Expected string",
    "errorType" : "INVALID_VALUE"
  }, {
    "absoluteKeywordLocation" : "app:///#/components/schemas/ProducerRecord/additionalProperties",
    "keywordLocation" : "#/properties/records/items/$ref/additionalProperties",
    "instanceLocation" : "#/records/0/value",
    "error" : "Property \"value\" does not match additional properties schema",
    "errorType" : "INVALID_VALUE"
  }, {
    "absoluteKeywordLocation" : "app:///#/additionalProperties",
    "keywordLocation" : "#/additionalProperties",
    "instanceLocation" : "#/records",
    "error" : "Property \"records\" does not match additional properties schema",
    "errorType" : "INVALID_VALUE"
  } ],
  "errorType" : "INVALID_VALUE"
}
```

As a result, simply upgrading the HTTP bridge to Vert.x 5.x is insufficient, support for `null` values in requests breaks unless the OpenAPI specification is revised to conform to the updated validation behavior.

For more details, please refer to this [discussion](https://github.com/eclipse-vertx/vertx-json-schema/issues/144) with the maintainer of the Vert.x JSON Schema component.

## Proposal

In light of the motivation outlined above, this proposal recommends updating the HTTP bridge to support OpenAPI 3.1.0 in conjunction with the migration to Vert.x 5.x.
This upgrade enables the OpenAPI specification to fully align with the JSON Schema standard by replacing the now unsupported `nullable` flag with the `type: null` declaration.

For example, the updated `RecordValue` schema would look like this:

```json
"RecordValue": {
  "title": "RecordValue",
  "description": "Value representation for a record. It can be an array, a JSON object or a string",
  "oneOf": [
    {
      "type": "array",
      "items": {}
    },
    {
      "type": ["object", "null"]
    },
    {
      "type": "string"
    },
    {
      "type": "null"
    }
  ]
}
```

A similar change would also be required for the `NewTopic` schema definition to ensure compatibility with the updated JSON Schema validation in Vert.x 5.x.

Of course, the OpenAPI version would also be updated within the specification itself, as shown below:

```json
"openapi": "3.1.0"
```

## Affected/not affected projects

The only affected project is the Strimzi HTTP bridge. 

## Compatibility

From the perspective of an HTTP client sending a record with `value: null` or creating a new topic without specifying the partition count and replication factor, this change introduces no behavioral differences.

For the HTTP bridge itself, aside from replacing the `nullable` flag with the JSON Schema-compliant `type: null` declaration, the rest of the OpenAPI specification remains largely compatible.

The main caveat with this upgrade is that, to address the compatibility issue, the project cannot follow the usual deprecation path where the existing OpenAPI 3.0.0 specification is supported alongside the new version for a transitional period before removal, as was done previously when migrating from OpenAPI 2.0.0 to 3.0.0.

However, this should not cause big issues since this update involves a minor version bump.
Apart from the removal of `nullable`, the OpenAPI 3.1.0 specification appears to be backward compatible with 3.0.0.

## Rejected alternatives

N/A
