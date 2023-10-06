![image](https://github.com/jnpr-jp-crdc/JCNR/assets/102126891/c21e6e2e-2d01-4692-9a47-ed3a2ab492e5)# JCNR Install (Single Kubernetes Node)手順
- 本手順はKVM上にRHEL VMを1デプロイし、Single Node Kubernetes Clusterを構築し、JCNRをインストールする手順となります。
- 本手順はJCNR 23.2をベースとしたインストール手順となります。

## 環境
Hypervisor: 
- OS: CentOS 7.9

JCNR VM:
- OS: RHEL8.6
- Kernel(RealTime) : 4.18.0-305.rt7.72.el8.x86_64
- K8S : 1.25.4
- Calico : 3.22.x
- Multus : 3.8
- Helm : 3.9.x
- Container-RT : Docker CE 20.10.11
- UIO Driver : VFIO-PCI
- NIC: ens3：172.27.115.12(Management), ens4：192.168.0.1/24(L3 Interface), ens5:none(L2 Interface)

※JCNRはNICをL2,L3 Modeのどちらで動作させるかを選択できます。必要数に応じてNICを追加してください。

## KVM 設定
### HugePage 設定
- PageSizeは1G必須
- Paze数は環境に合わせて変更
```
mkdir /dev/hugepages1G
mount -t hugetlbfs -o pagesize=1G none /dev/hugepages1G
echo 16 > /sys/devices/system/node/node1/hugepages/hugepages-1048576kB/nr_hugepages
echo 16 > /sys/devices/system/node/node0/hugepages/hugepages-1048576kB/nr_hugepages

mount -t hugetlbfs -o pagesize=1G none /dev/hugepages1G
```

/etc/fstab
```
none /dev/hugepages1G hugetlbfs pagesize=1G,size=32G 0 0
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
virt-install --name jcnr1 --vcpus=8 --ram=32768 --location=/home/images/rhel-8.6-x86_64-dvd.iso --disk path=/home/images/jcnr1.qcow2,format=qcow2,size=100 --network bridge=br-mng,model=virtio --network bridge=br01,model=virtio --network bridge=br02,model=virtio --virt-type kvm --os-variant rhel7 --serial pty --console pty,target_type=virtio --graphics vnc,listen=0.0.0.0 --noautoconsole
```
#### JCNR VMのHuagePage設定及びvCPU Pinning
virth edit jcnr1
```
  <cpu mode='custom' match='exact' check='partial'>
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
sed -i 's%/dev/mapper/rhel-swap%#/dev/mapper/rhel-swap%' /etc/fstab
```
#### Firewall
```
firewall-cmd --set-default-zone=trusted
firewall-cmd --reload
```
#### Hugepage
/etc/default/grub
```
GRUB_CMDLINE_LINUX="crashkernel=auto rd.lvm.lv=rhel/root rhgb quiet default_hugepagesz=1G hugepagesz=1G hugepages=8 intel_iommu=on iommu=pt"
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
echo 'vfio-pci' > /etc/modules-load.d/vfio-pci.conf
modprobe vfio-pci

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
modprobe vfio enable_unsafe_noiommu_mode=1
modprobe vfio-pci
```
```
chmod +x /etc/sysconfig/modules/vfio-noiommu.modules
echo 1 > /sys/module/vfio/parameters/enable_unsafe_noiommu_mode
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

## JCNR VM Kubernetesインストール
### 初期設定
```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```
```
modprobe overlay
modprobe br_netfilter
```
```
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
```
```
sysctl --system
```

### CRI Dockerインストール
```
yum install -y yum-utils
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum install -y docker-ce-20.10.11 docker-ce-cli-20.10.11 containerd.io
```
```
mkdir /etc/docker

cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF

systemctl daemon-reload
systemctl enable --now docker
```

### Kubeletインストール
```
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
```
```
yum install -y kubelet-1.22.4 kubeadm-1.22.4 kubectl-1.22.4 --disableexcludes=kubernetes
```
```
systemctl enable --now kubelet
systemctl restart kubelet
```

### Kubernetesデプロイ
```
kubeadm init --pod-network-cidr=172.30.0.0/16 --service-cidr=172.31.0.0/16
```
```
mkdir -p /root/.kube
cp -i /etc/kubernetes/admin.conf /root/.kube/config
chown $(id -u):$(id -g) /root/.kube/config
export KUBECONFIG=/etc/kubernetes/admin.conf
```
```
kubectl taint nodes --all node-role.kubernetes.io/master-
```

### CNI Calicoインストール
```
curl -OL -k https://projectcalico.docs.tigera.io/archive/v3.22/manifests/tigera-operator.yaml
```
```
kubectl create -f tigera-operator.yaml
```
```
curl -OL -k https://projectcalico.docs.tigera.io/archive/v3.22/manifests/custom-resources.yaml
```
custom-resources.yaml
```
    ipPools:
    - blockSize: 26
      cidr: 172.30.0.0/16
```
```
kubectl create -f custom-resources.yaml
```

### Multusインストール
```
yum -y install git
git clone https://github.com/k8snetworkplumbingwg/multus-cni.git
```
```
cat ./multus-cni/deployments/multus-daemonset-thick.yml | kubectl apply -f -
```

すべてのPodがRunningになっていることを確認
```
kubectl get pods -A
```

## JCNR インストール
### Helm Install
```
cd /root/
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```
### JCNR Imageの取得
- Software Download SiteからDownloadまたはJuniper社へお問い合わせください
```
tar xzvf Juniper_Cloud_Native_Router_23.2.tgz
cd Juniper_Cloud_Native_Router_23.2
docker load -i images/jcnr-images.tar.gz
```

helmchart/values.yaml
- "interface_mode"の設定がないInterfaceはL3 Interface, 設定があるInterfaceはL2 Interfaceとなる
```
global:
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
  cpu_core_mask: "0,1,2,3"
```

### JCNR License
- Trial Licenseの発行はJuniper社へお問い合わせください
- JCNR License(S-CRPD-x-x-PF)を取得
```
base64 -w 0 license.txt
```
```
echo 'ROOT_PASSWORD' | base64
```
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
```
kubectl apply -f secrets/jcnr-secrets.yaml
```

### JCNR Node Annotation設定
- Node AnnotationはJCNR Node毎に異なるパラメータを定義し、JCNRデプロイ時にConfigurationに反映
helmchart/cRPD_examples/node-annotation.yaml
```
---
apiVersion: v1
kind: Node
metadata:
  name: jcnr1
  annotations:
    jcnr.juniper.net/params: '{
        "isoLoopbackAddr": "49.0004.1000.0000.0001.00",
        "IPv4LoopbackAddr": "1.1.1.1",
        "srIPv4NodeIndex": "2001",
        "srIPv6NodeIndex": "3001",
        "BGPIPv4Neighbor": "1.1.1.2",
        "BGPLocalAsn": "64512"
    }'
