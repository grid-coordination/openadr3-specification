# OpenADR 3 Object Operation Notifications via Additional Protocols

## Version: 1.0.0, 2026-03-15

Copyright (c) 2026 Clark Communications Corporation. Licensed under the Apache License, Version 2.0.

---

# 1. Introduction

OpenADR 3 revisions 3.0 and 3.0.1 supported a single mechanism — webhooks via HTTP callbacks — for a Virtual Top Node (VTN) to push notifications to clients.  Revision 3.1.0 adds support for additional notification delivery protocols, with MQTT being the first such protocol defined.

This document provides a comprehensive, non-normative description of the additional notifications feature introduced in OpenADR 3.1.0.  It covers the protocol-agnostic notification framework, the MQTT notifier binding, and an exploration of how other protocols (such as websockets) might be supported in the future.

The normative specification is defined in the [OpenADR 3.1.0 Definition][Definition] and the [OpenADR 3.1.0 OpenAPI Specification (YAML)][YAML].  The [OpenADR 3.1.0 User Guide][User_Guide] provides usage scenarios and sequence diagrams.

# 2. Goals and Objectives

Define extensions to the OpenADR 3 (OA3) specification to **optionally** enable clients to subscribe to OA3 VTN objects and to (subsequently) receive "push" notifications of changes to subscribed objects, via mechanisms and transports other than webhooks.

The primary motivation for adding additional notification methods beyond OA3's existing webhook notifications is to enable VENs behind common Internet firewalls to feasibly receive push notifications.  It is extremely challenging to configure typical residential or small business Internet firewalls to receive inbound HTTP requests (as required to support webhooks).  Applications and services that need to obtain push notifications from the public Internet typically do so via client-initiated persistent connections, with MQTT being one popular option.

The resulting additions to the OpenADR REST API are protocol-agnostic, enabling specification and implementation of multiple notification protocols.  The scope of notification protocol independence is strictly limited to the OA3 REST API endpoints themselves, enabling OA3 clients to:

1. Discover which notification methods ("notifiers") a VTN supports
2. Query specific notifier bindings for binding-specific data (e.g. broker URIs, topic names, authentication methods)
3. Subscribe to and receive notifications via the binding's native protocol

After determination of a VTN's notifier binding support via the OA3 REST API, subsequent notification operations by the client utilize the notification binding protocol itself, typically via existing programming language libraries.  For example, if an OA3 client determines that a VTN supports notification via MQTT, after obtaining MQTT details from the VTN, the client will subsequently use the MQTT protocol (directly or via an MQTT client library) to receive notifications.

### 2.1 Evolution of the Design

The goals were refined and clarified during the design phase:

1. **MQTT focus**: Initial work focused on specifying a method for clients to receive notifications via MQTT.

2. **Protocol generalization**: The design was broadened to enable potential future support for other messaging protocols, specifically Kafka.

3. **Webhook unification**: We realized that the existing OA3 webhook-callback notifications can, and should be, considered as another notification mechanism within this framework.  The design was extended to incorporate webhooks as the `WEBHOOK` notifier binding.

4. **Beyond message brokers**: To accommodate notification transports not based on named-topic messaging and message-brokers (e.g. websockets), the terminology and specification of brokers and topics was moved into only those notifier bindings that use those concepts.

# 3. Definitions

- **Notifier Binding**: A specification (and implementation) of a specific notification protocol technology to the OA3 `/notifiers` endpoints.  Each notifier binding is specified in a companion specification, with the MQTT binding being the first.

- **Broker**: The standard term for the "server" component of a messaging protocol system.  VTNs implementing notifications via MQTT would include a MQTT broker.

- **Topic**: A category used to organize messages.  Each topic has a name that is unique to the broker.

- **Client**: An entity that connects to a broker to write or read messages.

- **Publisher/Producer**: A client that writes messages to a topic on a broker.  VTNs implementing notifications via MQTT would be publishing clients.

- **Subscriber/Consumer**: A client that reads messages from a topic on a broker.  Typically the subscribing client subscribes to a topic and subsequently receives messages published to that topic.  VENs (and BL-clients) utilizing a VTN notifier binding based on a messaging protocol would be subscribing clients.

- **Subscribe-able Object**: An OA3 object that can be subscribed to, resulting in subsequent notifications of changes to the subscribed object being sent to the subscriber.  OA3 subscribe-able objects include: PROGRAM, EVENT, REPORT, SUBSCRIPTION, VEN, and RESOURCE.  These are the same set of objects the REST API `/subscriptions` endpoint supports.

# 4. Motivation for Support of Multiple Notification Protocols

In a nutshell, to enable OA3 implementations (and their developers) to use "the right tool for the job."

Notifications via webhook are a useful and viable mechanism for delivery to cloud-based VENs — for example, a Demand Response Provider (DRP) that aggregates a large number of utility customers — and may be well-suited for notifications to BL-clients.

However, a VEN integrated into a residential appliance (e.g. a heat-pump water-heater) and a VTN serving millions of customers of an IOU are "completely different animals", differing vastly in scope and scale:

