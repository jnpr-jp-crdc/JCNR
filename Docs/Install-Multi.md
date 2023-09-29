# JCNR Install (Multi Worker Node Kubernetes Cluster)手順
- 本手順はKVM上にRHEL VMを3台デプロイし、Multi Worker Node Kubernetes Clusterを構築し、JCNRをインストールする手順となります。
- 本手順はJCNR 23.2をベースとしたインストール手順となります。
- JCNR Master NodeがBGP RRとなる

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
- Master NIC: ens3：１７２．２７．１１５．２０６/22(Management), ens4：192.168.0.1/24(L3 Interface), ens5:none(L2 Interface)
- Worker1 NIC: ens3：１７２．２７．１１５．２０7/22(Management), ens4：192.168.0.2/24(L3 Interface), ens5:none(L2 Interface)
- Worker2 NIC: ens3：１７２．２７．１１５．２０8/22(Management), ens4：192.168.0.3/24(L3 Interface), ens5:none(L2 Interface)

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
virt-install --name jcnr-master --vcpus=8 --ram=32768 --location=/home/images/rhel-8.6-x86_64-dvd.iso --disk path=/home/images/jcnr-master.qcow2,format=qcow2,size=100 --network bridge=br-mng,model=virtio --network bridge=br01,model=virtio --network bridge=br02,model=virtio --virt-type kvm --os-variant rhel7 --serial pty --console pty,target_type=virtio --graphics vnc,listen=0.0.0.0 --noautoconsole

virt-install --name jcnr-worker1 --vcpus=8 --ram=32768 --location=/home/images/rhel-8.6-x86_64-dvd.iso --disk path=/home/images/jcnr-worker1.qcow2,format=qcow2,size=100 --network bridge=br-mng,model=virtio --network bridge=br01,model=virtio --network bridge=br02,model=virtio --virt-type kvm --os-variant rhel7 --serial pty --console pty,target_type=virtio --graphics vnc,listen=0.0.0.0 --noautoconsole

virt-install --name jcnr-worker2 --vcpus=8 --ram=32768 --location=/home/images/rhel-8.6-x86_64-dvd.iso --disk path=/home/images/jcnr-worker2.qcow2,format=qcow2,size=100 --network bridge=br-mng,model=virtio --network bridge=br01,model=virtio --network bridge=br02,model=virtio --virt-type kvm --os-variant rhel7 --serial pty --console pty,target_type=virtio --graphics vnc,listen=0.0.0.0 --noautoconsole

