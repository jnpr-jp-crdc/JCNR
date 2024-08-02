# JCNR Install (Multi Worker Node Kubernetes Cluster)手順
- 本手順はKVM上にRHEL VMを3台デプロイし、Multi Worker Node Kubernetes Clusterを構築し、JCNRをインストールする手順となります。
- 本手順はJCNR 24.2をベースとしたインストール手順となります。
- JCNR Master NodeがBGP RRとなる

## 環境
Hypervisor: 
- OS: RHEL 8.6

JCNR VM:
- OS: RHEL8.6
- Kernel(RealTime) : 4.18.0-305.rt7.72.el8.x86_64
- K8S : 1.25.4
- Calico : 3.22.x
- Multus : 3.8
- Helm : 3.9.x
- Container-RT : Containerd 1.7.x
- UIO Driver : VFIO-PCI
- Master NIC: ens3：172.27.115.206/22(Management), ens4：192.168.0.1/24(L3 Interface), ens5:none(L2 Interface)
- Worker1 NIC: ens3：172.27.115.207/22(Management), ens4：192.168.0.2/24(L3 Interface), ens5:none(L2 Interface)
- Worker2 NIC: ens3：172.27.115.208/22(Management), ens4：192.168.0.3/24(L3 Interface), ens5:none(L2 Interface)

※JCNRはNICをL2,L3 Modeのどちらで動作させるかを選択できます。必要数に応じてNICを追加してください。

※ens4, ens5はJCNR側で設定するため、RHEL側での設定は不要

## KVM 設定
### HugePage 設定
- PageSizeは1G必須
- Paze数は環境に合わせて変更
```
mkdir /dev/hugepages1G
mount -t hugetlbfs -o pagesize=1G none /dev/hugepages1G
echo 32 > /sys/devices/system/node/node1/hugepages/hugepages-1048576kB/nr_hugepages
echo 32 > /sys/devices/system/node/node0/hugepages/hugepages-1048576kB/nr_hugepages
mount -t hugetlbfs -o pagesize=1G none /dev/hugepages1G
```

/etc/fstab
```
none /dev/hugepages1G hugetlbfs pagesize=1G,size=64G 0 0
```

/etc/libvirt/qemu.conf
```
hugetlbfs_mount = "/dev/hugepages1G"
```

/etc/default/qemu-kvm
```
KVM_HUGEPAGES=1
```

Libvirt Restart
```
systemctl restart libvirtd
```

### JCNR VM デプロイ
```
virt-install --name jcnr-master --vcpus=8 --ram=32768 --location=/home/images/rhel-8.6-x86_64-dvd.iso --disk path=/home/images/jcnr-master.qcow2,format=qcow2,size=100 --network bridge=br-mng,model=virtio --network bridge=br01,model=virtio --network bridge=br02,model=virtio --virt-type kvm --os-variant rhel7 --serial pty --console pty,target_type=virtio --graphics vnc,listen=0.0.0.0 --noautoconsole

virt-install --name jcnr-worker1 --vcpus=8 --ram=32768 --location=/home/images/rhel-8.6-x86_64-dvd.iso --disk path=/home/images/jcnr-worker1.qcow2,format=qcow2,size=100 --network bridge=br-mng,model=virtio --network bridge=br01,model=virtio --network bridge=br02,model=virtio --virt-type kvm --os-variant rhel7 --serial pty --console pty,target_type=virtio --graphics vnc,listen=0.0.0.0 --noautoconsole

virt-install --name jcnr-worker2 --vcpus=8 --ram=32768 --location=/home/images/rhel-8.6-x86_64-dvd.iso --disk path=/home/images/jcnr-worker2.qcow2,format=qcow2,size=100 --network bridge=br-mng,model=virtio --network bridge=br01,model=virtio --network bridge=br02,model=virtio --virt-type kvm --os-variant rhel7 --serial pty --console pty,target_type=virtio --graphics vnc,listen=0.0.0.0 --noautoconsole

```
#### JCNR VMのHuagePage設定及びvCPU Pinning
virth edit jcnr-XXX
```
  <cpu mode='host-passthrough' check='none'>
    <feature policy='force' name='pdpe1gb'/>
  </cpu>
  <cputune>
    <vcpupin vcpu='0' cpuset='1'/>
    <vcpupin vcpu='1' cpuset='2'/>
    <vcpupin vcpu='2' cpuset='3'/>
    <vcpupin vcpu='3' cpuset='4'/>
    <vcpupin vcpu='4' cpuset='5'/>
    <vcpupin vcpu='5' cpuset='6'/>
    <vcpupin vcpu='6' cpuset='7'/>
    <vcpupin vcpu='7' cpuset='8'/>
  </cputune>
  <memoryBacking>
    <hugepages>
      <page size='1048576' unit='KiB'/> 
    </hugepages>
  </memoryBacking>

```

