### 1. 環境

#### 1.1 OS  
>Ubuntu16.04 Server

#### 1.2 ホスト名  
>docker01 master  
docker02 worker  
docker03 master

#### 1.3 インストール直後の状態確認  
##### 1.3.1 docker01
**ホストのNIC確認**
```
# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    inet 10.0.2.15/24 brd 10.0.2.255 scope global enp0s3
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    inet 192.168.56.100/24 brd 192.168.56.255 scope global enp0s8
4: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
    inet 172.17.0.1/16 scope global docker0
9: docker_gwbridge: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    inet 172.18.0.1/16 scope global docker_gwbridge
11: veth9a70b79@if10: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker_gwbridge state UP group default
```
**namespaceの確認**
```
# ln -s /var/run/docker/netns /var/run/netns
# ip netns
1-2f4ug83ox8 (id: 0)
ingress_sbox (id: 1)
```

**ブリッジの確認**
```
# brctl show
bridge name	          bridge id	       STP enabled	  interfaces
docker0		        8000.02429b3e1eb0	        no		
docker_gwbridge		8000.0242d72aaa36	        no		veth9a70b79
```

**namespaceのNIC確認**
```
# ip netns exec 1-2f4ug83ox8 ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
2: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default
    inet 10.255.0.1/16 scope global br0
6: vxlan0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master br0 state UNKNOWN group default
8: veth0@if7: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master br0 state UP group default

# ip netns exec ingress_sbox ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
7: eth0@if8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default
    inet 10.255.0.2/16 scope global eth0
10: eth1@if11: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    inet 172.18.0.2/16 scope global eth1
```

**namespaceのNIC確認**
```
# ip netns exec 1-2f4ug83ox8 brctl show
bridge name	bridge id		STP enabled	interfaces
br0		    8000.829a270b3af8	no		veth0
							                  vxlan0

# ip netns exec ingress_sbox brctl show
empty
```
docker02,03についても同じ状態だった

**namespaceのiptablesを確認**
```
# ip netns exec 1-2f4ug83ox8 iptables-save
empty

# ip netns exec ingress_sbox iptables-save
*mangle
:PREROUTING ACCEPT [0:0]
:INPUT ACCEPT [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]
*nat
:PREROUTING ACCEPT [0:0]
:INPUT ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]
:DOCKER_OUTPUT - [0:0]
:DOCKER_POSTROUTING - [0:0]
-A OUTPUT -d 127.0.0.11/32 -j DOCKER_OUTPUT
-A POSTROUTING -d 127.0.0.11/32 -j DOCKER_POSTROUTING
-A DOCKER_OUTPUT -d 127.0.0.11/32 -p tcp -m tcp --dport 53 -j DNAT --to-destination 127.0.0.11:44532
-A DOCKER_OUTPUT -d 127.0.0.11/32 -p udp -m udp --dport 53 -j DNAT --to-destination 127.0.0.11:56147
-A DOCKER_POSTROUTING -s 127.0.0.11/32 -p tcp -m tcp --sport 44532 -j SNAT --to-source :53
-A DOCKER_POSTROUTING -s 127.0.0.11/32 -p udp -m udp --sport 56147 -j SNAT --to-source :53
*filter
:INPUT ACCEPT [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
```
