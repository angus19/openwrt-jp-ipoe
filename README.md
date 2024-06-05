# **Configuring OpenWrt to work with Japan NTT IPv6 (MAP-E) service**

## **The problem:**

ISPs with NTT mostly support both IPv4 & IPv6 implementations, while former one usually by using PPPoE which can introduce higher latency, during peak hours it can be also very slow in some busy districts. IPv6 is their newly promoted way to connect to internet which doesn't require PPPoE (note there is no PPPoE for 10G plan, IPoE is the only option),  they also claim this is a much faster option, with IPv4 over IPv6 together users should retain traditional IPv4 connectivity. Unfortunately if you subscribe the internet service without using Hikari Denwa (ひかり電話) residential phone service, you will end up getting /64 prefix address as well as without router advertisement (RA), if you don't use vendor provided router it would be extremely difficult to set up your IPv6 network with IPv4 over IPv6 connectivity.

There exist a few different implementations (DS-LITE/Transix/MAP-E)  in Japan, so not all providers can do the same way, here I am only referring to my own provider _**NTT ぷらら (plala**_**),** which is MAP-E implementation.

## **Setup**:

The whole setup was first tested with **GL-INET MT1300 (beryl, v22.03.3)** and verified with **NanoPi R4S (4GB, v22.03.3)** as well as **Linksys WRT3200ACM (rango, v22.03.2)** and **Jetway NF9HG-N2930 (x86, v22.03.3)** on 1G plan.

In October 2023 I have verified this setup is also working with **Netgear WAX206 (v23.05.0)** on upgraded 10G plan.