```
```
kubectl apply -f helmchart/cRPD_examples/node-annotation.yaml
```

### JCNR Custom Config File
- Node Annotationで指定したパラメータを元に、JCNRデプロイ時に設定する初期Configuration
- CalicoがBGP 179 Portを使用しているため、JCNRで使用するBGP Portは178に設定
helmchart/charts/jcnr-cni/files/jcnr-cni-custom-config.tmpl
```
apply-groups [custom];
system {
    processes {
        routing {
            bgp {
                tcp-listen-port 178;
            }
        }
    }
}
groups {
    custom {
        interfaces {
            lo0 {
                unit 0 {
                {{if .Params.isoLoopbackAddr}}
                    family iso {
                        address {{.Params.isoLoopbackAddr}};
                    }
                {{end}}
                    family inet {
                        address {{.Params.IPv4LoopbackAddr}};
                    }
                }
            }
        }
        routing-options {
            router-id {{.Params.IPv4LoopbackAddr}}
            route-distinguisher-id {{.Params.IPv4LoopbackAddr}}
        }
        protocols {
            isis {
                interface all;
                interface ens3 {
                    disable;
                }
                {{if and .Env.SRGB_START_LABEL .Env.SRGB_INDEX_RANGE}}
                source-packet-routing {
                    srgb start-label {{.Env.SRGB_START_LABEL}} index-range {{.Env.SRGB_INDEX_RANGE}};
                    node-segment {
                        {{if .Params.srIPv4NodeIndex}}
                        ipv4-index {{.Params.srIPv4NodeIndex}};
                        {{end}}
                        {{if .Params.srIPv6NodeIndex}}
                        ipv6-index {{.Params.srIPv6NodeIndex}};
                        {{end}}
                    }
                }
                {{end}}
                level 1 disable;
            }
            ldp {
                interface all;
                interface ens3 {
                    disable;
                }
            }
            mpls {
                interface all;
                interface ens3 {
                    disable;
                }
            }
        }
        policy-options {
            # policy to signal dynamic UDP tunnel attributes to BGP routes
            policy-statement udp-export {
                then community add udp;
            }
            community udp members encapsulation:0L:13;
        }
        protocols {
            bgp {
                group jcnrbgp1 {
                    type internal;
                    local-address {{.Params.IPv4LoopbackAddr}};
                    local-as {{.Params.BGPLocalAsn}};
                    neighbor {{.Params.BGPIPv4Neighbor}};
                    export udp-export;
                    tcp-connect-port 178;
                    family inet-vpn {
                        unicast;
                    }
                    family inet6-vpn {
                        unicast;
                    }
                }
            }
        }
        routing-options {
                dynamic-tunnels {
                dyn-tunnels {
                    source-address {{.Params.IPv4LoopbackAddr}};
                    udp;
                    destination-networks 0.0.0.0/0;
                }
            }
        }
    }
}
```

### JCNR デプロイ
```
helm install jcnr .
```
全てのPodがRunniningになっていることを確認
```
[root@jcnr1 ]# kubectl get pods -A
NAMESPACE          NAME                                       READY   STATUS    RESTARTS      AGE
calico-apiserver   calico-apiserver-f894c7976-mqr45           1/1     Running   1 (26h ago)   26h
calico-apiserver   calico-apiserver-f894c7976-xglq7           1/1     Running   1 (26h ago)   26h
calico-system      calico-kube-controllers-7c594948b5-nqb97   1/1     Running   1 (26h ago)   26h
calico-system      calico-node-6vnhl                          1/1     Running   1 (26h ago)   26h
calico-system      calico-typha-c8c57669b-gbsnv               1/1     Running   2 (26h ago)   26h
contrail-deploy    contrail-k8s-deployer-74f787b6c6-zcmmq     1/1     Running   0             7h43m
contrail           contrail-vrouter-masters-whtqr             3/3     Running   0             7h42m
jcnr               kube-crpd-worker-sts-0                     1/1     Running   0             7h43m
jcnr               syslog-ng-b2c86                            1/1     Running   0             7h43m
kube-system        coredns-78fcd69978-6qwbp                   1/1     Running   1 (26h ago)   26h
kube-system        coredns-78fcd69978-mctk8                   1/1     Running   1 (26h ago)   26h
kube-system        etcd-jcnr1                                 1/1     Running   1 (26h ago)   26h
kube-system        kube-apiserver-jcnr1                       1/1     Running   1 (26h ago)   26h
kube-system        kube-controller-manager-jcnr1              1/1     Running   4 (26h ago)   26h
kube-system        kube-multus-ds-vvgh4                       1/1     Running   1 (26h ago)   26h
kube-system        kube-proxy-95wj6                           1/1     Running   1 (26h ago)   26h
kube-system        kube-scheduler-jcnr1                       1/1     Running   4 (26h ago)   26h
tigera-operator    tigera-operator-866558f9c-slw8m            1/1     Running   5 (26h ago)   26h
```

### JCNR Login方法
cRPD Login
```
kubectl exec -n jcnr -it kube-crpd-worker-sts-XXXX -- bash
cli
```

vRouter Login
```
kubectl exec -n contrail -it contrail-vrouter-masters-XXXX -- bash
```

### JCNR 初期Config
```
root@jcnr1> show configuration | display set
set version 20230905.075640_builder.r1366729
set groups base apply-flags omit
set groups base apply-macro ht jcnr
set groups base system root-authentication encrypted-password "$6$J96HssqtrBSoJhF7$9Bb884hJQI1VkkU77pyv6.ZPZnwzjenjMT6cOZaQ.awvdFp31dGgZFo79fN5wk4Iy0zPtrsrGqrRYxTYblCqh/"
set groups base system commit xpath
set groups base system commit constraints direct-access
set groups base system commit notification configuration-diff-format xml
set groups base system scripts action max-datasize 256m
set groups base system scripts language python3
set groups base system services ssh root-login allow
set groups base system services ssh port 24
set groups base system services extension-service request-response grpc clear-text address 127.0.0.1
set groups base system services extension-service request-response grpc clear-text port 50051
set groups base system services extension-service request-response grpc skip-authentication
set groups base system services netconf ssh
set groups base system syslog host 127.0.0.1 any any
set groups base system syslog host 127.0.0.1 port 50055
set groups base system syslog host 127.0.0.1 transport udp
set groups base system syslog host 127.0.0.1 structured-data
set groups base system syslog file test any any
set groups base system syslog source-address 127.0.0.1
set groups base system license keys key "JUNOS892191212 aeaqie alakap hs6c2a eqaaad 5adsqo zm7quo 3hqrmy hg74fs lzcbra wvagfc zxvq7a nwwdy2 63hajk u26llp mbk4wv sidbdz"
set groups base interfaces irb unit 0 mac 48:5a:0d:44:2e:96
set groups base routing-options resolution rib :gribi.inet6.0 inet6-resolution-ribs :gribi.inet6.0
set groups base routing-options forwarding-table channel vrouter protocol protocol-type gRPC
set groups base routing-options forwarding-table channel vrouter protocol destination 127.0.0.1:50052
set groups custom interfaces lo0 unit 0 family inet address 1.1.1.1/32
set groups custom interfaces lo0 unit 0 family iso address 49.0004.1000.0000.0001.00
set groups custom policy-options policy-statement udp-export then community add udp
set groups custom policy-options community udp members encapsulation:0L:13
set groups custom routing-options route-distinguisher-id 1.1.1.1
set groups custom routing-options router-id 1.1.1.1
set groups custom routing-options dynamic-tunnels dyn-tunnels source-address 1.1.1.1
set groups custom routing-options dynamic-tunnels dyn-tunnels udp
set groups custom routing-options dynamic-tunnels dyn-tunnels destination-networks 0.0.0.0/0
set groups custom protocols bgp group jcnrbgp1 type internal
set groups custom protocols bgp group jcnrbgp1 local-address 1.1.1.1
set groups custom protocols bgp group jcnrbgp1 family inet-vpn unicast
set groups custom protocols bgp group jcnrbgp1 family inet6-vpn unicast
set groups custom protocols bgp group jcnrbgp1 export udp-export
set groups custom protocols bgp group jcnrbgp1 local-as 64512
set groups custom protocols bgp group jcnrbgp1 neighbor 1.1.1.2
set groups custom protocols bgp group jcnrbgp1 tcp-connect-port 178
set groups custom protocols isis interface all
set groups custom protocols isis interface ens3 disable
set groups custom protocols isis source-packet-routing srgb start-label 400000
set groups custom protocols isis source-packet-routing srgb index-range 4000
set groups custom protocols isis source-packet-routing node-segment ipv4-index 2001
set groups custom protocols isis source-packet-routing node-segment ipv6-index 3001
set groups custom protocols isis level 1 disable
set groups custom protocols ldp interface all
set groups custom protocols ldp interface ens3 disable
set groups custom protocols mpls interface all
set groups custom protocols mpls interface ens3 disable
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
set groups cni
set apply-groups custom
set apply-groups base
set apply-groups internal
set apply-groups cni
set system processes routing bgp tcp-listen-port 178
set interfaces ens5 native-vlan-id 100
set interfaces ens5 unit 0 family bridge interface-mode trunk
set interfaces ens5 unit 0 family bridge vlan-id-list 100
set interfaces ens5 unit 0 family bridge vlan-id-list 200
set interfaces ens5 unit 0 family bridge vlan-id-list 300
set interfaces ens5 unit 0 family bridge vlan-id-list 700-705
set interfaces ens5 unit 0 family bridge ce-facing
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
