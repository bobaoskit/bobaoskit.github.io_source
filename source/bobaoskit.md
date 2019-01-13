# BobaosKit - accessories

## About

Inspired by Apple HomeKit I was searching for android solution but couldn't find appropriate. To be able to work in LAN without cloud I started own project.

So, bobaoskit is a simple implementation of accessory system without any complication. So, there is no such things like pairing, crypto, binary protocol implemented. It works with simple Websocket protocol to implement network communications. Only five websocket requests are implemented and four broadcasted events.

Accessories only have `id`, `name`, `type`, `control`, `status` fields.

For example, simple switch that works with `bobaos`:

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
