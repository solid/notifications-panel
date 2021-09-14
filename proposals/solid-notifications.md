# Solid Notifications

**Status**: Proposal

## Introduction

Much of the Solid Protocol is built on a RESTful API. This allows Solid to make use of well-understood interactions with resources on the Web, and this satisfies many use cases for client applications in the Solid ecosystem. However, for interactive client applications with multiple participants, chat-apps being just one such example, a RESTful API is too limited, as clients need to rely on polling in order to discover changes in the underlying data.

A notification API for Solid would complement the core RESTful API so that clients can listen for updates to particular resources. In addition, a notification-based mechanism may reduce latency and load on a Pod server.

One challenge with (near) real-time notifications on the Web involves the diversity of technologies available. WebSockets and EventSource APIs are two options that are widely supported by browsers; ReadableStreams are becoming more widely supported. Furthermore, there are other, more asynchronous models that are relevant in certain cases: WebHooks, WebSub and Linked Data Notifications are three examples. A major goal of the model described in this document is to make it possible for client applications to navigate the diversity of these technologies while also being open to new notification mechanisms as they emerge.

Finally, a critically important part of this model is security. Any notification-based API not only needs the same level of security that is found in the RESTful API but also the authorization mechanism needs to be consistent across these endpoints.

## Use Cases

1. **Immediate notification of data changes** - for many clients, it is important to know when a resource state changes. This includes collaborative editing apps, chat apps or any client that expects the underlying data to change frequently based on updates made by others.
1. **Change frequency** - sometimes a client doesn't need to be alerted for every change. If a client only needs infrequent updates, for example, every 5 minutes, it should be possible to avoid floods of data that might otherwise place a higher burden on the client application. Ideally, a client should be able to define what that aggregation window would be: for some clients, 10 seconds might be appropriate; for others, several minutes might be more appropriate.
1. **Filtered notifications** - certain categories of notifications may be more or less interesting to clients. For example, a client may not care about UPDATE operations or may only care about notifications for resources of certain types.
1. **Time-bound subscriptions** - occasionally, a client only needs a subscription to be in effect for a limited period of time. While a client may easily close a WebSocket itself, subscriptions for other, more asynchronous forms of notifications, such as LDN alerts or WebHook operations, may be more difficult to terminate.
1. **Avoiding missing updates** - for WebSockets and protocols that rely on an active, live connection to a notification server, the notification protocol needs to make sure that clients do not miss notifications in the event of a dropped connection.
1. **Protocol negotiation** - a given Solid Notification server may support certain technologies (e.g. WebSockets and LDN) but not others (e.g. EventSource and WebSub). Likewise, a client may not support the same set of protocols that are implemented by a server. As such, there needs to be a mechanism for clients and servers to agree on a mutually supported protocol. In addition to simply determining the set of protocols that work for client and server, there may be particular features (e.g. notification aggregation, notification filtering) that are required for the client. This could be used to further filter the protocol selection.
1. **Security** - a client should not be able to subscribe to resources to which it does not have read access.

## Terminology

This document uses terms from the Solid Protocol specification, including "data pod". This document also uses terms from the OAuth2 specification, including "resource server", "authorization server" and "client", as well as terms from the WebSub specification, including "topic".

In addition, the following terms are defined:

**Notification Gateway API** -- an HTTP endpoint at which a client can negotiate an acceptable notification subscription location

**Notification Subscription API** -- an HTTP endpoint at which a client can initiate a subscription to notification events for a particular set of resources

**Solid Server Metadata Resource** - an RDF document that includes metadata about the Solid server

## High-level Flow

The following diagram shows the high level interactions involved in this flow. How a client retrieves an access token for step 5 is outside the scope of this document.

![High level flow](solid-notifications-flow.png)

## Discovery

For any resource in a data pod, a client can discover the associated notification gateway API by fetching the Solid Server Metadata Resource. This resource can be retrieved by appending /.well-known/solid to the base URL of the data pod.

```http
GET /.well-known/solid
```

The server `MUST` be capable of serializing this resource as Turtle or JSON-LD.

A sample representation of this resource might include the following:

```http
Content-Type: application/ld+json

{
    "@context": ["https://www.w3.org/ns/solid/notification/v1"],
    "notification_endpoint": "https://gateway.example",
    ...
}
```

