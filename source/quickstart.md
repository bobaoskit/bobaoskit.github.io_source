# Quick Start

## hardware configuration

Assuming that Raspberry Pi 3 is used.

The Raspberry Pi 3 does not use the correct baud rate in Raspian out of the box, so kBerry will not work. To fix this, the overlay pi3-miniuart-bt-overlay must be activated.


```
sudo sh -c "echo dtoverlay=pi3-miniuart-bt >>/boot/config.txt"
```

Open /boot/cmdline.txt and remove console=ttyAMA0,115200 from the file For Raspbian Jessie and newer the entry is console=serial0,115200

```
sudo nano /boot/cmdline.txt
```

Make sure the user has the rights to access the ttyAMA0

```
# Check the group of the device
ls -l /dev/ttyAMA0
```

```
crw-rw---- 1 root dialout 204, 64 Aug  4 11:33 /dev/ttyAMA0
```

```
# Add the user to the group seen above
sudo usermod -a -G dialout pi
```

After reboot proceed to next steps.

```
sudo reboot
```

## myps.broker

Bobaos.Kit communicates over publish/subscribe service, implemented in nodejs: myps(MyPubSub). So, install it at first and create a service.

```
sudo npm i -g myps.broker
```

Create service for systemd:

```
sudo nano /etc/systemd/system/myps.service
```

Add following content:

```
[Unit]
Description=PubSub service running on nodejs

[Service]
User=pi
ExecStart=/usr/bin/env myps-broker
Restart=on-failure
RestartSec=10
RuntimeDirectory=myps
RuntimeDirectoryMode=0775
# and now working dir.
# make sure it is located at `/run/${RuntimeDirectory}`
# which is in this example myps
WorkingDirectory=/run/myps

[Install]
WantedBy=multi-user.target
```

Service file contains **RuntimeDirectory** and **WorkingDirectory** config values, so with this service file, socket file **myipc.sock** can be found at **/var/run/myipc.sock**.

Enable and start service:

```
sudo systemctl daemon-reload
sudo systemctl enable myps.service
sudo systemctl start myps.service
```

Check status:

```
systemctl status myps.service
● myps.service - PubSub service running on nodejs
   Loaded: loaded (/etc/systemd/system/myps.service; enabled; vendor preset: disabled)
   Active: active (running) since Wed 2018-09-12 13:51:54 MSK; 2min 49s ago
 Main PID: 31066 (node)
    Tasks: 11 (limit: 4915)
   Memory: 12.0M
   CGroup: /system.slice/myps.service
           └─31066 node /usr/bin/myps-broker

Sep 12 13:51:54 pi systemd[1]: Started PubSub service running on nodejs.
```

Check if runtime directory was created and look inside it:

```
ls -hlatr /run
.......
drwxr-xr-x  7 root       root      160 Sep 12 13:21 udev
drwxr-xr-x 27 root       root      640 Sep 12 13:51 .
drwxrwxr-x  2 pi         lp         60 Sep 12 13:51 myps
drwxr-xr-x 18 root       root      440 Sep 12 13:52 systemd
```

```
ls -hlatr /run/myps
total 0
drwxr-xr-x 27 root       root 640 Sep 12 13:51 ..
srwxr-xr-x  1 pi         lp     0 Sep 12 13:51 myipc.sock
drwxrwxr-x  2 pi         lp    60 Sep 12 13:51 .
```

That's it. Proceed to next steps.

## myps.logviewer

All logging in bobaos.kit I implemented over myps service. Logs in real time can be viewed via util called **myps.logviewer**. Keep in mind, that it doesn't store any log data, just in real time.

```
sudo npm i -g myps.logviewer
```

```
myps-logviewer -c bobaos_logs
```

If bobaos.pub package is not installed and started yet then there will be only one word in console output: **connected**.

## bobaos.pub

Now, let install working service for BAOS sdk.

```
sudo npm i -g bobaos.pub --unsafe-perm
```

After that, edit config file located at **/usr/lib/node_modules/bobaos.pub/config.json** and replace default **/dev/ttyS1** to **/dev/ttyAMA0**. 

