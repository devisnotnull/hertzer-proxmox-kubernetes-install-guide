# Installing Kubernetes on Proxmox

For this example i shall be using a dedicated server from Hertzner.https://www.hetzner.de/en/. A shout out to hetzner if your looking for cheap and beefy dedicated hosting then these guys are your best bet.

# Setting up the Hertzer server

This guide assumes your server has Debian 8 (Jessie installed)

#### Config when tested

Intel Core i7-920

HDD2x HDD 2,0 TB SATA Enterprise

RAM6x RAM DDR3 8192 MB = 42 GB

## Step one - Install Proxmox

You will begin by creating a new apt source for Proxmox

```sh
vim /etc/apt/sources.list.d/proxmox.list
```

Once you have added the new apt soource you will add the repo key

```sh
wget -O- "http://download.proxmox.com/debian/key.asc" | apt-key add -
```

You will now need to update the system repos

```sh
apt-get update && apt-get dist-upgrade
```

Now you will need to install the Proxmox VE kernel

```sh
pve-firmware pve-kernel-4.4.8-1-pve pve-headers-4.4.8-1-pve
```

Now reboot the machine to load in the new kernel, 

Once the machine is in an up state you can install core the main Proxmox application.

```sh
apt-get install proxmox-ve
```

Once again reboot your machine, When the machine is once again in an upstate you will be able to access the web-ui for proxmox https://{YOUR_IP}:8006/. You will be able to login with your root credentials. 

WHOLLA you have Proxmox installed and running, You you will need to configure all the networking.

### Step two - Networking

Your eth0 interface should have already been pre-configured

```sh
auto  eth0
iface eth0 inet static
  address   PUBLIC_IP
  netmask   255.255.255.192
  gateway   GATEWAY_IP
  # default route to access subnet
  up route add -net NET_IP netmask 255.255.255.192 gw GATEWAY_IP eth0

```

Now we will need to create an internal network for virtual machines to connect and communicate with, This will be the backbone of the entire cluster. We will accomplish this by creating a linux bridge.

```sh
auto vmbr0
iface vmbr1 inet static
    address 10.20.30.1
    netmask 255.255.255.0
    bridge_ports none
    bridge_stp off
    bridge_fd 0
    post-up iptables -t nat -A POSTROUTING -s '10.20.30.0/24' -o eth0 -j MASQUERADE
    post-down iptables -t nat -D POSTROUTING -s '10.20.30.0/24' -o eth0 -j MASQUERADE

```

Now that we have the interfaces configured we need to configure the host server to act as our router, As such we need to make sure the kernel has packet forwarding enabled.

```sh
vim /etc/sysctl.conf
```

Only alter/uncomment these lines
```sh
net.ipv4.ip_forward=1
net.ipv6.conf.all.forwarding=1
```

Lastly you will need to ensure we dont send ICPM redirect messages

```sh
vim /etc/sysctl.conf
```

I need to give a shout out to https://blog.no-panic.at/2016/08/09/proxmox-on-debian-at-hetzner-with-multiple-ip-addresses/ this helped me massivly trying to figure this out

```sh
net.ipv4.conf.all.send_redirects=0
```

Finally reboot the host and your good to go !

### Step three - Creating the vms

Now you have your host networking setup your ready to create your virtual machines, for this setup we will be creating a cluster of 3.

The layout will be,

```sh
node-01 10.20.30.101
node-02 10.20.30.102
node-03 10.20.30.103
```

We will be using ubuntu 16.04 for each node, to start ssh into your main Proxmox box and download the Ubuntu ISO the Proxmox template directory, This is so Proxmox can see the ISO to mount it.

```sh
cd /var/lib/vz/template/iso
wget http://releases.ubuntu.com/16.04.1/ubuntu-16.04.1-server-amd64.iso?_ga=1.150784255.852713614.1480375703
mv ubuntu-16.04.1-server-amd64.iso?_ga=1.150784255.852713614.1480375703 ubuntu-16.04.1-server-amd64.iso
```

You will now be able to select this ISO when you create your VMS.

Now login to your Proxmox web-ui https://{MAIN_IP}:8006/ with your system credentials.

- Input the hostname, e.g node-01
- For OS select linux 4.X/3.X/2.6
- For CD/DVD select from the ISO select menu your downloaded ISO
- For Hard Disk i recommend 200GB per VM, Space permitting.
- For CPU use 1 core, Cpu spec permitting.
- For Memory 4GB, Again memory permitting.
- For Networking select NAT mode
- Then confirm

Now you will also need to add one more network adapter, This adapter will utilize the bridge we created in the previous section.

- Select the new VM from the "Server View"
- Find the Hardware option
- Select "Add" and select "Network Device"
- We need a "Bridged mode" interface, select bridge vmbr0.
- Change the "Model" to Intel E1000. These are issues with the standard virtualised network drivers.
- Add

Now you can turn on your VM.

Install ubuntu as you normally would, By be sure to use the NAT adapter when you install. We will configure the Bridge adapter later.

