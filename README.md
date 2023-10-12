# cerberus
## Ansible to deploy Kubernetes onto Fedora
-----------------------------------------------------------------
### This project is essentially this blog's process flow but ansible-ized: 
[Fedora 38: Kubernetes with Kubeadm](https://www.server-world.info/en/note?os=Fedora_38&p=kubernetes&f=1)
-------------------------------------------------------------------------------------------------------------------------------------------  	
For system requirements, each node has unique hostname, MAC address, and product_uuid.
MAC address and product_uuid are generally already unique if you installed OS on physical machine or virtual machine with common procedures.
You can see product_uuid with the command:
`dmidecode -s system-uuid`
Furthermore, based on this environment; firewalld is disabled.
This example is based on the environment like follows.
```
-----------+---------------------------+--------------------------+------------
           |                           |                          |            
       eth0|10.0.0.30              eth0|10.0.0.51             eth0|10.0.0.52   
+----------+-----------+   +-----------+----------+   +-----------+----------+ 
|   [ dlp.srv.world ]  |   | [ node01.srv.world ] |   | [ node02.srv.world ] | 
|     Control Plane    |   |      Worker Node     |   |      Worker Node     | 
+----------------------+   +----------------------+   +----------------------+`
```

[1] 	On all Nodes, Change settings for System requirements.
[root@dlp ~]# cat > /etc/sysctl.d/99-k8s-cri.conf <<EOF
net.bridge.bridge-nf-call-iptables=1
net.ipv4.ip_forward=1
net.bridge.bridge-nf-call-ip6tables=1
EOF
[root@dlp ~]# echo -e overlay\\nbr_netfilter > /etc/modules-load.d/k8s.conf
[root@dlp ~]# dnf -y install iptables-legacy

[root@dlp ~]# alternatives --config iptables


There are 2 programs which provide 'iptables'.

  Selection    Command
-----------------------------------------------
*+ 1           /usr/sbin/iptables-nft
   2           /usr/sbin/iptables-legacy

# switch to [iptables-legacy]
Enter to keep the current selection[+], or type selection number: 2

# set Swap off setting

[root@dlp ~]# touch /etc/systemd/zram-generator.conf
# disable [firewalld]

[root@dlp ~]# systemctl disable --now firewalld
# disable [systemd-resolved] (enabled by default)

[root@dlp ~]# systemctl disable --now systemd-resolved
[root@dlp ~]# vi /etc/NetworkManager/NetworkManager.conf
# add into [main] section

[main]
dns=default
[root@dlp ~]# unlink /etc/resolv.conf

[root@dlp ~]# touch /etc/resolv.conf
# restart to apply changes

[root@dlp ~]# reboot

[2] 	On all Nodes, Install required packages.
This example shows to use CRI-O for container runtime.
[root@dlp ~]# dnf module -y install cri-o:1.25/default
[root@dlp ~]# systemctl enable --now crio
[root@dlp ~]# dnf -y install kubernetes-kubeadm kubernetes-node kubernetes-client cri-tools iproute-tc container-selinux
[root@dlp ~]# vi /etc/kubernetes/kubelet
# line 5 : change

KUBELET_ADDRESS="--address=0.0.0.0
"
# line 8 : uncomment

KUBELET_PORT="--port=10250"
# line 11 : change to your hostname

KUBELET_HOSTNAME="--hostname-override=dlp.srv.world
"
[root@dlp ~]# vi /etc/systemd/system/kubelet.service.d/kubeadm.conf
# line 6 : add

Environment="KUBELET_EXTRA_ARGS=--cgroup-driver=systemd --container-runtime=remote --container-runtime-endpoint=unix:///var/run/crio/crio.sock
"
[root@dlp ~]# systemctl enable kubelet
[3] 	On all Nodes, if SELinux is enabled, change policy.
[root@dlp ~]# vi k8s.te
# create new

module k8s 1.0;

require {
        type cgroup_t;
        type iptables_t;
        class dir ioctl;
}

#============= iptables_t ==============
allow iptables_t cgroup_t:dir ioctl;

[root@dlp ~]# checkmodule -m -M -o k8s.mod k8s.te

[root@dlp ~]# semodule_package --outfile k8s.pp --module k8s.mod

[root@dlp ~]# semodule -i k8s.pp 
