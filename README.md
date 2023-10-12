# cerberus
![cerberus](assets/cerberus.png)
## Ansible to deploy Kubernetes onto Fedora
[Fedora 38: Kubernetes with Kubeadm](https://www.server-world.info/en/note?os=Fedora_38&p=kubernetes&f=1) [^1]
-------------------------------------------------------------------------------------------------------------------------------------------
1. [Install Kubeadm](#install-kubeadm)
2. [Configure Control Plane](#configure-control-plane)
### Install Kubeadm
----------------------
For system requirements; each node must have a unique hostname, MAC address, and product_uuid.\
MAC address and product_uuid are generally already unique if you installed OS on physical machine or virtual machine with common procedures.\
You can see product_uuid with the command:\
`dmidecode -s system-uuid`\
Furthermore, based on this environment; firewalld is disabled. [^2]\
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
----------------------------------------------------------------------------------------------------------------- 
-----------------------------------------------------------------------------------------------------------------
### Configure Control Plane
This example is based on the environment like follows.
```
-----------+---------------------------+--------------------------+------------
           |                           |                          |
       eth0|10.0.0.30              eth0|10.0.0.51             eth0|10.0.0.52
+----------+-----------+   +-----------+----------+   +-----------+----------+
|   [ dlp.srv.world ]  |   | [ node01.srv.world ] |   | [ node02.srv.world ] |
|     Control Plane    |   |      Worker Node     |   |      Worker Node     |
+----------------------+   +----------------------+   +----------------------+
```
[1] 	
Configure pre-requirements on all Nodes, refer to here.

Configure initial setup on Control Plane Node.\
For [control-plane-endpoint], specify the Hostname or IP address that Etcd and Kubernetes API server are run.\
For [--pod-network-cidr] option, specify network for pods.\
There are some plugins for Pod Network. (Refer to details in link below.)\
[Cluster Networking](https://kubernetes.io/docs/concepts/cluster-administration/networking/)\
On this example, it selects Calico.
##### Initialize Kubernetes with arguments.
```
kubeadm init --control-plane-endpoint=10.0.0.30 --pod-network-cidr=192.168.0.0/16 --cri-socket=unix:///var/run/crio/crio.sock
```
```
[init] Using Kubernetes version: v1.26.4
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [dlp.srv.world kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 10.0.0.30]

.....
.....

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of control-plane nodes by copying certificate authorities
and service account keys on each node and then running the following as root:

  kubeadm join 10.0.0.30:6443 --token 3wx2wn.0dkhtd9i26gsrc3t \
        --discovery-token-ca-cert-hash sha256:b430cc64408d3349fc368c1d611638a0b71bd10c1ef47254afe414ecfaa200ca \
        --control-plane
```
#### Set cluster admin. (If you set common user as cluster admin, login with it and run: (sudo cp/chown ***))
##### Make kube directory.
```
mkdir -p $HOME/.kube
```
##### Copy over kube admin conf.
```
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
```
##### Change ownership of copy.
```
chown $(id -u):$(id -g) $HOME/.kube/config
```
#### Configure Pod Network with Calico.
##### Download the calico conf from Github. (Included as Jinja template instead.)
```
wget https://raw.githubusercontent.com/projectcalico/calico/master/manifests/calico.yaml
```
##### Apply calico conf to deploy CNI.
```
kubectl apply -f calico.yaml
```
```
poddisruptionbudget.policy/calico-kube-controllers created
serviceaccount/calico-kube-controllers created
serviceaccount/calico-node created
serviceaccount/calico-cni-plugin created
configmap/calico-config created
customresourcedefinition.apiextensions.k8s.io/bgpconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/bgpfilters.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/bgppeers.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/blockaffinities.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/caliconodestatuses.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/clusterinformations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/felixconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworksets.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/hostendpoints.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamblocks.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamconfigs.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamhandles.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ippools.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipreservations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/kubecontrollersconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networksets.crd.projectcalico.org created
clusterrole.rbac.authorization.k8s.io/calico-kube-controllers created
clusterrole.rbac.authorization.k8s.io/calico-node created
clusterrole.rbac.authorization.k8s.io/calico-cni-plugin created
clusterrolebinding.rbac.authorization.k8s.io/calico-kube-controllers created
clusterrolebinding.rbac.authorization.k8s.io/calico-node created
clusterrolebinding.rbac.authorization.k8s.io/calico-cni-plugin created
daemonset.apps/calico-node created
deployment.apps/calico-kube-controllers created
```
##### Show kube state of nodes : OK if STATUS = Ready.
```
kubectl get nodes
```
```
NAME            STATUS   ROLES           AGE     VERSION
dlp.srv.world   Ready    control-plane   3m32s   v1.26.4
```
##### Show kube state of pods : OK if all are Running.
```
kubectl get pods -A
```
```
NAMESPACE     NAME                                      READY   STATUS    RESTARTS   AGE
kube-system   calico-kube-controllers-56dd5794f-6p9w9   1/1     Running   0          45s
kube-system   calico-node-bwg6l                         1/1     Running   0          45s
kube-system   coredns-787d4945fb-dnk67                  1/1     Running   0          3m34s
kube-system   coredns-787d4945fb-wzpvs                  1/1     Running   0          3m34s
kube-system   etcd-dlp.srv.world                        1/1     Running   0          3m50s
kube-system   kube-apiserver-dlp.srv.world              1/1     Running   0          3m50s
kube-system   kube-controller-manager-dlp.srv.world     1/1     Running   0          3m50s
kube-system   kube-proxy-9dx9j                          1/1     Running   0          3m35s
kube-system   kube-scheduler-dlp.srv.world              1/1     Running   0          3m44s
```
------------------------------------------------------------------------------------------------------------------------------------------------
[^1]: This process flow is based off the linked blog tutorial but ansible-lized.
[^2]: The ansible will disable firewalld, you do not need to manually.
