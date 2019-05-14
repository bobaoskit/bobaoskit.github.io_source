## bobaos.pub API

### bee-queue

To perform operation with KNX datapoints [bee-queue](https://github.com/bee-queue/bee-queue) job may sent to `config.ipc.request_channel` with data object, consist following fields:

* `method` is an API method.
* `payload` depends on method. It may be datapoint id, array of ids, value, or null.

Request:

```
{
  "method": "get parameter byte",
  "payload": [1,2,3,4]
}
```

Response is processes by `queue.on("job succeeded")`:

```
{
  "method": "success",
  "payload": [1,3,5,7]
}
```

In case of error, `method` field in response will contain "error" string and payload will describe error.

For service requests channel `config.ipc.service_channel` is used.

Bee-queue client is implemented in [bobaos.sub](https://github.com/bobaoskit/bobaos.sub) js package.

### redis pub/sub channels

Since version 2.0.8 this feature is supported to simplify job adding.

bobaos.pub listens request and service channels and on incoming request it add job to corresponding queue. So, request can be done just by publishing json message to redis channel.

Only difference from bee-queue request is that request should contain `response_channel` field, so bobaos.pub will know where to send response. This channels may be randomly generated, then subscribed, and after request was sent and response came, client may unsubscribe from this channel.

### service methods

#### ping

**payload**: null

If bobaos.pub service is running, true will be returned as a response payload.

#### get sdk state

**payload**: null

Response payload: "ready"/"stopped".

#### reset

**payload**: null

#### get version

**payload**: null

Get version of bobaos.pub npm package.

### baos methods

#### get description

**payload**: null/number/array of numbers

Returns descriptions for given object ids. If payload is null, return description for all configured datapoints.

Response payload example: 

```text
{
  "valid": "true",
  "id": "1",
  "length": "2",
  "dpt": "dpt9",
  "flag_priority": "low",
  "flag_communication": "true",
  "flag_read": "false",
  "flag_write": "true",
  "false": "false",
  "flag_readOnInit": "false",
  "flag_update": "true",
  "flag_transmit": "true",
  "value": "23.6",
  "raw": "DJw="
}
```

#### get value

**payload**: number/array of numbers

Returns values for given object ids. Sends data to BAOS via UART. When received, value is saved in redis, so can be got also by `get stored value` method. It is recommended to get values for multiple datapoints at once to reduce UART message count.

Response payload example:

```text
{
  "id":1,
  "value":23.6,
  "raw":"DJw="
}
```

#### get stored value

Returns value as `get value` method, but without data sending by UART. Values is taken from redis database.

#### poll values

**payload**: null

Returns values for all configured datapoints.  May take a while to accomplish. Response payload is the same as for `get values`/`get stored values`.

#### set value

**payload**: Object/Array of objects

Request payload object should be one of following: `{ id: <Number>, value: <Value> }`, `{ id: <Number>, raw: <String> }`, where raw is base64 encoded buffer. 

If payload is array of objects, it is possible to combine value types, so following payload is valid:

```text
[
  {id: 2, value: true}
  {id: 3, raw: "AA=="}
]
```

After values are sent to bus, bobaos.pub broadcasts them via pubsub channel `bobaos_bcast`.

#### read value

**payload**: number/array of numbers

#### get server item

**payload**: null/number/array of numbers

If payload is null, all server items to be returned in response.

Response payload example:

```text
[
  {
    "id": 1,
    "value": [ 0, 0, 197, 8, 0, 3],
    "raw": "AADFCAAD"
  },
  {
    "id": 2,
    "value": [ 16 ],
    "raw": "EA=="
  },
  {
    "id": 3,
    "value": [ 18 ],
    "raw": "Eg=="
  }
]
```

#### set programming mode

**payload**: 1/0/true/false

#### get programming mode

**payload**: null

Response payload: true/false

#### get parameter byte

**payload**: number/array of numbers

Response payload example: 

```text
[ 0, 1, 2, 5, 7, 3]
```

### broadcasted events

On incoming datapoint values or on service events(bus connected/disconnected/baos in programming mode) bobaos.pub sends json message to redis channel configured in `config.ipc.broadcast_channel`.

Message consists `method` and `payload` fields. 

Method may be `"datapoint value"/"server item"/"sdk state"`.

Payload for `"datapoint value"/"server item"` is a value object for datapoint/server item or array of such values. So, type checking is required. Payload for `"sdk state"` is one of following: `"ready"/"stop"`.

