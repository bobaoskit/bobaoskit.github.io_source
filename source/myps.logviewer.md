# myps.logviewer

Log viewer for selected channels.

Usage:

```
myps-logviewer -c mylogchannel
```

This app connects to myps.broker socket file and listens selected channel. Default socket file is `/run/myps/myipc.sock`. To change, use `-s --socketfile` commandline argument.


Main purpose of this app is debugging apps. So, it doesn't store any logs, just shows in present time.

For logging from your application take a look at myps.logger library.
