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

**namespaceのブリッジ確認**
```
# ip netns exec 1-2f4ug83ox8 brctl show
bridge name	bridge id		STP enabled	interfaces
br0		    8000.829a270b3af8	no		veth0
							                    vxlan0

# ip netns exec ingress_sbox brctl show
empty
```

**namespaceのiptablesを確認**
```
# ip netns exec 1-2f4ug83ox8 iptables-save
empty

# ip netns exec ingress_sbox iptables-save
*mangle
*nat
-A OUTPUT -d 127.0.0.11/32 -j DOCKER_OUTPUT
-A POSTROUTING -d 127.0.0.11/32 -j DOCKER_POSTROUTING
-A DOCKER_OUTPUT -d 127.0.0.11/32 -p tcp -m tcp --dport 53 -j DNAT --to-destination 127.0.0.11:44532
-A DOCKER_OUTPUT -d 127.0.0.11/32 -p udp -m udp --dport 53 -j DNAT --to-destination 127.0.0.11:56147
-A DOCKER_POSTROUTING -s 127.0.0.11/32 -p tcp -m tcp --sport 44532 -j SNAT --to-source :53
-A DOCKER_POSTROUTING -s 127.0.0.11/32 -p udp -m udp --sport 56147 -j SNAT --to-source :53
*filter
```
Docker用内部DNSへのNATの設定が入っている

**ホストのiptablesを確認**
```
# iptables-save
*mangle
*nat
-A PREROUTING -m addrtype --dst-type LOCAL -j DOCKER
-A OUTPUT ! -d 127.0.0.0/8 -m addrtype --dst-type LOCAL -j DOCKER
-A POSTROUTING -s 172.18.0.0/16 ! -o docker_gwbridge -j MASQUERADE
-A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE
-A DOCKER -i docker_gwbridge -j RETURN
-A DOCKER -i docker0 -j RETURN
*filter
-A FORWARD -j DOCKER-ISOLATION
-A FORWARD -o docker_gwbridge -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A FORWARD -o docker_gwbridge -j DOCKER
-A FORWARD -i docker_gwbridge ! -o docker_gwbridge -j ACCEPT
-A FORWARD -o docker0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A FORWARD -o docker0 -j DOCKER
-A FORWARD -i docker0 ! -o docker0 -j ACCEPT
-A FORWARD -i docker0 -o docker0 -j ACCEPT
-A FORWARD -i docker_gwbridge -o docker_gwbridge -j DROP
-A DOCKER-ISOLATION -i docker0 -o docker_gwbridge -j DROP
-A DOCKER-ISOLATION -i docker_gwbridge -o docker0 -j DROP
-A DOCKER-ISOLATION -j RETURN
```
docker02,03についても同じ状態だった

#### 1.4 コンテナ作成後の状態を確認
**コンテナ2台起動してみる**
```
# docker service create --name helloworld --replicas=2 -p 80:8000 momijiame/greeting:1.0
# docker service ps helloworld
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE            ERROR                       PORTS
a5priwyxm92p        helloworld.1        alpine:latest       docker03             Running             Running 14 minutes ago                               
xtga47qcta3m        helloworld.2        alpine:latest       docker01             Running             Running 14 minutes ago
```
##### 1.4.1 docker01で確認
**ホストのNIC確認**  
全部書くと冗長なので差分だけ記載していく
```
# ip addr
26: veth8f546c1@if25: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker_gwbridge state UP group default
```
**namespaceの確認**
```
# ip netns
686fc135d53d (id: 2)
```
コンテナ用のnamespaceが追加されている  
**ブリッジの確認**
```
# brctl show
bridge name	bridge id		STP enabled	interfaces
docker_gwbridge		8000.0242e933701f	no		vethc10c6e8
```
追加されたvethはdocker_gwbridgeに追加されている  
**追加されたnamespaceのNIC確認**
```
# ip netns exec 686fc135d53d ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
23: eth0@if24: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default
    inet 10.255.0.7/16 scope global eth0
    inet 10.255.0.5/32 scope global eth0
25: eth1@if26: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    inet 172.18.0.3/16 scope global eth1
```
eth1はdocker_gwbridgeに追加されたvethと繋がってる  
**vxlan用namespaceのNICとブリッジを確認**
```
# ip netns exec 1-flu1uxdh02 ip addr
24: veth1@if23: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master br0 state UP group default

# ip netns exec 1-flu1uxdh02 brctl show
bridge name	bridge id		STP enabled	interfaces
br0		8000.0aa8e892ebb7	no		veth0
							              veth1
							              vxlan0
```
追加されたNICはコンテナnamespaceのeth0と繋がってる