## JCNR VM 初期設定
### RHEL Subscription
- "Red Hat Enterprise Linux for Real Time"を含むSubscription
```
subscription-manager register
subscription-manager list
subscription-manager list --available
subscription-manager attach --pool=XXXXX
```

### RHEL Realtime Kernel Install
```
subscription-manager repos --enable rhel-8-for-x86_64-rt-rpms
yum -y install kernel-rt-4.18.0-305.rt7.72.el8.x86_64 kernel-headers-4.18.0-305.el8.x86_64 kernel-rt-devel-4.18.0-305.rt7.72.el8.x86_64 kernel-rt-modules-extra-4.18.0-305.rt7.72.el8.x86_64
yum --exclude=kernel* groupinstall RT -y
```

### RHEL OS 設定
#### NTP
```
yum install -y chrony
systemctl enable --now chronyd
chronyc sources
```
#### SELinux
```
sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
```
#### SWAP無効
```
sed -i 's%/dev/mapper/rhel-swap%#/dev/mapper/rhel-swap%' /etc/fstab
```
#### Firewall無効
```
systemctl stop firewalld
systemctl disable firewalld
```
#### Hugepage
/etc/default/grub
```
GRUB_CMDLINE_LINUX="crashkernel=auto rd.lvm.lv=rhel/root rhgb quiet default_hugepagesz=1G hugepagesz=1G hugepages=16 intel_iommu=on iommu=pt"
```
```
grub2-mkconfig -o /boot/grub2/grub.cfg
reboot
```
Reboot後HugePage確認
```
cat /proc/meminfo | grep Huge
cat /proc/sys/vm/nr_hugepages
lscpu | grep -i numa
```

#### JCNRに必要なDriverのLoad
```
echo '8021q' > /etc/modules-load.d/8021q.conf
modprobe 8021q

echo 'tun' > /etc/modules-load.d/tun.conf
modprobe tun

echo 'ipip' > /etc/modules-load.d/ipip.conf
modprobe ipip

echo 'ip_tunnel' > /etc/modules-load.d/ip_tunnel.conf
modprobe ip_tunnel

echo 'ip6_tunnel' > /etc/modules-load.d/ip6_tunnel.conf
modprobe ip6_tunnel

echo 'mpls_gso' > /etc/modules-load.d/mpls_gso.conf
modprobe mpls_gso

echo 'mpls_router' > /etc/modules-load.d/mpls_router.conf
modprobe mpls_router

echo 'mpls_iptunnel' > /etc/modules-load.d/mpls_iptunnel.conf
modprobe mpls_iptunnel

echo 'vrf' > /etc/modules-load.d/vrf.conf
modprobe vrf

echo 'vxlan' > /etc/modules-load.d/vxlan.conf
modprobe vxlan
```

/etc/sysconfig/modules/vfio-noiommu.modules
```
#!/bin/sh
echo Y > /sys/module/vfio/parameters/enable_unsafe_noiommu_mode
modprobe vfio enable_unsafe_noiommu_mode=1
modprobe vfio-pci
```
```
chmod +x /etc/sysconfig/modules/vfio-noiommu.modules
modprobe vfio enable_unsafe_noiommu_mode=1
```

/etc/sysctl.conf
```
net.ipv6.conf.default.addr_gen_mode=0
net.ipv6.conf.all.addr_gen_mode=0
net.ipv6.conf.default.autoconf=0
net.ipv6.conf.all.autoconf=0
```
```
sysctl -p /etc/sysctl.conf
```