- **Appliance-integrated VEN**: Might be implemented on a low-cost microcontroller (e.g. ESP32), with commensurate resource constraints.

- **Utility-scale VTN**: Might be implemented in the cloud using Internet-scale PaaS services.  Utility BL connections with the VTN may involve numerous complex integrations to legacy systems.

- **Home Energy Management System (HEMS)**: Incorporating "OA3-Gateway" functionality, might be implemented on a Raspberry Pi, engineered to handle tens of appliance-VENs, and easily capable of additionally hosting a MQTT broker for use by local VENs/clients.

These different implementations will make very different engineering technology trade-offs.  Notifications via MQTT messaging might be well-suited to residential VENs (appliance-integrated, HEMS/OA3-Gateway), and a utility-scale VTN might find notifications via Kafka advantageous for some BL-clients and notifications via MQTT advantageous for others.

# 5. Common Scenarios

During the development of this design, we evaluated our ideas against the subscription and notification requirements of the following VEN scenarios:

### 5.1 Utility Customer

Customer with electric service from a utility that offers an OpenADR 3 VTN.  The VEN is provisioned/configured with the VTN's URL, the customer's tariff (rate-plan), and the customer's authentication credentials.

After the VEN authenticates, registers itself, and obtains the program representing the customer's tariff from the VTN, subscriptions/notifications might include:

- Subscription to any changes in the program representing the customer's tariff
- Subscription to all events (and subsequent changes) associated with that tariff/program

### 5.2 Grid Monitoring and Tracking

A utility researcher/analyst monitors aspects of the grid "in the large" using a grid-monitoring-VEN.  The grid-monitoring-VEN connects to a cloud-based aggregating VTN, and:

- Subscribes to the creation, update, or deletion of any/all programs/tariffs
- Subscribes to program events of specific interest, e.g. real-time prices, or GHG data

### 5.3 Flexibility Aggregator Customer

A flexibility aggregator might have several programs available that customers can choose to participate in.  A customer would have multiple VENs.  Each VEN connects to the VTN and:

- Subscribes to the creation, update, or deletion of any events intended for this VEN
- A VEN must not receive any information detailing the creation, update, or deletion of events exclusively destined for other VENs, as this information may be commercially sensitive

BL/VTN subscription/notification scenarios are likely more complex and numerous than the above, but the design is intended to meet those requirements as well.

# 6. Subscription Semantics: OA3 REST vs. Messaging Protocols

The new (non-webhook) notifier bindings were designed for messaging protocols — initially MQTT, and potentially later Kafka.  These messaging protocols are typically organized around the concept of named topics: clients subscribe to topics and receive messages published to those topics.

OA3 REST subscriptions are not based on named topics.  Instead, the subscriber specifies the objects, operations-on-those-objects, and object-targets for which it requests notifications.  In effect, the OA3 client defines and creates a "custom" subscription on the VTN.

These very different semantics were not immediately obvious, and considerable time was spent attempting to provide OA3 REST subscription semantics within the constraints of existing messaging protocol functionality, without success.

After consultation with the broader OA3 community, the goals were revised:

- Subscriptions to subscribe-able objects receive notifications for the operations `CREATE`, `UPDATE`, and `DELETE`.  The exclusion of the `READ` operation is a conscious and deliberate omission.

- Fine-grained subscriber customization of notifications via object-targets is **NOT** supported by OA3 subscriptions made via messaging protocol notifier bindings.  However, object privacy is maintained through VEN-scoped topics (see Section 9): a VEN receives notifications only for objects it is authorized to access.  Clients requiring finer-grained target-based filtering may do so via the existing webhook-callback mechanism (the `WEBHOOK` notifier binding).

- Clients requiring notification of `READ` operations on objects may do so via the existing `WEBHOOK` notifier binding.

# 7. Obtaining VTN Notifier Details

The REST endpoint `GET /notifiers` returns an object describing supported notifier bindings, and for each binding (identified by a `notifierBindingKey`), the information a client needs to utilize that notifier.

### Example Request and Response

```
GET /notifiers
```

```json
{
  "WEBHOOK": true,
  "MQTT": {
    "URIS": ["mqtts://broker.vtn.company.com:8883"],
    "authentication": {"method": "ANONYMOUS"},
    "serialization": "JSON"
  }
}
```

This example response indicates the VTN supports two notifier bindings: `WEBHOOK` and `MQTT`.

**Implementation note:** The `GET /notifiers` request requires the client's authentication token, therefore the VTN knows the identity of the requestor and **MAY** provide a response specific to that client.

## 7.1 WEBHOOK Notifier

The `WEBHOOK` notifier binding key **MUST** be provided by the VTN.

OpenADR 3.0.0, 3.0.1, and 3.1.0 all require a VTN to support notification via webhooks.  The `WEBHOOK` key in the `/notifiers` response is not providing new information per se; its inclusion anticipates a possible future OpenADR version that does not require webhook notifications.  If that were to occur, the `/notifiers` endpoint would provide clients a standard method to determine webhook availability.

When webhooks are supported, the `true` value is all a client needs to know — the delivery of notifications via webhook callbacks is fully defined in the standard and requires no additional information.

## 7.2 MQTT Notifier