**Note**: the notification panel should define a particular JSON-LD @context for use with JSON-LD serializations.

**Note**: the use of a well-known resource is based on suggestions to avoid too many link headers, but that is not a settled issue.

## Protocol

The notification protocol involves two endpoints: a notification gateway API and a notification subscription API. A server `MUST` support JSON-LD in these interactions.

### Notification Gateway
Once the location of a notification gateway is discovered, an application will negotiate for a particular notification channel, based on acceptable features and protocols.

The `type` array advertises which subscription channels a client is able to understand. The `features` array advertises which notification features are required by the client. The server selects one of the proposed subscription types and responds with a JSON-LD document that includes the selected protocol, the endpoint for negotiating a subscription and the supported features offered by that protocol.

Authentication is not required at this endpoint.

A sample interaction is described below. The type and feature names are included as examples.

#### Request

```http
POST /gateway
Content-Type: application/ld+json

{
    "@context": ["https://www.w3.org/ns/solid/notification/v1"],
    "type": ["WebSocketSubscription2021", "WebHookSubscription2021"],
    "features": ["state", "rate"]
}
```

#### Response

```http
HTTP/2
Content-Type: application/ld+json

{
    "@context": ["https://www.w3.org/ns/solid/notification/v1"],
    "type": "WebSocketSubscription2021",
    "endpoint": "https://websocket.example/subscription",
    "features": ["state", "rate", "expiration", "feature1", "feature2"]
}
```

**Note**: the features array may not be needed as part of this negotiation.

### Notification Subscription

The details of subscribing to a particular endpoint depend on the technology used, but all will follow a similar pattern. For WebSockets, this involves sending an authenticated subscription request to the endpoint retrieved from the gateway service.

The client sends a JSON-LD payload to this endpoint via POST. The only required fields in this interaction are type and topic. The type field MUST contain the type of subscription being requested. The topic field MUST contain the URL of the resource a client wishes to subscribe to changes.

All other fields are defined by the particular subscription type. Some common features are described in the Features section of this document.

An example POST request using a DPoP bound access token is below:

```http
POST /subscription
Authorization: DPoP <token>
DPoP: <proof>
Content-Type: application/ld+json

{
    "@context": ["https://www.w3.org/ns/solid/notification/v1"],
    "type": "WebSocketSubscription2021",
    "topic": "https://pod.example/resource",
    "state": "opaque-state",
    "expiration": "2021-09-21T12:37:15Z",
    "rate": "PT10s"
}
```

A successful response will contain a URL that can be used directly with a JavaScript client

```http
HTTP/2
Content-Type: application/ld+json

{
    "@context": "https://www.w3.org/ns/solid/notification/v1",
    "type": "WebSocketSubscription2021",
    "endpoint": "wss://websocket.example/?auth=Ys3KiUq"
}
```

In JavaScript, a client can use the data in the response to establish a connection to the WebSocket endpoint.

```javascript
const ws = new WebSocket(endpoint, type)
```

A client will define how to handle events, especially the onclose and onmessage events.

```javascript
ws.onclose(evt => reconnect(evt));
ws.onmessage(evt => console.log("Message received: ", evt))
```

## Authentication and Authorization

The Subscription API requires authorization. This document does not define the specific technology used to authorize requests; rather, it defers to the Solid Protocol sections on Authentication and Authorization. It is out of scope for this document to describe how a client fetches an authorization token. Solid-OIDC is one example of an authentication mechanism that could be used with Solid Notifications.

Resource-based Access controls MUST be enforced for every subscription. A client MUST be permitted to read the resource to which it subscribes.

## Notification Data Model

The content of a Solid Notification is defined in terms of a W3C Activity Streams 2.0 document. As such, JSON-LD 1.0 is a required baseline serialization format, though other formats are not prohibited. JSON-LD version 1.1 is recommended when using JSON-LD serializations.

```javascript
{
   "@context":[
      "https://www.w3.org/ns/activitystreams",
      "https://www.w3.org/ns/solid/notification/v1"
   ],
   "id":"urn:uuid:<uuid>",
   "type":[
     "Update"
   ],
   "actor":[
      "<WebID>"
   ],
   "object":{
      "id":"https://pod.inrupt.com/<user>/some-container/",
      "type":[
         "http://www.w3.org/ns/ldp#BasicContainer",
         "http://www.w3.org/ns/ldp#Resource",
         "http://example.com/SomeType"
      ]
   },
   "state": "1234-5678-90ab-cdef-12345678",
   "published":"2021-08-05T01:01:49.550044Z"
}
```

