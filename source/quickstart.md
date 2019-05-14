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

Start bobaos.pub

```text
~ > bobaos-pub

  ()--()     hello, friend
    \"/_     here comes bobaos
     '  )    learn and enjoy
       ~

User config file does not exist
Default congig file created at
/home/pi/.config/bobaos/pub.json
...
```

Default serialport to be opened is `/dev/ttyS1`. If you need other serialport, then close application(`Ctrl-C`) and replace default value in config file located at `~/.config/bobaos/pub.json`.

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
bobaos> set 2: 1
20:27:06:239,    id: 2, value: true, raw: [AQ==]
bobaos> set [2: 0, 26: "hello, friend"]
23:41:05:171,    id: 2, value: false, raw: [AA==]
23:41:05:171,    id: 26, value: hello, friend, raw: [aGVsbG8sIGZyaWVuZAA=]
```

I would advice to use **multiple value sending** as a cool feature of BAOS protocol.

* Get values:

```text
bobaos> get 1 2 3 11 26
23:42:26:212,    id: 1, value: 21.7, raw: [DD0=]
23:42:26:213,    id: 2, value: false, raw: [AA==]
23:42:26:213,    id: 3, value: false, raw: [AA==]
23:42:26:213,    id: 11, value: 0, raw: [AA==]
23:42:26:214,    id: 26, value: hello, friend, raw: [aGVsbG8sIGZyaWVuZAA=]
```

It is possible to get value stored in RAM:

```text
bobaos> stored 1 2 3 11 26
23:42:53:630,    id: 1, value: 21.7, raw: [DD0=]
23:42:53:631,    id: 2, value: false, raw: [AA==]
23:42:53:631,    id: 3, value: false, raw: [AA==]
23:42:53:632,    id: 11, value: 0, raw: [AA==]
23:42:53:632,    id: 26, value: hello, friend, raw: [aGVsbG8sIGZyaWVuZAA=]

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
bobaos> watch [1: hide, 2: cyan, 3: bgRed]
datapoint 1 value is now in hide
datapoint 2 value is now in cyan
datapoint 3 value is now in bgRed
bobaos> unwatch 1 2 3
datapoint 1 value is now in default color
datapoint 2 value is now in default color
datapoint 3 value is now in default color
```

This feature is helpful when there is a need to catch by eyes certain datapoints from common output.

Coloring is done by [colors](https://github.com/Marak/colors.js) package. So list of supported colors can be found on it's github/npm page.

To hide datapoint use `watch 1: hide`/`watch 1: hidden` commands. To show again, apply watch command with color value differs from `hide/hidden` or use `unwatch`. 

Please keep in mind that  bobaos.tool creates config file `~/.config/bobaos/tool.json` and stores color settings there.

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

For detailed documentation look **API** page.

Following js script may be used as a simple template to start with:

```js
const Bobaos = require("bobaos.sub");

const bobaos = Bobaos();

bobaos.on("ready", _ => {
  console.log("bobaos ready");
});

const processBaosValue = payload => {
  if (Array.isArray(payload)) {
    return payload.forEach(processBaosValue);
  }

  let { id, value, raw } = payload;
  console.log(`hello, ${id}, ${value}, raw`);
};

bobaos.on("datapoint value", processBaosValue);
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