A VTN **MAY** support notifications via MQTT.  If so, it will include the `MQTT` notifier binding key in its response, with a value object containing:

- **`URIS`**: One or more URIs specifying how to connect to the VTN's MQTT broker
- **`authentication`**: The MQTT broker's authentication method (see Section 10)
- **`serialization`**: The message serialization format.  Currently `JSON` is the only supported format and is the assumed default.

## 7.3 Future Notifier Bindings

The framework is designed to accommodate future notifier bindings.  For example, a Kafka binding might appear as:

```json
{
  "WEBHOOK": true,
  "MQTT": { ... },
  "KAFKA": {
    "URIS": ["kafka-uri-1", "kafka-uri-2", "kafka-uri-3"],
    "serialization": "JSON"
  }
}
```

# 8. Obtaining VTN Notifier Binding Topics

VENs and VTN-clients using notifier bindings based on topic-based messaging protocols will need to obtain the topic names for the subscribe-able objects of interest.

All requests for topic names are performed via endpoints requiring authorization, so that the VTN knows the identity of the client and **MAY** provide customized topic names specific to that client.

A VTN that supports a notifier binding based on a topic-oriented protocol **MUST** support all of the topic-name `GET` endpoints detailed below.

## 8.1 Topic Name Response Format

The response from the VTN is a JSON object containing a `topics` key.  The value of `topics` is a JSON object where each key is an operation (`CREATE`, `UPDATE`, `DELETE`, or `ALL`) and the corresponding value is the topic name for that operation.

The `ALL` key is optional — if included, it provides a topic the client can use to subscribe to all operations on that object.  With careful topic design, the `ALL` topic name might utilize MQTT topic wildcard specifiers.  Not all messaging protocols support wildcard topics (MQTT does, Kafka does not), so some bindings may not provide the `ALL` topic name.

Formally: A VTN **MAY** include a topic name for `ALL`.

## 8.2 Topic Name Endpoints

### Programs — all

```
GET /notifiers/{notifierBindingKey}/topics/programs
```

Example:

```
GET /notifiers/mqtt/topics/programs
```

```json
{
  "topics": {
    "CREATE": "programs/create",
    "UPDATE": "programs/update",
    "DELETE": "programs/delete",
    "ALL":    "programs/+"
  }
}
```

### Programs — specific program

```
GET /notifiers/{notifierBindingKey}/topics/programs/{programID}
```

Example:

```
GET /notifiers/mqtt/topics/programs/44
```

```json
{
  "topics": {
    "UPDATE": "programs/44/update",
    "DELETE": "programs/44/delete",
    "ALL":    "programs/44/+"
  }
}
```

Note the absence of the `CREATE` topic: if the program hasn't been created, no `programID` exists, so a client cannot request notifications of the creation of a program that doesn't yet exist.

### Events — all

BL-clients may obtain notifications for all events, without regard to `programID`.

```
GET /notifiers/{notifierBindingKey}/topics/events
```

Example:

```
GET /notifiers/mqtt/topics/events
```

```json
{
  "topics": {
    "CREATE": "events/create",
    "UPDATE": "events/update",
    "DELETE": "events/delete",
    "ALL":    "events/+"
  }
}
```

### Events — for a specific program

```
GET /notifiers/{notifierBindingKey}/topics/programs/{programID}/events
```

Example:

```
GET /notifiers/mqtt/topics/programs/44/events
```

```json
{
  "topics": {
    "CREATE": "events/programs/44/create",
    "UPDATE": "events/programs/44/update",
    "DELETE": "events/programs/44/delete",
    "ALL":    "events/programs/44/+"
  }
}
```

It is not possible for a client (VEN) to subscribe to a subset of a program's event types; the client must subscribe to all of the program's events and filter/ignore events in which it is not interested.

### Reports — all

```
GET /notifiers/{notifierBindingKey}/topics/reports
```

```json
{
  "topics": {
    "CREATE": "reports/create",
    "UPDATE": "reports/update",
    "DELETE": "reports/delete",
    "ALL":    "reports/+"
  }
}
```

### Reports — for a specific program

```
GET /notifiers/{notifierBindingKey}/topics/programs/{programID}/reports
```

### Subscriptions — all

```
GET /notifiers/{notifierBindingKey}/topics/subscriptions
```

### Subscriptions — for a specific program

```
GET /notifiers/{notifierBindingKey}/topics/programs/{programID}/subscriptions
```

### VENs — all

```
GET /notifiers/{notifierBindingKey}/topics/vens
```

### VENs — specific VEN

```
GET /notifiers/{notifierBindingKey}/topics/vens/{venID}
```

### Resources — all

```
GET /notifiers/{notifierBindingKey}/topics/resources
```

### Resources — for a specific VEN

```
GET /notifiers/{notifierBindingKey}/topics/vens/{venID}/resources
```

### VEN-scoped topic endpoints and object privacy

The OpenADR 3.1.0 specification defines topic endpoints scoped to a specific VEN's `venID`.  These endpoints exist to support **object privacy** for notifications delivered via messaging protocols, and are the mechanism by which the notifier framework maintains the access controls that OpenADR 3 requires (see Section 9 below).

