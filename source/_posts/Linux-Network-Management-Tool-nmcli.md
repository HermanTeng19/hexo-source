---
title: Linux Network Management Tool nmcli
date: 2020-12-14 13:21:16
categories: [IT]
tags: [Linux, centos, ubuntu, network]
toc: true
cover: /img/linuxnetworkribbon.jpg
thumbnail: /img/linuxnetworkicon.png
---

There are over 60 Linux networking commands you can utilize to do all the system network configurations, some of them are well known and widely used such as `ifconfig`, `ip addr`, `traceroute`, `netstat` and `ping` etc.., one command is very useful but relatively few being used, that is `nmcli` which is used for controlling your network, just like its name network manager and also can do all the thing to configure your network like displaying network device status, create, edit, activate/deactivate and delete network connection.

## syntax and options

`nmcli` command has 2 arguments, one is _option_ and the other one is _object_. 

```bash
nmcli [options] object {command | help}
```

<!-- more -->

You can quick check help to get the information `nmcli -h`

> OPTIONS
>   -t[erse]                                       terse output
>   -p[retty]                                      pretty output
>   -m[ode] tabular|multiline                      output mode
>   -c[olors] auto|yes|no                          whether to use colors in output
>   -f[ields] <field1,field2,...>|all|common       specify fields to output
>   -g[et-values] <field1,field2,...>|all|common   shortcut for -m tabular -t -f
>   -e[scape] yes|no                               escape columns separators in values
>   -a[sk]                                         ask for missing parameters
>   -s[how-secrets]                                allow displaying passwords
>   -w[ait] <seconds>                              set timeout waiting for finishing operations
>   -v[ersion]                                     show program version
>   -h[elp]                                        print this help
>
> OBJECT
>   g[eneral]       NetworkManager's general status and operations
>   n[etworking]    overall networking control
>   r[adio]         NetworkManager radio switches
>   c[onnection]    NetworkManager's connections
>   d[evice]        devices managed by NetworkManager
>   a[gent]         NetworkManager secret agent or polkit agent
>   m[onitor]       monitor NetworkManager changes	

From command syntax, you can tell 

1. the options can be multiple but object is only one, because nmcli only return information or do configure for one object a time.
2. the place of options and object can not be switched, must follow option first and object second. 

## Examples

### Show network status

```bash
nmcli -p general status
```

![networkstatus.png](/img/screenshots/networkstatus.png)

### nmcli permission

```bash
nmcli general permission
```

![nmclipermission.png](/img/screenshots/nmclipermission.png)

### Enable and disable network

```bash
nmcli networking on | off | connectivity
```

### Radio wifi transmission control

```bash
nmcli radio wifi | wwan | all
```

### Show local network connection

```bash
nmcli connection show
```

![networkconn.png](/img/screenshots/networkconn.png)

### Show network device status

```bas
nmcli device status
```

![networkdevicestatus.png](/img/screenshots/networkdevicestatus.png)

### Show overall network and device information

```bash
nmcli device show
```

that command will show device, device type, network connection, gateway, route, ip4, ip6, dns

### Show wifi connection status

```bash
nmcli device wifi list
```

![wificonn.png](/img/screenshots/wificonn.png)

### Configure wifi connections for Ubuntu

By default, Ubuntu wifi connection is disabled, so if you want to use wifi, something need to be done to be able to connect to WAN.

Firstly, show your wifi device

```bash
nmcli device status
```

Secondly, turn on the wifi radio transmission

```bash
nmcli radio wifi on
```

Thirdly, show all your wifi networks

```bash
nmcli device wifi list
```

Finally, connect your wifi network by password

```bash
nmcli device wifi connect <your wifi network name> password <your password>
```





























