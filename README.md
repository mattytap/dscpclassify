# DSCP Classify

An nftables based service for applying DSCP classifications to connections, compatible with OpenWrt's firewall4 for dynamically setting DSCP packet marks (this only works in OpenWrt 22.03 and above).

This should be used in conjunction with layer-cake SQM queue with ctinfo configured to restore DSCP on the device ingress.
The dscpclassify service uses the last 8 bits of the conntrack mark (0x000000ff).

## Classification modes

The service supports three modes for classifying and DSCP marking connections.

### User rules

The service will first attempt to classify new connections using rules specified by the user in the config file.<br />
These follow a similar syntax to the OpenWrt firewall config and can match upon source/destination ports and IPs, firewall zones etc.<br />
The rules support the use of nft sets, which could be dynamically updated from external sources such as dsnmasq. <br />

### Client DSCP hinting

The service can be configured to apply the DSCP mark supplied by a non WAN originating client.<br />
This function ignores CS6 and CS7 classes to avoid abuse from inappropriately configed LAN clients such as IoT devices.

### Dynamic classification

Connections that do not match a pre-specified rule will be dynamically classified by the service via two mechanisms:

* Multi-connection client port detection for detecting P2P traffic
  * These connections are classified as Low Effort (LE) by default and therefore prioritised below Best Effort traffic when using the layer-cake qdisc.
* Multi-threaded service detection for identifying high-throughput downloads from services such as Steam/Windows Update
  * These connections are classified as High-Throughput (AF13) by default and therefore prioritised as follows by cake:
    * diffserv3/4: Equal to besteffort (CS0) traffic.
    * diffserv8: Below besteffort (CS0) traffic, but above low effort (LE) traffic.

### External classification

The service will respect DSCP classification stored by an external service in a connection's conntrack bits, this could include services such as netifyd.

## Service architecture