The VEN-scoped topic endpoints are:

- `GET /notifiers/mqtt/topics/vens/{venID}/events` — topic names for operations on events targeted for a VEN
- `GET /notifiers/mqtt/topics/vens/{venID}/programs` — topic names for operations on programs targeted for a VEN
- `GET /notifiers/mqtt/topics/vens/{venID}/resources` — topic names for operations on resources belonging to a VEN

These endpoints return topic names that are specific to the identified VEN.  For example:

```
GET /notifiers/mqtt/topics/vens/ven-abc-123/events
```

```json
{
  "topics": {
    "CREATE": "events/vens/ven-abc-123/create",
    "UPDATE": "events/vens/ven-abc-123/update",
    "DELETE": "events/vens/ven-abc-123/delete",
    "ALL":    "events/vens/ven-abc-123/+"
  }
}
```

A VEN subscribing to these VEN-scoped topics will receive notifications only for objects that the VTN has determined that VEN is authorized to see.  The VTN publishes notifications to VEN-specific topics based on target resolution (see Section 9).

Note the security scopes in the OpenAPI specification: the "all objects" topic endpoints (e.g. `GET /notifiers/mqtt/topics/events`) require `read_bl` (BL-only access), while VEN-scoped endpoints require `read_ven_objects`.  This ensures VENs can only request topic names for their own objects.

# 9. Object Privacy and Notifications via Messaging Protocols

OpenADR 3 defines object privacy rules to prevent unintended access to objects.  BL clients have unrestricted access, but VEN clients are limited to accessing objects they have created or been granted access to via targeting.  These rules must be maintained for notifications delivered via messaging protocols, not just REST API responses.

## 9.1 The Challenge

With REST API subscriptions and webhook notifications, the VTN evaluates targets at notification time — it uses the subscription's `clientID` to discover the associated ven and resource objects, processes targets, and only sends the webhook callback if the targets match.

With messaging protocol notifications (e.g. MQTT), a VTN publishes messages to topics.  If all VENs subscribe to the same shared topics (e.g. `events/create`), every VEN would receive every event notification, violating object privacy.  A VEN must not receive notifications for events exclusively destined for other VENs, as this information may be commercially sensitive.

## 9.2 The Solution: VEN-scoped Topics

The specification addresses this through **VEN-scoped topic endpoints** (described in Section 8.2 above).  The VTN exposes per-VEN topic paths — topic names whose paths include the VEN's `venID` — and restricts subscription to these topics based on client identity.

The mechanism works as follows:

1. **Topic discovery**: A VEN requests its VEN-scoped topic names via the REST API (e.g. `GET /notifiers/mqtt/topics/vens/{venID}/events`).  The VTN authenticates the request and returns topic names specific to that VEN.

2. **Topic subscription**: The VEN subscribes to its VEN-scoped topics via the messaging protocol (e.g. MQTT subscribe).

3. **Targeted notification publishing**: When a BL-created object with targets (e.g. an event) is created, updated, or deleted, the VTN resolves the object's targets to determine which VENs should be notified.  The VTN then publishes the notification to each authorized VEN's private topic.

4. **Subscription enforcement**: The VTN **MUST** ensure that a VEN cannot subscribe to topics it is not authorized to access.  The VTN/broker **MUST** prevent VEN-A from subscribing to VEN-B's private topics.

### Example

Consider an event with `targets=["group1"]`, and two VENs — `venA` and `venB` — both with `targets=["group1"]` in their ven objects:

- The VTN exposes VEN-scoped event topics for both VENs:
  - `events/vens/venA/create`, `events/vens/venA/update`, `events/vens/venA/delete`
  - `events/vens/venB/create`, `events/vens/venB/update`, `events/vens/venB/delete`

- When the event is created, the VTN resolves the targets, determines both `venA` and `venB` should receive the notification, and publishes to both sets of topics.

- `venA` can only subscribe to `events/vens/venA/*` topics, and `venB` can only subscribe to `events/vens/venB/*` topics.  Neither can see the other's notifications.

## 9.3 Scope of Topics by Client Type

| Topic Type | Accessible By | Examples |
|---|---|---|
| All-objects topics (no VEN scope) | BL-clients only | `events/create`, `programs/update` |
| Program-scoped topics | BL-clients and VENs (per authorization) | `events/programs/44/create` |
| VEN-scoped topics | The specific VEN only | `events/vens/ven-abc-123/create` |

## 9.4 VTN Implementation Responsibilities

The specification defines the **policy** (what must happen) but not the **mechanism** (how to implement it).  VTN implementations are responsible for:

- **Target resolution**: When an object with targets is created/updated/deleted, resolving those targets to a set of VEN `venID`s that should receive the notification.

- **Per-VEN publishing**: Publishing the notification message to each authorized VEN's private topic(s).

- **Subscription enforcement**: Configuring and enforcing broker-level access controls so that VENs can only subscribe to their own topics.  The means by which the VTN/broker enforces this (e.g. broker authorization plugins, ACL configuration, JWT-based authorization) is an implementation detail not specified by OpenADR 3.

