---
title: "Multi-vendor EVPN VXLAN setup with Containerlab"
date: 2022-03-03
draft: true
tags: [containerlab, cumulus, arista, junos, bgp, vxlan, evpn, cisco, nexus]
description:
---

## Introduction and topology

This post continues to build and showcase the power of containerlab. In this post, we will create a multivendor EVPN topology for L2 overlays. The vendors included are Arista (vEOS), Juniper (vQFX), Cumulus (Cumulus VX) and Cisco (N9Kv). Every piece of software used here is free (some might be behind a login, so you'll have to register to get access to image downloads).

The topology is as follows:

![topology1](/images/multivendor/multivendor_1/topology_1.jpg)

Arista vEOS is used for both the spines and leaf1. Cumulus VX is leaf2, Cisco N9Kv is leaf3 and Juniper vQFX is leaf4. Behind each leaf is a host, and all hosts are in the same 10.100.100.0/24 subnet. Router-IDs for each spine and leaf, along with the host details are as follows:

![topology2](/images/multivendor/multivendor_1/topology_2.jpg)

The endgoal is to get all the hosts to talk to each other, and also understand if these vendors play nice with each other when deploying L2 overlays with BGP EVPN and VXLAN (w're in for a few interesting surprises here).

## Building the docker images
### Juniper vQFX

Juniper vQFX can be built using their qcow2 images that are available publicly. I've already written about this in an earlier blog post with complete instructions on how this is built and what is happening behind the scenes.

### Cumulus VX

Docker images are not officially maintained for this by Cumulus Networks (now Nvidia). However, Michael (@networkop) maintains some unofficial images [here](https://hub.docker.com/r/networkop/cx/tags). You can simply pull this using 'docker pull'.

```
root@aninchat-ubuntu:~/clabs# docker pull networkop/cx:4.4.0
4.4.0: Pulling from networkop/cx
bd8f6a7501cc: Pull complete
cd45a4bb2831: Pull complete
3267eb82436d: Pull complete
43c69aa0f07d: Pull complete
5467689b757b: Pull complete
47813b34fad7: Pull complete
dffd6a1e6cf7: Pull complete
c5ff024177cd: Pull complete
8fbd9fcb1a0b: Pull complete
b24d53edac2f: Pull complete
a287a0d978c0: Pull complete
050552fdbe9c: Pull complete
12c6d13747e2: Pull complete
7a90f1a90542: Pull complete
d4e73cf57d80: Pull complete
e90926507c70: Pull complete
2476c9c7d51c: Pull complete
28e6b7764b01: Pull complete
81c55abe901f: Pull complete
9072c910401f: Pull complete
f4e46c63c0c6: Pull complete
4347adc92cde: Pull complete
1e2015807ed1: Pull complete
bb63128af12d: Pull complete
ff47aad4ed38: Pull complete
8956d77e1873: Pull complete
41a3ceebbb60: Pull complete
440c42b04957: Pull complete
bb3ae1d72db8: Pull complete
085ab76472c5: Pull complete
e851254ed5e5: Pull complete
6122afe252b1: Pull complete
2156d27e047f: Pull complete
d1b2f513878e: Pull complete
836eb1aec088: Pull complete
5aa925411d8b: Pull complete
Digest: sha256:2ef73abf91c2214ceec08df00580c8754d7a4163391841fe2ad714596a361a4a
Status: Downloaded newer image for networkop/cx:4.4.0
docker.io/networkop/cx:4.4.0
```

You should now see this image available for use:

```
root@aninchat-ubuntu:~/clabs# docker images | grep cx
networkop/cx         5.0.1       4d6152fa636b   2 weeks ago     805MB
networkop/cx         4.4.0       468cdd1a4be5   7 months ago    772MB
```

He maintains a fairly up-to-date list of corresponding docker images, but I've pulled an older one because I am not too fond of the new NVUE interface that Cumulus Linux has shifted to - I prefer the older NCLU, which 4.4.x still runs.

### Arista vEOS

Arista vEOS can be downloaded for free from Arista's software download site. This is locked behind a guest registration though, so you still need to go through that entire process if you'd like to download this. Once you have the vEOS image (it should be a vmdk file), place it in the vrnet/veos folder.

```
root@aninchat-ubuntu:~/vrnetlab/veos# pwd
/root/vrnetlab/veos
root@aninchat-ubuntu:~/vrnetlab/veos# ls -l
total 491984
-rw-r--r-- 1 root root      1028 Feb  5 15:41 Makefile
-rw-r--r-- 1 root root      1281 Feb  5 15:41 README.md
drwxr-xr-x 2 root root      4096 Feb 16 06:35 docker
-rw-r--r-- 1 root root 503775232 Feb 16 06:35 vEOS-lab-4.27.2F.vmdk
```

You can now trigger the docker image build using 'make'. Since I already have the image built, it doesn't do much for me at this point.

```
root@aninchat-ubuntu:~/vrnetlab/veos# make
Makefile:18: warning: overriding recipe for target 'docker-pre-build'
../makefile.include:18: warning: ignoring old recipe for target 'docker-pre-build'
for IMAGE in vEOS-lab-4.27.2F.vmdk; do \
	echo "Making $IMAGE"; \
	make IMAGE=$IMAGE docker-build; \
done
Making vEOS-lab-4.27.2F.vmdk
make[1]: Entering directory '/root/vrnetlab/veos'
Makefile:18: warning: overriding recipe for target 'docker-pre-build'
../makefile.include:18: warning: ignoring old recipe for target 'docker-pre-build'
rm -f docker/*.qcow2* docker/*.tgz* docker/*.vmdk* docker/*.iso
# checking if ZTP config contains a string (DISABLE=True) in the file /zerotouch-config
# if it does, we don't need to write this file
Checking ZTP status
ZTPOFF=DISABLE=True; \
echo "docker-pre-build: ZTPOFF is $ZTPOFF" && \
    if [ "$ZTPOFF" != "DISABLE=True" ]; then \
      echo "Disabling ZTP" && docker run --rm -it -e LIBGUESTFS_DEBUG=0 -v $(pwd):/work cmattoon/guestfish -a vEOS-lab-4.27.2F.vmdk -m /dev/sda2 write /zerotouch-config "DISABLE=True"; \
    fi
docker-pre-build: ZTPOFF is DISABLE=True
Building docker image using vEOS-lab-4.27.2F.vmdk as vrnetlab/vr-veos:4.27.2F
cp ../common/* docker/
make IMAGE=$IMAGE docker-build-image-copy
make[2]: Entering directory '/root/vrnetlab/veos'
Makefile:18: warning: overriding recipe for target 'docker-pre-build'
../makefile.include:18: warning: ignoring old recipe for target 'docker-pre-build'
cp vEOS-lab-4.27.2F.vmdk* docker/
make[2]: Leaving directory '/root/vrnetlab/veos'
(cd docker; docker build --build-arg http_proxy= --build-arg https_proxy= --build-arg IMAGE=vEOS-lab-4.27.2F.vmdk -t vrnetlab/vr-veos:4.27.2F .)
Sending build context to Docker daemon  503.8MB
Step 1/11 : FROM ubuntu:20.04
 ---> 54c9d81cbb44
Step 2/11 : LABEL maintainer="Kristian Larsson <kristian@spritelink.net>"
 ---> Using cache
 ---> b8a0857a144e
Step 3/11 : LABEL maintainer="Roman Dodin <dodin.roman@gmail.com>"
 ---> Using cache
 ---> a22af11cc083
Step 4/11 : ARG DEBIAN_FRONTEND=noninteractive
 ---> Using cache
 ---> 1735e5bccc44
Step 5/11 : RUN apt-get update -qy  && apt-get upgrade -qy  && apt-get install -y     bridge-utils     iproute2     python3-ipy     socat     qemu-kvm     tcpdump     tftpd-hpa     ssh     inetutils-ping     dnsutils     openvswitch-switch     iptables     telnet  && rm -rf /var/lib/apt/lists/*
 ---> Using cache
 ---> 43deb024677e
Step 6/11 : ARG IMAGE
 ---> Using cache
 ---> 36e30f700548
Step 7/11 : COPY $IMAGE* /
 ---> Using cache
 ---> 858ae07ca107
Step 8/11 : COPY *.py /
 ---> Using cache
 ---> 62058063f86c
Step 9/11 : EXPOSE 22 80 161/udp 443 830 5000 6030 10000-10099 57400
 ---> Using cache
 ---> 53c2102098d6
Step 10/11 : HEALTHCHECK CMD ["/healthcheck.py"]
 ---> Using cache
 ---> 9b3ebcc4ab71
Step 11/11 : ENTRYPOINT ["/launch.py"]
 ---> Using cache
 ---> 11aed84d6be0
Successfully built 11aed84d6be0
Successfully tagged vrnetlab/vr-veos:4.27.2F
make[1]: Leaving directory '/root/vrnetlab/veos'
```

You should now see this image available to use:

```
root@aninchat-ubuntu:~/vrnetlab/n9kv# docker images | grep veos
vrnetlab/vr-veos     4.27.2F     11aed84d6be0   2 weeks ago     932MB
```

### Cisco N9Kv

Cisco's N9Kv is also available for free (again, locked behind an account registration). Similar to the earlier processes, download the qcow2 image and move it to the vrnetlab/n9kv folder.

```
root@aninchat-ubuntu:~/vrnetlab/n9kv# pwd
/root/vrnetlab/n9kv
root@aninchat-ubuntu:~/vrnetlab/n9kv# ls -l
total 1934160
-rw-r--r-- 1 root root        266 Feb  5 15:41 Makefile
-rw-r--r-- 1 root root        585 Feb  5 15:41 README.md
drwxr-xr-x 2 root root       4096 Feb 16 15:54 docker
-rw-r--r-- 1 root root 1980563456 Feb 16 06:51 nxosv.9.3.9.qcow2
```

Be sure to use the 'n9kv' folder and not the 'nxos' folder - the 'nxos' folder is for the older titanium images. Once the image is copied here, trigger 'make' to build the docker image for this.

```
root@aninchat-ubuntu:~/vrnetlab/n9kv# make
for IMAGE in nxosv.9.3.9.qcow2; do \
	echo "Making $IMAGE"; \
	make IMAGE=$IMAGE docker-build; \
done
Making nxosv.9.3.9.qcow2
make[1]: Entering directory '/root/vrnetlab/n9kv'
rm -f docker/*.qcow2* docker/*.tgz* docker/*.vmdk* docker/*.iso
Building docker image using nxosv.9.3.9.qcow2 as vrnetlab/vr-n9kv:9.3.9
cp ../common/* docker/
make IMAGE=$IMAGE docker-build-image-copy
make[2]: Entering directory '/root/vrnetlab/n9kv'
cp nxosv.9.3.9.qcow2* docker/
make[2]: Leaving directory '/root/vrnetlab/n9kv'
(cd docker; docker build --build-arg http_proxy= --build-arg https_proxy= --build-arg IMAGE=nxosv.9.3.9.qcow2 -t vrnetlab/vr-n9kv:9.3.9 .)
Sending build context to Docker daemon  1.985GB
Step 1/12 : FROM ubuntu:20.04
 ---> 54c9d81cbb44
Step 2/12 : LABEL maintainer="Kristian Larsson <kristian@spritelink.net>"
 ---> Using cache
 ---> b8a0857a144e
Step 3/12 : LABEL maintainer="Roman Dodin <dodin.roman@gmail.com>"
 ---> Using cache
 ---> a22af11cc083
Step 4/12 : ARG DEBIAN_FRONTEND=noninteractive
 ---> Using cache
 ---> 1735e5bccc44
Step 5/12 : RUN apt-get update -qy  && apt-get upgrade -qy  && apt-get install -y     bridge-utils     iproute2     python3-ipy     socat     qemu-kvm     tcpdump     tftpd-hpa     ssh     inetutils-ping     dnsutils     openvswitch-switch     iptables     telnet  && rm -rf /var/lib/apt/lists/*
 ---> Using cache
 ---> 43deb024677e
Step 6/12 : ARG IMAGE
 ---> Using cache
 ---> 36e30f700548
Step 7/12 : COPY $IMAGE* /
 ---> Using cache
 ---> 66fbaa5d2045
Step 8/12 : COPY OVMF.fd /
 ---> Using cache
 ---> 87c96b2ebf08
Step 9/12 : COPY *.py /
 ---> Using cache
 ---> 756902fd2036
Step 10/12 : EXPOSE 22 80 161/udp 443 830 5000 6030 10000-10099 57400
 ---> Using cache
 ---> df3be4a02195
Step 11/12 : HEALTHCHECK CMD ["/healthcheck.py"]
 ---> Using cache
 ---> 5143e36e8998
Step 12/12 : ENTRYPOINT ["/launch.py"]
 ---> Using cache
 ---> 0eb634fdcd16
Successfully built 0eb634fdcd16
Successfully tagged vrnetlab/vr-n9kv:9.3.9
make[1]: Leaving directory '/root/vrnetlab/n9kv'
```

You should now see this image available to use:

```
root@aninchat-ubuntu:~/vrnetlab/n9kv# docker images | grep n9k
vrnetlab/vr-n9kv     9.3.9       0eb634fdcd16   2 weeks ago     2.41GB
```

### End hosts

For hosts, I like to use Michael's host docker image (networkop/host) which are based on Ubuntu 20.04. You can pull the latest tag ('ifreload') from dockerhub.

```
root@aninchat-ubuntu:~# docker pull networkop/host:ifreload
ifreload: Pulling from networkop/host
f7ec5a41d630: Pull complete
dcf92054a263: Pull complete
3179c3d1124f: Pull complete
eea897900801: Pull complete
717337aab6d8: Pull complete
23ffec7efd97: Pull complete
6bdc05c9f934: Pull complete
Digest: sha256:ed38567dc55f88cda67a208eae36f6a6569b461c767f7be6db04298ae89f3fc5
Status: Downloaded newer image for networkop/host:ifreload
docker.io/networkop/host:ifreload
```

This image should now be available to use:

```
root@aninchat-ubuntu:~# docker images | grep host
networkop/host       ifreload    b73cafd09ad8   7 months ago    283MB
```

We have all our vendor images built and ready to go now. 

## The Containerlab topology

The following topology file describes the nodes and the interconnections between them for containerlab:

```
root@aninchat-ubuntu:~/clabs/multivendor# cat evpn-multivendor.yml
name: evpn-l2-multivendor
topology:
  nodes:
    spine1:
            kind: vr-veos
            image: vrnetlab/vr-veos:4.27.2F
            mgmt_ipv4: 172.20.20.101
    spine2:
            kind: vr-veos
            image: vrnetlab/vr-veos:4.27.2F
            mgmt_ipv4: 172.20.20.102
    leaf1:
            kind: vr-veos
            image: vrnetlab/vr-veos:4.27.2F
            mgmt_ipv4: 172.20.20.11
    leaf2:
            kind: cvx
            image: networkop/cx:4.4.0
    leaf3:
            kind: vr-n9kv
            image: vrnetlab/vr-n9kv:9.3.9
            mgmt_ipv4: 172.20.20.13
    leaf4:
            kind: vr-vqfx
            image: vrnetlab/vr-vqfx:20.2R1.10
            mgmt_ipv4: 172.20.20.14
    h1:
            kind: linux
            image: networkop/host:ifreload
            binds:
                    - hosts/h1_interfaces:/etc/network/interfaces
            mgmt_ipv4: 172.20.20.21
    h2:
            kind: linux
            image: networkop/host:ifreload
            binds:
                    - hosts/h2_interfaces:/etc/network/interfaces
            mgmt_ipv4: 172.20.20.22
    h3:
            kind: linux
            image: networkop/host:ifreload
            binds:
                    - hosts/h3_interfaces:/etc/network/interfaces
            mgmt_ipv4: 172.20.20.23
    h4:
            kind: linux
            image: networkop/host:ifreload
            binds:
                    - hosts/h4_interfaces:/etc/network/interfaces
            mgmt_ipv4: 172.20.20.24
  links:
          - endpoints: ["leaf1:eth1", "spine1:eth1"]
          - endpoints: ["leaf1:eth2", "spine2:eth1"]
          - endpoints: ["leaf2:swp1", "spine1:eth2"]
          - endpoints: ["leaf2:swp2", "spine2:eth2"]
          - endpoints: ["leaf3:eth1", "spine1:eth3"]
          - endpoints: ["leaf3:eth2", "spine2:eth3"]
          - endpoints: ["leaf4:eth1", "spine1:eth4"]
          - endpoints: ["leaf4:eth2", "spine2:eth4"]
          - endpoints: ["leaf1:eth3", "h1:eth1"]
          - endpoints: ["leaf2:swp3", "h2:eth1"]
          - endpoints: ["leaf3:eth3", "h3:eth1"]
          - endpoints: ["leaf4:eth3", "h4:eth1"]
```

Note that the 'binds' block allows us to bind a file in the host OS to the container itself. In this case, I am simply pre-configuring the network interfaces for the fabric hosts by copying relevant configuration into the /etc/network/interfaces file of the container.

For example, let's take host h1 and it's file, hosts/h1_interfaces:

```
root@aninchat-ubuntu:~/clabs/multivendor# cat hosts/h1_interfaces
auto lo
iface lo inet loopback

auto eth1
iface eth1 inet static
    address 10.100.100.1
    netmask 255.255.255.0
```

All my hosts are in the same subnet, so h2 is 10.100.100.2/24, h3 is 10.100.100.3/24 and h4 is 10.100.100.4/24. We can now deploy this lab using 'containerlab deploy':

```
root@aninchat-ubuntu:~/clabs/multivendor# containerlab deploy --topo evpn-multivendor.yml
INFO[0000] Containerlab v0.23.0 started
INFO[0000] Parsing & checking topology file: evpn-multivendor.yml
INFO[0000] Creating lab directory: /root/clabs/multivendor/clab-evpn-l2-multivendor
INFO[0000] Creating docker network: Name='clab', IPv4Subnet='172.20.20.0/24', IPv6Subnet='2001:172:20:20::/64', MTU='1500'
INFO[0000] Creating container: h4
INFO[0000] Creating container: h3
INFO[0000] Creating container: spine2
INFO[0000] Creating container: leaf1
INFO[0000] Creating container: leaf3
INFO[0000] Creating container: leaf4
INFO[0000] Creating container: h1
INFO[0000] Creating container: h2
INFO[0000] Creating container: spine1
INFO[0001] Creating virtual wire: leaf1:eth3 <--> h1:eth1
INFO[0001] Creating virtual wire: leaf4:eth3 <--> h4:eth1
INFO[0001] Creating virtual wire: leaf4:eth2 <--> spine2:eth4
INFO[0001] Creating virtual wire: leaf1:eth2 <--> spine2:eth1
INFO[0002] Creating virtual wire: leaf1:eth1 <--> spine1:eth1
INFO[0002] Creating virtual wire: leaf4:eth1 <--> spine1:eth4
INFO[0002] Creating virtual wire: leaf3:eth1 <--> spine1:eth3
INFO[0002] Creating virtual wire: leaf3:eth2 <--> spine2:eth3
INFO[0002] Creating virtual wire: leaf3:eth3 <--> h3:eth1
INFO[0002] Networking is handled by "docker-bridge"
INFO[0002] Started Firecracker VM "b01b941136a18c36" in a container with ID "86191b18f7f9fd90011aa9627ee972c882beb0d2fdb43a98e45347ccd2ca4014"
INFO[0002] Creating virtual wire: leaf2:swp1 <--> spine1:eth2
INFO[0002] Creating virtual wire: leaf2:swp2 <--> spine2:eth2
INFO[0002] Creating virtual wire: leaf2:swp3 <--> h2:eth1
INFO[0003] Adding containerlab host entries to /etc/hosts file
INFO[0003] 🎉 New containerlab version 0.24.1 is available! Release notes: https://containerlab.srlinux.dev/rn/0.24/#0241
Run 'containerlab version upgrade' to upgrade or go check other installation options at https://containerlab.srlinux.dev/install/
+----+---------------------------------+--------------+------------------------------+---------+---------+------------------+----------------------+
| #  |              Name               | Container ID |            Image             |  Kind   |  State  |   IPv4 Address   |     IPv6 Address     |
+----+---------------------------------+--------------+------------------------------+---------+---------+------------------+----------------------+
|  1 | clab-evpn-l2-multivendor-h1     | e54edb974d39 | networkop/host:ifreload      | linux   | running | 172.20.20.21/24  | 2001:172:20:20::4/64 |
|  2 | clab-evpn-l2-multivendor-h2     | 11a73b7514c0 | networkop/host:ifreload      | linux   | running | 172.20.20.22/24  | 2001:172:20:20::5/64 |
|  3 | clab-evpn-l2-multivendor-h3     | 9f22eff7b0ca | networkop/host:ifreload      | linux   | running | 172.20.20.23/24  | 2001:172:20:20::8/64 |
|  4 | clab-evpn-l2-multivendor-h4     | a02ead147994 | networkop/host:ifreload      | linux   | running | 172.20.20.24/24  | 2001:172:20:20::3/64 |
|  5 | clab-evpn-l2-multivendor-leaf1  | ff2871985b4e | vrnetlab/vr-veos:4.27.2F     | vr-veos | running | 172.20.20.11/24  | 2001:172:20:20::2/64 |
|  6 | clab-evpn-l2-multivendor-leaf2  | 86191b18f7f9 | docker.io/networkop/cx:4.4.0 | cvx     | running | 172.17.0.2/24    | N/A                  |
|  7 | clab-evpn-l2-multivendor-leaf3  | 8a317c31929d | vrnetlab/vr-n9kv:9.3.9       | vr-n9kv | running | 172.20.20.13/24  | 2001:172:20:20::9/64 |
|  8 | clab-evpn-l2-multivendor-leaf4  | 690414536e2b | vrnetlab/vr-vqfx:20.2R1.10   | vr-vqfx | running | 172.20.20.14/24  | 2001:172:20:20::6/64 |
|  9 | clab-evpn-l2-multivendor-spine1 | b628b4de29b9 | vrnetlab/vr-veos:4.27.2F     | vr-veos | running | 172.20.20.101/24 | 2001:172:20:20::a/64 |
| 10 | clab-evpn-l2-multivendor-spine2 | 3f426a4c2db4 | vrnetlab/vr-veos:4.27.2F     | vr-veos | running | 172.20.20.102/24 | 2001:172:20:20::7/64 |
+----+---------------------------------+--------------+------------------------------+---------+---------+------------------+----------------------+
```

Remember to wait till the containers are reported as healthy. If they remain unhealthy, then something was wrong with the docker build and you should probably re-visit that. 

```
root@aninchat-ubuntu:~/clabs/multivendor# docker ps
CONTAINER ID   IMAGE                        COMMAND                  CREATED         STATUS                   PORTS                                                                                       NAMES
86191b18f7f9   networkop/ignite:dev         "/usr/local/bin/igni…"   8 minutes ago   Up 7 minutes                                                                                                         ignite-b01b941136a18c36
b628b4de29b9   vrnetlab/vr-veos:4.27.2F     "/launch.py --userna…"   8 minutes ago   Up 7 minutes (healthy)   22/tcp, 80/tcp, 443/tcp, 830/tcp, 5000/tcp, 6030/tcp, 10000-10099/tcp, 57400/tcp, 161/udp   clab-evpn-l2-multivendor-spine1
8a317c31929d   vrnetlab/vr-n9kv:9.3.9       "/launch.py --userna…"   8 minutes ago   Up 7 minutes (healthy)   22/tcp, 80/tcp, 443/tcp, 830/tcp, 5000/tcp, 6030/tcp, 10000-10099/tcp, 57400/tcp, 161/udp   clab-evpn-l2-multivendor-leaf3
ff2871985b4e   vrnetlab/vr-veos:4.27.2F     "/launch.py --userna…"   8 minutes ago   Up 8 minutes (healthy)   22/tcp, 80/tcp, 443/tcp, 830/tcp, 5000/tcp, 6030/tcp, 10000-10099/tcp, 57400/tcp, 161/udp   clab-evpn-l2-multivendor-leaf1
3f426a4c2db4   vrnetlab/vr-veos:4.27.2F     "/launch.py --userna…"   8 minutes ago   Up 7 minutes (healthy)   22/tcp, 80/tcp, 443/tcp, 830/tcp, 5000/tcp, 6030/tcp, 10000-10099/tcp, 57400/tcp, 161/udp   clab-evpn-l2-multivendor-spine2
690414536e2b   vrnetlab/vr-vqfx:20.2R1.10   "/launch.py --userna…"   8 minutes ago   Up 7 minutes (healthy)   22/tcp, 830/tcp, 5000/tcp, 10000-10099/tcp, 161/udp                                         clab-evpn-l2-multivendor-leaf4
e54edb974d39   networkop/host:ifreload      "/entrypoint.sh"         8 minutes ago   Up 7 minutes                                                                                                         clab-evpn-l2-multivendor-h1
11a73b7514c0   networkop/host:ifreload      "/entrypoint.sh"         8 minutes ago   Up 7 minutes                                                                                                         clab-evpn-l2-multivendor-h2
a02ead147994   networkop/host:ifreload      "/entrypoint.sh"         8 minutes ago   Up 7 minutes                                                                                                         clab-evpn-l2-multivendor-h4
9f22eff7b0ca   networkop/host:ifreload      "/entrypoint.sh"         8 minutes ago   Up 7 minutes                                                                                                         clab-evpn-l2-multivendor-h3
```

## Automating the fabric bringup

In order to automate the fabric bringup, I've written a (terrible) Ansible script. The script can also be found [here](). The inventory for the script is built from the IP addresses that were statically assigned in the containerlab topology file. The inventory is also crucial to grouping together devices, allowing for group variables to kick in:

```
root@aninchat-ubuntu:~/clabs/multivendor# cat inventory.yml
---

all:
        children:
                leafs:
                        hosts:
                                leaf1:
                                        ansible_host: 172.20.20.11
                                leaf2:
                                        ansible_host: 172.17.0.2
                                leaf3:
                                        ansible_host: 172.20.20.13
                                leaf4:
                                        ansible_host: 172.20.20.14
                spines:
                        hosts:
                                spine1:
                                        ansible_host: 172.20.20.101
                                spine2:
                                        ansible_host: 172.20.20.102
                nxos:
                        hosts:
                                leaf3:
                junos:
                        hosts:
                                leaf4:
                eos:
                        hosts:
                                spine1:
                                spine2:
                                leaf1:
                cumulus:
                        hosts:
                                leaf2:
```

