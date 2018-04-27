# CaaSP KVM

This directory contains the caasp-kvm tool, aimed at replacing the existing caasp-devenv
and terraform repositories.

A primary goal of caasp-kvm is to reduce the number of places where our development
enviroments differ from a customer enviroment, allowing us to more easily ensure our
changes really will work in the final product.

## Requirements

Follow this README in order from **Zypper setup** to **CLI syntax** to provision 
a cluster starting from a fresh OS install. 

# Zypper setup

Add the containers repository for the correct version of Docker, among other things. 
See Notes section.   
```
sudo zypper ar obs://Virtualization:containers/openSUSE_Leap_42.2 Virtualization:containers
```
You must also have SUSE's CA certificates installed, as documented on
http://ca.suse.de . Replace distribution 42.2 with your version if different. 
```
zypper ar --refresh http://download.suse.de/ibs/SUSE:/CA/openSUSE_Leap_42.2/SUSE:CA.repo
zypper in -y ca-certificates-suse p11-kit-nss-trust
```
The first time you install a package from a new repository zypper will ask you to accept or 
reject the GPG key. Select 'a' for 'always'.


# Packages and services
Run the following as root or via sudo:   
```
zypper in -y git jq docker python-requests python-tox python-devel python-openstackclient python-novaclient python-heatclient qemu-kvm libvirt-daemon-qemu terraform terraform-provider-libvirt
systemctl enable libvirtd
systemctl start libvirtd
systemctl enable docker
systemctl start docker
```

