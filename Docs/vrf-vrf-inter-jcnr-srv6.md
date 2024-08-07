# VRF - VRF L3VPN / SRv6 : JCNR間Routing
- VRF Instance Type : vrf
- JCNR WorkerNode間でPOD間SRv6によるL3接続
- POD / JCNR vRouter間はL3接続

## VRF L3VPN / SR-MPLS - JCNR間Routing
<img src="https://github.com/jnpr-jp-crdc/JCNR/blob/main/Docs/Images/vrf-srv6.png" width=800>

### JCNR Configlet
[JCNR1 Configlet Sample Yaml](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Manifests/configlet-srv6-jcnr1.yaml)

[JCNR2 Configlet Sample Yaml](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Manifests/configlet-srv6-jcnr2.yaml)


### Network Attachment Definition　作成
[VRF L3VPN NAD JCNR1 Sample Yaml](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Manifests/vrf1-nad-jcnr1.yaml)

[VRF L3VPN NAD JCNR2 Sample Yaml](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Manifests/vrf1-nad-jcnr2.yaml)

#### NAD Option
- InstanceType : "vrf"を使用
- vrfTarget : VRF間接続時はvrfTargetを揃える
- InterfaceType : 
  - veth : non DPDK Application接続時に使用
  - virtio : DPDK Application接続時に使用