```
#### JCNR VMのHuagePage設定及びvCPU Pinning
virth edit jcnr-XXX
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

### Kubletインストール
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

### Kubernetesデプロイ (Master Node Only)
```
kubeadm init --pod-network-cidr=172.30.0.0/16 --service-cidr=172.31.0.0/16
```
```
mkdir -p /root/.kube
cp -i /etc/kubernetes/admin.conf /root/.kube/config
chown $(id -u):$(id -g) /root/.kube/config
export KUBECONFIG=/etc/kubernetes/admin.conf
```

### CNI Calicoインストール (Master Node Only)
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

### Multusインストール (Master Node Only)
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

## JCNR インストール (Master Node Only)
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
  - eth4:
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
  name: jcnr-master
  annotations:
    jcnr.juniper.net/params: '{
        "isoLoopbackAddr": "49.0004.1000.0000.0001.00",
        "ClusterID": "1.1.1.1",
        "IPv4LoopbackAddr": "1.1.1.1",
        "srIPv4NodeIndex": "2001",
        "srIPv6NodeIndex": "3001",
        "BGPIPv4Neighbor": "1.1.1.2",
        "BGPLocalAsn": "64512"
    }'
---
apiVersion: v1
kind: Node
metadata:
  name: jcnr-worker1
  annotations:
    jcnr.juniper.net/params: '{
        "isoLoopbackAddr": "49.0004.1000.0000.0002.00",
        "IPv4LoopbackAddr": "1.1.1.2",
        "srIPv4NodeIndex": "2002",
        "srIPv6NodeIndex": "3002",
        "BGPIPv4Neighbor": "1.1.1.1",
        "BGPLocalAsn": "64512"
    }'
---
apiVersion: v1
kind: Node
metadata:
  name: jcnr-worker2
  annotations:
    jcnr.juniper.net/params: '{
        "isoLoopbackAddr": "49.0004.1000.0000.0003.00",
        "IPv4LoopbackAddr": "1.1.1.3",
        "srIPv4NodeIndex": "2003",
        "srIPv6NodeIndex": "3003",
        "BGPIPv4Neighbor": "1.1.1.1",
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
    		        {{if .Node.isoLoopbackAddr}}
                        family iso {
                            address {{.Node.isoLoopbackAddr}};
                        }
    		        {{end}}
                        family inet {
                            address {{.Node.IPv4LoopbackAddr}};
                        }
                    }
                }
            }
            routing-options {
               router-id {{.Node.IPv4LoopbackAddr}}
               route-distinguisher-id {{.Node.IPv4LoopbackAddr}}
            }
            protocols {
                isis {
                   interface all;
                   {{if and .Env.SRGB_START_LABEL .Env.SRGB_INDEX_RANGE}}
                   source-packet-routing {
                       srgb start-label {{.Env.SRGB_START_LABEL}} index-range {{.Env.SRGB_INDEX_RANGE}};
                       node-segment {
                           {{if .Node.srIPv4NodeIndex}}
                           ipv4-index {{.Node.srIPv4NodeIndex}};
                           {{end}}
                           {{if .Node.srIPv6NodeIndex}}
                           ipv6-index {{.Node.srIPv6NodeIndex}};
                           {{end}}
                       }
                   }
                   {{end}}
                   level 1 disable;
                }
                ldp {
                    interface all;
                }
                mpls {
                    interface all;
                }
                bgp {
                    interface all;
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
                        {{if .Node.ClusterID}}
                        cluster {{.Node.ClusterID}}
                        {{end}}
                        local-address {{.Node.IPv4LoopbackAddr}};
                        local-as {{.Node.BGPLocalAsn}};
                        neighbor {{.Node.BGPIPv4Neighbor}};
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
                        source-address {{.Node.IPv4LoopbackAddr}};
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
[root@jcnr-master ~]# kubectl get pods -A
NAMESPACE          NAME                                       READY   STATUS    RESTARTS        AGE
calico-apiserver   calico-apiserver-bd5c44b7b-pkwgg           1/1     Running   12              69d
calico-apiserver   calico-apiserver-bd5c44b7b-rsbtb           1/1     Running   12 (56d ago)    69d
calico-system      calico-kube-controllers-7c594948b5-wnnpk   1/1     Running   12 (56d ago)    69d
calico-system      calico-node-fqpbn                          1/1     Running   12 (2d8h ago)   69d
calico-system      calico-node-vhgc4                          1/1     Running   14 (2d8h ago)   69d
calico-system      calico-node-vnrh2                          1/1     Running   15 (2d8h ago)   69d
calico-system      calico-typha-85565bbd56-wx99m              1/1     Running   11 (56d ago)    69d
calico-system      calico-typha-85565bbd56-xz275              1/1     Running   17 (56d ago)    69d
contrail-deploy    contrail-k8s-deployer-74f787b6c6-qk6nn     1/1     Running   0               30h
contrail           contrail-vrouter-masters-vvjcx             3/3     Running   0               30h
contrail           contrail-vrouter-nodes-97f97               3/3     Running   0               30h
contrail           contrail-vrouter-nodes-s5sgq               3/3     Running   0               30h
jcnr               kube-crpd-worker-sts-0                     1/1     Running   0               30h
jcnr               kube-crpd-worker-sts-1                     1/1     Running   0               30h
jcnr               kube-crpd-worker-sts-2                     1/1     Running   0               30h
jcnr               syslog-ng-5nd8m                            1/1     Running   0               30h
jcnr               syslog-ng-8pdpw                            1/1     Running   0               30h
kube-system        coredns-78fcd69978-gsmr5                   1/1     Running   10 (56d ago)    69d
kube-system        coredns-78fcd69978-rpmjt                   1/1     Running   10 (56d ago)    69d
kube-system        etcd-jcnr-master                           1/1     Running   11 (56d ago)    69d
kube-system        kube-apiserver-jcnr-master                 1/1     Running   13 (56d ago)    69d
kube-system        kube-controller-manager-jcnr-master        1/1     Running   17 (2d8h ago)   69d
kube-system        kube-multus-ds-2jpv2                       1/1     Running   10 (56d ago)    69d
kube-system        kube-multus-ds-j9m2n                       1/1     Running   10 (56d ago)    69d
kube-system        kube-multus-ds-mpkcj                       1/1     Running   10 (56d ago)    69d
kube-system        kube-proxy-4mh62                           1/1     Running   10 (56d ago)    69d
kube-system        kube-proxy-llgr6                           1/1     Running   11 (56d ago)    69d
kube-system        kube-proxy-twk6l                           1/1     Running   10              69d
kube-system        kube-scheduler-jcnr-master                 1/1     Running   17 (2d8h ago)   69d
tigera-operator    tigera-operator-866558f9c-8ksht            1/1     Running   27 (2d8h ago)   69d
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