Formally: A VTN **MUST** ensure that object privacy is maintained for notifications delivered via messaging protocols.  A VTN **MUST** prevent a VEN from subscribing to topics that would expose objects the VEN is not authorized to access.

# 10. Comparison of REST API and Notifier Binding Subscription and Notification

The OpenADR 3 REST API provides endpoints to read, list, create, modify, and delete subscriptions, with resulting notifications transported via webhooks.

An OpenADR client may utilize REST subscriptions or a VTN-supported notifier binding (or both), but **there is no relationship or interaction between the notifications delivered by notifier bindings and REST subscriptions**.

Notifier binding subscriptions operate as described above, and REST subscriptions operate as defined in the OpenADR 3 specification.

That said, a BL-client **MAY** subscribe to operations on VTN `subscriptions` via the topic name endpoints and subsequently receive notifications about operations on those subscriptions.

| Feature | Webhook (REST Subscriptions) | Messaging Protocol Notifiers |
|---|---|---|
| Object targets / filtering | Supported via subscription targets | Supported via VEN-scoped topics |
| READ operation notifications | Supported | Not supported |
| Custom callback URL | Yes | N/A — messages delivered via broker |
| Firewall traversal | Requires inbound HTTP | Client-initiated outbound connection |
| Multiple subscribers | Each needs own subscription | Many clients subscribe to same topic |
| Object privacy | Enforced at notification time | Enforced via per-VEN topics and broker ACLs |

---

# Part II: MQTT Notifier Binding

# 11. MQTT Binding Overview

The MQTT binding defines and specifies how the OA3 notifier framework maps to the MQTT messaging protocol.  This is the first non-webhook notifier binding specified for OpenADR 3.

## 11.1 MQTT Background

"MQTT is the most commonly used messaging protocol for the Internet of Things (IoT).  MQTT stands for MQ Telemetry Transport.  The protocol is a set of rules that defines how IoT devices can publish and subscribe to data over the Internet.  MQTT is used for messaging and data exchange between IoT and industrial IoT (IIoT) devices, such as embedded devices, sensors, industrial PLCs, etc.  The protocol is event-driven and connects devices using the publish/subscribe (Pub/Sub) pattern.  The sender (Publisher) and the receiver (Subscriber) communicate via Topics and are decoupled from each other.  The connection between them is handled by the MQTT broker.  The MQTT broker filters all incoming messages and distributes them correctly to the Subscribers."