**ingress用namespaceのNICとブリッジは変更なし**

**コンテナ用namespaceのiptablesを確認**
```
*mangle
*nat
-A PREROUTING -d 10.255.0.7/32 -p tcp -m tcp --dport 80 -j REDIRECT --to-ports 8000
-A OUTPUT -d 127.0.0.11/32 -j DOCKER_OUTPUT
-A POSTROUTING -d 127.0.0.11/32 -j DOCKER_POSTROUTING
-A DOCKER_OUTPUT -d 127.0.0.11/32 -p tcp -m tcp --dport 53 -j DNAT --to-destination 127.0.0.11:35163
-A DOCKER_OUTPUT -d 127.0.0.11/32 -p udp -m udp --dport 53 -j DNAT --to-destination 127.0.0.11:33418
-A DOCKER_POSTROUTING -s 127.0.0.11/32 -p tcp -m tcp --sport 35163 -j SNAT --to-source :53
-A DOCKER_POSTROUTING -s 127.0.0.11/32 -p udp -m udp --sport 33418 -j SNAT --to-source :53
*filter
-A INPUT -d 10.255.0.7/32 -p tcp -m tcp --dport 8000 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
-A INPUT -d 10.255.0.7/32 -p udp -j DROP
-A INPUT -d 10.255.0.7/32 -p tcp -j DROP
-A OUTPUT -s 10.255.0.7/32 -p tcp -m tcp --sport 8000 -m conntrack --ctstate ESTABLISHED -j ACCEPT
-A OUTPUT -s 10.255.0.7/32 -p udp -j DROP
-A OUTPUT -s 10.255.0.7/32 -p tcp -j DROP
```
10.255.0.7:80宛のtcpパケットは10.255.0.7:8000に変換される  


**ingress用namespaceのiptablesを確認**
```
*mangle
-A PREROUTING -p tcp -m tcp --dport 80 -j MARK --set-xmark 0x100/0xffffffff
-A OUTPUT -d 10.255.0.5/32 -j MARK --set-xmark 0x100/0xffffffff
*nat
-A OUTPUT -d 10.255.0.5/32 -p icmp -m icmp --icmp-type 8 -j DNAT --to-destination 127.0.0.1
-A POSTROUTING -d 10.255.0.0/16 -m ipvs --ipvs -j SNAT --to-source 10.255.0.2
```
**ホストのiptablesを確認**
```
*mangle
*nat
-A PREROUTING -m addrtype --dst-type LOCAL -j DOCKER-INGRESS
-A OUTPUT -m addrtype --dst-type LOCAL -j DOCKER-INGRESS
-A POSTROUTING -o docker_gwbridge -m addrtype --src-type LOCAL -j MASQUERADE
-A DOCKER-INGRESS -p tcp -m tcp --dport 80 -j DNAT --to-destination 172.18.0.2:80
-A DOCKER-INGRESS -j RETURN
*filter
-A FORWARD -j DOCKER-INGRESS
-A DOCKER-INGRESS -p tcp -m tcp --dport 80 -j ACCEPT
-A DOCKER-INGRESS -p tcp -m state --state RELATED,ESTABLISHED -m tcp --sport 80 -j ACCEPT
-A DOCKER-INGRESS -j RETURN
```
