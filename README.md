# Channel Channels HTTP Go Library

[![Build Status](https://github.com/duolacloud/channel-go/workflows/Tests/badge.svg)](https://github.com/duolacloud/channel-go/actions?query=workflow%3ATests+branch%3Amaster) [![Coverage Status](https://coveralls.io/repos/github/duolacloud/channel-go/badge.svg?branch=master)](https://coveralls.io/github/duolacloud/channel-go?branch=master) [![Go Reference](https://pkg.go.dev/badge/github.com/duolacloud/channel-go/logo.svg)](https://pkg.go.dev/github.com/duolacloud/channel-go)

The Golang library for interacting with the Channel Channels HTTP API.

This package lets you trigger events to your client and query the state of your channels. When used with a server, you can validate Channels webhooks and authenticate `private-` or `presence-` channels.

Register for free at <https://webpubsub.com/channels> and use the application credentials within your app as shown below.

## Supported Platforms

* Go - supports **Go 1.11 or greater**.

## Table of Contents

- [安装](#installation)
- [快速入门](#getting-started)
- [配置](#configuration)
  - [附加选项](#additional-options)
  - [Google App Engine](#google-app-engine)
- [使用](#usage)
  - [触发事件](#triggering-events)
  - [Authenticating Channels](#authenticating-channels)
  - [应用状态](#application-state)
  - [Webhook 验证](#webhook-validation)
- [支持的特性](#feature-support)
- [Developing the Library](#developing-the-library)
  - [运行测试](#running-the-tests)
- [许可](#license)

## 安装

```sh
$ go get github.com/duolacloud/channel-go
```

## 快速入门

```go
package main

import (
  "github.com/webpubsub/webpubsub-go"
)

func main(){
    // instantiate a client
    webpubsubClient := webpubsub.Client{
        AppID:   "APP_ID",
        Key:     "APP_KEY",
        Secret:  "APP_SECRET",
        Cluster: "APP_CLUSTER",
    }

    data := map[string]string{"message": "hello world"}

    // trigger an event on a channel, along with a data payload
    err := webpubsubClient.Trigger("my-channel", "my_event", data)

    // All trigger methods return an error object, it's worth at least logging this!
    if err != nil {
        panic(err)
    }
}
```

## 配置

The easiest way to configure the library is by creating a new `Channel` instance:

```go
webpubsubClient := webpubsub.Client{
    AppID:   "APP_ID",
    Key:     "APP_KEY",
    Secret:  "APP_SECRET",
    Cluster: "APP_CLUSTER",
}
```

### 附加选项

#### 通过 URL 实例化

```go
webpubsubClient := webpubsub.ClientFromURL("http://<key>:<secret>@api.<cluster>.webpubsub.com/apps/app_id")
```

注意: URL根据创建应用程序所在的群集而异:

```
http://key:secret@api.eu.webpubsub.com/apps/app_id
http://key:secret@api.ap1.webpubsub.com/apps/app_id
```

#### 通过环境变量实例化

```go
webpubsubClient := webpubsub.ClientFromEnv("CHANNEL_URL")
```

This is particularly relevant if you are using Channel Channels as a Heroku add-on, which stores credentials in a `"WEBPUBSUB_URL"` environment variable.

#### HTTPS

To ensure requests occur over HTTPS, set the `Secure` property of a `webpubsub.Client` to `true`.

```go
webpubsubClient.Secure = true
```

This is `false` by default.

#### 请求超时

If you wish to set a time-limit for each HTTP request, create a `http.Client` instance with your specified `Timeout` field and set it as the Channels instance's `Client`:

```go
httpClient := &http.Client{Timeout: time.Second * 3}

webpubsubClient.HTTPClient = httpClient
```

If you do not specifically set a HTTP client, a default one is created with a timeout of 5 seconds.

#### 更改 Host

Changing the `webpubsub.Client`'s `Host` property will make sure requests are sent to your specified host.

```go
webpubsubClient.Host = "foo.bar.com"
```

By default, this is `"api.webpubsub.com"`.

#### 更改集群(Cluster)

Setting the `webpubsub.Client`'s `Cluster` property will make sure requests are sent to the cluster where you created your app.

*NOTE! If `Host` is set then `Cluster` will be ignored.*

```go
webpubsubClient.Cluster = "eu" // in this case requests will be made to api.eu.webpubsub.com.
```
#### 端到端加密

This library supports end to end encryption of your private channels. This means that only you and your connected clients will be able to read your messages. Channel cannot decrypt them. You can enable this feature by following these steps:

1. You should first set up Private channels. This involves [creating an authentication endpoint on your server](https://webpubsub.com/docs/authenticating_users).

2. Next, generate a 32 byte master encryption key, base64 encode it and store
   it securely.

   This is secret and you should never share this with anyone. Not even Channel.

   To generate a suitable key from a secure random source, you could use:

   ```bash
   openssl rand -base64 32
   ```

3. Pass the encoded key when constructing your webpubsub.Client

   ```go
   webpubsubClient := webpubsub.Client{
       AppID:                    "APP_ID",
       Key:                      "APP_KEY",
       Secret:                   "APP_SECRET",
       Cluster:                  "APP_CLUSTER",
       EncryptionMasterKeyBase64 "<output from command above>",
   }
   ```
4. Channels where you wish to use end to end encryption should be prefixed with `private-encrypted-`.

5. Subscribe to these channels in your client, and you're done! You can verify it is working by checking out the debug console on the https://dashboard.webpubsub.com/ and seeing the scrambled ciphertext.

**Important note: This will not encrypt messages on channels that are not prefixed by private-encrypted-.**

### Google App Engine

As of version 1.0.0, this library is compatible with Google App Engine's urlfetch library. Pass in the HTTP client returned by `urlfetch.Client` to your Channel Channels initialization struct.

```go
package helloworldapp

import (
    "appengine"
    "appengine/urlfetch"
    "fmt"
    "github.com/duolacloud/channel-go"
    "net/http"
)

func init() {
    http.HandleFunc("/", handler)
}

func handler(w http.ResponseWriter, r *http.Request) {
    c := appengine.NewContext(r)
    urlfetchClient := urlfetch.Client(c)

    webpubsubClient := webpubsub.Client{
        AppID:      "APP_ID",
        Key:        "APP_KEY",
        Secret:     "APP_SECRET",
        HTTPClient: urlfetchClient,
    }

    webpubsubClient.Trigger("my-channel", "my_event", map[string]string{"message": "hello world"})

    fmt.Fprint(w, "Hello, world!")
}
```

## 使用

### 触发事件

It is possible to trigger an event on one or more channels. Channel names can contain only characters which are alphanumeric, `_` or `-` and have to be at most 200 characters long. Event name can be at most 200 characters long too.

#### Custom Types

**webpubsub.Event**

```go
type TriggerParams struct {
    SocketID *string
    Info     *string
}
```

注意: `Info` is part of an [experimental feature](https://webpubsub.com/docs/lab#experimental-program).

#### Single channel

##### `func (c *Client) Trigger`

| Argument |Description   |
| :-: | :-: |
| channel `string` | The name of the channel you wish to trigger on. |
| event `string` | The name of the event you wish to trigger. |
| data `interface{}` | The payload you wish to send. Must be marshallable into JSON. |

###### 示例

```go
data := map[string]string{"hello": "world"}
webpubsubClient.Trigger("greeting_channel", "say_hello", data)
```

##### `func (c *Client) TriggerWithParams`

Allows additional parameters to be included as part of the request body.
The complete list of parameters are documented [here](https://webpubsub.com/docs/channels/library_auth_reference/rest-api#request).

| Argument |Description   |
| :-: | :-: |
| channel `string` | The name of the channel you wish to trigger on. |
| event `string` | The name of the event you wish to trigger. |
| data `interface{}` | The payload you wish to send. Must be marshallable into JSON. |
| params `TriggerParams` | Any additional parameters. |

| Return Value | Description |
| :-: | :-: |
| channels `TriggerChannelsList` | A struct representing channel attributes for the requested `TriggerParams.Info` |
| err `error` | Any errors encountered|

###### Example

```go
data := map[string]string{"hello": "world"}
socketID := "1234.12"
attributes := "user_count"
params := webpubsub.TriggerParams{SocketID: &socketID, Info: &attributes}
channels, err := webpubsubClient.TriggerWithParams("presence-chatroom", "say_hello", data, params)

// channels => &{Channels:map[presence-chatroom:{UserCount:4}]}
```

#### Multiple channels

##### `func (c. *Client) TriggerMulti`

| Argument | Description |
| :-: | :-: |
| channels `[]string` | A slice of channel names you wish to send an event on. The maximum length is 10. |
| event `string` | As above. |
| data `interface{}` | As above. |

###### Example

```go
webpubsubClient.TriggerMulti([]string{"a_channel", "another_channel"}, "event", data)
```

##### `func (c. *Client) TriggerMultiWithParams`

| Argument | Description |
| :-: | :-: |
| channels `[]string` | A slice of channel names you wish to send an event on. The maximum length is 10. |
| event `string` | As above. |
| data `interface{}` | As above. |
| params `TriggerParams` | As above. |

| Return Value | Description |
| :-: | :-: |
| channels `TriggerChannelsList` | A struct representing channel attributes for the requested `TriggerParams.Info` |
| err `error` | Any errors encountered|

###### Example

```go
data := map[string]string{"hello": "world"}
socketID := "1234.12"
attributes := "user_count"
params := webpubsub.TriggerParams{SocketID: &socketID, Info: &attributes}
channels, err := webpubsubClient.TriggerMultiWithParams([]string{"presence-chatroom", "presence-notifications"}, "event", data, params)

// channels => &{Channels:map[presence-chatroom:{UserCount:4} presence-notifications:{UserCount:31}]}
```

#### Batches

##### `func (c. *Client) TriggerBatch`

| Argument | Description |
| :-: | :-: |
| batch `[]Event` | A list of events to publish |

| Return Value | Description |
| :-: | :-: |
| batch `TriggerBatchChannelsList` | A struct representing channel attributes for the requested `TriggerParams.Info` |
| err `error` | Any errors encountered|

###### Custom Types

**webpubsub.Event**

```go
type Event struct {
    Channel  string
    Name     string
    Data     interface{}
    SocketID *string
    Info     *string
}
```

Note: `Info` is part of an [experimental feature](https://webpubsub.com/docs/lab#experimental-program).

###### Example

```go
socketID := "1234.12"
attributes := "user_count"
batch := []webpubsub.Event{
  { Channel: "a-channel", Name: "event", Data: "hello world" },
  { Channel: "presence-b-channel", Name: "event", Data: "hi my name is bob", SocketID: &socketID, Info: &attributes },
}
response, err := webpubsubClient.TriggerBatch(batch)

for i, attributes := range response.Batch {
    if attributes.UserCount != nil {
        fmt.Printf("channel: %s, name: %s, user_count: %d\n", batch[i].Channel, batch[i].Name, *attributes.UserCount)
    } else {
        fmt.Printf("channel: %s, name: %s\n", batch[i].Channel, batch[i].Name)
    }
}

// channel: a-channel, name: event
// channel: presence-b-channel, name: event, user_count: 4
```

### Authenticating Channels

Application security is very important so Channel Channels provides a mechanism for authenticating a user’s access to a channel at the point of subscription.

This can be used both to restrict access to private channels, and in the case of presence channels notify subscribers of who else is also subscribed via presence events.

This library provides a mechanism for generating an authentication signature to send back to the client and authorize them.

For more information see our [docs](http://webpubsub.com/docs/authenticating_users).

#### Private channels

##### `func (c *Client) AuthenticatePrivateChannel`

| Argument | Description |
| :-: | :-: |
| params `[]byte` | The request body sent by the client |

| Return Value | Description |
| :-: | :-: |
| response `[]byte` | The response to send back to the client, carrying an authentication signature |
| err `error` | Any errors generated |

###### 示例

```go
func channelAuth(res http.ResponseWriter, req *http.Request) {
    params, _ := ioutil.ReadAll(req.Body)
    response, err := webpubsubClient.AuthenticatePrivateChannel(params)
    if err != nil {
        panic(err)
    }

    fmt.Fprintf(res, string(response))
}

func main() {
    http.HandleFunc("/channel/auth", channelAuth)
    http.ListenAndServe(":5000", nil)
}
```

###### Example (JSONP)

```go
func channelJsonpAuth(res http.ResponseWriter, req *http.Request) {
    var (
        callback, params string
    )

    {
        q := r.URL.Query()
        callback = q.Get("callback")
        if callback == "" {
            panic("callback missing")
        }
        q.Del("callback")
        params = []byte(q.Encode())
    }

    response, err := webpubsubClient.AuthenticatePrivateChannel(params)
    if err != nil {
        panic(err)
    }

    res.Header().Set("Content-Type", "application/javascript; charset=utf-8")
    fmt.Fprintf(res, "%s(%s);", callback, string(response))
}

func main() {
    http.HandleFunc("/channel/auth", channelJsonpAuth)
    http.ListenAndServe(":5000", nil)
}
```

#### Authenticating presence channels

Using presence channels is similar to private channels, but in order to identify a user, clients are sent a user_id and, optionally, custom data.

##### `func (c *Client) AuthenticatePresenceChannel`

| Argument | Description |
| :-: | :-: |
| params `[]byte` | The request body sent by the client |
| member `webpubsub.MemberData` | A struct representing what to assign to a channel member, consisting of a `UserID` and any custom `UserInfo`. See below |

###### Custom Types

**webpubsub.MemberData**

```go
type MemberData struct {
    UserID   string
    UserInfo map[string]string
}
```

###### Example

```go
params, _ := ioutil.ReadAll(req.Body)

presenceData := webpubsub.MemberData{
    UserID: "1",
    UserInfo: map[string]string{
        "twitter": "jamiepatel",
    },
}

response, err := webpubsubClient.AuthenticatePresenceChannel(params, presenceData)

if err != nil {
    panic(err)
}

fmt.Fprintf(res, response)
```

### Application state

This library allows you to query our API to retrieve information about your application's channels, their individual properties, and, for presence-channels, the users currently subscribed to them.

#### Get the list of channels in an application

##### `func (c *Client) Channels`

| Argument | Description |
| :-: | :-: |
| params `ChannelsParams` | The query options. The field `FilterByPrefix` will filter the returned channels. To get the number of users subscribed to a presence-channel, specify an the `Info` field with value `"user_count"`. Pass in `nil` if you do not wish to specify any query attributes. |

| Return Value | Description |
| :-: | :-: |
| channels `ChannelsList` | A struct representing the list of channels. See below. |
| err `error` | Any errors encountered|

###### Custom Types

**webpubsub.ChannelsParams**

```go
type ChannelsParams struct {
    FilterByPrefix *string
    Info           *string
}
```

**webpubsub.ChannelsList**

```go
type ChannelsList struct {
    Channels map[string]ChannelListItem
}
```

**webpubsub.ChannelsListItem**

```go
type ChannelListItem struct {
    UserCount int
}
```

###### Example

```go
prefixFilter := "presence-"
attributes := "user_count"
params := webpubsub.ChannelsParams{FilterByPrefix: &prefixFilter, Info: &attributes}
channels, err := webpubsubClient.Channels(params)

// channels => &{Channels:map[presence-chatroom:{UserCount:4} presence-notifications:{UserCount:31}]}
```

#### Get the state of a single channel

##### `func (c *Client) Channel`

| Argument | Description |
| :-: | :-: |
| name `string` | The name of the channel |
| params `ChannelParams` | The query options. The field `Info` can have comma-separated values of `"user_count"`, for presence-channels, and `"subscription_count"`, for all-channels. To use the `"subscription_count"` value, first check the "Enable subscription counting" checkbox in your App Settings on [your Channel Channels dashboard](https://dashboard.webpubsub.com). Pass in `nil` if you do not wish to specify any query attributes. |

| Return Value | Description |
| :-: | :-: |
| channel `Channel` | A struct representing a channel. See below. |
| err `error` | Any errors encountered |

###### Custom Types

**webpubsub.ChannelParams**

```go
type Channel struct {
    Info *string
}
```

**webpubsub.Channel**

```go
type Channel struct {
    Name              string
    Occupied          bool
    UserCount         int
    SubscriptionCount int
}
```

###### Example

```go
attributes := "user_count,subscription_count"
params := webpubsub.ChannelParams{Info: &attributes}
channel, err := client.Channel("presence-chatroom", params)

// channel => &{Name:presence-chatroom Occupied:true UserCount:42 SubscriptionCount:42}
```

#### Get a list of users in a presence channel

##### `func (c *Client) GetChannelUsers`

| Argument | Description |
| :-: | :-: |
| name `string` | The channel name |

| Return Value | Description |
| :-: | :-: |
| users `Users` | A struct representing a list of the users subscribed to the presence-channel. See below |
| err `error` | Any errors encountered. |

###### Custom Types

**webpubsub.Users**

```go
type Users struct {
    List []User
}
```

**webpubsub.User**

```go
type User struct {
    ID string
}
```

###### Example

```go
users, err := webpubsubClient.GetChannelUsers("presence-chatroom")

// users => &{List:[{ID:13} {ID:90}]}
```

### Webhook 验证

On your [dashboard](http://app.webpubsub.com), you can set up webhooks to POST a payload to your server after certain events. Such events include channels being occupied or vacated, members being added or removed in presence-channels, or after client-originated events. For more information see <https://webpubsub.com/docs/webhooks>.

This library provides a mechanism for checking that these POST requests are indeed from Channel, by checking the token and authentication signature in the header of the request.

##### `func (c *Client) Webhook`

| Argument | Description |
| :-: | :-: |
| header `http.Header` | The header of the request to verify |
| body `[]byte` | The body of the request |

| Return Value | Description |
| :-: | :-: |
| webhook `*webpubsub.Webhook` | If the webhook is valid, this method will return a representation of that webhook that includes its timestamp and associated events. If invalid, this value will be `nil`. |
| err `error` | If the webhook is invalid, an error value will be passed. |

###### Custom Types

**webpubsub.Webhook**

```go
type Webhook struct {
    TimeMs int
    Events []WebhookEvent
}
```

**webpubsub.WebhookEvent**

```go
type WebhookEvent struct {
    Name     string
    Channel  string
    Event    string
    Data     string
    SocketID string
}
```

###### Example

```go
func channelWebhook(res http.ResponseWriter, req *http.Request) {
    body, _ := ioutil.ReadAll(req.Body)
    webhook, err := webpubsubClient.Webhook(req.Header, body)
    if err != nil {
        fmt.Println("Webhook is invalid :(")
    } else {
        fmt.Printf("%+v\n", webhook.Events)
    }
}
```

## Feature Support

Feature                                    | Supported
-------------------------------------------| :-------:
Trigger event on single channel            | *&#10004;*
Trigger event on multiple channels         | *&#10004;*
批量触发事件                                 | *&#10004;*
Excluding recipients from events           | *&#10004;*
Fetching info on trigger                   | *&#10004;*
认证 private channels                       | *&#10004;*
认证 presence channels                      | *&#10004;*
获取应用的所有 channels 列表                  | *&#10004;*
Get the state of a single channel          | *&#10004;*
Get a list of users in a presence channel  | *&#10004;*
WebHook 验证                                | *&#10004;*
Heroku add-on support                      | *&#10004;*
Debugging & Logging                        | *&#10004;*
Cluster configuration                      | *&#10004;*
Timeouts                                   | *&#10004;*
HTTPS                                      | *&#10004;*
HTTP Proxy configuration                   | *&#10008;*
HTTP KeepAlive                             | *&#10008;*

## Helper Functionality

These are helpers that have been implemented to to ensure interactions with the HTTP API only occur if they will not be rejected e.g. [channel naming conventions](https://webpubsub.com/docs/channels/using_channels/channels#channel-naming-conventions).

Helper Functionality                      | Supported
----------------------------------------- | :-------:
Channel name validation                   | &#10004;
Limit to 10 channels per trigger          | &#10004;
Limit event name length to 200 chars      | &#10004;

## Developing the Library

Feel more than free to fork this repo, improve it in any way you'd prefer, and send us a pull request :)

### Running the tests

```sh
$ go test
```

## License

This code is free to use under the terms of the MIT license.
