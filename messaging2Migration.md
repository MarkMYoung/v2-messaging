# Messaging 2.0 Migration guide

## Assumptions

* You have both your [Voice and Messaging](https://app.bandwidth.com) & your [Phone Number Dashboard](https://dashboard.bandwidth.com) setup and configured.
* You have a storage solution for message recall that meets your needs.

### API Credentials
* API Credentials work the same way they do in the V1 Messaging API. Use your API Token and Secret with Basic Auth when making API requests to send messages. [See here for more details](accountCredentials.md).

## Key Differentiators

| Difference                                  | v1/messages                                                                                                                                                                                    | v2/messages                                                                                                                                                                                                         |
|:--------------------------------------------|:-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|:--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Message Storage                             | All messages are stored within the API and can be fetched                                                                                                                                      | Messages **are not** stored at all within the API and must be stored per your needs                                                                                                                                 |
| Outbound Message `callbackUrl`              | Messages could be sent with a combination of `callbackUrl` and `receiptRequested` in order to get status updates as the message progresses throughout the system                               | **ALL** messages must include the `applicationId` and all status events are automatically sent to the `callbackUrl` specified in the [application](https://dashboard.bandwidth.com/portal-v2/report/#application:/) |
| Creating message `http` code                | The API returns `201 - Created` when sending the message, however the message may still fail with an [error code](https://dev.bandwidth.com/ap-docs/methods/messages/getMessagesMessageId.html) | The API returns a `202 - Accepted` and will send a callback for [every event](./events/messageEvents.md) (*including delivered and error*)                                                                          |
| Creating message response                   | The API returns with the `{message-id}` in the `Location` Header                                                                                                                               | The API returns a JSON object that contains information about the request                                                                                                                   |
| Outbound message `to` value                 | Can only include a single phone number as a string                                                                                                                                             | Will accept both a string and an array. If an array with more than one phone number is sent, the message will be treated as a group message                                                                         |
| **Any** Message Callback                    | Message callbacks are sent as a single JSON object for both [sms and mms](https://dev.bandwidth.com/ap-docs/apiCallbacks/messagingEvents.html)                                                  | Message callbacks are sent as an array of objects for [every event](./events/messageEvents.md)                                                                                                                      |
| HTTP Errors                                 | See the [errors](https://dev.bandwidth.com/ap-docs/methods/messages/getMessagesMessageId.html) for possible responses                                                                           | See the [v2 errors](./codes.md) for possible responses                                                                                                                                                              |
| [Segment Count](concepts.md#segment-counts) | Messages sent or receved with the v1/messages API do not contain the number of segments in the callbacks                                                                                       | In the v2 API, the message segment is returned in the callbacks.                                                                                                                                                    |

### Sending Message

{% codetabs name="v1/messages", type="http" -%}
POST https://api.catapult.inetwork.com/v1/users/{userId}/messages HTTP/1.1
Content-Type: application/json; charset=utf-8
Authorization: {apiToken:apiSecret}

{
    "from" : "+12525089000",
    "to"   : "+15035555555",
    "text" : "Hello there from Bandwidth!"
}
{%- language name="v2/messages", type="http" -%}
POST https://messaging.bandwidth.com/api/v2/users/{accountId}/messages HTTP/1.1
Content-Type: application/json; charset=utf-8
Authorization: {apiToken:apiSecret}

{
    "to"            : ["+12345678902"],
    "from"          : "+12345678901",
    "text"          : "Hey, check this out!",
    "applicationId" : "93de2206-9669-4e07-948d-329f4b722ee2",
    "tag"           : "test message"
}
{%- endcodetabs %}

#### Responds

{% codetabs name="v1/messages response", type="http" -%}
HTTP/1.1 201 Created
Location: https://api.catapult.inetwork.com/v1/users/{userId}/messages/{message-id}
{%- language name="v2/messages response", type="http" -%}
Status: 202 Accepted
Content-Type: application/json; charset=utf-8

{
  "id"            : "14762070468292kw2fuqty55yp2b2",
  "time"          : "2016-09-14T18:20:16Z",
  "to"            : [
    "+12345678902",
    "+12345678903"
  ],
  "from"          : "+12345678901",
  "text"          : "Hey, check this out!",
  "applicationId" : "93de2206-9669-4e07-948d-329f4b722ee2",
  "tag"           : "test message",
  "owner"         : "+12345678901",
  "direction"     : "out",
  "segmentCount"  : 1
}
{%- endcodetabs %}

### Callback Example

{% codetabs name="v1/messages Callback", type="http" -%}
POST /your_url HTTP/1.1
Content-Type: application/json; charset=utf-8
User-Agent: BandwidthAPI/v1 ({CURRENT_BUILD_TIMESTAMP})

{
   "eventType"  : "sms",
   "direction"  : "out",
   "messageId"  : "{messageId}",
   "messageUri" : "https://api.catapult.inetwork.com/v1/users/{userId}/messages/{messageId}",
   "from"       : "+13233326955",
   "to"         : "+13865245000",
   "text"       : "Example",
   "time"       : "2012-11-14T16:13:06.076Z",
   "state"      : "sent"
}
{%- language name="v2/messages Callback", type="http" -%}
POST /your_url HTTP/1.1
Content-Type: application/json; charset=utf-8
User-Agent: bandwidth-api

[
  {
    "type"        : "message-received",
    "time"        : "2016-09-14T18:20:16Z",
    "description" : "Incoming message received",
    "to"          : "+12345678902",
    "message"     : {
      "id"            : "14762070468292kw2fuqty55yp2b2",
      "time"          : "2016-09-14T18:20:16Z",
      "to"            : ["+12345678902"],
      "from"          : "+12345678901",
      "text"          : "Hey, check this out!",
      "applicationId" : "93de2206-9669-4e07-948d-329f4b722ee2",
      "media"         : [
        "https://messaging.bandwidth.com/api/v2/users/{accountId}/media/14762070468292kw2fuqty55yp2b2/0/bw.png",
        "https://messaging.bandwidth.com/api/v2/users/{accountId}/media/14762070468292kw2fuqty55yp2b2/1/bandwidth_logo.png",
        "https://messaging.bandwidth.com/api/v2/users/{accountId}/media/14762070468292kw2fuqty55yp2b2/2/Bandwidth_Contact.png"
      ],
      "owner"         : "+12345678902",
      "direction"     : "in",
      "segmentCount"  : 1
    }
  }
]
{%- endcodetabs %}
