# Schemas 101

Snowplow enforces data integrity by using self-describing JSONs.

Self-describing JSONs have two parts, `schema` and `data`:

```json
{
	"schema": "iglu:com.snowplowanalytics.snowplow/ad_click/jsonschema/1-0-0",
	"data": {
		"targetUrl": "http://www.google.com"
	}
}
```

The `schema` field holds a URI - an identifier that unambiguously describes a resource.

We can use it to search an Iglu repository for a schema.

A schema is something that specifies what form our events will take. It's correct to think of a schema as a contract.

When we create events, we need to make sure they follow the format we agreed upon beforehand.

## Writing a schema

Let's open up this website:
https://www.jsonschemavalidator.net/

Here's a schema. Copy it into the left pane of the website.

```json
{
	"$schema": "http://iglucentral.com/schemas/com.snowplowanalytics.self-desc/schema/jsonschema/1-0-0#",
	"description": "A schema I'm using for a demo",
	"self": {
		"vendor": "com.mike",
		"name": "message",
		"format": "jsonschema",
		"version": "1-0-0"
	},
	"type": "object",
	"properties": {
		"name": {
			"type": "string"
		},
		"message": {
			"type": "string"
		}
	},
	"required": ["name", "message"],
	"additionalProperties": false,
	"minItems": 2
}
```

Paste something like this into the right pane to validate it against the schema:

```json
{
	"name": "mike",
    "message": "hey!"
}
```

Normally, an event will look something like this when it gets received by the collector:

```json
{
	"data": [
		{
			"dtm": "1556757326183",
			"e": "ue",
			"eid": "3214d6cd-7740-4339-b5b3-c47fb0aee5f1",
			"p": "pc",
			"stm": "1556757326000",
			"tv": "py-0.8.2",
			"ue_px": "eyJzY2hlbWEiOiAiaWdsdTpjb20uc25vd3Bsb3dhbmFseXRpY3Muc25vd3Bsb3cvdW5zdHJ1Y3RfZXZlbnQvanNvbnNjaGVtYS8xLTAtMCIsICJkYXRhIjogeyJzY2hlbWEiOiAiaWdsdTpjb20uc25vd3Bsb3cvYXdheV93ZWVrX2RlbW9fbWVzc2FnZS9qc29uc2NoZW1hLzEtMC0wIiwgImRhdGEiOiB7Im5hbWUiOiAibWlrZSIsICJtZXNzYWdlIjogImhleSwgSSBqdXN0IHNlbnQgYW4gZXZlbnQhIn19fQ=="
		}
	],
	"schema": "iglu:com.snowplowanalytics.snowplow/payload_data/jsonschema/1-0-4"
}
```

When we decode the mess of letters under `ue_px`, we'll get:

```json
{
	"schema": "iglu:com.snowplowanalytics.snowplow/unstruct_event/jsonschema/1-0-0",
	"data": {
		"schema": "iglu:com.mike/message/jsonschema/1-0-0",
		"data": {
			"name": "mike",
			"message": "hey!"
		}
	}
}
```

The command to decode the payload is this: `echo "<ue_px goes here>" | base64 -D`

## Under-the-hood

Our trackers use HTTP requests to send their events to a collector. There's two methods we use: `POST` and `GET`.

When we're sending POST requests, the request carries a JSON filled with the event. No magic here. It's perhaps obscure, but good to know that we have a schema that describes this JSON: [tracker protocol schema](https://github.com/snowplow/iglu-central/blob/master/schemas/com.snowplowanalytics.snowplow/payload_data/jsonschema/1-0-4).

With GET requests, we're passing events to the server by packing all the information in the URL that we request. This is done by passing key-values in a query string - the part after `i` in `http://1.2.3.4/i?e=ue&ue_pr=...`. The question mark marks the beginning of the query string, equal signs are placed between keys and values, and the ampersands separate key-value pairs.

With any unstructured event, we have fields that are used for when the event is encoded and unencoded: `ue_pr` and `ue_px`. We use `ue_px` when the event data is [base64 encoded](https://developer.mozilla.org/en-US/docs/Web/HTTP/Basics_of_HTTP/Data_URIs#Encoding_data_into_base64_format).

To encode the event data, we turn the event JSON into a string (called stringifying - we need to do this cause a JSON is an object) and then base64 encode the string. URL-safe base64 encoding makes sure we can represent it in a way that only contains valid characters allowed in a URL.