**Note**: the Solid JSON-LD context resource does not currently exist. An implementation may choose to augment the context definition.

The content of these serialized notifications may be extended, but the example here provides a minimal baseline that clients can expect.

## Subscription Types

This document defines four initial subscription types:

* `WebSocketSubscription2021`
* `EventSourceSubscription2021`
* `LinkedDataNotificationSubscription2021`
* `WebHookSubscription2021`

**Note**: the naming convention used here follows a pattern used by the Verifiable Credentials community, making it possible to distinguish between different versions of these subscription types. Other naming conventions can achieve the same end.

### WebSocketSubscription2021

The `WebSocketSubscription2021` type defines the following properties:

* `endpoint`: this property is used in the body of the subscription response. The value of this property MUST be a URI, using the wss scheme. A JavaScript client would use the entire value as input to the WebSocket constructor.

### EventSourceSubscription2021

The `EventSourceSubscription2021` type defines the following properties:

* `endpoint`: this property is used in the body of the subscription response. The value of this property MUST be a URI, using the https scheme. A JavaScript client would use the entire value as input to the EventSource constructor.

### LinkedDataNotificationSubscription2021

The `LinkedDataNotificationSubscription2021` type defines the following properties:

* `inbox`: this property is used in the body of the subscription request. The value of this property MUST be a URI, using the https scheme. The inbox URI MUST conform to the receiver portion of the Linked Data Notification specification.
* `webid`: this property is used in the body of the subscription response. The value of this property MUST be an HTTPS URI conforming to the WebID draft specification. All LDN notifications will be sent to the defined inbox using the agent identity specified by this WebID. A resource owner should ensure that the agent identified in this property is permitted to send authenticated HTTP requests to the defined inbox.

### WebHookSubscription2021

The `WebHookSubscription2021` type defines the following properties:

* `target`: this property is used in the body of the subscription request. The value of this property MUST be a URI, using the https scheme.

## Features

Features allow clients to customize the details of a subscription. Some features are specific to particular subscription types while others may be used across all types. These shared features are listed as an initial baseline, though not all subscription types are required to implement these.

* `expiration`: an ISO 8601 datetime value indicating a proposed expiration point for a subscription. A server may choose another value.
* `state`: an opaque value representing the last known state of a resource. If the resource state is known to have changed when the client establishes a subscription, an initial notification must be sent to the client. This value should correspond to a resource's ETag header value.
* `rate`: an ISO 8601 duration value indicating the minimum amount of time to elapse between notifications sent to the client
* `accept`: a MIME-Type value that indicates the desired serialization of the notification. For instance, text/turtle may be acceptable.

Individual subscription types may define type-specific features.

**Note**: The list of common, shared features is not intended to be exhaustive.

## Extensions

There are two main axes of extension for this design: subscription type (e.g. `WebSocketSubscription2021` vs. `LinkedDataNotificationSubscription2021`) and notification features (e.g. `expiration` vs. `state`).

## References

* Solid Protocol: https://solidproject.org/TR/protocol
* Solid-OIDC: https://solid.github.io/solid-oidc/
* W3C Activity Streams 2.0: https://www.w3.org/TR/activitystreams-core/
* W3C JSON-LD 1.1: https://www.w3.org/TR/json-ld11/
* W3C Linked Data Notifications: https://www.w3.org/TR/ldn/
* W3C RDF 1.1 Concepts and Abstract Syntax: https://www.w3.org/TR/rdf11-concepts/
* W3C RDF 1.1 Turtle: https://www.w3.org/TR/turtle/
* W3C WebSub: https://www.w3.org/TR/websub/
* W3C WebID: https://www.w3.org/2005/Incubator/webid/spec/identity/
* The OAuth 2.0 Authorization Framework (RFC 6749): https://datatracker.ietf.org/doc/html/rfc6749
* OAuth 2.0 Demonstrating Proof-of-Possession at the Application Layer (DPoP): https://datatracker.ietf.org/doc/html/draft-ietf-oauth-dpop-03
* Web sockets: https://html.spec.whatwg.org/multipage/web-sockets.html

