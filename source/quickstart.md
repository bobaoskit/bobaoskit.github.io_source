# Quick Start

## hardware configuration

Assuming that Raspberry Pi 3 is used.

The Raspberry Pi 3 does not use the correct baud rate in Raspian out of the box, so kBerry will not work. To fix this, the overlay pi3-miniuart-bt-overlay must be activated.


```text
sudo sh -c "echo dtoverlay=pi3-miniuart-bt >>/boot/config.txt"
```

Open /boot/cmdline.txt and remove console=ttyAMA0,115200 from the file For Raspbian Jessie and newer the entry is console=serial0,115200

```text
sudo nano /boot/cmdline.txt
```

Make sure the user has the rights to access the ttyAMA0

```text
# Check the group of the device
ls -l /dev/ttyAMA0
```

```text
crw-rw---- 1 root dialout 204, 64 Aug  4 11:33 /dev/ttyAMA0
```

```text
# Add the user to the group seen above
sudo usermod -a -G dialout pi
```

After reboot proceed to next steps.

```text
sudo reboot
```

## Node.js

Install [nodejs](https://nodejs.org)

```text
curl -sL https://deb.nodesource.com/setup_10.x | sudo -E bash -
sudo apt-get install -y nodejs
```

[More info](https://github.com/nodesource/distributions/blob/master/README.md)

## Redis

bobaos.pub broadcasts events over publish/subscribe and uses queues backed by [Redis](https://redis.io/). So, install it at first and start service.

```text
sudo apt install redis-server
sudo systemctl daemon-reload
sudo systemctl enable redis.service
sudo systemctl start redis.service
```

Check with `redis-cli` if it is running. If not, reboot and try testing with `redis-cli` again.

## bobaos.pub

Now, let install working service for BAOS sdk.

```text
sudo npm i -g bobaos.pub --unsafe-perm
```

After that, edit config file located at **/usr/lib/node_modules/bobaos.pub/config.json** and replace default **/dev/ttyS1** to **/dev/ttyAMA0**. 

Create service file for systemd:

```text
sudo nano /etc/systemd/system/bobaos_pub.service
```

Use following content for this file:

```text
[Unit]
Description=PubSub service running on nodejs
After=redis.service

[Service]
User=pi
ExecStart=/usr/bin/env bobaos-pub
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Reload systemd daemon, enable and start service.

```text
sudo systemctl daemon-reload
sudo systemctl enable bobaos_pub.service
sudo systemctl start bobaos_pub.service
```

Proceed to final steps.

## bobaos.tool

To control datapoint values from shell, bobaos.tool package is used.

```text
sudo npm i -g bobaos.tool
```

Run and ping sdk:

```text
~$ bobaos-tool
hello, friend
connecting to /var/run/myps/myipc.sock
connected to ipc, still not subscribed to channels
ready to send requests
bobaos> help
:: To work with datapoints
::  set ( 1: true | [2: "hello", 3: 42] )
::> get ( 1 2 3 | [1, 2, 3] )
::  stored ( 1 2 3 | [1, 2, 3] )
::> read ( 1 2 3 | [1, 2, 3] )
::  description ( * | 1 2 3 | [1, 2, 3] )

:: Helpful in bus monitoring:
::> watch ( 1: red | [1: red, 2: green, 3: underline] )
::  unwatch ( 1 2 3 | [1, 2, 3] )

:: BAOS services:
::> getbyte ( 1 2 3 | [1, 2, 3] )
::  getitem ( * | 1 2 ..17 | [1, 2, ..17] )
::> progmode ( ? | true/false/1/0 )

:: General:
::> ping
::  state
::> reset
::  help
bobaos> ping
ping: true
bobaos>
```


Some command examples:

* Programming mode

First **cool feature**: there is no need to press programming button.

To get/set programming mode following command is used:

```text
bobaos> progmode ?
BAOS module in programming mode: false
bobaos> progmode 1
BAOS module in programming mode: true
bobaos> progmode 0
BAOS module in programming mode: false
bobaos> progmode true
BAOS module in programming mode: true
bobaos> progmode false
BAOS module in programming mode: false
bobaos>
```

* Get description for datapoints:

```text
bobaos> description 1 2 3
#1: length = 2, dpt = dpt9,  prio: low  flags: [C-WTU]
#2: length = 1, dpt = dpt5,  prio: low  flags: [C-WTU]
#3: length = 1, dpt = dpt5,  prio: low  flags: [C-WTU]
bobaos>
```

To list all datapoints, run:

```text
bobaos> description *
```

* Set value:

```text
bobaos> set 2: 0
20:27:06:239,    id: 2, value: 0, raw: [0]
bobaos> set [2: 0, 3: 128, 999: "hello, friend"]
20:28:48:586,    id: 2, value: 0, raw: [0]
20:28:48:592,    id: 3, value: 128, raw: [128]
20:28:48:593,    id: 999, value: "hello, friend, raw: [34,104,101,108,108,111,44,32,102,114,105,101,110,100]
```

I would advice to use **multiple value sending** as a cool feature of BAOS protocol.

* Get values:

```text
bobaos> get 1 2 3 4 5 101 102 103
20:38:02:250,    id: 1, value: 23, raw: [12,126]
20:38:02:260,    id: 2, value: 0, raw: [0]
20:38:02:268,    id: 3, value: 128, raw: [128]
20:38:02:277,    id: 4, value: 0, raw: [0]
20:38:02:286,    id: 5, value: 0, raw: [0]
20:38:02:295,    id: 101, value: false, raw: [0]
20:38:02:304,    id: 102, value: false, raw: [0]
20:38:02:312,    id: 103, value: false, raw: [0]
```

It is possible to get value stored in RPi's RAM:

```text
bobaos> stored  1 2 3
20:39:54:963,    id: 1, value: 22.9, raw: [12,121]
20:39:54:974,    id: 2, value: 0, raw: [0]
20:39:54:982,    id: 3, value: 128, raw: [128]
```

This feature can help to get datapoint values and don't send a lot of requests over UART.

* Read values:

```text
bobaos> read 1 2 3 105
20:42:12:540,    id: 1, value: 22.9, raw: [12,121]
20:42:12:642,    id: 105, value: false, raw: [0]
bobaos>
```

In this example only two datapoint indications came from bus. That happened because there is no KNX objects configured with flag **Read** for datapoints 2 and 3(these are control). Datapoint 1 is a temperature sensor and datapoint 105 is binary output feedback.

If group objects configured right, but there is no read response, check update flag in BAOS ETS config.

```text
bobaos> description 1 2 3 105
#1: length = 2, dpt = dpt9,  prio: low  flags: [C-WTU]
#2: length = 1, dpt = dpt5,  prio: low  flags: [C-WTU]
#3: length = 1, dpt = dpt5,  prio: low  flags: [C-WTU]
#105: length = 1, dpt = dpt1,  prio: low  flags: [C-WTU]
bobaos>
```

In this example, **U** flag assigned for all of these objects.

* Monitoring datapoint values with colors:

```text
bobaos> watch 1: green
datapoint 1 value is now in green
bobaos> watch [1: green, 105: underline]
datapoint 1 value is now in green
datapoint 105 value is now in underline
bobaos> read 1 105
20:50:18:077,    id: 1, value: 22.9, raw: [12,121]
20:50:18:123,    id: 105, value: false, raw: [0]
bobaos> unwatch 1 105
datapoint 1 value is now in default color
datapoint 105 value is now in default color
bobaos> read 1 105
20:50:41:901,    id: 1, value: 23, raw: [12,126]
20:50:41:944,    id: 105, value: false, raw: [0]
bobaos>
```

This feature is helpful when there is a need to catch by eyes certain datapoints from common output.

* Other BAOS services: server items, parameter bytes

```text
bobaos> getitem *
[ { id: '1', value: [ 0, 0, 197, 8, 0, 3 ], raw: 'AADFCAAD' },
  { id: '2', value: [ 16 ], raw: 'EA==' },
  { id: '3', value: [ 18 ], raw: 'Eg==' },
  { id: '4', value: [ 0, 197 ], raw: 'AMU=' },
  { id: '5', value: [ 0, 197 ], raw: 'AMU=' },
  { id: '6', value: [ 8, 5 ], raw: 'CAU=' },
  { id: '7', value: [ 16 ], raw: 'EA==' },
  { id: '8', value: [ 0, 197, 1, 1, 118, 183 ], raw: 'AMUBAXa3' },
  { id: '9', value: [ 42, 39, 107, 116 ], raw: 'KidrdA==' },
  { id: '10', value: [ 1 ], raw: 'AQ==' },
  { id: '11', value: [ 0, 250 ], raw: 'APo=' },
  { id: '12', value: [ 0, 0 ], raw: 'AAA=' },
  { id: '13', value: [ 1 ], raw: 'AQ==' },
  { id: '14', value: [ 0, 250 ], raw: 'APo=' },
  { id: '15', value: [ 0 ], raw: 'AA==' },
  { id: '16', value: [ 32 ], raw: 'IA==' },
  { id: '17', value: [ 1 ], raw: 'AQ==' } ]
bobaos>
```

```text
bobaos> getbyte 1 2 3
[ 1, 3, 5 ]
bobaos>
```
To get parameter byte values.

* General commands: **ping, state, reset**

```text
bobaos> ping
ping: true
bobaos> state
get sdk state: ready
bobaos> reset
reset request sent
bobaos> state
get sdk state: ready
bobaos>
```

## bobaos.sub

This library control BAOS datapoints from nodejs app.

```text
npm i --save bobaos.sub
```

Then in script:

```js
const BobaosSub = require('bobaos.sub');
 
let my = BobaosSub();
 
my.on("connect", _ => {
  console.log("connected to ipc, still not subscribed to channels");
});
 
my.on("ready", async _ => {
  try {
    console.log("hello, friend");
    console.log("ping:", await my.ping());
    console.log("get sdk state:", await my.getSdkState());
    console.log("get value:", await my.getValue([1, 107, 106]));
    console.log("get stored value:", await my.getValue([1, 107, 106]));
    console.log("get server item:", await my.getServerItem("SerialNumber"));
    console.log("set value:", await my.setValue({id: 103, value: 0}));
    console.log("read value:", await my.readValue([1, 103, 104, 105]));
    console.log("get programming mode:", await my.getProgrammingMode());
    console.log("set programming mode:", await my.setProgrammingMode(true));
    console.log("get parameter byte", await my.getParameterByte([1, 2, 3, 4]));
    console.log("reset", await my.reset());
  } catch(e) {
    console.log("err", e.message);
  }
});
 
my.on("datapoint value", payload => {
  console.log("broadcasted datapoint value: ", payload);
});
 
my.on("server item", payload => {
  console.log("broadcasted server item: ", payload);
});
 
my.on("sdk state", payload => {
  console.log("broadcasted sdk state: ", payload);
});
```

## bobaos.ws

This package is used to add WebSocket support

### Installation

```text
sudo npm i -g bobaos.ws
```

You can run it:

```text
pi@pi:~$ bobaos-ws
Starting bobaos.ws
bobaos sdk: ready
```

Try to send from any websocket client following request on port 49190

```json
{"request_id": 42, "method": "ping", "payload": null}
```

If everything is ok, create service file `/etc/systemd/system/bobaos_ws.service:

```text
[Unit]
Description=WebSocket server for bobaos.pub
After=bobaos_pub.service

[Service]
User=pi
ExecStart=/usr/bin/env bobaos-ws
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Reload systemd, start and enable daemon:

```text
sudo systemctl daemon-reload
sudo systemctl start bobaos_ws.service
sudo systemctl enable bobaos_ws.service
```

After that, to configure `bobaos.ws` edit `/usr/lib/node_modules/bobaos.ws/config.json` file.

### Protocol

#### Overview


Published messages to websocket are serialized JSON objects, containing required fields `request_id, method, payload`.
For broadcasted messages `request_id` field is not used.

For outgoing requests:

* `request_id` is used to receive response exactly to this request. If it is not defined then you will not receive response from server.
* `method` is an API method.
* `payload` depends on method. It may be datapoint id, array of ids, value, or null.

Request:

```json
{
  "request_id": 420,
  "method": "get parameter byte",
  "payload": [1,2,3,4]
}
```

Response:

```json
{
  "response_id":420,
  "method":"success",
  "payload":[1,3,5,7]
}
```

#### API methods

* Method: `ping`. Payload: `null`. 

    Returns `true/false` depending on running state of bobaos.pub service.
    
* Method: `get sdk state`. Payload: `null`.
    
    Check if connected to BAOS.

* Method: `reset`. 

    Restart SDK. Reload datapoints/server items.

* Method: `get description`. Payload: `null/id/[id1, .. idN]`.

    Get description for datapoints with given ids. If payload is null then description for all datapoints will be returned.
   
* Method: `get value`. Payload: `id/[id1, .. idN]`.

    Get value for datapoints with given ids.
    
* Method: `get stored value`. Payload: `id/[id1, .. idN]`.

    Get value stored in bobaos.pub service. Do not send any data via UART.
    
* Method: `set value`. Payload: `{id: i, value: v}/[{id: i1, value: v1}, .. {id: iN, value: vN}]`

    Set datapoints value and send to bus. Keep in mind that after successful request, new datapoint value will be broadcasted to all websocket clients, including sender.
    
* Method: `read value`. Payload: `id/[id1, .. idN]`.

    Send read requests to KNX bus. Keep in mind that datapoint object should have active Update flag.
    If reading was successful then datapoint value will be broadcasted.
    
* Method: `get server item`. Payload: `null/id/[id1, .. idN]`.

    Get server items with given id/ids. To get all server items use `null` as a payload.
    
* Method: `set programming mode`. Payload: `1/0/true/false`.

    Send to BAOS request to go into programming mode.
    
* Method: `get programming mode`. Payload: `null`.

    Return current value of ProgrammingMode sever item.
    
* Method: `get parameter byte`. Payload: `id/[id1, .. idN]`.

    Get parameter byte values for given ids.
    
#### Communication example

```text
{"request_id": 42, "method": "ping", "payload": null}
{"response_id":42,"method":"success","payload":true}
{"request_id": 42, "method": "get sdk state", "payload": null}
{"response_id":42,"method":"success","payload":"ready"}
{"request_id": 42, "method": "reset", "payload": null}
{"method":"sdk state","payload":"stop"}
{"method":"sdk state","payload":"ready"}
{"response_id":42,"method":"success","payload":null}
{"method":"datapoint value","payload":{"id":1,"value":23.6,"raw":"DJw="}}
{"request_id": 42, "method": "get description", "payload": 1}
{"response_id":42,"method":"success","payload":{"valid":"true","id":"1","length":"2","dpt":"dpt9","flag_priority":"low","flag_communication":"true","flag_read":"false","flag_write":"true","false":"false","flag_readOnInit":"false","flag_update":"true","flag_transmit":"true","value":"23.6","raw":"DJw="}}
{"request_id": 42, "method": "get value", "payload": 1}
{"response_id":42,"method":"success","payload":{"id":1,"value":23.6,"raw":"DJw="}}
{"method":"datapoint value","payload":{"id":1,"value":23.6,"raw":"DJw="}}
{"request_id": 42, "method": "get stored value", "payload": 1}
{"response_id":42,"method":"success","payload":{"id":1,"value":23.6,"raw":"DJw="}}
{"request_id": 42, "method": "set value", "payload": {"id": 103, "value": false}}
{"method":"datapoint value","payload":[{"id":103,"value":false,"raw":"AA=="}]}
{"response_id":42,"method":"success","payload":[{"id":103,"value":false,"raw":"AA=="}]}
{"method":"datapoint value","payload":{"id":107,"value":false,"raw":"AA=="}}
{"method":"datapoint value","payload":{"id":1,"value":23.6,"raw":"DJw="}}
{"request_id": 42, "method": "set value", "payload": [{"id": 102, "value": true}, {"id": 103, "value": true}]}
{"method":"datapoint value","payload":[{"id":102,"value":true,"raw":"AQ=="},{"id":103,"value":true,"raw":"AQ=="}]}
{"response_id":42,"method":"success","payload":[{"id":102,"value":true,"raw":"AQ=="},{"id":103,"value":true,"raw":"AQ=="}]}
{"method":"datapoint value","payload":{"id":107,"value":true,"raw":"AQ=="}}
{"request_id": 42, "method": "read value", "payload": [1, 105, 106]}
{"response_id":42,"method":"success","payload":null}
{"method":"datapoint value","payload":{"id":1,"value":22.7,"raw":"DG8="}}
{"method":"datapoint value","payload":{"id":105,"value":false,"raw":"AA=="}}
{"method":"datapoint value","payload":{"id":106,"value":true,"raw":"AQ=="}}
{"request_id": 42, "method": "get server item", "payload": [1, 2, 3]}
{"response_id":42,"method":"success","payload":[{"id":1,"value":[0,0,197,8,0,3],"raw":"AADFCAAD"},{"id":2,"value":[16],"raw":"EA=="},{"id":3,"value":[18],"raw":"Eg=="}]}
{"method":"datapoint value","payload":{"id":1,"value":22.9,"raw":"DHk="}}
{"method":"datapoint value","payload":{"id":1,"value":23.2,"raw":"DIg="}}
``` 

Thank you for attention.
