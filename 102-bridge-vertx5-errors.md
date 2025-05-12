# Additional OpenAPI schema validation error details with Vert.x 5

This proposal outlines an enhancement to the Strimzi HTTP Bridge error responses, adding detailed information for OpenAPI schema validation failures.
The change is prompted by updates introduced in Vert.x as part of the upcoming upgrade from version 4.x to 5.x.

## Current situation

The Strimzi HTTP bridge uses Vert.x 4.x, consistent with all Strimzi operators.
By leveraging the `vertx-web-openapi` component, together with the underlying JSON schema validation dependency, the bridge can validate incoming HTTP requests against the OpenAPI v3 specification before processing them.
When a schema validation error occurs, the HTTP response body returned to the client has the following format:

```json
{
    "error_code": 400,
    "message": "<description of the schema validation error>"
}
```

The content of the `message` field is derived from the `ValidationException` raised during JSON schema validation by the `vertx-web-openapi` component and is caught by the bridge while handling the request.

For example, if an HTTP client attempts to create a new Kafka consumer but provides the `enable.auto.commit` field as a string value `"true"` instead of the expected boolean `true`, the schema validation will fail, and the following error will be returned:

```json
{
    "error_code": 400,
    "message": "Validation error on: /enable.auto.commit - input don't match type BOOLEAN"
}
```

The error message clearly indicates the issue in the request body, allowing users to quickly identify and fix the problem in their client.

## Motivation

The Vert.x team is preparing to release Vert.x 5.x, which introduces several significant changes.
One major change, particularly relevant for OpenAPI v3 support in the bridge, is the transition from the `vertx-web-openapi` component to the new `vertx-web-openapi-router` and `vertx-openapi` components.
This shift also includes an update to the underlying JSON schema validation library.
The new validation mechanism offers more comprehensive error reporting, although it also changes the structure of the output.

In Vert.x 5.x, the new `SchemaValidationException` class replaces the previous `ValidationException` used in Vert.x 4.x.
While it still includes an error message field, the format and content have changed.
This is due to the introduction of a new `OutputUnit` field, which provides a list of detailed validation errors.
However, the Vert.x implementation currently extracts only the last error from this array to use as the error message.
This change stems from improvements in the JSON schema validator, which now detects multiple hierarchical validation issues in a single pass and reports all of them, unlike the previous Vert.x version.

For example, using the same scenario from the previous section, where an HTTP client attempts to create a new Kafka consumer but sends the `enable.auto.commit` field as the string `"true"` instead of the boolean `true`, the validation would fail and return the following message:

```json
{
    "error_code": 400,
    "message": "Validation error on: The value of the request body is invalid. Reason: Property \"enable.auto.commit\" does not match additional properties schema at #/enable.auto.commit"
}
```

This message originates from the last entry in the list of validation errors, which actually looks like this:

```json
{
  "absoluteKeywordLocation" : "app:///#/properties",
  "keywordLocation" : "#/properties",
  "instanceLocation" : "#/enable.auto.commit",
  "error" : "Property \"enable.auto.commit\" does not match schema",
  "errorType" : "INVALID_VALUE"
},
{
  "absoluteKeywordLocation" : "app:///#/properties/enable.auto.commit/type",
  "keywordLocation" : "#/properties/enable.auto.commit/type",
  "instanceLocation" : "#/enable.auto.commit",
  "error" : "Instance type string is invalid. Expected boolean",
  "errorType" : "INVALID_VALUE"
},
{
  "absoluteKeywordLocation" : "app:///#/additionalProperties",
  "keywordLocation" : "#/additionalProperties",
  "instanceLocation" : "#/enable.auto.commit",
  "error" : "Property \"enable.auto.commit\" does not match additional properties schema",
  "errorType" : "INVALID_VALUE"
}
```

These results reflect a more thorough validation process. Specifically:

* The first error indicates that the property doesn’t match the expected schema.
* The next error pinpoints the type mismatch (string vs. expected boolean).
* the last error is about the validator detecting the property as a new one, because not within the schema (the type is different), and it's not allowed by the bridge OpenAPI specification, as per this [line](https://github.com/strimzi/strimzi-kafka-bridge/blob/main/src/main/resources/openapi.json#L1759) by having `"additionalProperties": false`.