# VPN access
Many of the required resources are hosted inside SUSE's private R&D network; in
order to access and connect to these resouces, you may need a SUSE R&D openvpn
connection; see the
[Micro Focus internal wiki](https://wiki.microfocus.net/index.php?title=SUSE-Development/OPS/Services/OpenVPN)
for details.


# User settings
The user running `caasp-kvm` must be a member of the dependent service groups. 
Log out and back in for changes to take effect. 


# Project dependencies
Clone these repositories:
```
git clone git@github.com:kubic-project/salt.git
git clone git@github.com:kubic-project/velum.git
git clone git@github.com:kubic-project/caasp-container-manifests.git
git clone git@github.com:kubic-project/automation.git
```

# VM and storage setup

Initialize the default storage pool for your VMs. 

default-pool.xml:
```
<pool type='dir'>
  <name>default</name>
  <target>
    <path>/path/to/libvirt/images</path>
  </target>
</pool>
```

```
sudo virsh pool-create default-pool.xml
sudo virsh pool-autostart default
sudo virsh pool-start default
```

## CLI Syntax

    Usage:

      * Building a cluster

        -b|--build             Run the CaaSP KVM Build Step
        -m|--masters <INT>     Number of masters to build (Default: CAASP_NUM_MASTERS=1)
        -w|--workers <INT>     Number of workers to build (Default: CAASP_NUM_WORKERS=2)
        -i|--image <STR>       Image to use (Default: CAASP_IMAGE=channel://devel)
        --vanilla              Do not inject devenv code, use vanilla caasp (Default: false)

      * Destroying a cluster

        -d|--destroy                Run the CaaSP KVM Destroy Step

      * Common options

        -p|--parallelism            Set terraform parallelism (Default: CAASP_PARALLELISM)
        -P|--proxy                  Set HTTP proxy (Default: CAASP_HTTP_PROXY)
        -L|--location               Set location used for downloads (Default: CAASP_LOCATION or 'default')

      * Local git checkouts

         --salt-dir <DIR>           the Salt repo checkout (Default: CAASP_SALT_DIR)
         --manifests-dir <DIR>      the manifests repo checkout (Default: CAASP_MANIFESTS_DIR)
         --velum-dir <DIR>          the Velum repo checkout (Default: CAASP_VELUM_DIR)

      * Advanced Options

        --plan                 Run the CaaSP KVM Plan Step
        --admin-ram <INT>      RAM to allocate to admin node (Default: CAASP_ADMIN_RAM=4096)
        --admin-cpu <INT>      CPUs to allocate to admin node (Default: CAASP_ADMIN_CPU=4)
        --master-ram <INT>     RAM to allocate to master node(s) (Default: CAASP_MASTER_RAM=2048)
        --worker-ram <INT>     CPUs to allocate to master node(s) (Default: CAASP_MASTER_CPU=2)
        --master-cpu <INT>     RAM to allocate to worker node(s) (Default: CAASP_WORKER_RAM=2048)
        --worker-cpu <INT>     CPUs to allocate to worker node(s) (Default: CAASP_WORKER_CPU=2)
        --extra-repo <STR>     URL of a custom repository on the master(s)/worker(s) (Default: CAASP_EXTRA_REPO)

      * Examples:

      Build a 1 master, 2 worker cluster

      ./caasp-kvm --build -m 1 -w 2

      Build a 1 master, 2 worker cluster using the latest kubic image

      ./caasp-kvm --build -m 1 -w 2 --image channel://kubic

      Build a 1 master, 2 worker cluster using the latest staging A image

      ./caasp-kvm --build -m 1 -w 2 --image channel://staging_a

      Build a 1 master, 2 worker cluster and add a custom repository on the master(s)/worker(s)

      ./caasp-kvm --build -m 1 -w 2 --extra-repo https://download.opensuse.org/repositories/devel:/CaaSP:/Head:/ControllerNode:/TestUpdates

      Destroy a cluster

      ./caasp-kvm --destroy

# Notes

The [Terraform](https://github.com/hashicorp/terraform) and the
[terraform-provider-libvirt](https://github.com/dmacvicar/terraform-provider-libvirt)
providers must be installed.

We maintain packages for both of them inside of the
[Virtualization:containers](https://build.opensuse.org/project/show/Virtualization:containers)
project on OBS.

## Using a cluster with custom rpm packages from a repository

In the devenv we can already inject code from a few components (e.g. velum, salt, manifests)

But to test e.g. a different docker image, or another version of kubernetes you can add a custom repository url to the commandline and this repo will then be added to either the master(s), the worker(s) or both:

    ./caasp-kvm --build -m 1 -w 2 --extra-repo https://download.opensuse.org/repositories/devel:/CaaSP:/Head:/ControllerNode:/TestUpdates

After the cluster is up and running, you can either head over to the velum user interface and trigger the update (after transactional-update has run), or install the new packages on master/worker nodes via:

    # on the host
    RESOLV_CONF=$(< /etc/resolv.conf) transactional-update shell
    # in the transactional-update shell
    echo "$RESOLV_CONF" > /etc/resolv.conf
    zypper -n dup --allow-vendor-change
    exit
    # on the host
    reboot

Writing again the resolv.conf is necessary as the there is no overlayfs in the chroot environment of the transactional-update shell

## Using a cluster from the hypervisor node

The nodes booted will be given fixed/deterministic IP addresses, it it recommended
you create a ~/.ssh/config with the following content, adjusting the SSH key path
as necessary:

    Host 10.17.*
        User root
        IdentityFile /home/kiall/SUSE/caasp/automation/misc-files/id_shared
        UserKnownHostsFile /dev/null
        StrictHostKeyChecking no

Each node will have an determinisic IP address, and additional nodes will have the
final octet incremented by 1:

* Admin node will have 10.17.1.0
* First master node will have 10.17.2.0
* First worker node will have 10.17.3.0

## Using a cluster from a non-hypervisor node

It's possible to interact with the whole cluster even from machines that are
**not** the hypervisor node running all the VMs.

This can be useful when your development machine is not powerful enough to
run an entire cluster. In this case we can have all the VMs running on a
powerful workstation and access them from your laptop.

First of all you have to pick a different subnet for the caasp network. This
is done by creating a `terraform.tfvars` with the following line:

```hcl
caasp_net_network = "route"
caasp_net_network = "172.30.0.0/22"
```

You have to be really careful with the subnet you are going to use. The default
one is conflicting with the SUSE R&D network. If your remote machine is
connected to the corporate VPN all the requests made against your cluster will
be swallowed by the VPN.
At the same time you cannot pick networks like `172.16.0.0/13` or
`172.17.0.0/24` because the first one is being used by flannel, while the second
one is used by docker.
A working subnet is `172.30.0.0/22`.

Finally you have to add a route to the cluster on the development machine. The
easiest way is to use the following command:

```
ip -4 route add 172.30.0.0/22 via 192.168.1.39
```

Where `192.168.1.39` is the IP address of the hypervisor node.

You can achieve the same result by setting the rule on your router, pushing
the rule through a local dnsmasq instance,...

### Remote devenv setup

This example shows how to have a remote devenv setup. The goals of this setup
are:

  * Run the cluster on an hypervisor nodes.
  * Develop the code on a laptop.
  * Access the cluster from any local computer.

All the source code is checked out on the laptop inside of
`~/code/kubic`. This is where all the changes are done.

The whole `~/code/kubic` directory is automatically sent to the workstation
(hostname `workstation`) and kept in sync using
[lsyncd](https://github.com/axkibe/lsyncd) (which has official openSUSE
packages BTW).

The configuration file for `lsyncd` is saved inside of
`~/code/kubic/lsyncd.conf`:

```lua
settings {
  logfile    = "/tmp/lsyncd.log",
  statusFile = "/tmp/lsyncd.status",
  nodaemon   = true,
}

sync {
  default.rsyncssh,
  source    = "/home/developer/code/kubic",
  host      = "developer@workstation",
  targetdir = "/home/developer/code/kubic",
  exclude   = {
    "automation/downloads/**",
    "*.tfstate",
    "*.tfstate.backup",
    "*.terraform" }
}
```

Now it's just a matter of going to `~/code/kubic` and leave this command running:

```
lsyncd lsyncd.conf
```

Now you can go to the workstation and use caasp-kvm as usual.

## Running development tests against velum

To run velum's rspec tests:

* ssh into the admin node
* execute: `docker exec $(docker ps | grep dashboard | awk {'print $1'}) entrypoint.sh bash -c "RAILS_ENV=test bundle exec rspec"`

To run velum's rubocop:

* ssh into the admin node
* execute: `docker exec $(docker ps | grep dashboard | awk {'print $1'}) bundle exec rubocop`
