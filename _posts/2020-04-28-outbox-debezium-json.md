---
layout: post
title:  "Outbox pattern in JSON with Debezium"
date:   2020-04-28 19:00:00
meta_description: "This post describes how to produce pure JSON payload using the outbox pattern with debezium"
tags:
  - JSON
  - Kafka-connect
  - MySQL
  - Outbox
categories: 
  - CDC
  - Outbox pattern
---

If you are not familiar with the outbox pattern, this [article](https://debezium.io/blog/2019/02/19/reliable-microservices-data-exchange-with-the-outbox-pattern/) gives a very rich introduction with step by step instruction.

_Note_ that what follows applies only if you want to work with JSON-formatted Kafka messages.

If you follow it thoroughly, you will obtain a payload which itself contains the stringify JSON payload of your original outbox table. This is hardly usable from a consumer point of view. What you may want instead is the original JSON payload, from your outbox table field `payload`, as a simple string and all other attributes as Kafka headers for example.

The ["Outbox Event Router" SMT](https://debezium.io/documentation/reference/1.1/configuration/outbox-event-router.html) from the Debezium project partially supports it. The idea is to keep only the payload field content in the kafka message and move everything else in the header with the configuration flag `additional.placement` as such:

```json
"transforms.outbox.type": "io.debezium.transforms.outbox.EventRouter",
"transforms.outbox.additional.placement": "type:header:eventType"
```

This will move the `eventType` (`type`) information from the envelope to the headers leaving the actual kafka message left with only `payload`.

At this point, you will notice that the output is actually the JSON-serialized content of the (internal) Kafka Connect data and not a pure and simple JSON as a string, it should look something like this:

```json
{
  "schema": {
    "type": "string",
    "optional": true
  },
  "payload": "{\"hello\":\"monde\"}"
}
```

You may be tempted to configure the JSON converter to disable schema, like this:

```json
  "key.converter": "org.apache.kafka.connect.json.JsonConverter",
  "key.converter.schemas.enable": "false",
  "value.converter": "org.apache.kafka.connect.json.JsonConverter",
```

which will result in:

```
"{\"hello\":\"monde\"}"
```

quite not what we want.

Since our payload is now just a string really, we can simply use the `StringConvert` as such:

```json
  "value.converter": "org.apache.kafka.connect.storage.StringConverter",
  "key.converter": "org.apache.kafka.connect.storage.StringConverter",
```

which, this time, gives the expected output:

```json
{"hello": "monde"}
```