You will need to repeat this step three times for each VM. Or you can create a template from the first VM you created and clone it three times.


### Step four - Configure VM Network

Once ubuntu is installed you will need to setup the networking for each VM. Connect to node-01 with VNC from the web-ui and login.

Next you will need to configure the adapters, Open up /etc/network/interfaces

```sh
vim /etc/network/interfaces

auto ens18
iface ens18 inet dhcp
```

Your NAT adapter should have already been configured for you, We will now add the Bridged adapter. Add the below to the config.

```sh
vim /etc/network/interfaces

ADD THIS

auto ens19
iface ens19 inet static
        address 10.20.30.101
        netmask 255.255.255.0
        gateway 10.20.30.1
```

Now restart the networking service

```sh
sudo serivce restart networking
```

Try to ping the Hertzer host

```sh
ping 10.20.30.1
```

If you are able to ping the host then !! It worked, Your Virtual machine is connected to the main host via the network bridge with its own adapter.

You can confirm this by connecting to the new VM with an SSH tunnel through the hertzer host

```sh
ADDRESSS = 10.20.30.101 OR 10.20.30.102 or 10.20.30.103
ssh -A -t root@{MAIN_IP} ssh -A -t {VM_USER}@{ADDRESSS}
```

You will need to first input the hertzer hosts password then your new VMS credential. if everything was setup properly then you should be able to SSH into your new VM.

This model uses the NAT adapter to connect to the internet, But you could remove the NAT adapter and just use the private network and use 10.20.30.1 as the network gateway.

You will need to repeat this for each of the VMs assigning each VM a diffrent IP.

```sh
node-01 10.20.30.101
node-02 10.20.30.102
node-03 10.20.30.103
```

You now have a 3 node VM cluster connected via a private network that you can ssh into.


### Step four - Install kubernetes

Now you will need to install Kuberntees on each of your nodes, So for each VM repeat this process.

Add the kubernetes repo to your sources list

```sh
vim /etc/apt/sources.list.d/kubernetes.list

deb http://apt.kubernetes.io/ kubernetes-xenial main
```

Load in the new repo list

```sh
apt-get update
```

Install base packages

```sh
apt-get install -y docker.io kubelet kubeadm kubectl kubernetes-cni
```

Once these packages are installed we can create our base cluster.

We will use node-01 as the master 

```sh
ssh -A -t root@{MAIN_IP} ssh -A -t {VM_USER}@10.20.30.101
```

On the master we can used the installed kubeadm tool

```sh
kubeadm init
```

This will take several minutes to configure, Once the process finished you will be given a list of detailed that you MUST SAVE.

```sh
{VM_USER}@node-01:~# kubeadm init
<master/tokens> generated token: "7fa96f.ddb39492a1874689"
<master/pki> created keys and certificates in "/etc/kubernetes/pki"
<util/kubeconfig> created "/etc/kubernetes/admin.conf"
<util/kubeconfig> created "/etc/kubernetes/kubelet.conf"
<master/apiclient> created API client configuration
<master/apiclient> created API client, waiting for the control plane to become ready
<master/apiclient> all control plane components are healthy after 23.098433 seconds
<master/apiclient> waiting for at least one node to register and become ready
<master/apiclient> first node is ready after 10.034029 seconds
<master/discovery> created essential addon: kube-discovery, waiting for it to become ready
<master/discovery> kube-discovery is ready after 30.44947 seconds
<master/addons> created essential addon: kube-proxy
<master/addons> created essential addon: kube-dns

Kubernetes master initialised successfully!

You can now join any number of machines by running the following on each node:

kubeadm join --token 7fa96f.ddb39492a1874689 10.20.30.1
```

The most important command is the kubeadm join command, You need to keep this secret if someone get the token and your master IP they will be able to automatically add nodes to your cluster.

Now install the POD network

```sh
kubectl apply -f https://git.io/weave-kube
```

Becuase we have a small number of nodes we also want to use our master server as a minion

```sh
kubectl taint nodes --all dedicated-
```

Now you are ready to connect your minions nodes to the master, Assuming you have installed the base packes on node-02 and node-3 simply run,

```sh
kubeadm join --token 7fa96f.ddb39492a1874689 10.20.30.1
```

This will configure each node.

To check that the nodes are all checked in run,

```sh
kubectl get nodes

NAME      STATUS    AGE
node-01   Ready     7h
node-02   Ready     7h
node-03   Ready     5h
```

You should be an output like the one above, Congrats you have a kuberntes cluster.

### Step five - Setup kubectl on your local machine

TODO

```sh
kubectl config set-credentials ubuntu --username={KUBE_USER} --password={KUBE_PASSWORD}
kubectl config set-cluster personal --server=http://{MAIN_IP}:8080
kubectl config set-context personal-context --cluster=personal --user=ubuntu
kubectl config use-context personal-context
kubectl config set contexts.personal-context.namespace the-right-prefix
kubectl config view
```