#### JCNRで使用するNICのNetworkManager無効化
/etc/NetworkManager/conf.d/crpd.conf
```
[keyfile]
unmanaged-devices+=interface-name:ens4;interface-name:ens5
```
```
systemctl restart NetworkManager
```

/etc/rc.local
```
ip link set up ens4
ip link set up ens5
```
```
chmod +x /etc/rc.local
```

## JCNR VM Kubernetesインストール
- Kubesprayを使用し、K8Sをインストール
- 本手順ではJCNRのRequirementを満たしていないが、必要に応じてKubesprayでインストールする各種Versionを揃える
### 初期設定 (Master Node Only)
```
ssh-keygen
ssh-copy-id 172.27.115.206
ssh-copy-id 172.27.115.207
ssh-copy-id 172.27.115.208
```
```
yum install -y python39 git
git clone https://github.com/kubernetes-sigs/kubespray.git -b v2.23.0
cd kubespray
pip3 install -r requirements.txt
```
```
cp -rfp inventory/sample inventory/mycluster
declare -a IPS=(172.27.115.206 172.27.115.207 172.27.115.208)
CONFIG_FILE=inventory/mycluster/hosts.yml python3 contrib/inventory_builder/inventory.py ${IPS[@]}
```

inventory/mycluster/hosts.yml
```
all:
  hosts:
    jcnr-master:
      ansible_host: 172.27.115.206
      ip: 172.27.115.206
      access_ip: 172.27.115.206
    jcnr-worker1:
      ansible_host: 172.27.115.207
      ip: 172.27.115.207
      access_ip: 172.27.115.207
    jcnr-worker2:
      ansible_host: 172.27.115.208
      ip: 172.27.115.208
      access_ip: 172.27.115.208
  children:
    kube_control_plane:
      hosts:
        jcnr-master:
    kube_node:
      hosts:
        jcnr-master:
        jcnr-worker1:
        jcnr-worker2:
    etcd:
      hosts:
        jcnr-master:
    k8s_cluster:
      children:
        kube_control_plane:
        kube_node:
    calico_rr:
      hosts: {}
```

roles/kubespray-defaults/defaults/main.yaml
```
kube_network_plugin_multus: true
enable_dual_stack_networks: true
```

roles/network_plugin/calico/defaults/main.yml

※JCNR cRPDとCalicoが使用するBGPのPortが重複するため、Calico側のPort番号を変更
```
calico_bgp_listen_port: 178
```

Kubespray Bug対応

roles/kubernetes/preinstall/tasks/0040-verify-settings.yml
```
 - name: Stop if either kube_control_plane or kube_node group is empty assert:
-    that: "groups.get('{{ item }}')"
+    that: "groups.get( item )"
```

### K8S Install (Master Node Only)
```
ansible-playbook -i inventory/mycluster/hosts.yml cluster.yml
```

### Multusインストール (Master Node Only)
```
git clone https://github.com/k8snetworkplumbingwg/multus-cni.git
```
```
cd multus-cni
cat ./multus-cni/deployments/multus-daemonset-thick.yml | kubectl apply -f -
```

すべてのPodがRunningになっていることを確認
```
kubectl get pods -A
```

## JCNR インストール
### Helm Install (Master Node Only)
```
cd /root/
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```
### JCNR Imageの取得 (All Node)
- Software Download SiteからDownloadまたはJuniper社へお問い合わせください
```
tar xzvf Juniper_Cloud_Native_Router_24.2-354.tar.gz
cd Juniper_Cloud_Native_Router_24.2-354
gunzip images/jcnr-images.tar.gz
ctr -n k8s.io images import images/jcnr-images.tar
```

### JCNR Helmchart編集 (Master Node Only)
helmchart/jcnr/values.yaml
- "interface_mode"の設定がないInterfaceはL3 Interface, 設定があるInterfaceはL2 Interfaceとなる
```
global:
  replicas: "3"
  fabricInterface:
  # L2L3
  - ens4:
      ddp: "off"
  - ens5:
      ddp: "off" 
      interface_mode: trunk
      vlan-id-list: [100, 200, 300, 700-705]
      storm-control-profile: rate_limit_pf1
      native-vlan-id: 100
      no-local-switching: true
jcnr-vrouter:
  cpu_core_mask: "1,2,3,4"
contrail-tools:
  install: true
```