Create service file for systemd:

```
sudo nano /etc/systemd/system/bobaos_pub.service
```

Use following content for this file:

```
[Unit]
Description=PubSub service running on nodejs
After=myps.service

[Service]
User=pi
ExecStart=/usr/bin/env bobaos-pub
Restart=on-failure
RestartSec=10
# and now working dir.
# make sure it is located at `/run/${RuntimeDirectory}`
# which is in this example myps
WorkingDirectory=/run/myps

[Install]
WantedBy=multi-user.target
```

Reload systemd daemon, enable and start service.

```
sudo systemctl daemon-reload
sudo systemctl enable bobaos_pub.service
sudo systemctl start bobaos_pub.service
```

If **myps.logviewer** package is running then it will log output into console:

```
bobaos_ipc:: info:: 20:22:07:: ["Connected to pubsub as 655590131100"]
bobaos_ipc:: info:: 20:22:07:: ["Listening topic bobaos_req"]
bobaos.pub:: info:: 20:22:07:: ["IPC ready"]
bobaos_sdk:: info:: 20:22:07:: ["Serialport opened"]
bobaos_sdk:: info:: 20:22:07:: ["Loading server items"]
bobaos_sdk:: info:: 20:22:07:: ["Server items loaded"]
bobaos_sdk:: info:: 20:22:07:: ["Loading datapoints. Setting indication to false"]
bobaos_sdk:: info:: 20:22:08:: ["All datapoints [79] loaded. Return indications to true"]
bobaos_sdk:: info:: 20:22:08:: ["Bobaos SDK ready. Emitting event"]
bobaos.pub:: info:: 20:22:08:: ["Connected to BAOS. SDK ready."]
```

Proceed to final steps.

## bobaos.tool

To control datapoint values from shell, bobaos.tool package is used.

```
sudo npm i -g bobaos.tool
```

Run and ping sdk:

```
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
::  getitem ( * | ServerItem1 Item2... | [Item1, Item2, ..] )
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

On every request bobaos.pub will log debug info that can be viewer in myps.logviewer:

```
bobaos.pub:: debug:: 20:22:57:: ["Incoming request: "]
bobaos.pub:: debug:: 20:22:58:: ["method: ping"]
bobaos.pub:: debug:: 20:22:58:: ["payload: null"]
bobaos.pub:: debug:: 20:22:58:: ["response_channel: bobaos_res_26807"]
```

It is not necessary to view this logs but may be helpful in debugging.

But let's return to **bobaos-tool**. Some command examples:

* Programming mode

First **cool feature**: there is no need to press programming button.

To get/set programming mode following command is used:

```
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

```
bobaos> description 1 2 3
#1: length = 2, dpt = dpt9,  prio: low  flags: [C-WTU]
#2: length = 1, dpt = dpt5,  prio: low  flags: [C-WTU]
#3: length = 1, dpt = dpt5,  prio: low  flags: [C-WTU]
bobaos>
```

To list all datapoints, run:

```
bobaos> description *
```

* Set value:

```
bobaos> set 2: 0
20:27:06:239,    id: 2, value: 0, raw: [0]
bobaos> set [2: 0, 3: 128, 999: "hello, friend"]
20:28:48:586,    id: 2, value: 0, raw: [0]
20:28:48:592,    id: 3, value: 128, raw: [128]
20:28:48:593,    id: 999, value: "hello, friend, raw: [34,104,101,108,108,111,44,32,102,114,105,101,110,100]
```

I would advice to use **multiple value sending** as a cool feature of BAOS protocol.

* Get values:

