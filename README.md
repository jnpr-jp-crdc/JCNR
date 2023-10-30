# Juniper Cloud Native Router (JCNR) 概要

Juniper Cloud Native Router (JCNR)は、コントロールプレーンにcRPD、データプレーンにContrail vRouterを使用したコンテナ型ルータです。

オンプレ環境(K8S, OpenShift, Windriver)とクラウド環境(EKS, GCP, AKS)で利用でき、スペースやスペックが限られた環境においてハイパフォーマンスで高機能なスイッチング・ルーティング機能を提供します。

JCNRでは２つの動作モードをサポート
- CNI Mode
  クラウドネイティブ環境においてSecondary CNIとして動作し、マニフェストファイル及びJUNOS CLIからRouting Instance(VRF)の作成、PODへの仮想NWへのアタッチが可能です。
  
- CNF Mode (Transit Gateway)
  User Application(POD)を使用せず、JCNRをコンテナ環境におけるコンテナルータとして稼働させることが可能です。

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
    - VRF EVPN/VXLAN Type2 : Roadmap
    - MPLS/E-LINE : Roadmap
    - EVPN/E-LAN, E-LINE : Roadmap

      
  - L3
    - [VRF Lite - JCNR間接続](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Docs/vrf-vrouter-inter-jcnr.md)
    - [VRF Lite - VirtualRouter間接続](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Docs/vrf-vrouter-inter-vrouter.md)
    - [VRF Lite - Sub Interface](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Docs/vrf-vrouter-sub-int.md)
    - [VRF L3VPN/SR-MPLS - JCNR間接続](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Docs/vrf-vrf-inter-jcnr-srmpls.md)
    - [VRF L3VPN/MPLSoUDP - JCNR間接続](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Docs/vrf-vrf-inter-jcnr-mplsoudp.md)
    - [VRF EVPN/VXLAN Type5 - JCNR間接続](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Docs/vrf-vrf-inter-jcnr-evpn.md)
    - L3 ACL : Roadmap
    - SRv6 : Roadmap
    - FlexArgo : Roadmap
- JCNR CNF Mode (Transit Gateway)
    - [VRF L3VPN/SR-MPLS](https://github.com/jnpr-jp-crdc/JCNR/blob/main/Docs/cnf-srmpls.md)
