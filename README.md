# cerberus
![cerberus](assets/cerberus.png)
## Ansible to deploy Kubernetes onto Fedora
-----------------------------------------------------------------
### This project is essentially this blog's process flow but ansible-ized: 
[Fedora 38: Kubernetes with Kubeadm](https://www.server-world.info/en/note?os=Fedora_38&p=kubernetes&f=1)
-------------------------------------------------------------------------------------------------------------------------------------------  	
For system requirements; each node must have a unique hostname, MAC address, and product_uuid.\
MAC address and product_uuid are generally already unique if you installed OS on physical machine or virtual machine with common procedures.\
You can see product_uuid with the command:\
`dmidecode -s system-uuid`\
Furthermore, based on this environment; firewalld is disabled. [^1]\
You will be root for all of this unless stated. Assume `root@hostname ~#` for all commands.\
This example is based on the environment that follows:
```
-----------+---------------------------+--------------------------+------------
           |                           |                          |            
       eth0|10.0.0.30              eth0|10.0.0.51             eth0|10.0.0.52   
+----------+-----------+   +-----------+----------+   +-----------+----------+ 
|   [ dlp.srv.world ]  |   | [ node01.srv.world ] |   | [ node02.srv.world ] | 
|     Control Plane    |   |      Worker Node     |   |      Worker Node     | 
+----------------------+   +----------------------+   +----------------------+`
```

#### On all nodes, change settings for system requirements.
-----------------------------------------------------------------
##### Create a conf for sysctl to enable bridging and forwarding.
```
cat > /etc/sysctl.d/99-k8s-cri.conf <<EOF
net.bridge.bridge-nf-call-iptables=1
net.ipv4.ip_forward=1
net.bridge.bridge-nf-call-ip6tables=1
EOF
```
##### Add overlay and br_netfilter as modules for etcd to load.
```
echo -e overlay\\nbr_netfilter > /etc/modules-load.d/k8s.conf
```
##### Install iptables-legacy with dnf.
```
dnf -y install iptables-legacy
```
##### Switch default for iptables to iptables-legacy.
```
alternatives --config iptables
```
```
There are 2 programs which provide 'iptables':
  Selection    Command
-----------------------------------------------
 + 1           /usr/sbin/iptables-nft
   2           /usr/sbin/iptables-legacy

Enter to keep the current selection[+], or type selection number: 2
```
##### Set swap off setting.
```
touch /etc/systemd/zram-generator.conf
```
##### Disable firewalld.
```
systemctl disable --now firewalld
```
##### Disable systemd-resolved.
```
systemctl disable --now systemd-resolved
```
##### Edit NetworkManager conf to set dns to default.
```
vi /etc/NetworkManager/NetworkManager.conf
```
```
[main]
dns=default
```
##### Stop symbolic link on resolv conf.
```
unlink /etc/resolv.conf
```
##### Create empty conf to stop resolv from working still.
```
touch /etc/resolv.conf
```

##### Restart to apply changes.
```
reboot
```
#### On all nodes, install required packages.
-----------------------------------------------------------------
We're gonna use CRI-O for our cri.\
##### Installs a modularity appstream for CRI-O with a default profile.
```
dnf module -y install cri-o:1.25/default
```
##### Enables CRI-O.
```
systemctl enable --now crio
```
##### Install required packages.
```
dnf -y install kubernetes-kubeadm kubernetes-node kubernetes-client cri-tools iproute-tc container-selinux
```
##### Edit kubelet conf.
```
vi /etc/kubernetes/kubelet
```
```
# line 5 : change
KUBELET_ADDRESS="--address=0.0.0.0"

# line 8 : uncomment
KUBELET_PORT="--port=10250"

# line 11 : change to your hostname
KUBELET_HOSTNAME="--hostname-override=dlp.srv.world"
```
##### Edit kubeadm conf.
```
vi /etc/systemd/system/kubelet.service.d/kubeadm.conf
```
```
# line 6 : add
Environment="KUBELET_EXTRA_ARGS=--cgroup-driver=systemd --container-runtime=remote --container-runtime-endpoint=unix:///var/run/crio/crio.sock"
```
##### Enable kubelet (This one does not have the --now flag!)
```
systemctl enable kubelet
```
#### On all nodes, if SELinux is enabled, change policy.
-----------------------------------------------------------------
##### Create a file to configure some SELinux policy.
```
vi k8s.te
```
```
module k8s 1.0;

require {
        type cgroup_t;
        type iptables_t;
        class dir ioctl;
}

#============= iptables_t ==============
allow iptables_t cgroup_t:dir ioctl;
```
##### Check for errors in created policy changes.
```
checkmodule -m -M -o k8s.mod k8s.te
```
##### Create a SELinux package out of the checked module.
```
semodule_package --outfile k8s.pp --module k8s.mod
```
##### Integrate the new SELinux policy.
```
semodule -i k8s.pp
```
======================================================================================================================================================================================== 
[^1]: The ansible will disable firewalld, you do not need to manually.
