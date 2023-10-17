# Juniper Cloud Native Router (JCNR) 概要

XXXXXX

# チュートリアル
- JCNR Install
  - [Single Node K8S Cluster](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Docs/Install-Single.md)
  - [Multi Node K8S Cluster](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Docs/Install-Multi.md)

- JCNR CNI Mode
  - L2
    - [VRF vSwitch - Pod Vlan Access](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Docs/vrf-vswitch-access.md)
    - [VRF vSwitch - Pod Vlan Trunk](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Docs/vrf-vswitch-trunk.md)
    - [VRF vSwitch - Pod Vlan Sub Interface](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Docs/vrf-vswitch-sub-int.md)
    - [VRF vSwitch - MAC Filter](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Docs/vrf-vswitch-mac-filter.md)
    - VRF EVPN/VXLAN : 未サポート(NADで使用するVRF TypeにMacVRFが使用できないため)
      
  - L3
    - [VRF Lite - JCNR間接続](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Docs/vrf-vrouter-inter-jcnr.md)
    - [VRF Lite - VirtualRouter間接続](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Docs/vrf-vrouter-inter-vrouter.md)
    - [VRF Lite - Sub Interface](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Docs/vrf-vrouter-sub-int.md)
    - [VRF L3VPN/SR-MPLS - JCNR間接続](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Docs/vrf-vrf-inter-jcnr-srmpls.md)
    - [VRF L3VPN/MPLSoUDP - JCNR間接続](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Docs/vrf-vrf-inter-jcnr-mplsoudp.md)
    - [VRF EVPN/VXLAN - JCNR間接続](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Docs/vrf-vrf-inter-jcnr-evpn.md)