Note: I didn't notice that my upgraded 10G plan is slightly different from the original 1G plan, the old one comes with /64 prefix without prefix delegation (PD), while ***the 10G plan already give you /56 prefix with PD even you don't pay for the Hikari phone service!*** Here is a [discussion](https://github.com/fakemanhk/openwrt-jp-ipoe/discussions/27) about it.


*   **System > Software**: Install the required add-on package _**map**_ for MAP-E/MAP-T support, you will need to reboot before you can use it.
*   Enable WAN6 with **DHCPv6**, firewall setting you probably need to add this to **WAN Zone** (same as IPv4 WAN) for protection.

> Under **DHCP Server > IPv6** Setting, follow these settings:
> 
> *   Designated master ON
> *   RA-Service: relay mode
> *   DHCPv6-Service: relay mode
> *   NDP-Proxy: relay mode
> *   Learn routes: ON

![](https://user-images.githubusercontent.com/21307353/212850790-a2c4ac8b-3bed-4941-8f1a-49e9e5597c3f.png)

Save & Apply setting, you should see a public IPv6 address being assigned to your _WAN6_ interface (usually starting with 2400)

*   _WAN6_ interface also needs to add _**Routed Prefix Information**_ due to missing RA on WAN6 side (note: If your service plan comes with prefix delegation, like having Hikari phone service, or using some of the 10G internet plan, it could be coming with a shorter prefix like /60 or /56 together with PD, then this step can be ommited):

   * Use SSH to login router, edit **/etc/config/network,** add the line marked in _**Italics**_ under _WAN6_ interface section, note the _**2400:aaaa:bbbb:cccc**_ is your WAN IP prefix (first 64 bit), this will give the WAN6 interface proper IPv6-PD:

> config interface 'wan6'
> 
>             option device 'eth1'
> 
>             option proto 'dhcpv6'
> 
>             option reqaddress 'try'
> 
>             option reqprefix 'auto'
> 
>             _**option ip6prefix '2400:aaaa:bbbb:cccc::/64'**_

Note: Previously I had failed my setup because of missing this step, it wasn't mentioned in most resources I found on web, and I eventually got a _**MAP rule invalid**_ error.

*   Next, configure LAN interface, under **DHCP Server > IPv6** settings, basically very similar to WAN6 but **Designated master OFF**

![](https://user-images.githubusercontent.com/21307353/212852863-7fb85f4e-a04c-4ed6-b936-2b71f2631019.png)

You clients can probably get public IPv6 addresses (*) from router now! But this will be IPv6 access only and you are still missing the IPv4 connectivity, another MAP-E interface is required to fill the gap.

(*) Some clients might not work with DHCPv6, and you'll need SLAAC, please refer to the discussion [here](https://github.com/fakemanhk/openwrt-jp-ipoe/discussions/2) to change the settings.


*   Before we create the MAP-E interface, the parameter calculation for MAP-E is needed, in reference section I have attached the IETF information but someone has created a page to calculate, you can copy the public IPv6 address from WAN6 interface and use this [online MAP-E rule calculator](http://ipv4.web.fc2.com/map-e.html):

![](https://user-images.githubusercontent.com/21307353/212853420-6ce2090f-98f1-4f34-9f44-4db2d3bbddca.png)

Note: Before I ran the above calculator, I used the Buffalo router that came with ISP to connect the internet service, logged into that router and from status page I can see that at least the IPv4 address and port numbers are the same as above, so I believe the parameters I get from the calculator should be correct, you might want to do this as a verification.

*   Next will be setting up new MAP-E interface, create a new interface and name it (e.g. _WAN6MAPE_), and fill the parameters using above generated parameters:
    *   Protocol: MAP/LW4over6
    *   Type: MAP-E
    *   BR/DMR/AFTR: _\[peeraddr\]_
    *   IPv4 prefix: _\[ipaddr\]_
    *   IPv4 prefix length: _\[ip4prefixlen\]_
    *   IPv6 prefix: _\[ip6prefix\]_
    *   IPv6 prefix length: _\[ip6prefixlen\]_
    *   EA-bit length: _\[ealen\]_
    *   PSID-bits length: _\[psidlen\]_
    *   PSID offset: _\[offset\]_
    *   From advanced settings, make sure it has **WAN6 as Tunnel Link**, and check the box **Use legacy MAP**:

![](https://user-images.githubusercontent.com/21307353/212856884-d6d627a4-37b9-4002-99a7-2795dccac2cd.png)

Note: Don't forget to add this _WAN6MAPE_ interface to same firewall zone as WAN/WAN6 since this is also part of WAN.

## ADVANCED CUSTOM CONFIGURATION

MAP-E with IPv4 sharing from ISP is designed to share same IPv4 address with many customers, with different ports being assigned based on IETF rules, the above linked parameter calculator already shown the assigned ports, usually it's divided into groups of 16 ports, according to [this discussion](https://github.com/fakemanhk/openwrt-jp-ipoe/discussions/10) JPNE assigns 15 groups (240 ports), while OCN/plala assign 63 groups (1008 ports). In most cases this should be enough for most home uses (since only IPv4 connections will use them), however a recent test with well known IPv4 based [website](https://nichiban.co.jp) that uses many sessions showing a significant lagging while loading. After investigation the OpenWrt firewall statistics indicating only first group of assigned ports (i.e. only 16 ports) being used and this is the reason of lagging when a large number of simultaneous IPv4 sessions opening, also IPv4 PING is not working. Not sure if it's because Japan ISP MAP-E configuration has something MAP package can't deal with, as a result a system change is required for _**/lib/netifd/proto/map.sh.**_ 

You can download the whole file [here](https://github.com/fakemanhk/openwrt-jp-ipoe/blob/main/map.sh.new) and replace it, _**don't forget to turn on the execute bit**_ of the file after replacement.

After editing, please restart IPv6 interface, or simply reboot router, you'll see that IPv4 PING is working as well as observing more port groups passing traffic now.

Eventually you should see the following screen under **Network > Interfaces**, _WAN6MAPE_ should get the IPv4 exactly the same as using the ISP provided router, there is also a _**Virtual dynamic interface**_ automatically created when MAP-E interface started correctly.

![](https://user-images.githubusercontent.com/21307353/212857199-21f283c9-e9e2-43b2-8d58-955126076744.png)

From **Status > Overview** you'll see both **IPv4 Upstream** and **IPv6 Upstream** information:

_(Note: If your plan comes with prefix delegation, your IPv6 Upstream might not show you any address, only a prefix will be shown, this is NORMAL)_

![](https://user-images.githubusercontent.com/21307353/212858791-e21a621e-0a5a-40a9-952f-ec9d759b6a9e.png)

Testing with my Linux laptop by visiting the [OCN connectivity verification](https://v6test.ocn.ne.jp/) page, both IPv4/IPv6 addresses should be the same as above upstream informations:

![](https://user-images.githubusercontent.com/21307353/212859123-0590650f-29f1-412c-99a7-19d5516a8d22.png)

## **SUCCESS!!**

Some speed test [results](https://github.com/fakemanhk/openwrt-jp-ipoe/discussions/4)

From time to time, you might observe `ip6_tunnel: map-MAPE xmit: Local address not yet configured!` in kernel log, this can be ignored and you don't need to worry about it.


### Reference materials:

https://datatracker.ietf.org/doc/html/draft-ietf-softwire-map-03#page-6

[https://www.labohyt.net/blog/lan/post-6760/](https://www.labohyt.net/blog/lan/post-6760/)

[https://zenn.dev/yakumo/articles/19cbc6309d8143cc9349b2fb0d29771e](https://zenn.dev/yakumo/articles/19cbc6309d8143cc9349b2fb0d29771e)

[https://blog.hinaloe.net/2020/03/14/openwrt-mape-ocn/](https://blog.hinaloe.net/2020/03/14/openwrt-mape-ocn/)

_First draft: 17 Jan 2023_

_Last Edit: 05 June 2024_
