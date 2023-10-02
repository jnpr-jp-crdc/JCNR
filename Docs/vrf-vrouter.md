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
                        "172.30.79.46"
                    ],
                    "default": true,
                    "dns": {}
                },{
                    "name": "default/vrouter1",
                    "interface": "mynet10",
                    "ips": [
                        "10.0.0.5"
                    ],
                    "mac": "02:00:00:D5:C8:DD",
                    "dns": {}
                }]
              k8s.v1.cni.cncf.io/networks:
                [
                  {
                    "name": "vrouter1",
                    "interface":"mynet10",
                    "cni-args": {
                      "interfaceType":"veth"
                    }
                  }
                ]
Status:       Running
--- snip ---
```