![image](https://user-images.githubusercontent.com/46714706/188151111-9167e54d-482e-4584-b43b-0759e0ad7561.png)

## Service installation

To install the dscpclassify service via command line you can use the following:

```bash
repo="https://raw.githubusercontent.com/mattytap/dscpclassify/mattytap"
mkdir -p "/etc/dscpclassify.d"
if [ ! -f "/etc/config/dscpclassify" ]; then
    wget "$repo/etc/config/dscpclassify" -O "/etc/config/dscpclassify"
else
    wget "$repo/etc/config/dscpclassify" -O "/etc/config/dscpclassify_git"
fi
wget "$repo/etc/dscpclassify.d/main.nft" -O "/etc/dscpclassify.d/main.nft"
wget "$repo/etc/dscpclassify.d/maps.nft" -O "/etc/dscpclassify.d/maps.nft"
wget "$repo/etc/dscpclassify.d/verdicts.nft" -O "/etc/dscpclassify.d/verdicts.nft"
wget "$repo/etc/hotplug.d/iface/21-dscpclassify" -O "/etc/hotplug.d/iface/21-dscpclassify"
wget "$repo/etc/init.d/dscpclassify" -O "/etc/init.d/dscpclassify"
chmod +x "/etc/init.d/dscpclassify"
/etc/init.d/dscpclassify enable
/etc/init.d/dscpclassify start
```

**SQM should be installed and configured on your device**

The OpenWrt guide for configuring this is here <https://openwrt.org/docs/guide-user/network/traffic-shaping/sqm>

Ingress DSCP marking requires the SQM queue setup script 'layer_cake_ct.qos' and the package 'kmod-sched-ctinfo'.

To install these via command line you can use the following:

```bash
repo="https://raw.githubusercontent.com/mattytap/dscpclassify/mattytap"
opkg update
opkg install kmod-sched-ctinfo
wget "$repo/usr/lib/sqm/autorate-ct.qos" -O "/usr/lib/sqm/autorate-ct.qos"
wget "$repo/usr/lib/sqm/autorate-ct.qos.help" -O "/usr/lib/sqm/autorate-ct.qos.help"
wget "$repo/usr/lib/sqm/layer_cake_ct.qos" -O "/usr/lib/sqm/layer_cake_ct.qos"
wget "$repo/usr/lib/sqm/layer_cake_ct.qos.help" -O "/usr/lib/sqm/layer_cake_ct.qos.help"
```

## Service configuration

The user rules in '/etc/config/dscpclassify' use the same syntax as OpenWrt's firewall config, the 'class' option is used to specified the desired DSCP.

A working default configuration is provided with the service.

**The service supports the following global classification options:**

|  Config option | Description  | Type  | Default  |
|---|---|---|---|
| class_bulk | The class applied to threaded bulk clients | string | le |
| class_high_throughput | The class applied to threaded high-throughput services | string | af13 |
| client_hints | Adopt the DSCP class supplied by a non-WAN client (this exludes CS6 and CS7 classes to avoid abuse) | boolean | 1 |
| threaded_client_min_bytes | The total bytes before a threaded client port (i.e. P2P) is classified as bulk | uint | 10000 |
| threaded_client_min_connections | The number of established connections for a client port to be considered threaded | uint | 10 |
| threaded_service_min_bytes | The total bytes before a threaded service's connection is classed as high-throughput | uint | 1000000 |
| threaded_service_min_connections | The number of established connections for a service to be considered threaded | uint | 3 |
| lan_device | Manually specify devices that the service should treat as LAN | string |  |
| lan_zone | Manually specify firewall zones that the service should treat as LAN | string | lan |
| wan_device | Manually specify devices that the service should treat as WAN | string |  |
| wan_zone | Manually specify firewall zones that the service should treat as WAN | string | wan |
| wmm | When enabled the service will mark LAN bound packets with DSCP values respective of WMM (RFC-8325) | boolean |  0 |

**Below is an example user rule:**

```
config rule
 option name 'DNS'
 list proto 'tcp'
 list proto 'udp'
 list dest_port '53'
 list dest_port '853'
 list dest_port '5353'
 option class 'cs5'
 option counter '0'
```

The OpenWrt firewall syntax is outlined here <https://openwrt.org/docs/guide-user/firewall/firewall_configuration>

The counter option can be enabled to count the number of matched connections for a rule.

## SQM configuration

The **'layer_cake_ct.qos'** queue setup script must be selected for your wan device in SQM setup,

It is important that **Squash DSCP** and **Ignore DSCP** on ingress are **not enabled** in SQM setup otherwise cake will ignore the service's DSCP classes.

![image](https://user-images.githubusercontent.com/46714706/190709086-c2e820ed-11ed-4be4-8e57-fba4ab6db190.png)
![image](https://user-images.githubusercontent.com/46714706/210797512-a2419605-5bd4-469b-8c99-2d881c2c8706.png)

<br />

**Below is a tested working SQM config for use with the service:**

| Config parameter | Value |
| ----------- | ----------- |
| qdisc_advanced | 1 |
| **squash_dscp** | 0, to ensure cake does not remove ingress packet DSCP values|
| **squash_ingress** | 0, to ensure cake looks at packet marks on ingress |
| qdisc_really_really_advanced | 1 |
| iqdisc_opts | nat dual-dsthost ingress diffserv4 |
| eqdisc_opts | nat dual-srchost ack-filter diffserv4 |
| **script** | layer_cake_ct.qos |

## Development Journey - using a repository such as jeverley/dscpclassify

This is a fork of the excellent [jeverley/dscpclassify](https://github.com/jeverley/dscpclassify) nftables based service for applying DSCP classifications to connections. This fork provides a simplified approach focused on IPV4 only, with no sets, no negation, and no error checking or parsing. We owe a lot to their initial work and the foundation DSCPClassify provided.

### DSCPClassify Overview

DSCPClassify is a service for OpenWRT that applies DSCP classification to connections using nftables. It can be used in combination with an SQM QoS script for tc commands, such as layer_cake.qos or simplest.qos, which have been configured to restore ingress DSCP marks from connection tracking. It was designed to work with OpenWRT 22.03 or later and with firewall4, the nftables-based firewall application included with OpenWRT.

DSCPClassify supports three modes of operation, which can be used separately or in combination:

* User rules: Allow you to specify rules for classifying connections in /etc/config/dscpclassify.
* Client DSCP hinting: If enabled, the service will apply the DSCP mark supplied by the client, unless the mark is in the list of ignored DSCP classes.
* Dynamic classification: If enabled, the service will classify connections that don't match user rules based on destination port, destination IP address, and the number of connections from the client to the server.

DSCPClassify can also respect DSCP marks stored in connection tracking by other services, such as netifyd.

### DSCPClassify Installation

To install DSCPClassify, follow these steps:

* Install the required packages
* Enable and start the service

For more details, refer to the [original repository](https://github.com/jeverley/dscpclassify).

### Usage

DSCPClassify is configured through /etc/config/dscpclassify. See [config.example](https://github.com/jeverley/dscpclassify/blob/main/config.example) in the repository for examples of how to configure the service.

### Limitations

* DSCPClassify only supports IPv4 connections.
* The service requires the nftables conntrack module, which is not available on all platforms.

## Acknowledgements and Thanks

We would like to express our gratitude to the authors of OpenWRT SQM (Michael D. Taht, Toke Høiland-Jørgensen, and Sebastian Moeller), as well as the creators of OpenWRT firewall4, jeverley/dscpclassify, lynxthecat/cake-qos-simple, lynxthecat/cake-autorate, and Lochnair/tsping.

Special thanks to [jeverley](https://github.com/jeverley) for their original dscpclassify project which this fork is based on.

We also thank stintel for their support of the qoriq platform.
