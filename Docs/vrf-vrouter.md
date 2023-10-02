# VRF - VRF Lite
- VRF Instance Type : Virtual Router
- JCNR WorkerNode内でPOD間接続

## VRF Lite - Pod Vlan Access
<img src="https://github.com/jnpr-jp-crdc/JCNR/blob/main/Docs/Images/vrf-vrouter1.png" width=400>

### Network Attachment Definition　作成
[VRF Lite NAD Sample Yaml](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Manifests/vrouter1-nad.yaml)

#### NAD Option
- InstanceType : VRF Liteは"virtual-router"を使用
- InterfaceType : 
  - veth : non DPDK Application接続時に使用
  - virtio : DPDK Application接続時に使用
- Ipam Type:
  - static : 全てのPODに同一IPを付与
  - host-local : 同一Host内でユニークなIPをPODに付与
  - whereabouts : 異なるHostでもユニークなIPをPODに付与 (https://github.com/k8snetworkplumbingwg/whereabouts)

### POD 作成
[Pod1 Sample Yaml](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Manifests/vrouter1-pod1.yaml)

[Pod2 Sample Yaml](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Manifests/vrouter1-pod2.yaml)

#### Pod Interface
- Secondary InterfaceにVRFが接続される
- IP AddressはNADで指定した方式に従い払い出される
- NADで指定したRouteがStatic Routeとして反映される
```
# kubectl describe pod vrouter1-pod1
Name:         vrouter1-pod1
Namespace:    default
Priority:     0
Node:         jcnr1/172.27.115.12
Start Time:   Mon, 02 Oct 2023 01:57:04 -0400
Labels:       <none>
Annotations:  cni.projectcalico.org/containerID: 702595e6eb2f05aaf5ca0b79c7fcc3344bfadc2603a0dd405130e2fdcfc3d003
              cni.projectcalico.org/podIP: 172.30.79.46/32
              cni.projectcalico.org/podIPs: 172.30.79.46/32
              k8s.v1.cni.cncf.io/network-status:
                [{
                    "name": "k8s-pod-network",
                    "ips": [
                        "172.30.79.53"
                    ],
                    "default": true,
                    "dns": {}
                },{
                    "name": "default/vrouter1",
                    "interface": "net1",
                    "ips": [
                        "10.0.0.9",
                        "abcd::a00:4"
                    ],
                    "mac": "02:00:00:17:98:B8",
                    "dns": {}
                }]
              k8s.v1.cni.cncf.io/networks:
                [
                  {
                    "name": "vrouter1",
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
[root@vrouter1-pod1 /]# ip a
--- snip ---
88: net1@if89: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:00:00:17:98:b8 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.0.0.9/24 brd 10.0.0.255 scope global net1
       valid_lft forever preferred_lft forever
    inet6 abcd::a00:4/120 scope global
       valid_lft forever preferred_lft forever
    inet6 fe80::ff:fe17:98b8/64 scope link
       valid_lft forever preferred_lft forever

[root@vrouter1-pod1 /]# ip route
default via 169.254.1.1 dev eth0
10.0.0.0/24 via 10.0.0.1 dev net1
10.0.0.1 dev net1 scope link
20.0.0.0/24 via 10.0.0.1 dev net1
169.254.1.1 dev eth0 scope link

[root@vrouter1-pod1 /]# ip -6 route
abcd::a00:1 dev net1 metric 1024 pref medium
abcd::a00:0/120 via abcd::a00:1 dev net1 metric 1024 pref medium
abcd::1400:0/120 via abcd::a00:1 dev net1 metric 1024 pref medium
fe80::/64 dev eth0 proto kernel metric 256 pref medium
fe80::/64 dev net1 proto kernel metric 256 pref medium
```

cRPD Config確認
```
set groups cni routing-instances vrouter1 instance-type virtual-router
set groups cni routing-instances vrouter1 routing-options rib vrouter1.inet6.0 static route abcd::a00:4/128 qualified-next-hop abcd::a00:4 interface jvknet1-6103a34
set groups cni routing-instances vrouter1 routing-options static route 10.0.0.9/32 qualified-next-hop 10.0.0.9 interface jvknet1-6103a34
set groups cni routing-instances vrouter1 interface jvknet1-6103a34
```

vRouter VIF確認
```
bash-5.1# vif --list
vif0/6      Ethernet: jvknet1-6103a34 NH: 19 MTU: 9160
            Type:Virtual HWaddr:00:00:5e:00:01:00 IPaddr:10.0.0.9
            IP6addr:abcd::a00:4
            DDP: OFF SwLB: ON
            Vrf:1 Mcast Vrf:1 Flags:PL3DVofProxyEr QOS:-1 Ref:11
            RX port   packets:47 errors:0
            RX queue errors to lcore 0 0 0 0 0 0 0 0 0 0 0 0 0 0
            RX packets:47  bytes:6069 errors:0
            TX packets:2  bytes:172 errors:0
            Drops:45
            TX port   packets:2 errors:0
```

## VRF Lite - Pod Vlan Trunk
<img src="https://github.com/jnpr-jp-crdc/JCNR/blob/main/Docs/Images/vrf-vrouter2.png" width=400>
