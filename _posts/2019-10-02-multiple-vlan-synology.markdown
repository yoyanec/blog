---
layout: post
title: Multiple vlans on Synology LACP Bond
date: 2019-10-02 21:18:00 +0200
description: How to connect Synology to multiple vlans true LACP Bond network interface
img: synology-multiple-vlan.png
tags: [synology, lacp, vlan]
---

In some cases you may want to connect your Synology NAS to multiple vlans. You may have different network cards, but why would you like to give away the advantage of loadbalancing/failover of an aggregated link, right? It is not possible using web interface, but it is fairly easy.

## Requirements

Create LACP Bond device and configure you network as usual. Please make use you check `Enable VLAN (802.1Q)`
![VLAN config screenshot](/assets/img/synology-vlan-config.png)


## Configuration
Then login into Synology using `ssh` and change the directory to `/etc/sysconfig/network-scripts`:
{% highlight Bash %}
ssh superadmin@synologynas.local

cd /etc/sysconfig/network-scripts
{% endhighlight %}

In this directory you find config for your network interfaces. Names of this configs looks like `ifcfg-bond0.40`. From first part `ifcfg` Synology knows, thas it is netwrok config file, second part `bond0` is the name of vlan and last part `.40` is VLAN id. So if you want to create new vlan interface, just copy this file, rename it and change VLAN ID in config:

{% highlight Bash %}
cp ifcfg-bond0.40 ifcfg-bond0.120

vi ifcfg-bond0.120
{% endhighlight %}

example of config file:
{% highlight Bash %}
DEVICE=bond0.120
VLAN_ROW_DEVICE=bond0
VLAN_ID=120
ONBOOT=yes
BOOTPROTO=dhcp
IPV6INIT=off
USERCTL=no
{% endhighlight %}

Make sure that you set the `DEVICE` and `VLAN_ID` appropriately to the match VLAN. To activate the new bond interface just restart Synology NAS.

After restart you should see new network interface in web interface:
![VLAN config screenshot](/assets/img/synology-multiple-vlan-conf.png)

## Troubleshooting

If you for some reason can't connect to Synology NAS after restart, just reset network and start over. Don't worry, your files and apps stay untouched.

[https://www.synology.com/en-global/knowledgebase/DSM/tutorial/General_Setup/How_to_reset_my_Synology_NAS](https://www.synology.com/en-global/knowledgebase/DSM/tutorial/General_Setup/How_to_reset_my_Synology_NAS)