```
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

```
bobaos> stored  1 2 3
20:39:54:963,    id: 1, value: 22.9, raw: [12,121]
20:39:54:974,    id: 2, value: 0, raw: [0]
20:39:54:982,    id: 3, value: 128, raw: [128]
```

This feature can help to get datapoint values and don't send a lot of requests over UART.

* Read values:

```
bobaos> read 1 2 3 105
20:42:12:540,    id: 1, value: 22.9, raw: [12,121]
20:42:12:642,    id: 105, value: false, raw: [0]
bobaos>
```

In this example only two datapoint indications came from bus. That happened because there is no KNX objects configured with flag **Read** for datapoints 2 and 3(these are control). Datapoint 1 is a temperature sensor and datapoint 105 is binary output feedback.

If group objects configured right, but there is no read response, check update flag in BAOS ETS config.

```
bobaos> description 1 2 3 105
#1: length = 2, dpt = dpt9,  prio: low  flags: [C-WTU]
#2: length = 1, dpt = dpt5,  prio: low  flags: [C-WTU]
#3: length = 1, dpt = dpt5,  prio: low  flags: [C-WTU]
#105: length = 1, dpt = dpt1,  prio: low  flags: [C-WTU]
bobaos>
```

In this example, **U** flag assigned for all of these objects.

* Monitoring datapoint values with colors:

```
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

```
bobaos> getitem *
[ { id: 1, name: 'HardwareType', value: [ 0, 0, 197, 8, 0, 3 ] },
  { id: 2, name: 'HardwareVersion', value: [ 16 ] },
  { id: 3, name: 'FirmwareVersion', value: [ 18 ] },
  { id: 4, name: 'KnxManufacturerCodeDev', value: [ 0, 197 ] },
  { id: 5, name: 'KnxManufacturerCodeApp', value: [ 0, 197 ] },
  { id: 6, name: 'EtsAppId', value: [ 8, 5 ] },
  { id: 7, name: 'EtsAppId', value: [ 16 ] },
  { id: 8,
    name: 'SerialNumber',
    value: [ 0, 197, 1, 1, 112, 182 ] },
  { id: 9, name: 'TimeSinceReset', value: 417399370 },
  { id: 10, name: 'BusConnectionState', value: true },
  { id: 11, name: 'MaximumBufferSize', value: 250 },
  { id: 12, name: 'DescStringLen', value: 0 },
  { id: 13, name: 'BaudRate', value: 19200 },
  { id: 14, name: 'CurrentBufferSize', value: 250 },
  { id: 15, name: 'ProgrammingMode', value: false },
  { id: 16, name: 'ProtocolVersion', value: 32 },
  { id: 17, name: 'IndicationSending', value: true } ]
bobaos> getitem BusConnectionState
[ { id: 10, name: 'BusConnectionState', value: true } ]
bobaos>
```

Keep in mind, that server items only can be get by name parameter or asterisk for all items.

```
bobaos> getbyte 1 2 3
[ 1, 3, 5 ]
bobaos>
```
To get parameter byte values.

* General commands: **ping, state, reset**

```
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

After reset request, bobaos.pub logs into bobaos_logs channel following info:

```
bobaos.pub:: debug:: 21:01:30:: ["Incoming request: "]
bobaos.pub:: debug:: 21:01:30:: ["method: reset"]
bobaos.pub:: debug:: 21:01:30:: ["payload: null"]
bobaos.pub:: debug:: 21:01:30:: ["response_channel: bobaos_res_16566"]
bobaos.pub:: info:: 21:01:30:: ["SDK has stopped"]
bobaos_sdk:: debug:: 21:01:30:: ["Resetting SDK."]
bobaos_sdk:: info:: 21:01:30:: ["Loading server items"]
bobaos_sdk:: info:: 21:01:30:: ["Server items loaded"]
bobaos_sdk:: info:: 21:01:30:: ["Loading datapoints. Setting indication to false"]
bobaos_sdk:: info:: 21:01:31:: ["All datapoints [79] loaded. Return indications to true"]
bobaos_sdk:: info:: 21:01:32:: ["Bobaos SDK ready. Emitting event"]
bobaos.pub:: info:: 21:01:32:: ["Connected to BAOS. SDK ready."]
```

## bobaos.sub

This library control BAOS datapoints from nodejs app.

```
npm i --save bobaos.sub
```

Then in script:

```js
const BobaosSub = require('bobaos.sub');
 
let my = BobaosSub({socketFile: "/var/run/myps/myipc.sock"});
 
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

There will be more info on API, but for now this is the end. The very end, my friend.

Thank you for attention.