References:
- [MQTT protocol version specifications](https://mqtt.org/mqtt-specification/)
- [MQTT client library implementations](https://en.wikipedia.org/wiki/Comparison_of_MQTT_implementations)
- [MQTT Version 3.1.1 specification](https://docs.oasis-open.org/mqtt/mqtt/v3.1.1/mqtt-v3.1.1.html)
- [MQTT Version 5.0 specification](https://docs.oasis-open.org/mqtt/mqtt/v5.0/mqtt-v5.0.html)

## 11.2 Protocol and Transport Requirements

- MQTT brokers and clients **MUST** support MQTT version 3.1.1 (or later)
- MQTT brokers and clients **SHOULD** support MQTT version 5.0 (or later)
- MQTT connections **SHOULD** be over TLS (`mqtts`)

# 12. MQTT Authentication

Each notifier binding returned in the response to `GET /notifiers` **MUST** provide the client with the `authentication` `method` (if any) and any additional details the client requires.

For the MQTT notifier, the following authentication methods are defined.  A VTN's MQTT broker **MUST** support one of these methods:

## 12.1 ANONYMOUS

VTNs **MAY** support unauthenticated MQTT broker connections.  This might be appropriate for VTNs designed to operate only within a private network (e.g. a home's LAN), or for a VTN providing public information such as tariffs.

A client connecting to a VTN's MQTT broker supporting anonymous connections needs no further details:

```json
{"method": "ANONYMOUS"}
```

## 12.2 OAUTH2_BEARER_TOKEN

OA3 specifies VTN authorization of REST API clients via the OAuth2 Client Credential Flow, resulting in the client obtaining an `access_token`.  VTNs **MAY** support MQTT broker authentication using this same token.

The MQTT protocol definition predates OAuth2, so the use of OAuth2 is not defined by the MQTT standard.  However, many MQTT brokers provide implementation-specific support.  The common convention is that the OAuth2 token is provided by the client as the `password` for the MQTT connect operation.

The required `username` varies between broker implementations.  Research indicates the `username` is typically either a fixed/static string defined by the broker, or the `clientID` used when obtaining the OAuth2 token.

The MQTT notifier binding specifies the broker's required `username` as follows:

- If the value is `"{clientID}"` (note the curly-brace characters), the client should provide its `clientID` value as the `username`
- For any other value, the client should use that string as-is

Examples:

```json
{"method": "OAUTH2_BEARER_TOKEN", "username": "{clientID}"}
```

```json
{"method": "OAUTH2_BEARER_TOKEN", "username": "oauth2"}
```

## 12.3 CERTIFICATE

Many MQTT brokers support authentication via mutual TLS (mTLS), which requires the client to have its own certificate.  The MQTT broker obtains the identity of the connecting client via the CN within the certificate provided by the client.

A VTN **MAY** support MQTT broker authentication via mTLS.  If so, the VTN's response to `GET /notifiers` **MUST** provide the required certificate authority (CA), client certificate, and client certificate key:

```json
{
  "method": "CERTIFICATE",
  "caCert": "string containing the Certificate Authority certificate",
  "clientCert": "string containing the client certificate",
  "clientKey": "string containing the client private key"
}
```

# 13. MQTT Authorization

A VTN **SHOULD** configure and enforce access to the MQTT broker's topics in accordance with its security and access policy (e.g. via the broker's authorization API), and this is out-of-band with respect to OA3 itself.

# 14. Client Connection to VTN's MQTT Broker

After obtaining the VTN's MQTT broker connection details via the REST `GET /notifiers` request, the OA3 client uses the MQTT protocol to create a connection to the broker, typically via an existing MQTT client library.

The MQTT connect operation may require authentication, as specified by the `authentication` `method` in the notifier binding response.

Implementation notes:

- Many MQTT client libraries accept an optional **keepalive** parameter on the connect call, which may be useful for detecting termination of the connection (in addition to standard TCP connection termination mechanisms).
- Many MQTT client libraries accept an optional **callback function** that is invoked on termination/disconnection, or may be configured to reconnect automatically.

# 15. Subscribing to MQTT Topics

After obtaining a connection to the MQTT broker and obtaining the topic names of interest (via the OA3 REST endpoints detailed in Section 8), an OpenADR client uses the MQTT protocol to:

1. Subscribe to the topics of choice

MQTT client libraries provide a subscribe call that takes a topic parameter and typically a callback function parameter that is invoked when a message is received.

The MQTT client can set the QoS (Quality of Service) level for topic subscriptions.

# 16. Receiving Notifications via MQTT

After a successful MQTT subscribe, the client will receive messages published to the topic.

The format and content of the notification messages will be the same as the existing notification JSON response format specified in OpenADR 3 (which is also the same as what is returned when the object is obtained via an OA3 REST `GET` request).

### 16.1 Retained Messages

MQTT supports the **retained flag** for messages.  It is expected (but not required) that VTN implementations of the MQTT binding will publish messages with the retain flag set to `true`, so that connecting VENs will receive the "last" message published to the topic.

Formally: A VTN **MAY** specify the retain flag as either `true` or `false` when publishing notification messages to a topic.

### 16.2 Reconnection and State Recovery

Clients that want to obtain the current state of an OA3 object upon (re)connection to the VTN **SHOULD** query (`GET`) those objects using the OA3 REST API, and not rely on retained messages or other implementation-specific features of a notifier binding.

# 17. MQTT Notification Sequence

The following sequence diagram illustrates the complete flow for a VEN receiving notifications via MQTT:

```
VEN                     VTN REST API              VTN MQTT Broker
 |                          |                          |
 |--- GET /notifiers ------>|                          |
 |<-- Notifier bindings ----|                          |
 |    (including MQTT)      |                          |
 |                          |                          |
 |--- GET /notifiers/mqtt/  |                          |
 |    topics/programs ----->|                          |
 |<-- Topic names ----------|                          |
 |                          |                          |
 |--- Connect (with auth, if required) --------------->|
 |--- Subscribe to topic name(s) --------------------->|
 |                          |                          |
 |<------------ Notification of change of state -------|
 |<------------ Notification of change of state -------|
 |                          |                          |
```

---

# Part III: Design Considerations for Future Notifier Bindings

# 18. Websocket Notifier — A Thought Experiment

This section explores whether a notification transport based on websockets might be supported by the notifier framework.  This is **not** a proposal to add a websocket notifier, but a thought experiment to validate that the framework design is amenable to protocols beyond topic-based messaging.

## 18.1 Websockets vs. Pub/Sub Messaging Protocols

"A WebSocket is a persistent connection between a client and server.  WebSockets provide a bidirectional, full-duplex communications channel that operates over HTTP through a single TCP/IP socket connection.  At its core, the WebSocket protocol facilitates message passing between a client and server."

In contrast to messaging protocols like MQTT and Kafka, websockets:

- Do not define the messages that a client and server exchange
- Do not directly support the publish-subscribe pattern
- Are not based on named topics
- Have no concept of a message broker

## 18.2 Conceptual Design

### Websocket Notifier Binding

The `GET /notifiers` response would include a binding for a `WEBSOCKET` notifier:

```json
{
  "WEBHOOK": true,
  "WEBSOCKET": {
    "URIS": ["wss://ws.vtn.company.com:8080"],
    "serialization": "JSON"
  }
}
```

As with all notifiers, the websocket notifier provides the `URIS` the client would use to establish a websocket connection to the VTN.

### Topic Name Endpoints

A websocket notifier would have no concept of, or need for, topics.  Therefore the topic name endpoints would not be specified or supported for a websocket notifier binding.

### Subscriptions

A VTN-client utilizing websocket notifications has a unique and specific channel for notifications that is not shared with other clients.  Therefore, notifications via websocket would be conceptually more similar to OA3 notifications via webhook.

Clients utilizing the websocket notifier would need to explicitly indicate to the VTN which `objectTypes` and operations they wish to receive notifications for.  The subscription operation for websockets would occur within the websocket connection itself, as "subscription command messages" sent by the client to the VTN.

### Authentication

Websockets are created via HTTP(S) requests, so a VTN supporting a websocket notifier would almost certainly utilize the client's existing OA3 OAuth2 `access_token` to authenticate websocket creation requests.

## 18.3 Conclusion

This design sketch gives confidence that the notifications framework would support a websocket notifier, and potentially other notification mechanisms conceptually similar to websockets (e.g. [Mercure](https://mercure.rocks/spec) or [Server-Sent Events](https://html.spec.whatwg.org/multipage/server-sent-events.html)).

The key insight from this exercise is that the notifier framework design successfully separates the protocol-agnostic discovery and configuration layer (the `/notifiers` endpoints) from the protocol-specific subscription and notification mechanisms, making it extensible to fundamentally different transport paradigms.

# 19. Kafka Notifier — Considerations

While a Kafka binding has not been specified, the framework design accommodates it.  Key considerations:

- Kafka topic names are **flat**, not hierarchical — unlike MQTT, Kafka does not support wildcard topic subscriptions.  A Kafka binding may therefore not provide the `ALL` topic name; clients would need to subscribe to each operation topic individually.

- Kafka is actually a distributed log store, not a messaging system, although it acts similarly to a message queue.  Kafka topics often provide access to previous messages depending on the retention policy.

- A Kafka notifier binding would follow the same pattern as MQTT: provide connection details in `GET /notifiers`, and topic names via the `/notifiers/kafka/topics/...` endpoints.

---

# Part IV: API Reference Summary

# 20. OpenAPI Schema Objects

The following schema objects are defined in the OpenADR 3.1.0 OpenAPI specification to support the notifier framework:

### notifiersResponse

The top-level response object from `GET /notifiers`.  Contains:

- **`WEBHOOK`** (boolean, required): Currently **MUST** be `true`
- **`MQTT`** (`mqttNotifierBindingObject`, optional): Present if VTN supports MQTT notifications

### mqttNotifierBindingObject

Details of the MQTT binding:

- **`URIS`** (array of strings, required): MQTT broker connection URIs
- **`serialization`** (string enum, required): Currently always `JSON`
- **`authentication`** (oneOf, optional): One of:
  - `mqttNotifierAuthenticationAnonymous`
  - `mqttNotifierAuthenticationOauth2BearerToken`
  - `mqttNotifierAuthenticationCertificate`

### notifierTopicsResponse

The response object from topic name endpoints:

- **`topics`** (`notifierOperationsTopics`): The topic names object

### notifierOperationsTopics

Topic names for operations on subscribe-able objects:

- **`CREATE`** (string, optional): Topic path for CREATE operations.  Not provided for a specific object ID (e.g. until a program is created, clients cannot request notification of its creation).
- **`UPDATE`** (string, required): Topic path for UPDATE operations
- **`DELETE`** (string, required): Topic path for DELETE operations
- **`ALL`** (string, optional): Topic path for ALL operations, if supported by the VTN

# 21. REST Endpoint Summary

| Endpoint | Description |
|---|---|
| `GET /notifiers` | List all notifier bindings supported by the server |
| `GET /notifiers/mqtt/topics/programs` | Topic names for operations on all programs |
| `GET /notifiers/mqtt/topics/programs/{programID}` | Topic names for operations on a specific program |
| `GET /notifiers/mqtt/topics/events` | Topic names for operations on all events |
| `GET /notifiers/mqtt/topics/programs/{programID}/events` | Topic names for operations on a program's events |
| `GET /notifiers/mqtt/topics/reports` | Topic names for operations on all reports |
| `GET /notifiers/mqtt/topics/subscriptions` | Topic names for operations on all subscriptions |
| `GET /notifiers/mqtt/topics/vens` | Topic names for operations on all VENs |
| `GET /notifiers/mqtt/topics/vens/{venID}` | Topic names for operations on a specific VEN |
| `GET /notifiers/mqtt/topics/resources` | Topic names for operations on all resources |
| `GET /notifiers/mqtt/topics/vens/{venID}/events` | Topic names for operations on a VEN's events |
| `GET /notifiers/mqtt/topics/vens/{venID}/programs` | Topic names for operations on a VEN's programs |
| `GET /notifiers/mqtt/topics/vens/{venID}/resources` | Topic names for operations on a VEN's resources |

---

# Part V: Design Decisions and Rationale

# 22. Key Design Decisions

This section documents the significant design decisions made during the development of the additional notifications feature, along with the reasoning behind each.

### 22.1 Protocol Agnosticism in the REST API

**Decision:** The `/notifiers` REST endpoints are protocol-agnostic; protocol-specific details are encapsulated within each binding's response object.

**Rationale:** This enables the framework to support fundamentally different notification transports (message brokers, websockets, SSE, etc.) without changes to the core API structure.  The discovery mechanism (`GET /notifiers`) works identically regardless of the notification protocol.

### 22.2 Topic Names Provided by the VTN, Not Hardcoded

**Decision:** Clients obtain topic names from the VTN via REST endpoints rather than deriving them from a fixed naming convention.

**Rationale:** This gives VTN implementations flexibility in topic naming and organization.  It also allows VTNs to provide client-specific topic names (since the request includes the client's auth token), enabling per-client topic isolation for privacy and security.

### 22.3 No OA3 REST Subscription Required for Messaging Protocol Notifications

**Decision:** Clients using messaging protocol notifiers subscribe directly via the protocol (e.g. MQTT subscribe), without creating an OA3 REST subscription object.

**Rationale:** OA3 REST subscriptions include semantics (object-targets, callback URLs) that don't map naturally to messaging protocol subscriptions.  Forcing clients to create REST subscriptions for MQTT notifications would add complexity without benefit, and would conflate two different subscription models.

### 22.4 Object Privacy via VEN-scoped Topics

**Decision:** Rather than attempting to replicate the webhook notifier's per-subscription target filtering within the messaging protocol, object privacy is achieved through VEN-scoped topics — topic paths that include a VEN's `venID` and to which only that VEN may subscribe.

**Rationale:** Messaging protocols publish to topics; per-subscriber message filtering is not a native capability of pub/sub systems.  The alternative — creating per-subscriber filtered views at the broker level — would negate the architectural benefits of pub/sub.  VEN-scoped topics provide a natural mapping: the VTN resolves an object's targets to the set of authorized VENs, then publishes the notification to each VEN's private topic.  The broker enforces subscription-level access control, preventing unauthorized VENs from subscribing to other VENs' topics.  This approach separates the privacy policy (defined by the spec) from the enforcement mechanism (an implementation detail of the VTN/broker).

### 22.5 Exclusion of Fine-grained Object-Target Filtering

**Decision:** Within a VEN's scoped topics, further filtering by object-targets is not supported.  A VEN subscribed to its event topics receives all events it is authorized to see, and must filter client-side for events of specific interest.

**Rationale:** The VEN-scoped topics ensure a VEN only sees objects it is authorized to access (satisfying object privacy), but do not provide finer-grained selection within that authorized set.  Clients requiring more granular target-based filtering should use the webhook notifier binding, which supports the full subscription targets mechanism.

### 22.6 Exclusion of READ Operation Notifications

**Decision:** Messaging protocol notifiers only support notifications for `CREATE`, `UPDATE`, and `DELETE` operations, not `READ`.

**Rationale:** READ operations do not change object state and generating notifications for them would create excessive noise.  Clients needing READ notifications can use the webhook notifier.

### 22.7 Webhooks as a Notifier Binding

**Decision:** The existing webhook notification mechanism is represented as the `WEBHOOK` notifier binding within the framework.

**Rationale:** This unifies all notification methods under a single discovery mechanism.  It also future-proofs the API for a potential OpenADR version that doesn't require webhooks — clients would discover this via `GET /notifiers`.

### 22.8 Broker and Topic Terminology Scoped to Bindings

**Decision:** Concepts like "broker" and "topic" are not part of the core notifier framework vocabulary; they appear only in bindings that use those concepts.

**Rationale:** Non-broker-based notification transports (websockets, SSE) have no concept of brokers or topics.  Keeping this terminology in the bindings ensures the framework remains genuinely protocol-agnostic.

### 22.9 MQTT Version Requirements

**Decision:** MQTT version 3.1.1 (or later) is required (**MUST**); MQTT version 5.0 (or later) is recommended (**SHOULD**).

**Rationale:** The initial design required MQTT 5 exclusively, but this was relaxed to allow MQTT 3.1.1 as the minimum.  Many existing IoT deployments and resource-constrained devices use MQTT 3.1.1, and requiring version 5 would have excluded these implementations.  MQTT 5 is recommended because it provides important features — including enhanced authentication, shared subscriptions, message expiry, and topic aliases — that are valuable for robust OA3 notification implementations, but these features are not strictly necessary for basic operation.

---

# References

- [OpenADR 3.1.0 Definition][Definition]
- [OpenADR 3.1.0 User Guide][User_Guide]
- [OpenADR 3.1.0 OpenAPI Specification][YAML]
- [IETF RFC 2119 "Key words for use in RFCs to Indicate Requirement Levels"](https://www.ietf.org/rfc/rfc2119.txt)
- [Uniform Resource Identifier (URI): Generic Syntax (RFC 3986)](https://www.rfc-editor.org/rfc/rfc3986)
- [MQTT Version 5.0 Specification](https://docs.oasis-open.org/mqtt/mqtt/v5.0/mqtt-v5.0.html)
- [MQTT Protocol Specifications](https://mqtt.org/mqtt-specification/)
- [MQTT Client Library Implementations](https://en.wikipedia.org/wiki/Comparison_of_MQTT_implementations)
- [AsyncAPI Reference](https://www.asyncapi.com/docs/reference)

This document's use of requirements keywords in **BOLD** is modeled after, and consistent with, IETF RFC 2119.

[Definition]: ../3.1.0/Definition.md
[User_Guide]: ../3.1.0/User_Guide.md
[YAML]: ../3.1.0/openadr3.yaml