### JCNR License
- Trial Licenseの発行はJuniper社へお問い合わせください
- JCNR License(S-CRPD-x-x-PF)を取得
```
base64 -w 0 license.txt
```
※license.txt内のデータは改行なしのlicense code (以下サンプル)

E20231213001 aeaqie ajagap jafuaa aaacql cfjs2q 2skbcc 2mjqfv atclkq iywtkd an4kaj ystvnz uxazls 4kaj2b zilqzw owdobt 7jwigm rrzyhi 7pqqxk vgculm vzpl4c o6uya5 wlo7fm 3raot4 k6kxz2 kaza
```
echo 'ROOT_PASSWORD' | base64
```
※ROOT_PASSWORDはRHELのroot password

secrets/jcnr-secrets.yaml
- BASE64_ROOT_PASSWORD, BASE64_JCNR_LICENSEを上記値に編集
```
---
apiVersion: v1
kind: Namespace
metadata:
  name: jcnr
---
apiVersion: v1
kind: Secret
metadata:
  name: jcnr-secrets
  namespace: jcnr
data:
  root-password: BASE64_ROOT_PASSWORD
  crpd-license: |
    BASE64_JCNR_LICENSE
```

### JCNR デプロイ
```
helm install jcnr .
```
全てのPodがRunniningになっていることを確認
```
[root@jcnr-master ~]# kubectl get pods -A
NAMESPACE         NAME                                         READY   STATUS    RESTARTS      AGE
contrail-deploy   contrail-k8s-deployer-7d874fd4c8-fpp6t       1/1     Running   0             17h
contrail          contrail-tools-js6zw                         1/1     Running   0             17h
contrail          contrail-tools-q975k                         1/1     Running   0             17h
contrail          contrail-tools-vwhjq                         1/1     Running   0             17h
contrail          jcnr-0-contrail-vrouter-nodes-jjznw          2/2     Running   0             17h
contrail          jcnr-0-contrail-vrouter-nodes-vrdpdk-ppmld   1/1     Running   0             17h
contrail          jcnr-1-contrail-vrouter-nodes-84tcq          2/2     Running   0             17h
contrail          jcnr-1-contrail-vrouter-nodes-vrdpdk-lckks   1/1     Running   0             17h
contrail          jcnr-2-contrail-vrouter-nodes-qttm9          2/2     Running   0             17h
contrail          jcnr-2-contrail-vrouter-nodes-vrdpdk-fskt9   1/1     Running   0             17h
jcnr              jcnr-0-crpd-0                                2/2     Running   0             17h
jcnr              jcnr-1-crpd-0                                2/2     Running   0             17h
jcnr              jcnr-2-crpd-0                                2/2     Running   0             17h
jcnr              syslog-ng-6nmc6                              1/1     Running   0             17h
jcnr              syslog-ng-fv9fx                              1/1     Running   0             17h
jcnr              syslog-ng-nz8hn                              1/1     Running   0             17h
kube-system       calico-kube-controllers-794577df96-4mxnx     1/1     Running   1 (17h ago)   18h
kube-system       calico-node-8g7mz                            1/1     Running   1 (17h ago)   18h
kube-system       calico-node-ns5z9                            1/1     Running   1 (17h ago)   18h
kube-system       calico-node-q6nbk                            1/1     Running   1 (17h ago)   18h
kube-system       coredns-5c469774b8-667hj                     1/1     Running   1 (17h ago)   18h
kube-system       coredns-5c469774b8-kkchx                     1/1     Running   1 (17h ago)   18h
kube-system       dns-autoscaler-f455cf558-s4c5p               1/1     Running   1 (17h ago)   18h
kube-system       kube-apiserver-jcnr-master                   1/1     Running   2 (17h ago)   18h
kube-system       kube-controller-manager-jcnr-master          1/1     Running   3 (17h ago)   18h
kube-system       kube-multus-ds-6bpd7                         1/1     Running   0             98m
kube-system       kube-multus-ds-sn8p9                         1/1     Running   0             98m
kube-system       kube-multus-ds-xk5g9                         1/1     Running   0             98m
kube-system       kube-proxy-8n7l4                             1/1     Running   1 (17h ago)   18h
kube-system       kube-proxy-ddgzq                             1/1     Running   1 (17h ago)   18h
kube-system       kube-proxy-fmjr7                             1/1     Running   1 (17h ago)   18h
kube-system       kube-scheduler-jcnr-master                   1/1     Running   2 (17h ago)   18h
kube-system       nginx-proxy-jcnr-worker1                     1/1     Running   1 (17h ago)   18h
kube-system       nginx-proxy-jcnr-worker2                     1/1     Running   1 (17h ago)   18h
kube-system       nodelocaldns-49bjn                           1/1     Running   1 (17h ago)   18h
kube-system       nodelocaldns-6ldl7                           1/1     Running   1 (17h ago)   18h
kube-system       nodelocaldns-786bm                           1/1     Running   1 (17h ago)   18h
```