While this enhanced validation gives developers more insights, the downside is that only the last error is exposed through the `SchemaValidationException`, which may not be the most informative one, in this case, the type mismatch is arguably more useful.

For more details please have a look at the [issue](https://github.com/eclipse-vertx/vertx-openapi/issues/100) I opened on the `vertx-openapi` repository, together with a similar [discussion](https://github.com/vert-x3/vertx-web/issues/2044#issuecomment-1779412178) which was raised in the past within the Vert.x community.

Ultimately, this is a breaking change we’ll need to adapt to, adjusting our error handling logic to align with the new way JSON schema validation results are structured and reported.

## Proposal

This proposal suggests enhancing the HTTP error responses returned by Vert.x 5.x when an OpenAPI schema validation error occurs. Specifically, it introduces a new `validation_errors` field in the response body, providing clients with detailed insight into what caused the schema validation failure.
The additional field would bring the error messages array coming from the validation.
The current `message` field would provide just a generic "Schema validation error" text instead.

For example, still taking into account the same scenario, the outcome would be:

```json
{
    "error_code": 400,
    "message": "Schema validation error",
    "validation_errors": [
        "Property \"enable.auto.commit\" does not match schema",
        "Instance type string is invalid. Expected boolean",
        "Property \"enable.auto.commit\" does not match additional properties schema"
      ]
}
```

When the error is unrelated to schema validation, the `validation_errors` field will be omitted.
In such cases, the `message` field will continue to convey the nature of the error.

As a result, the current `"Error"` [component definition](https://github.com/strimzi/strimzi-kafka-bridge/blob/main/src/main/resources/openapi.json#L1712) within the bridge OpenAPI v3 specification would change to the following:

```json
{
  "Error": {
    "title": "Error",
    "type": "object",
    "properties": {
      "error_code": {
        "type": "integer",
        "format": "int32"
      },
      "message": {
        "type": "string"
      },
      "validation_errors": {
        "type": "array",
        "items": {
          "type": "string"
        }
      }
    },
    "example": {
      "error_code": 400,
      "message": "Schema validation error",
      "validation_errors": [
        "Property \"enable.auto.commit\" does not match schema",
        "Instance type string is invalid. Expected boolean",
        "Property \"enable.auto.commit\" does not match additional properties schema"
      ]
    }
  }
}
```

## Affected/not affected projects

The only affected project is the Strimzi HTTP bridge. 

## Compatibility

From the perspective of an HTTP client that relies on the `message` field for error details, this change introduces a potential breaking behavior.
With the new structure, the specific cause of a schema validation failure will no longer appear directly in the `message` field, which will now contain only a generic "Schema validation error" message.
Instead, clients will need to inspect the new `validation_errors` array to obtain detailed information about the issue.
For errors unrelated to schema validation, the behavior remains unchanged and the `message` field will continue to convey the appropriate error description.

## Rejected alternatives

Instead of returning only the error messages with details of the schema validation, the HTTP response body could include the entire JSON structure as shown below:

```json
{
    "error_code": 400,
    "message": "Schema validation error",
    "validation_errors": [
        {
            "absoluteKeywordLocation" : "app:///#/properties",
            "keywordLocation" : "#/properties",
            "instanceLocation" : "#/enable.auto.commit",
            "error" : "Property \"enable.auto.commit\" does not match schema",
            "errorType" : "INVALID_VALUE"
        },
        {
            "absoluteKeywordLocation" : "app:///#/properties/enable.auto.commit/type",
            "keywordLocation" : "#/properties/enable.auto.commit/type",
            "instanceLocation" : "#/enable.auto.commit",
            "error" : "Instance type string is invalid. Expected boolean",
            "errorType" : "INVALID_VALUE"
        },
        {
            "absoluteKeywordLocation" : "app:///#/additionalProperties",
            "keywordLocation" : "#/additionalProperties",
            "instanceLocation" : "#/enable.auto.commit",
            "error" : "Property \"enable.auto.commit\" does not match additional properties schema",
            "errorType" : "INVALID_VALUE"
        }
    ]
}
```

This approach would align more closely with the evolution of the Vert.x JSON schema validation. However, it would require updating the bridge's OpenAPI specification every time Vert.x changes, which could add an additional maintenance burden.
