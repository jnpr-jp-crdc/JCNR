# VRF - vSwitch : Pod Vlan Trunk
- VRF Instance Type : Virtual Switch
- JCNR WorkerNode間のL2接続
- POD / JCNR vRouter間はVlan Trunk接続
- VLANはJCNRデプロイ時にL2 Interfaceに指定したVLANしか使用できません

## VRF vSwitch - Pod VLAN Access 
<img src="https://github.com/jnpr-jp-crdc/JCNR/blob/main/Docs/Images/vrf-vswitch2.png" width=600>

### Network Attachment Definition　作成
[VRF vSwitch NAD JCNR1 Sample Yaml](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Manifests/vswitch2-nad-jcnr1.yaml)

[VRF vSwitch NAD JCNR2 Sample Yaml](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Manifests/vswitch2-nad-jcnr2.yaml)




#### NAD Option
- InstanceName : "vswitch"のまま。変更不可
- InstanceType : VRF Liteは"virtual-switch"を使用
- vlanIdList: VLAN ID
- InterfaceType : 
  - veth : non DPDK Application接続時に使用
  - virtio : DPDK Application接続時に使用
- Ipam Type:
  - static : 全てのPODに同一IPを付与
  - host-local : 同一Host内でユニークなIPをPODに付与
  - whereabouts : 異なるHostでもユニークなIPをPODに付与 (https://github.com/k8snetworkplumbingwg/whereabouts)

### POD 作成
[JCNR1 Pod1 Sample Yaml](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Manifests/vswitch2-pod1-jcnr1.yaml)

[JCNR2 Pod2 Sample Yaml](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Manifests/vswitch2-pod2-jcnr2.yaml)

#### Pod Interface
- Secondary InterfaceにVRFが接続される
- IP AddressはNADで指定した方式に従い払い出される
```
[root@jcnr1 yaml]# kubectl describe pod vswitch2-pod1
Name:         vswitch2-pod1
Namespace:    default
Priority:     0
Node:         jcnr1/172.27.115.12
Start Time:   Tue, 03 Oct 2023 04:22:33 -0400
Labels:       <none>
Annotations:  cni.projectcalico.org/containerID: c6475f0237f252f9fafda7b8adea57cf5228c26abedd6e365cd8e42545c39375
              cni.projectcalico.org/podIP: 172.30.79.27/32
              cni.projectcalico.org/podIPs: 172.30.79.27/32
              k8s.v1.cni.cncf.io/network-status:
                [{
                    "name": "k8s-pod-network",
                    "ips": [
                        "172.30.79.27"
                    ],
                    "default": true,
                    "dns": {}
                },{
                    "name": "default/vswitch2",
                    "interface": "net1",
                    "ips": [
                        "10.0.0.2",
                        "abcd::a00:2"
                    ],
                    "mac": "02:00:00:CC:1F:78",
                    "dns": {}
                }]
              k8s.v1.cni.cncf.io/networks:
                [
                  {
                    "name": "vswitch2",
                    "interface":"net1",
                    "cni-args": {
                      "interfaceType":"veth"
                    }
                  }
                ]
Status:       Running
--- snip ---
```
```
[root@jcnr2 yaml]# kubectl describe pod vswitch2-pod2
Name:         vswitch2-pod2
Namespace:    default
Priority:     0
Node:         jcnr2/172.27.115.13
Start Time:   Tue, 03 Oct 2023 04:24:47 -0400
Labels:       <none>
Annotations:  cni.projectcalico.org/containerID: 9c6962ae26a69f0503d15f0e96a2cfd6ef441869f74965e73d6e073344fac176
              cni.projectcalico.org/podIP: 172.30.147.232/32
              cni.projectcalico.org/podIPs: 172.30.147.232/32
              k8s.v1.cni.cncf.io/network-status:
                [{
                    "name": "k8s-pod-network",
                    "ips": [
                        "172.30.147.232"
                    ],
                    "default": true,
                    "dns": {}
                },{
                    "name": "default/vswitch2",
                    "interface": "net1",
                    "ips": [
                        "10.0.0.3",
                        "abcd::a00:3"
                    ],
                    "mac": "02:00:00:6B:E7:12",
                    "dns": {}
                }]
              k8s.v1.cni.cncf.io/networks:
                [
                  {
                    "name": "vswitch2",
                    "interface":"net1",
                    "cni-args": {
                      "interfaceType":"veth"
                    }
                  }
                ]
Status:       Running
--- snip ---
```

POD IP/Route 確認
```
[root@vswitch2-pod1 /]# ip a
--- snip ---
189: net1@if190: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:00:00:cc:1f:78 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.0.0.2/24 brd 10.0.0.255 scope global net1
       valid_lft forever preferred_lft forever
    inet6 abcd::a00:2/120 scope global
       valid_lft forever preferred_lft forever
    inet6 fe80::ff:fecc:1f78/64 scope link
       valid_lft forever preferred_lft forever

[root@vswitch2-pod1 /]# ip link add link net1 name net1.701 type vlan id 701
[root@vswitch2-pod1 /]# ip addr add dev net1.701 71.0.0.1/24
[root@vswitch2-pod1 /]# ip link set net1.701 up
[root@vswitch2-pod1 /]# ip a
--- snip ---
10: net1.701@net1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 02:00:00:cc:1f:78 brd ff:ff:ff:ff:ff:ff
    inet 71.0.0.1/24 scope global net1.701
       valid_lft forever preferred_lft forever
    inet6 fe80::ff:fecc:1f78/64 scope link
       valid_lft forever preferred_lft forever

```
```
[root@vswitch2-pod2 /]# ip a
--- snip ---
74: net1@if75: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:00:00:6b:e7:12 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.0.0.3/24 brd 10.0.0.255 scope global net1
       valid_lft forever preferred_lft forever
    inet6 abcd::a00:3/120 scope global
       valid_lft forever preferred_lft forever
    inet6 fe80::ff:fe6b:e712/64 scope link
       valid_lft forever preferred_lft forever

[root@vswitch2-pod2 /]# ip link add link net1 name net1.701 type vlan id 701
[root@vswitch2-pod2 /]# ip addr add dev net1.701 71.0.0.2/24
[root@vswitch2-pod2 /]# ip link set net1.701 up
[root@vswitch2-pod2 /]# ip a
10: net1.701@net1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 02:00:00:6b:e7:12 brd ff:ff:ff:ff:ff:ff
    inet 71.0.0.2/24 scope global net1.701
       valid_lft forever preferred_lft forever
    inet6 fe80::ff:fe6b:e712/64 scope link
       valid_lft forever preferred_lft forever
```


cRPD Config確認
```
root@jcnr1> show configuration | display set
--- snip ---
set groups cni interfaces jvknet1-c0e9ad3 unit 0 family bridge interface-mode trunk
set groups cni interfaces jvknet1-c0e9ad3 unit 0 family bridge vlan-id-list 701-703
set groups cni routing-instances vswitch instance-type virtual-switch
set groups cni routing-instances vswitch interface jvknet1-c0e9ad3
```
```
root@jcnr2> show configuration | display set
--- snip ---
set groups cni interfaces jvknet1-62bf6d7 unit 0 family bridge interface-mode trunk
set groups cni interfaces jvknet1-62bf6d7 unit 0 family bridge vlan-id-list 701-703
set groups cni routing-instances vswitch instance-type virtual-switch
set groups cni routing-instances vswitch interface jvknet1-62bf6d7
```

cRPD L2 Default設定
- JCNRデプロイ時に指定したInterface VLAN設定が以下の設定の通り反映されている
```
set routing-instances vswitch instance-type virtual-switch
set routing-instances vswitch bridge-domains bd100 vlan-id 100
set routing-instances vswitch bridge-domains bd200 vlan-id 200
set routing-instances vswitch bridge-domains bd300 vlan-id 300
set routing-instances vswitch bridge-domains bd700 vlan-id 700
set routing-instances vswitch bridge-domains bd701 vlan-id 701
set routing-instances vswitch bridge-domains bd702 vlan-id 702
set routing-instances vswitch bridge-domains bd703 vlan-id 703
set routing-instances vswitch bridge-domains bd704 vlan-id 704
set routing-instances vswitch bridge-domains bd705 vlan-id 705
set routing-instances vswitch interface ens5
```

vRouter VIF確認
```
[root@jcnr1]# kubectl exec -it contrail-vrouter-masters-flnlk -n contrail -- bash
bash-5.1# vif --list
vif0/6      Ethernet: jvknet1-c0e9ad3 MTU: 9160
            Type:Virtual HWaddr:00:00:5e:00:01:00
            DDP: OFF SwLB: ON
            Vrf:0 Flags:L2Vof QOS:-1 Ref:8
            RX port   packets:38 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            Vlan Mode: Trunk  Vlan: 701-703
            RX packets:77  bytes:21936 errors:25
            TX packets:12  bytes:956 errors:0
            Drops:90
            TX queue  packets:12 errors:0
            TX port   packets:12 errors:0
```
```
[root@jcnr2 yaml]# kubectl exec -it contrail-vrouter-masters-lxhhw -n contrail -- bash
bash-5.1# vif --list
vif0/6      Ethernet: jvknet1-62bf6d7 MTU: 9160
            Type:Virtual HWaddr:00:00:5e:00:01:00
            DDP: OFF SwLB: ON
            Vrf:0 Flags:L2Vof QOS:-1 Ref:8
            RX port   packets:39 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            Vlan Mode: Trunk  Vlan: 701-703
            RX packets:49  bytes:11538 errors:26
            TX packets:13  bytes:1030 errors:0
            Drops:63
            TX queue  packets:13 errors:0
            TX port   packets:13 errors:0
```

POD間疎通確認
```
[root@vswitch2-pod1 /]# ping 71.0.0.2
PING 71.0.0.2 (71.0.0.2) 56(84) bytes of data.
64 bytes from 71.0.0.2: icmp_seq=1 ttl=64 time=0.544 ms
64 bytes from 71.0.0.2: icmp_seq=2 ttl=64 time=0.240 ms
```