### JCNR Login方法
cRPD Login
```
kubectl exec -n jcnr -it jcnr-X-crpd-0 -- bash
cli
```

vRouter Login
```
kubectl exec -n contrail -it jcnr-0-contrail-vrouter-nodes-XXXX -- bash
```

### JCNR 初期Config
```
root@jcnr-master> show configuration | display set
set version 20240621.103832_builder.r1429411
set groups base apply-flags omit
set groups base apply-macro ht jcnr
set groups base system root-authentication encrypted-password "$6$QrpuHcYfYOiR4aPg$uKpKRXTycSAJCb7gQJhQPN1hlYWEXzXizq7bxp.kAo55kgzFMhxlXwQHOxpdtR1VoZYIvmovyWVAUo6BgN/Io0"
set groups base system commit xpath
set groups base system commit constraints direct-access
set groups base system commit notification configuration-diff-format xml
set groups base system scripts action max-datasize 256m
set groups base system scripts language python3
set groups base system services netconf ssh
set groups base system services ssh root-login allow
set groups base system services ssh port 24
set groups base system services extension-service request-response grpc clear-text address 127.0.0.1
set groups base system services extension-service request-response grpc clear-text port 50053
set groups base system services extension-service request-response grpc skip-authentication
set groups base system syslog host 127.0.0.1 any any
set groups base system syslog host 127.0.0.1 kernel none
set groups base system syslog host 127.0.0.1 interactive-commands notice
set groups base system syslog host 127.0.0.1 port 50055
set groups base system syslog host 127.0.0.1 transport udp
set groups base system syslog host 127.0.0.1 match-strings license-check
set groups base system syslog host 127.0.0.1 match-strings jcnr-init
set groups base system syslog host 127.0.0.1 match-strings cli
set groups base system syslog host 127.0.0.1 match-strings _LOG
set groups base system syslog host 127.0.0.1 structured-data
set groups base system syslog source-address 127.0.0.1
set groups base system license keys key "E20231213001 aeaqie ajagap jafuaa aaacql cfjs2q 2skbcc 2mjqfv atclkq iywtkd an4kaj ystvnz uxazls 4kaj2b zilqzw owdobt 7jwigm rrzyhi 7pqqxk vgculm vzpl4c o6uya5 wlo7fm 3raot4 k6kxz2 kaza"
set groups base interfaces irb unit 0 mac 48:5a:0d:ab:14:78
set groups base routing-options route-distinguisher-id 172.27.115.206
set groups base routing-options resolution rib :gribi.inet6.0 inet6-resolution-ribs :gribi.inet6.0
set groups base routing-options forwarding-table channel vrouter protocol protocol-type gRPC
set groups base routing-options forwarding-table channel vrouter protocol destination 127.0.0.1:50052
set groups cni apply-flags omit
set groups cni apply-macro ht jcnr
set groups internal apply-flags omit
set groups internal apply-macro ht jcnr
set groups internal system commit xpath
set groups internal system commit constraints direct-access
set groups internal system commit notification configuration-diff-format xml
set groups internal system scripts language python3
set groups internal event-options policy save_config events UI_COMMIT_COMPLETED
set groups internal event-options policy save_config then execute-commands commands "start shell sh command /config/scripts/save-config.sh"
set groups internal event-options policy save_config then execute-commands output-filename save_config
deactivate groups internal event-options policy save_config then execute-commands output-filename
set groups internal event-options policy save_config then execute-commands destination local
deactivate groups internal event-options policy save_config then execute-commands destination local
set groups internal event-options policy save_config then execute-commands output-format text
set groups internal event-options destinations local archive-sites /var/tmp
deactivate groups internal event-options destinations
set groups l2-config interfaces ens5 native-vlan-id 100
set groups l2-config interfaces ens5 unit 0 family bridge interface-mode trunk
set groups l2-config interfaces ens5 unit 0 family bridge vlan-id-list 100
set groups l2-config interfaces ens5 unit 0 family bridge vlan-id-list 200
set groups l2-config interfaces ens5 unit 0 family bridge vlan-id-list 300
set groups l2-config interfaces ens5 unit 0 family bridge vlan-id-list 700-705
set groups l2-config interfaces ens5 unit 0 family bridge ce-facing
set groups l2-config routing-instances vswitch instance-type virtual-switch
set groups l2-config routing-instances vswitch bridge-domains bd100 vlan-id 100
set groups l2-config routing-instances vswitch bridge-domains bd200 vlan-id 200
set groups l2-config routing-instances vswitch bridge-domains bd300 vlan-id 300
set groups l2-config routing-instances vswitch bridge-domains bd700 vlan-id 700
set groups l2-config routing-instances vswitch bridge-domains bd701 vlan-id 701
set groups l2-config routing-instances vswitch bridge-domains bd702 vlan-id 702
set groups l2-config routing-instances vswitch bridge-domains bd703 vlan-id 703
set groups l2-config routing-instances vswitch bridge-domains bd704 vlan-id 704
set groups l2-config routing-instances vswitch bridge-domains bd705 vlan-id 705
set groups l2-config routing-instances vswitch interface ens5
set apply-groups base
set apply-groups internal
set apply-groups l2-config
```

