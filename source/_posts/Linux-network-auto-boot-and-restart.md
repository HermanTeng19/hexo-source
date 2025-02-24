---
title: Linux network auto boot and restart
date: 2020-11-25 13:59:22
categories: [IT]
tags: [Linux, Centos, network, config]
cover: /img/centosribbon.jpg
thumbnail: /img/centosicon.png
---

Centos 7 network is disabled by default after installation and initialization, which causes network connection can not be made until you manually turn on it

![centos7_network_icon.png](/img/screenshots/centos7_network_icon.png)

That is such _**annoying**_ when you usually use SSH tool remote connect to centos workstation or server, so it's quite necessary to turn on the network automatically every time reboot machine or VM. To accomplish that, we need to modify the network configuration file.

<!-- more -->

### Auto Boot Network

#### VIM open config file: /etc/sysconfig/network-scripts/ifcfg-ens33

```bash
vim /etc/sysconfig/network-scripts/ifcfg-ens33
```

#### Revise the `ONBOOT` value to be "yes"

![centos7_network_setting.png](/img/screenshots/centos7_network_setting.png)

#### Save and quit config file by `:wq` vim command

### Restart Network

To be able to make that change taking effect, network needs to be restarted by below command

```bash
systemctl restart network.service
```

something trick here is some blog articles mention that part is only `systemctl restart`, which won't work if there is no `network.service`, I confuse and spend many time to figure out that, thanks to [CodeSheep](https://www.codesheep.cn/) whose video shed the light on it and help me to solve that problem.



