[TOC]

------

# 主要网络配置

编辑网络配置`sudo nano /etc/network/interfaces`
刚安装的一般内容如下，我这里有四个板载网卡分别是eno1~eno4
准备拿eno1做对外的WAN口，其他三个做内网LAN

```
# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
auto eno1
allow-hotplug eno1
iface eno1 inet dhcp
# set mtu to 1492 to adapte PPPoE in future
        up ip link set eno1 mtu 1492
# This is an autoconfigured IPv6 interface
iface eno1 inet6 auto

# This line to include more config from directory /etc/network/interfaces.d/
source /etc/network/interfaces.d/*
```



# 建立LAN网络桥

建立一个新网络配置作为eno2-eno4的网络桥
`sudo nano /etc/network/interfaces.d/br0`
打开STP功能并且也设置mtu为1492

```
auto br0
iface br0 inet static
        address 192.168.10.1/24
        bridge_ports eno2 eno3 eno4
        up /usr/sbin/brctl stp br0 on; ip link set dev br0 address $(cat /sys/class/net/eno2/address); ip link set br0 mtu 1492
```


# 配置PPPoE

建立一个新网络配置启动pppoe
`sudo nano /etc/network/interfaces.d/ppp0`

```
auto ppp0
        iface ppp0 inet ppp
        provider ChinaTelecom
```

添加拨号配置
`sudo nano /etc/ppp/peers/ChinaTelecom`
内容如下：
```
#Change eno1 to your WAN interface
plugin rp-pppoe.so eno1
noipdefault
defaultroute
hide-password
#lcp-echo-interval 30
#lcp-echo-failure 4
noauth
persist
#mtu 1492
#persist
#maxfail 0
#holdoff 20
noaccomp
default-asyncmap
user "username from ISP"
usepeerdns
# This line to enable IPv6 over PPPoE
+ipv6 ipv6cp-use-ipaddr
```
将这个中国电信的拨号配置设置为默认配置
```bash
sudo mv /etc/ppp/peers/provider /etc/ppp/peers/provider.orig
sudo ln -s /etc/ppp/peers/ChinaTelecom /etc/ppp/peers/provider
```
编辑拨号密码配置文件
`sudo nano /etc/ppp/chap-secrets`
```
# Secrets for authentication using CHAP
# client        server  secret                  IP addresses
"username from ISP" * "password from ISP"
```
有些运营商可能还需要建立一个`/etc/ppp/pap-secrets`配置文件，内容同上



# 配置dnsmasq

## 安装dnsmasq

`sudo apt install dnsmasq`
安装过程可能会提示启动失败，53端口被占用
可能是因为systemd-resolved已经绑定到53端口，因此dnsmasq不能使用它。
首先需要禁用此systemd服务：

```bash
sudo systemctl stop systemd-resolved
sudo systemctl disable systemd-resolved
```
然后`sudo systemctl start dnsmasq`启动dnsmasq。

修改`/etc/dnsmasq.conf`，找到`conf-dir=/etc/dnsmasq.d/,*.conf` 所在行，取消注释
conf-dir 是设置额外配置文件所在的路径，这样我们可以把配置文件分类放入/etc/dnsmasq.d，方便管理

## 设置DHCP

`sudo nano /etc/dnsmasq.d/10-dhcp.conf`
内容如下：

```
interface=br0
#IP range
dhcp-range=192.168.10.2,192.168.10.254,255.255.255.0,12h
# Static IP （给指定MAC地址分配静态ip）
dhcp-host=04:D4:C4:54:91:D9,192.168.10.2
```
## 设置DNS

`sudo nano /etc/dnsmasq.d/10-dns.conf`
内容如下：

```
expand-hosts
#resolv-file=/etc/dhcpc/resolv.conf
#resolv-file=/etc/ppp/resolv.conf
```



# 配置网络转发

## 开启内核转发功能

创建`/etc/sysctl.d/10-ipforward.conf`，内容如下：
```
net.ipv4.ip_forward = 1
net.ipv6.config.default.forwarding = 1
net.ipv6.config.all.forwarding = 1
```
`sudo sysctl -p`或者重启生效

## 配置iptables报文转发

创建WAN口为eno1时的转发，此时WAN一般是DHCP或static IP形式接上层网络。

 `sudo nano /etc/iptables/10-nat.rules`

```
*nat
:PREROUTING ACCEPT [0:0]
:INPUT ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]
-A POSTROUTING -o eno1 -j MASQUERADE
COMMIT

*filter
:INPUT ACCEPT [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
-A FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A FORWARD -i br0 -o eno1 -j ACCEPT
COMMIT
```

创建WAN口为PPPoE时的转发

`sudo nano /etc/iptables/20-nat-pppoe.rules`

```
*nat
:PREROUTING ACCEPT [0:0]
:INPUT ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]
-A POSTROUTING -o ppp0 -j MASQUERADE
COMMIT

*filter
:INPUT ACCEPT [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
-A FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A FORWARD -i br0 -o ppp0 -j ACCEPT
COMMIT

*mangle
:PREROUTING ACCEPT [0:0]
:INPUT ACCEPT [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]
-A FORWARD -o ppp0 -p tcp -m tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
COMMIT
```

`sudo nano /etc/network/if-up.d/nat`

```bash
#!/bin/sh
if [ "$IFACE" = br0 ]; then
    iptables-restore /etc/iptables/10-nat.rules
fi
if [ "$IFACE" = ppp0 ]; then
    iptables-restore /etc/iptables/20-nat-pppoe.rules
fi
```
`sudo chmod +x /etc/network/if-up.d/nat`

`sudo nano /etc/network/if-down.d/nat`

```bash
#!/bin/sh
if [ "$IFACE" = ppp0 ]; then
    iptables-restore /etc/iptables/10-nat.rules
fi
```
`sudo chmod +x /etc/network/if-down.d/nat`