### JCNR Configlet設定
- ConfigletはJCNR Node毎に異なるパラメータを定義し、JCNRデプロイ後にConfigurationに追加、変更、削除が可能
- 以下はSR-MPLSのサンプル設定
/root/configlet-jcnr-master.yaml
```
apiVersion: configplane.juniper.net/v1
kind: Configlet
metadata:
  name: configlet-jcnr-master
  namespace: jcnr
spec:
  config: |-
    set interfaces lo0 unit 0 family iso address 49.0001.0000.0000.0001.00
    set interfaces lo0 unit 0 family inet address 1.1.1.1/32
    set interfaces ens4 unit 0 family inet address 192.168.1.1/24
    set routing-options router-id 1.1.1.1
    set routing-options route-distinguisher-id 1.1.1.1
    set protocols isis interface ens4
    set protocols isis interface lo0.0
    set protocols isis source-packet-routing node-segment ipv4-index 1
    set protocols isis source-packet-routing srgb start-label 400000
    set protocols isis source-packet-routing srgb index-range 99999
    set protocols mpls interface ens4
    set protocols bgp group jcnrbgp1 type internal
    set protocols bgp group jcnrbgp1 local-address 1.1.1.1
    set protocols bgp group jcnrbgp1 local-as 65000
    set protocols bgp group jcnrbgp1 neighbor 1.1.1.2
    set protocols bgp group jcnrbgp1 neighbor 1.1.1.3
    set protocols bgp group jcnrbgp1 cluster 1.1.1.1
    set protocols bgp group jcnrbgp1 family inet-vpn unicast
    set protocols bgp group jcnrbgp1 family inet6-vpn unicast
  crpdSelector:
    matchLabels:
      kubernetes.io/hostname: jcnr-master

```
```
kubectl apply -f /root/configlet-jcnr-master.yaml
```