- Ipam Type:
  - static : 全てのPODに同一IPを付与
  - host-local : 同一Host内でユニークなIPをPODに付与
  - whereabouts : 異なるHostでもユニークなIPをPODに付与 (https://github.com/k8snetworkplumbingwg/whereabouts)

### POD 作成
[JCNR1 Pod1 Sample Yaml](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Manifests/vrf1-pod1-jcnr1.yaml)

[JCNR2 Pod2 Sample Yaml](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Manifests/vrf1-pod2-jcnr2.yaml)

#### Pod Interface
- Secondary InterfaceにVRFが接続される
- IP AddressはNADで指定した方式に従い払い出される
- NADで指定したRouteがStatic Routeとして反映される
```
[root@jcnr1]# kubectl describe pod vrf1-pod1
Name:             vrf1-pod1
Namespace:        default
Priority:         0
Service Account:  default
Node:             jcnr-worker1/172.27.115.207
Start Time:       Wed, 07 Aug 2024 04:24:58 -0400
Labels:           <none>
Annotations:      cni.projectcalico.org/containerID: ed226f2a4715bfdadde79976594103c39d4698f6fe4fd551a23f26f00e68d734
                  cni.projectcalico.org/podIP: 10.233.78.211/32
                  cni.projectcalico.org/podIPs: 10.233.78.211/32
                  k8s.v1.cni.cncf.io/network-status:
                    [{
                        "name": "k8s-pod-network",
                        "ips": [
                            "10.233.78.211"
                        ],
                        "default": true,
                        "dns": {}
                    },{
                        "name": "default/vrf1",
                        "interface": "net1",
                        "ips": [
                            "10.0.0.10",
                            "abcd::a00:a"
                        ],
                        "mac": "02:00:00:B9:3C:7B",
                        "dns": {}
                    }]
                  k8s.v1.cni.cncf.io/networks:
                    [
                      {
                        "name": "vrf1",
                        "interface":"net1",
                        "cni-args": {
                          "interfaceType":"veth",
                          "vrfTarget": "65000:1"
                        }
                      }
                    ]
Status:           Running
--- snip ---
```
```
[root@jcnr2]# kubectl describe pod vrf1-pod2
Name:             vrf2-pod1
Namespace:        default
Priority:         0
Service Account:  default
Node:             jcnr-worker2/172.27.115.208
Start Time:       Wed, 07 Aug 2024 04:25:07 -0400
Labels:           <none>
Annotations:      cni.projectcalico.org/containerID: e1082cddb47dfb33e71854c18044e70ad3747878d037a444fb0dc475a896ff65
                  cni.projectcalico.org/podIP: 10.233.81.90/32
                  cni.projectcalico.org/podIPs: 10.233.81.90/32
                  k8s.v1.cni.cncf.io/network-status:
                    [{
                        "name": "k8s-pod-network",
                        "ips": [
                            "10.233.81.90"
                        ],
                        "default": true,
                        "dns": {}
                    },{
                        "name": "default/vrf2",
                        "interface": "net1",
                        "ips": [
                            "20.0.0.12",
                            "abcd::1400:c"
                        ],
                        "mac": "02:00:00:55:0E:9C",
                        "dns": {}
                    }]
                  k8s.v1.cni.cncf.io/networks:
                    [
                      {
                        "name": "vrf2",
                        "interface":"net1",
                        "cni-args": {
                          "interfaceType":"veth",
                          "vrfTarget": "65000:1"
                        }
                      }
                    ]
Status:           Running
--- snip ---
```

POD IP/Route 確認
```
[root@vrf1-pod1 /]# ip a
--- snip ---
26: net1@if27: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:00:00:b9:3c:7b brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.0.0.10/24 brd 10.0.0.255 scope global net1
       valid_lft forever preferred_lft forever
    inet6 abcd::a00:a/120 scope global
       valid_lft forever preferred_lft forever
    inet6 fe80::ff:feb9:3c7b/64 scope link
       valid_lft forever preferred_lft forever

[root@vrf1-pod1 /]# ip route
default via 10.0.0.1 dev net1
10.0.0.0/24 via 10.0.0.1 dev net1
10.0.0.1 dev net1 scope link
20.0.0.0/24 via 10.0.0.1 dev net1
169.254.1.1 dev eth0 scope link
```
```
[root@vrf2-pod1 /]# ip a
--- snip ---
28: net1@if29: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:00:00:55:0e:9c brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 20.0.0.12/24 brd 20.0.0.255 scope global net1
       valid_lft forever preferred_lft forever
    inet6 abcd::1400:c/120 scope global
       valid_lft forever preferred_lft forever
    inet6 fe80::ff:fe55:e9c/64 scope link
       valid_lft forever preferred_lft forever

[root@vrf2-pod1 /]# ip route
default via 169.254.1.1 dev eth0
10.0.0.0/24 via 20.0.0.1 dev net1
20.0.0.0/24 via 20.0.0.1 dev net1
20.0.0.1 dev net1 scope link
169.254.1.1 dev eth0 scope link
```

cRPD Config確認
```
root@jcnr1> show configuration | display set
--- snip ---
set groups cni routing-instances vrf1 instance-type vrf
set groups cni routing-instances vrf1 routing-options rib vrf1.inet6.0 static route abcd::a00:a/128 qualified-next-hop abcd::a00:a interface jvknet1-365ab9c
set groups cni routing-instances vrf1 routing-options static route 10.0.0.10/32 qualified-next-hop 10.0.0.10 interface jvknet1-365ab9c
set groups cni routing-instances vrf1 interface jvknet1-365ab9c
set groups cni routing-instances vrf1 vrf-target target:65000:1
```
```
root@jcnr2> show configuration | display set
--- snip ---
set groups cni routing-instances vrf2 instance-type vrf
set groups cni routing-instances vrf2 routing-options rib vrf2.inet6.0 static route abcd::1400:c/128 qualified-next-hop abcd::1400:c interface jvknet1-81e476e
set groups cni routing-instances vrf2 routing-options static route 20.0.0.12/32 qualified-next-hop 20.0.0.12 interface jvknet1-81e476e
set groups cni routing-instances vrf2 interface jvknet1-81e476e
set groups cni routing-instances vrf2 vrf-target target:65000:1
```

cRPD Route確認
```
root@jcnr-worker1> show isis adjacency
Interface             System         L State         Hold (secs) SNPA
ens4                  jcnr-master    1  Up                    22  52:54:0:c0:11:df
ens4                  jcnr-master    2  Up                    24  52:54:0:c0:11:df
ens4                  jcnr-worker2   1  Up                    19  52:54:0:60:8b:33
ens4                  jcnr-worker2   2  Up                    19  52:54:0:60:8b:33

root@jcnr-worker1> show isis database detail
IS-IS level 1 link-state database:

jcnr-master.00-00 Sequence: 0x3, Checksum: 0xe240, Lifetime: 572 secs
   IS neighbor: jcnr-worker1.02               Metric:       10
   V6 prefix: 2001:1:1:1::1/128               Metric:        0 Internal Up
   V6 prefix: 2001:192:168:1::/64             Metric:       10 Internal Up
   V6 prefix: 2001:db8:4600::/48              Metric:        0 Internal Up
   V6 prefix: fe80::a82a:d3ff:fee3:4097/128   Metric:        0 Internal Up

jcnr-worker1.00-00 Sequence: 0x6, Checksum: 0xa839, Lifetime: 690 secs
   IS neighbor: jcnr-worker1.02               Metric:       10
   V6 prefix: 2001:1:1:1::2/128               Metric:        0 Internal Up
   V6 prefix: 2001:192:168:1::/64             Metric:       10 Internal Up
   V6 prefix: 2001:db8:4700::/48              Metric:        0 Internal Up
   V6 prefix: fe80::44f4:17ff:fe2f:29ad/128   Metric:        0 Internal Up

jcnr-worker1.02-00 Sequence: 0x4, Checksum: 0xbd4b, Lifetime: 694 secs
   IS neighbor: jcnr-master.00                Metric:        0
   IS neighbor: jcnr-worker1.00               Metric:        0
   IS neighbor: jcnr-worker2.00               Metric:        0

jcnr-worker2.00-00 Sequence: 0x6, Checksum: 0xd487, Lifetime: 668 secs
   IS neighbor: jcnr-worker1.02               Metric:       10
   V6 prefix: 2001:1:1:1::3/128               Metric:        0 Internal Up
   V6 prefix: 2001:192:168:1::/64             Metric:       10 Internal Up
   V6 prefix: 2001:db8:4800::/48              Metric:        0 Internal Up
   V6 prefix: fe80::54d7:5ff:fef2:bdee/128    Metric:        0 Internal Up

IS-IS level 2 link-state database:

jcnr-master.00-00 Sequence: 0x5, Checksum: 0xe08c, Lifetime: 572 secs
   IS neighbor: jcnr-worker1.02               Metric:       10
   V6 prefix: 2001:1:1:1::1/128               Metric:        0 Internal Up
   V6 prefix: 2001:1:1:1::2/128               Metric:       10 Internal Up
   V6 prefix: 2001:1:1:1::3/128               Metric:       10 Internal Up
   V6 prefix: 2001:192:168:1::/64             Metric:       10 Internal Up
   V6 prefix: 2001:db8:4600::/48              Metric:        0 Internal Up
   V6 prefix: 2001:db8:4700::/48              Metric:       10 Internal Up
   V6 prefix: 2001:db8:4800::/48              Metric:       10 Internal Up
   V6 prefix: fe80::44f4:17ff:fe2f:29ad/128   Metric:       10 Internal Up
   V6 prefix: fe80::54d7:5ff:fef2:bdee/128    Metric:       10 Internal Up
   V6 prefix: fe80::a82a:d3ff:fee3:4097/128   Metric:        0 Internal Up

jcnr-worker1.00-00 Sequence: 0x8, Checksum: 0x1212, Lifetime: 690 secs
   IS neighbor: jcnr-worker1.02               Metric:       10
   V6 prefix: 2001:1:1:1::1/128               Metric:       10 Internal Up
   V6 prefix: 2001:1:1:1::2/128               Metric:        0 Internal Up
   V6 prefix: 2001:1:1:1::3/128               Metric:       10 Internal Up
   V6 prefix: 2001:192:168:1::/64             Metric:       10 Internal Up
   V6 prefix: 2001:db8:4600::/48              Metric:       10 Internal Up
   V6 prefix: 2001:db8:4700::/48              Metric:        0 Internal Up
   V6 prefix: 2001:db8:4800::/48              Metric:       10 Internal Up
   V6 prefix: fe80::44f4:17ff:fe2f:29ad/128   Metric:        0 Internal Up
   V6 prefix: fe80::54d7:5ff:fef2:bdee/128    Metric:       10 Internal Up
   V6 prefix: fe80::a82a:d3ff:fee3:4097/128   Metric:       10 Internal Up

jcnr-worker1.02-00 Sequence: 0x4, Checksum: 0xbd4b, Lifetime: 690 secs
   IS neighbor: jcnr-master.00                Metric:        0
   IS neighbor: jcnr-worker1.00               Metric:        0
   IS neighbor: jcnr-worker2.00               Metric:        0

jcnr-worker2.00-00 Sequence: 0x8, Checksum: 0xfb21, Lifetime: 607 secs
   IS neighbor: jcnr-worker1.02               Metric:       10
   V6 prefix: 2001:1:1:1::1/128               Metric:       10 Internal Up
   V6 prefix: 2001:1:1:1::2/128               Metric:       10 Internal Up
   V6 prefix: 2001:1:1:1::3/128               Metric:        0 Internal Up
   V6 prefix: 2001:192:168:1::/64             Metric:       10 Internal Up
   V6 prefix: 2001:db8:4600::/48              Metric:       10 Internal Up
   V6 prefix: 2001:db8:4700::/48              Metric:       10 Internal Up
   V6 prefix: 2001:db8:4800::/48              Metric:        0 Internal Up
   V6 prefix: fe80::44f4:17ff:fe2f:29ad/128   Metric:       10 Internal Up
   V6 prefix: fe80::54d7:5ff:fef2:bdee/128    Metric:        0 Internal Up
   V6 prefix: fe80::a82a:d3ff:fee3:4097/128   Metric:       10 Internal Up

root@jcnr-worker1> show route

inet.0: 9 destinations, 10 routes (9 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.233.0.1/32      *[Local/0] 00:30:02
                       Reject
10.233.0.3/32      *[Local/0] 00:30:02
                       Reject
10.233.7.156/32    *[Local/0] 00:30:02
                       Reject
10.233.10.71/32    *[Local/0] 00:30:02
                       Reject
10.233.78.192/32   *[Local/0] 00:29:55
                       Reject
169.254.25.10/32   *[Local/0] 00:29:42
                       Reject
169.254.157.182/32 *[Direct/0] 00:30:02
                    >  via eth0
                    [Local/0] 00:30:02
                       Local via eth0
172.27.112.0/22    *[Direct/0] 00:30:02
                    >  via ens3
172.27.115.207/32  *[Local/0] 00:30:02
                       Local via ens3

vrf1.inet.0: 3 destinations, 3 routes (3 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.1/32        *[Local/0] 00:27:58
                       Local via jvknet1-365ab9c
10.0.0.10/32       *[Static/5] 00:27:58
                    >  via jvknet1-365ab9c
20.0.0.12/32       *[BGP/170] 00:27:48, localpref 100, from 2001:1:1:1::1
                      AS path: I, validation-state: unverified
                    >  to fe80::5054:ff:fe60:8b33 via ens4, SRv6 SID: 2001:db8:4800:e001::, SRV6-Tunnel, Dest: 2001:db8:4800::

iso.0: 1 destinations, 1 routes (1 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

49.0001.0000.0000.0002/72
                   *[Direct/0] 00:29:51
                    >  via lo0.0

bgp.l3vpn.0: 1 destinations, 1 routes (1 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

172.27.115.208:3:20.0.0.12/32
                   *[BGP/170] 00:27:48, localpref 100, from 2001:1:1:1::1
                      AS path: I, validation-state: unverified
                    >  to fe80::5054:ff:fe60:8b33 via ens4, SRv6 SID: 2001:db8:4800:e001::, SRV6-Tunnel, Dest: 2001:db8:4800::

inet6.0: 28 destinations, 30 routes (28 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

::/96              *[Direct/0] 00:30:02
                    >  via sit0
                    [Direct/0] 00:30:02
                    >  via sit0
                    [Direct/0] 00:30:02
                    >  via sit0
::127.0.0.1/128    *[Local/0] 00:30:02
                       Local via sit0
::169.254.157.182/128
                   *[Local/0] 00:30:02
                       Local via sit0
::172.27.115.207/128
                   *[Local/0] 00:30:02
                       Local via sit0
2001:1:1:1::1/128  *[IS-IS/15] 00:29:07, metric 10
                    >  to fe80::5054:ff:fec0:11df via ens4
2001:1:1:1::2/128  *[Direct/0] 00:29:52
                    >  via lo0.0
2001:1:1:1::3/128  *[IS-IS/15] 00:29:37, metric 10
                    >  to fe80::5054:ff:fe60:8b33 via ens4
2001:192:168:1::/64*[Direct/0] 00:29:52
                    >  via ens4
2001:192:168:1::2/128
                   *[Local/0] 00:29:52
                       Local via ens4
2001:db8:4600::/48 *[IS-IS/15] 00:29:07, metric 10
                    >  to fe80::5054:ff:fec0:11df via ens4
2001:db8:4700::/48 *[IS-IS/15] 00:29:51, metric 0
                       Receive
2001:db8:4700:e001::/80
                   *[SRV6/171] 00:27:59
                       to table vrf1.inet.0
2001:db8:4800::/48 *[IS-IS/15] 00:29:37, metric 10
                    >  to fe80::5054:ff:fe60:8b33 via ens4
2001:db8:e001::/64 *[SRV6/171] 00:27:59
                       to table vrf1.inet.0
fe80::1/128        *[Direct/0] 00:30:02
                    >  via lo
fe80::200:ff:fe00:0/128
                   *[Local/0] 00:30:02
                       Local via ip6tnl0
fe80::44f4:17ff:fe2f:29ad/128
                   *[Local/0] 00:30:01
                       Local via lo0.0
fe80::4a5a:dff:fe3c:dae3/128
                   *[Local/0] 00:29:52
                       Local via irb
fe80::5054:ff:fe96:c35f/128
                   *[Local/0] 00:30:02
                       Local via ens3
fe80::5054:ff:fec6:79e8/128
                   *[Local/0] 00:29:59
                       Local via ens4
fe80::549a:16ff:feca:951c/128
                   *[Local/0] 00:29:50
                       Local via __crpd-brd2
fe80::54d7:5ff:fef2:bdee/128
                   *[IS-IS/15] 00:29:37, metric 10
                    >  to fe80::5054:ff:fe60:8b33 via ens4
fe80::6cc7:d5ff:fe44:1a62/128
                   *[Local/0] 00:29:50
                       Local via __crpd-tap2
fe80::a82a:d3ff:fee3:4097/128
                   *[IS-IS/15] 00:29:07, metric 10
                    >  to fe80::5054:ff:fec0:11df via ens4
fe80::dc27:64ff:fef8:1321/128
                   *[Local/0] 00:30:02
                       Local via eth0
fe80::ecee:eeff:feee:eeee/128
                   *[Local/0] 00:27:59
                       Local via cali75ffc0facfd
fe80::fc41:36ff:feb9:255f/128
                   *[Local/0] 00:30:02
                       Local via lsi
ff02::2/128        *[INET6/0] 00:30:02
                       MultiRecv

inet6.3: 2 destinations, 2 routes (2 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

2001:db8:4600::/48 *[SRV6-ISIS/14] 00:29:07, metric 10
                    >  to fe80::5054:ff:fec0:11df via ens4, SRV6-Tunnel, Dest: 2001:db8:4600::
2001:db8:4800::/48 *[SRV6-ISIS/14] 00:29:37, metric 10
                    >  to fe80::5054:ff:fe60:8b33 via ens4, SRV6-Tunnel, Dest: 2001:db8:4800::

vrf1.inet6.0: 3 destinations, 3 routes (3 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

abcd::a00:1/128    *[Local/0] 00:27:58
                       Local via jvknet1-365ab9c
abcd::a00:a/128    *[Static/5] 00:27:58
                    >  via jvknet1-365ab9c
ff02::2/128        *[INET6/0] 00:27:58
                       MultiRecv

root@jcnr-worker1> show bgp summary
Threading mode: BGP I/O
Default eBGP mode: advertise - accept, receive - accept
Groups: 1 Peers: 1 Down peers: 0
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
bgp.l3vpn.0
                       1          1          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
2001:1:1:1::1         65000         61         61       0       0       26:00 Establ
  bgp.l3vpn.0: 1/1/1/0
  vrf1.inet.0: 1/1/1/0
```
```
root@jcnr-worker2> show isis adjacency
Interface             System         L State         Hold (secs) SNPA
ens4                  jcnr-master    1  Up                    23  52:54:0:c0:11:df
ens4                  jcnr-master    2  Up                    22  52:54:0:c0:11:df
ens4                  jcnr-worker1   1  Up                     7  52:54:0:c6:79:e8
ens4                  jcnr-worker1   2  Up                     8  52:54:0:c6:79:e8

root@jcnr-worker2> show isis database detail
IS-IS level 1 link-state database:

jcnr-master.00-00 Sequence: 0x4, Checksum: 0xe041, Lifetime: 1091 secs
   IS neighbor: jcnr-worker1.02               Metric:       10
   V6 prefix: 2001:1:1:1::1/128               Metric:        0 Internal Up
   V6 prefix: 2001:192:168:1::/64             Metric:       10 Internal Up
   V6 prefix: 2001:db8:4600::/48              Metric:        0 Internal Up
   V6 prefix: fe80::a82a:d3ff:fee3:4097/128   Metric:        0 Internal Up

jcnr-worker1.00-00 Sequence: 0x6, Checksum: 0xa839, Lifetime: 492 secs
   IS neighbor: jcnr-worker1.02               Metric:       10
   V6 prefix: 2001:1:1:1::2/128               Metric:        0 Internal Up
   V6 prefix: 2001:192:168:1::/64             Metric:       10 Internal Up
   V6 prefix: 2001:db8:4700::/48              Metric:        0 Internal Up
   V6 prefix: fe80::44f4:17ff:fe2f:29ad/128   Metric:        0 Internal Up

jcnr-worker1.02-00 Sequence: 0x4, Checksum: 0xbd4b, Lifetime: 495 secs
   IS neighbor: jcnr-master.00                Metric:        0
   IS neighbor: jcnr-worker1.00               Metric:        0
   IS neighbor: jcnr-worker2.00               Metric:        0

jcnr-worker2.00-00 Sequence: 0x6, Checksum: 0xd487, Lifetime: 474 secs
   IS neighbor: jcnr-worker1.02               Metric:       10
   V6 prefix: 2001:1:1:1::3/128               Metric:        0 Internal Up
   V6 prefix: 2001:192:168:1::/64             Metric:       10 Internal Up
   V6 prefix: 2001:db8:4800::/48              Metric:        0 Internal Up
   V6 prefix: fe80::54d7:5ff:fef2:bdee/128    Metric:        0 Internal Up

IS-IS level 2 link-state database:

jcnr-master.00-00 Sequence: 0x6, Checksum: 0xde8d, Lifetime: 1091 secs
   IS neighbor: jcnr-worker1.02               Metric:       10
   V6 prefix: 2001:1:1:1::1/128               Metric:        0 Internal Up
   V6 prefix: 2001:1:1:1::2/128               Metric:       10 Internal Up
   V6 prefix: 2001:1:1:1::3/128               Metric:       10 Internal Up
   V6 prefix: 2001:192:168:1::/64             Metric:       10 Internal Up
   V6 prefix: 2001:db8:4600::/48              Metric:        0 Internal Up
   V6 prefix: 2001:db8:4700::/48              Metric:       10 Internal Up
   V6 prefix: 2001:db8:4800::/48              Metric:       10 Internal Up
   V6 prefix: fe80::44f4:17ff:fe2f:29ad/128   Metric:       10 Internal Up
   V6 prefix: fe80::54d7:5ff:fef2:bdee/128    Metric:       10 Internal Up
   V6 prefix: fe80::a82a:d3ff:fee3:4097/128   Metric:        0 Internal Up

jcnr-worker1.00-00 Sequence: 0x8, Checksum: 0x1212, Lifetime: 492 secs
   IS neighbor: jcnr-worker1.02               Metric:       10
   V6 prefix: 2001:1:1:1::1/128               Metric:       10 Internal Up
   V6 prefix: 2001:1:1:1::2/128               Metric:        0 Internal Up
   V6 prefix: 2001:1:1:1::3/128               Metric:       10 Internal Up
   V6 prefix: 2001:192:168:1::/64             Metric:       10 Internal Up
   V6 prefix: 2001:db8:4600::/48              Metric:       10 Internal Up
   V6 prefix: 2001:db8:4700::/48              Metric:        0 Internal Up
   V6 prefix: 2001:db8:4800::/48              Metric:       10 Internal Up
   V6 prefix: fe80::44f4:17ff:fe2f:29ad/128   Metric:        0 Internal Up
   V6 prefix: fe80::54d7:5ff:fef2:bdee/128    Metric:       10 Internal Up
   V6 prefix: fe80::a82a:d3ff:fee3:4097/128   Metric:       10 Internal Up

jcnr-worker1.02-00 Sequence: 0x4, Checksum: 0xbd4b, Lifetime: 492 secs
   IS neighbor: jcnr-master.00                Metric:        0
   IS neighbor: jcnr-worker1.00               Metric:        0
   IS neighbor: jcnr-worker2.00               Metric:        0

jcnr-worker2.00-00 Sequence: 0x9, Checksum: 0xf922, Lifetime: 1193 secs
   IS neighbor: jcnr-worker1.02               Metric:       10
   V6 prefix: 2001:1:1:1::1/128               Metric:       10 Internal Up
   V6 prefix: 2001:1:1:1::2/128               Metric:       10 Internal Up
   V6 prefix: 2001:1:1:1::3/128               Metric:        0 Internal Up
   V6 prefix: 2001:192:168:1::/64             Metric:       10 Internal Up
   V6 prefix: 2001:db8:4600::/48              Metric:       10 Internal Up
   V6 prefix: 2001:db8:4700::/48              Metric:       10 Internal Up
   V6 prefix: 2001:db8:4800::/48              Metric:        0 Internal Up
   V6 prefix: fe80::44f4:17ff:fe2f:29ad/128   Metric:       10 Internal Up
   V6 prefix: fe80::54d7:5ff:fef2:bdee/128    Metric:        0 Internal Up
   V6 prefix: fe80::a82a:d3ff:fee3:4097/128   Metric:       10 Internal Up

root@jcnr-worker2> show bgp summary
Threading mode: BGP I/O
Default eBGP mode: advertise - accept, receive - accept
Groups: 1 Peers: 1 Down peers: 0
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
bgp.l3vpn.0
                       1          1          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
2001:1:1:1::1         65000         64         64       0       0       27:36 Establ
  bgp.l3vpn.0: 1/1/1/0
  vrf2.inet.0: 1/1/1/0

root@jcnr-worker2> show route

inet.0: 9 destinations, 11 routes (9 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.233.0.1/32      *[Local/0] 00:29:42
                       Reject
10.233.0.3/32      *[Local/0] 00:29:42
                       Reject
10.233.7.156/32    *[Local/0] 00:29:42
                       Reject
10.233.10.71/32    *[Local/0] 00:29:42
                       Reject
10.233.81.64/32    *[Direct/0] 00:29:42
                    >  via vxlan.calico
                    [Local/0] 00:29:42
                       Local via vxlan.calico
169.254.25.10/32   *[Local/0] 00:29:42
                       Reject
169.254.73.216/32  *[Direct/0] 00:29:42
                    >  via eth0
                    [Local/0] 00:29:42
                       Local via eth0
172.27.112.0/22    *[Direct/0] 00:29:42
                    >  via ens3
172.27.115.208/32  *[Local/0] 00:29:42
                       Local via ens3

vrf2.inet.0: 3 destinations, 3 routes (3 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.10/32       *[BGP/170] 00:26:58, localpref 100, from 2001:1:1:1::1
                      AS path: I, validation-state: unverified
                    >  to fe80::5054:ff:fec6:79e8 via ens4, SRv6 SID: 2001:db8:4700:e001::, SRV6-Tunnel, Dest: 2001:db8:4700::
20.0.0.1/32        *[Local/0] 00:26:57
                       Local via jvknet1-81e476e
20.0.0.12/32       *[Static/5] 00:26:57
                    >  via jvknet1-81e476e

iso.0: 1 destinations, 1 routes (1 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

49.0001.0000.0000.0003/72
                   *[Direct/0] 00:29:02
                    >  via lo0.0

bgp.l3vpn.0: 1 destinations, 1 routes (1 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

172.27.115.207:3:10.0.0.10/32
                   *[BGP/170] 00:26:58, localpref 100, from 2001:1:1:1::1
                      AS path: I, validation-state: unverified
                    >  to fe80::5054:ff:fec6:79e8 via ens4, SRv6 SID: 2001:db8:4700:e001::, SRV6-Tunnel, Dest: 2001:db8:4700::

inet6.0: 28 destinations, 30 routes (28 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

::/96              *[Direct/0] 00:29:42
                    >  via sit0
                    [Direct/0] 00:29:42
                    >  via sit0
                    [Direct/0] 00:29:42
                    >  via sit0
::127.0.0.1/128    *[Local/0] 00:29:42
                       Local via sit0
::169.254.73.216/128
                   *[Local/0] 00:29:42
                       Local via sit0
::172.27.115.208/128
                   *[Local/0] 00:29:42
                       Local via sit0
2001:1:1:1::1/128  *[IS-IS/15] 00:28:01, metric 10
                    >  to fe80::5054:ff:fec0:11df via ens4
2001:1:1:1::2/128  *[IS-IS/15] 00:28:45, metric 10
                    >  to fe80::5054:ff:fec6:79e8 via ens4
2001:1:1:1::3/128  *[Direct/0] 00:29:03
                    >  via lo0.0
2001:192:168:1::/64*[Direct/0] 00:29:03
                    >  via ens4
2001:192:168:1::3/128
                   *[Local/0] 00:29:03
                       Local via ens4
2001:db8:4600::/48 *[IS-IS/15] 00:28:01, metric 10
                    >  to fe80::5054:ff:fec0:11df via ens4
2001:db8:4700::/48 *[IS-IS/15] 00:28:45, metric 10
                    >  to fe80::5054:ff:fec6:79e8 via ens4
2001:db8:4800::/48 *[IS-IS/15] 00:29:02, metric 0
                       Receive
2001:db8:4800:e001::/80
                   *[SRV6/171] 00:26:58
                       to table vrf2.inet.0
2001:db8:e001::/64 *[SRV6/171] 00:26:58
                       to table vrf2.inet.0
fe80::1/128        *[Direct/0] 00:29:42
                    >  via lo
fe80::200:ff:fe00:0/128
                   *[Local/0] 00:29:42
                       Local via ip6tnl0
fe80::8af:5bff:fe91:3f88/128
                   *[Local/0] 00:29:42
                       Local via eth0
fe80::44f4:17ff:fe2f:29ad/128
                   *[IS-IS/15] 00:28:45, metric 10
                    >  to fe80::5054:ff:fec6:79e8 via ens4
fe80::5054:ff:fe60:8b33/128
                   *[Local/0] 00:29:42
                       Local via ens4
fe80::5054:ff:fe8f:967d/128
                   *[Local/0] 00:29:42
                       Local via ens3
fe80::54d7:5ff:fef2:bdee/128
                   *[Local/0] 00:29:42
                       Local via lo0.0
fe80::6450:80ff:fe8f:7a8/128
                   *[Local/0] 00:29:42
                       Local via vxlan.calico
fe80::6804:74ff:fef1:a2f3/128
                   *[Local/0] 00:29:42
                       Local via lsi
fe80::a82a:d3ff:fee3:4097/128
                   *[IS-IS/15] 00:28:01, metric 10
                    >  to fe80::5054:ff:fec0:11df via ens4
fe80::e4a7:16ff:fe4f:6efa/128
                   *[Local/0] 00:29:00
                       Local via __crpd-brd2
fe80::e853:f4ff:feef:d65d/128
                   *[Local/0] 00:29:01
                       Local via __crpd-tap2
fe80::ecee:eeff:feee:eeee/128
                   *[Local/0] 00:29:42
                       Local via calidfbad1972db
ff02::2/128        *[INET6/0] 00:29:42
                       MultiRecv

inet6.3: 2 destinations, 2 routes (2 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

2001:db8:4600::/48 *[SRV6-ISIS/14] 00:28:01, metric 10
                    >  to fe80::5054:ff:fec0:11df via ens4, SRV6-Tunnel, Dest: 2001:db8:4600::
2001:db8:4700::/48 *[SRV6-ISIS/14] 00:28:45, metric 10
                    >  to fe80::5054:ff:fec6:79e8 via ens4, SRV6-Tunnel, Dest: 2001:db8:4700::

vrf2.inet6.0: 3 destinations, 3 routes (3 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

abcd::1400:1/128   *[Local/0] 00:26:57
                       Local via jvknet1-81e476e
abcd::1400:c/128   *[Static/5] 00:26:57
                    >  via jvknet1-81e476e
ff02::2/128        *[INET6/0] 00:26:57
                       MultiRecv
```


vRouter VIF確認
```
[root@jcnr-master ~]# kubectl exec -it jcnr-1-contrail-vrouter-nodes-vrdpdk-lckks -n contrail -- bash

bash-5.1# flow --match 10.0.0.10
Flow table(size 161218560, entries 629760)

Entries: Created 6 Added 6 Deleted 10 Changed 10Processed 6 Used Overflow entries 0
(Created Flows/CPU: 0 0 0 0 0 0 0 0 0 0 0 6 0 0 0)(oflows 0)

Action:F=Forward, D=Drop N=NAT(S=SNAT, D=DNAT, Ps=SPAT, Pd=DPAT, L=Link Local Port)
 Other:K(nh)=Key_Nexthop, S(nh)=RPF_Nexthop
 Flags:E=Evicted, Ec=Evict Candidate, N=New Flow, M=Modified Dm=Delete Marked
TCP(r=reverse):S=SYN, F=FIN, R=RST, C=HalfClose, E=Established, D=Dead
 Stats:Packets/Bytes

Listing flows matching ([10.0.0.10]:*)

    Index                Source:Port/Destination:Port                      Proto(V)
-----------------------------------------------------------------------------------
   345212<=>493600       10.0.0.10:45                                        1 (2)
                         20.0.0.12:0
(Gen: 1, K(nh):2, Action:F, Flags:, QOS:-1, S(nh):20,  Stats:182/17836,
 SPort 52076, TTL 0, Sinfo 7.0.0.0)

   493600<=>345212       20.0.0.12:45                                        1 (2)
                         10.0.0.10:0
(Gen: 1, K(nh):2, Action:F, Flags:, QOS:-1, S(nh):23,  Stats:182/15288,
 SPort 64915, TTL 0, Sinfo 0.0.0.0)


bash-5.1# rt --get 20.0.0.12/32 --vrf 2
Match 20.0.0.12/32 in vRouter inet4 table 0/2/unicast

Flags: L=Label Valid, P=Proxy ARP, T=Trap ARP, F=Flood ARP, Ml=MAC-IP learnt route
vRouter inet4 routing table 0/2/unicast
Destination           PPL        Flags        Label         Nexthop    Stitched MAC(Index)
20.0.0.12/32            0          LPT          -             23        -


bash-5.1# nh --get 23
Id:23         Type:Tunnel         Fmly:AF_INET6  Rid:0  Ref_cnt:2          Vrf:0
              Flags:Valid, Policy, Etree Root, SRv6,
              Oif:1 Len:14 Data:52 54 00 60 8b 33 52 54 00 c6 79 e8 86 dd
              Sip: 2001:1:1:1::2
              Block Len:32 Block: 2001:db8::
              Number of Containers:1
              Container Dips:[1]: 2001:db8:4800:e001::
```

POD間疎通確認
```
[root@vrf1-pod1 /]# ping 20.0.0.12
PING 20.0.0.12 (20.0.0.12) 56(84) bytes of data.
64 bytes from 20.0.0.12: icmp_seq=1 ttl=62 time=0.908 ms
64 bytes from 20.0.0.12: icmp_seq=2 ttl=62 time=0.165 ms
64 bytes from 20.0.0.12: icmp_seq=3 ttl=62 time=0.122 ms
```
