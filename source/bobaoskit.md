# BobaosKit - accessories

## About

Inspired by Apple HomeKit I was searching for android solution but couldn't find appropriate. To be able to work in LAN without cloud I started own project.

So, bobaoskit is a simple implementation of accessory system without any complication. So, there is no such things like pairing, authorization, crypto, binary protocol implemented. It works with simple Websocket protocol to implement network communications. Only five websocket requests are implemented and four broadcasted events.

Accessories only have `id`, `name`, `type`, `control`, `status` fields.

Following example shows how switch accessory can be implemented. It has only `state` both control and status field that reflects switch relay state.

```js
const Bobaos = require("bobaos.sub");
const BobaosKit = require("bobaoskit.accessory");

// init bobaos and bobaoskit sdk
const bobaos = Bobaos();
const sdk = BobaosKit.Sdk();

const switchDatapointControl = 3;
const switchDatapointStatus = 7;

const switch1 = BobaosKit.Accessory({
  id: "switch1",
  name: "Simple Switch",
  type: "switch",
  control: ["state"],
  status: ["state"],
  sdk: sdk
});

switch1.on("ready", async _ => {
  let value = (await bobaos.getValue(switchDatapointStatus)).value;
  return switch1.updateStatusValue({field: "state", value: value});
});

const processOneControlAccValue = payload => {
  let {field, value} = payload;
  // send datapoint value to KNX bus
  return bobaos.setValue({id: switchDatapointControl, value: value});
};

// on incoming event from bobaoskit.worker
// e.g. control request sent from mobile app
switch1.on("control accessory value", async (payload, cb) => {
  if (Array.isArray(payload)) {
    await Promise.all(payload.map(processOneControlAccValue));
    return;
  }

  await processOneControlAccValue(payload);
});

const processOneDatapointValue = payload => {
  let {id, value} = payload;
  if (id === switchDatapointStatus) {
    return switch1.updateStatusValue({field: "state", value: value});
  }
};

// register listener for incoming value from KNX bus
// after switch command state of relay was changed
bobaos.on("datapoint value", payload => {
  if (Array.isArray(payload)) {
    return payload.forEach(processOneDatapointValue);
  }

  processOneDatapointValue(payload);
});

// on exit, unregister accessory
const inTheEnd = async _ => {
  console.log("removing accessory");
  await switch1.unregisterAccessory();
  process.exit();
};
process.on('SIGINT', inTheEnd);
process.on('SIGTERM', inTheEnd);
```

## How it works

### Accessory backend service

First of all, there should be running instance of `bobaoskit.worker` on pc. 

`bobaoskit.worker` script uses `redis` and `bee-queue` job manager to process incoming requests.
There is possible requests as clear/add/remove accessory, get accessory info, get/update status value, control accessory value. Events are broadcasted over Redis Pub/Sub channel defined in `config.json`.

`bobaoskit.worker` listens port defined in `config.json` for WebSocket connections. Websocket API exposes request methods like get accessory info, get status/control accessory value

`bobaoskit.worker` uses `dnssd` module to advertise itself in local network. Advertised service name is defined in `config.json`.

## Accessory SDK

`bobaoskit.accessory` nodejs module is used to register and manage accessories. So, use it to create accessory instance, receive control commands, control your devices and update accessory state.

Take a look at following code

```js
const BobaosKit = require("bobaoskit.accessory");

// params:
//   redis: "redis_url"/client object
//   job_channel: "bobakit_job"
//   broadcast_channel: "bobaoskit_bcast"
// by default redis is undefined
// channels are from comments above,
// same as default in config.json
const sdk = BobaosKit.Sdk();


// sdk param is not required but 
// if you want to add multiple accessories
// it is better to create one sdk instance 
// and use it for all accessories
const accessory = BobaosKit.Accessory({
  id: "accessory",
  name: "Simple Switch",
  type: "switch",
  control: ["state"],
  status: ["state"],
  sdk: sdk
});
```
Now, sdk has following methods that is used by accessory instance:

  * ping()
  * getGeneralInfo()
  * clearAccessories()
  * addAccessory(payload)
  * removeAccessory(payload)
  * getAccessoryInfo(payload)
  * getStatusValue(payload)
  * updateStatusValue(payload)
  * controlAccessoryValue(payload)

Also, sdk emits `broadcasted event` with `(method, payload)` as a params.

`BobaosKit.Accessory(..)` creates object that represents accessory. It uses `sdk` to communicate via `bee-queue` with bobaoskit worker and creates own `bee-queue` instance to accept incoming requests. Exposed methods:

  * getAccessoryInfo()
  * getStatusValue(payload)
  * updateStatusValue(payload)
  * unregisterAccessory()

Accessory instance emits event `control accessory value` event with `(method, payload)` params when request to control accessory value is received. So, listen for this event and process it.

## Client

Client application makes a dnssd discovery and  after resolving host creates WebSocket connection to given port. Then `get accessory info` with `null` payload is sent and response contains list of all accessories.

To control accessory field/fields client sends `control accessory value` request with payload `{id: accId, control: {field: field1, value: value}/[{field: .. value: ..}]}/[{id: ...}, ...]`.

## Installation

Currently, no npm package is published.

So, clone git repositories:

```text
git clone https://github.com/bobaoskit/bobaoskit.worker
git clone https://github.com/bobaoskit/bobaoskit.accessory
```

Install dependencies for worker and run

```text
cd bobaoskit.worker
npm install
./bin/bobaos-kit.js
```

Install dependencies for bobaoskit.accessory

```text
cd bobaoskit.accessory
npm install
```
Now, take a look at `bobaoskit.accessory/examples/` folder.

You may be able to run radio player accessory scripted in `mpvsw.js`. Make sure you have `mpv` and latest version of `youtube-dl` installed. 

Install dependencies for mpv client

```text
npm install mympvspawn
npm install mympvclient
```

Now, start it with node

```text
node ./examples/mpvsw.js
```

## Mobile application

For mobile application flutter is used. It uses mdns plugin for dns service discovery, currently this plugin is not published, so there is git dependency in `pubspec.yaml`.

Install flutter(I use v1.0.0), dart sdk(2.1.0), all sdk(Android/XCode) to compile app.

Clone repository

```text
git clone https://github.com/shabunin/bobaflu
```

Install packages

```text
flutter packages get
```

To run, debug and build use your favourite editor/ide, I prefer IntelliJ IDEA.

I run it only on Android device, so, it will be great if somebody takes responsibility to compile it for iOS.

Note: Android emulator won't discover any service due to isolated emulator network. Still, you are able to make direct connection.
