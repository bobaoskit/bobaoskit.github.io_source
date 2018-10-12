# myps.broker

myps stands for MyPubSub, a small service that implementing publish/subscribe model over  Unix Domain Socket

## Usage

```
$ myps.broker --help
Options:
  --help          Show help                                            [boolean]
  --version       Show version number                                  [boolean]
  --sockfile, -s  path to socket file. Default: ${process.cwd()}/myipc.sock'
  --debug, -d     debug output true/false
```

## Protocol

### Overview

It uses JSON messages encapsulated into BDSM frame to communicate between broker and clients.

`|B|D|S|M|<LB>|<LE>|<DATA>|<C>|`

where DATA is JSON message, `<L>` is two byte field in BE that contains length of `<DATA>`, `<C>` is checksum of `<DATA>` (sum of all bytes modulo 256). `BDSM` is a string header of message.

### JSON message

`DATA` field is a JSON message that uses `method` and `payload` fields as required. For requests from client also `request_id` field may be used.


####  Messages sent from server. `method` field.

##### `greeting` sends server when client connected. As a payload it transmits connection id. `{"method": "greeting", "payload": 1317678194084}`.
##### `message` client receives when a message on subscribed topic occured. `{"method": "message", "payload": {"topic": "given topic", "message": "given message", "connection_id": 42}}`.
##### `success` signals that request was successful `{"response_id": 42, "method": "success", "payload": "something"}`.
##### `error` signals that something went wrong with request `{"response_id": 42, "method": "error", "payload": "error description"}`.

####  Messages sent from client to server. `method` field.

##### `subscribe` server receives when client wants to subscribe to topic given in `payload`. `{"request_id": 42, "method": "subscribe", "payload": "given topic"}`.
##### `unsubscribe` when client want to unsubscribe `{"request_id": 42, "method": "unsubscribe", "payload": "given topic"}`.
##### `publish` client sends when want to publish message to topic. `{"request_id": 42, "method": "publish", "payload": {"topic": "given topic", "message": "given string message or object"}`.

## Installing

```
sudo npm i -g myps.broker@latests
```

I would advice to run myps.broker as a systemd daemon using followind service file:

```
vim /etc/systemd/system/myps.service
```

Edit content of this file:

```
[Unit]
Description=PubSub service running on nodejs

[Service]
User=yourusername
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

Sep 12 13:51:54 the-undefeated systemd[1]: Started PubSub service running on nodejs.
```

Check if runtime directory was created:

```
ls -hlatr /run
.......
drwxr-xr-x  7 root       root      160 Sep 12 13:21 udev
drwxr-xr-x 27 root       root      640 Sep 12 13:51 .
drwxrwxr-x  2 undefeated lp         60 Sep 12 13:51 myps
drwxr-xr-x 18 root       root      440 Sep 12 13:52 systemd
```

As you can see in example, there is myps directory, so look inside it:

```
ls -hlatr /run/myps
total 0
drwxr-xr-x 27 root       root 640 Sep 12 13:51 ..
srwxr-xr-x  1 undefeated lp     0 Sep 12 13:51 myipc.sock
drwxrwxr-x  2 undefeated lp    60 Sep 12 13:51 .
```

That's it. Now we can use it for own purposes. 
