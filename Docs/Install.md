# JCNR Install Step
- 本手順はKVM上にRHELをデプロイし、Single Node Kubernetes Clusterを構築し、JCNRをインストールするStepとなります。

## KVM 設定
### HugePage 設定
'''
mkdir /dev/hugepages1G
mount -t hugetlbfs -o pagesize=1G none /dev/hugepages1G
echo 32 > /sys/devices/system/node/node1/hugepages/hugepages-1048576kB/nr_hugepages
echo 32 > /sys/devices/system/node/node0/hugepages/hugepages-1048576kB/nr_hugepages

mount -t hugetlbfs -o pagesize=1G none /dev/hugepages1G
'''

/etc/fstab
'''
none /dev/hugepages1G hugetlbfs pagesize=1G,size=32G 0 0
'''

/etc/libvirt/qemu.conf
'''
hugetlbfs_mount = "/dev/hugepages1G"
'''

/etc/default/qemu-kvm
'''
KVM_HUGEPAGES=1
'''

