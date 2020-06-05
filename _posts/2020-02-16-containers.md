---
layout:      post
date:        2019-06-05 10:40:00 +0000
title:       Containers, what is a container and how do they work?
description: "How does docker internally work? Container standards and specifications: OCI, CRI, CNI. Server install: container vs virtual machine."
tags:        container docker cri-o containerd runc CRI OCI CNI gvisor
h1_title:    Containers
h2_desc:     What are containers and how do they work?
image:       /assets/image/containers-thumbnail.png
author:      tramlot
social:
  name: Tim Ramlot
  links:
    - https://www.linkedin.com/in/timramlot/
    - https://github.com/inteon
---

# The problem
When powering up a computing device all resources are allocated to the first program that starts up the device. On a microcontroller for example, the main program gets loaded into memory and is executed by the CPU, where all further actions are performed by that one program. On more complex systems, operating system (OS) software is used to allocate computer resources to software processes running on the device.

Optimal allocation of CPU, memory and IO (including disk and network) resources to software processes (or groups of processes) is a problem that is extensively addressed in computer science however always leading to trade off decisions to be made between operational, economical and governance (e.g. security risks) aspects of a system's design. 

## Sharing resources
In order to share resources an OS creates virtual subresources by partitioning available resources. This can be done multiple times, creating a hierarchy. New resources can also be created by combining 2 parent resources (e.g. a software disk raid). In general we recognize two types of resource partitioning, namely temporal (time-based) partitioning (e.g. 10% and 90% of CPU execution time for process A and process B respectively) and spatial (space-based) partitioning (e.g. CPU core 1 and core 2 for process A and process B respectively). The OS's process scheduler, virtual memory system and hardware drivers are all concretisations of this concept.

**Virtual Machines (VM)** were made to allocate a set of (sub)resources to a certain OS. This enables us to improve resource allocation efficiency (e.g. a linux webserver and a database server) while maintaining isolation (e.g. different virtual disks for both guest operating systems). The host OS (Virtual Machine Manager) performs the first level of resource mapping, namely from the physical to virtual devices, while the guest OSs repartition these virtual resources as if it were the physical versions. The guest OS has no knowledge or access to the physical parent resources or its neighbour virtual resources. Moreover, different types of guest operating systems (e.g. Windows and Linux) can be combined on the same host machine, which brings economical and operational benefits.

**Containers** are an alternative solution to the same problem. Using the process isolation features of the OS, we can allocate resources to a group of processes. The Linux kernel enables us to create such a setup by utilising their chroot (change root), cgroup (control group) and namespace APIs (Application Programing Interface). Child processes can be created such that they think by themselves as the root (init) process, without knowledge or access to their parent process resouces. Containers do, unlike VMs, share the OS kernel. As all containers must use the same OS type (eg. Windows or Linux) and the same kernel version, this allows us to have way less overhead than a VM with additional economical benefits.

{% include picture.html img="/assets/image/traditional-virtual-container" ext="png" alt="Traditional vs virtual vs container" %}

In practice, containers are often combined with virtual machines, to benefit from both technologies at the cost of the VM and container overhead.

# Containers
The container concept includes a low-level runtime, a high-level runtime and a container management solution. A low-level runtime manages Namespaces and cgroups. The high-level runtime controls the low-level runtime and includes a container image packaging and distribution client as well as a network and storage manager. A command line inteface to interact with the high-level runtime is typically available to manage the containers.

## Resource restrictions
Linux containers make use of control groups (cgroups) to limit the resources (memory, CPU, I/O, network, etc.) consumed by a container. This prevents a container from consuming all host resources and starving other processes.

## Isolation
Linux namespaces allow containers to have isolation on 7 levels:

{% include picture.html img="/assets/image/namespaces" ext="png" alt="Namespaces" class="width-400" %}

* **UTS**: isolates hostname
* **IPC**: isolates System V IPC, POSIX message queues
* **Network**: isolates network interface controllers, iptables firewall rules, routing tables, ...
* **Mount**: isolates mountpoints
* **PID**: isolates process IDs
* **User**: isolates user and group IDs
* **cgroup**: isolates cgroup root directory (container processes cannot see/ change resource limitations for that container)

Isolation means that groups of processes are separated such that they cannot "see" resources in other groups. Each time a new virtual hierarchy is created for the software processes, unaware of the parent hierarchial structure.
Possible example situations:
* Processes in different UTS namespaces see their dedicated hostname and can edit their hostname independently.
* Processes in different User namespaces see a their dedicates list of users and can add or remove users without affecting other processes.
* Processes get a different PID for each PID namespaces they are part of, each PID namespace has its own PID tree. Processes in two non-ovelapping namespaces do not see eachother, only a process in their common ancestor PID namespace can see both processes.

### Filesystem
Containers use a layered filesystem, better known as a union filesystem. Changes are being overlayed on top of a base image, creating a new overlay image. This way, base images can be shared across containers and storage efficiency can be increased. At the lowest level all images start from the "scratch" image (an empty image). A scripting file (e.g. Dockerfile, Gockerfile) specifies what changes to that image must be done relative to its parent base image. Consequently, images are built and all changes are being registered by the overlay file system and packed with a reference to the base image. The built images contain the binaries that are used as starting point when starting a container. On startup we always start from the built image mounted as the root of our filesystem contained within a mount namespace. These images are stateless: after a container restarts, all changes to the file system are discarded. If we want to run stateful applications in containers (e.g. database), we need to mount a stateful filesystem into this container on startup. This can be a network share or a folder on the host filesystem.

{% include picture.html img="/assets/image/container-image" ext="png" alt="Container image" class="width-500" %}

### Networking
In order to isolate our networking, we use networking namespaces. This means that every networking namespace needs at least a device used to communicate with the external world. Veth pairs, i.e. interconnected virtual ethernet devices, create such a setup, as if an ethernet cable connects them together. One of the ends is part of the networking namespace of the container. The other end is connected to a switch in the global networking interface. This switch interconnects all namespaces and can additionaly be connected to a physical interface in order to allow the containers to access the network/ internet.

## Security
Containers are secured by default. Although VMs have a security benefit over containers, their attack surface is smaller. VM security concerns preventing intruders from impacting other VMs or the hypervisor, for containers we want to prevent them to have any impact on other containers or host system. The linux kernel is exposed to the processes running inside the container and enlarge the set of possible attack vectors. By well utilising some of the features containers offer, they can be made more secure. Processes in the container can be run as non-privileged user. Binaries can be stripped from the image, reducing the tools available for an intruder. Also, the volotile nature of a container helps to protects its security. Additionally, a variety of tools designed to improve the security of containers can be used. These tools often add operational and/or configuration overhead, thus adding them might not always be the best solution. Here is a subset of the tools that are popular nowadays:

* **gvisor**: a virtual kernel runs in userspace and only utilises a limited set of systemcalls, thus limits the number of possible attack vectors
* **static analysis of images**: scanning the container image registry for vulnerable images, outdated dependencies, ... (e.g. Clair)
* **dynamic analysis of containers**: unexpected behaviour can be detected, shells that are spawn within containers, a process spawns a child process of an unexpected type, ... (eg. Falco)
* **signing container images**: signing your container images reduces the possibility of a compromised image registry, compromising your whole deloyment (e.g. Docker Notary)
* **kernel security**: the Linux kernel allows us to use additional tooling to restrict a process's behaviour
    * seccomp (short for secure computing) restricting the system calls a process can use
    * Linux security modules (LSM) like SELinux and AppArmor add fine grained access control

Due to the higher complexity of containers, misconfigurations are more likely to occure and might currently impose the biggest security risk linked to containers.

## Implementation
The **Open Container Initiative (OCI)** is a Linux Foundation project created to design an open standard for containers. Currently they published 3 specifications:
* *OCI Image Format Spec*: defines an OCI Image consisting of a manifest, an image index (optional), a set of filesystem layers and a configuration
* *OCI Runtime Spec*: specifies the configuration, execution environment and lifecycle of a container
* *OCI Distribution Spec*: defines an API protocol to facilitate distribution of images

These specifications for an OCI implementation are mostly based on the Docker implementation.

**Container Runtime Interface (CRI)** is another specification and defines an API that is used by e.g. kubernetes (kubelet) to interact with containers.

Lastly, we have **Container Networking Interface (CNI)**, a specification defining how to interact with a networking implementation.

Ideally, we have an implementation compatible with the OCI, CRI and CNI spec.

{% include picture.html img="/assets/image/container-api" ext="png" alt="API overview" %}

### Available software components
Following list contains a variety of specifications, implementations and tools for the APIs above: <https://github.com/inteon/awesome-container>. Some of them are further addressed below:

#### Runc (Low-level runtime)
Runc is the OCI Runtime reference implementation.

#### Containerd (High-level runtime)
Containerd is part of docker and used by dockerd to run and manage containers, images, storage and networking. In order to do so it is able to interact with OCI Runtimes and OCI Images (or Docker container images). One of its plugins makes containerd CRI compatible. Currently it is still the more popular CRI implementation.

#### Cri-o (High-level runtime)
Cri-o is an alternative to containerd. It is more optimized towards kubernetes (read less features) and is meant to provide a CRI interface to OCI conformant runtimes. It therefore utilises the containers/image libraries, the containers/storage libraries and a CNI client.
* containers/image: pull and push images and support conversion of docker container images to OCI images
* containers/storage: manages three types of items: layers, images, and containers. (From the github page:)
    - A **layer** is a copy-on-write filesystem which is notionally stored as a set of changes relative to its parent layer, if it has one. A given layer can only have one parent, but any layer can be the parent of multiple layers. Layers which are parents of other layers should be treated as read-only.
    - An **image** is a reference to a particular layer (its top layer), along with other information manageable for the library for the convenience of its caller. This information typically includes configuration templates for running a binary contained within the image's layers and may include cryptographic signatures. Multiple images can reference the same layer, as the differences between two images may not be in their layer contents.
    - A **container** is a read-write layer which is a child of an image's top layer, along with information manageable for the library for the convenience of its caller. This information typically includes configuration information for running the specific container. Multiple containers can be derived from a single image.
* CNI: setup networking to connect the container to a network as specified via CRI

# Contributions/ comments
Found an error, got any tips? Feel free to create a git issue/ pull request on <https://github.com/ramlot/blog>.

Author:
<div style="text-align: left;">
<strong style="font-size: 20px;">Tim Ramlot</strong><br>
<em>Engineering Student at Ghent University</em>
{% include picture.html img="/assets/image/ghent-university-faculty-of-engineering" ext="png" alt="Ghent university faculty of engineering" class="logo-right" %}
<a href="https://www.linkedin.com/in/timramlot/">https://www.linkedin.com/in/timramlot/</a>
</div>

# References and useful links
* [chroot-cgroups-and-namespaces](https://itnext.io/chroot-cgroups-and-namespaces-an-overview-37124d995e3d)
* [overlayfs](https://jvns.ca/blog/2019/11/18/how-containers-work--overlayfs/)
* [docker-supported-filesystems](https://docs.docker.com/storage/storagedriver/select-storage-driver/)
* [namespaces](https://www.toptal.com/linux/separation-anxiety-isolating-your-system-with-linux-namespaces)
* [tun-tap-and-veth-pairs](https://www.fir3net.com/Networking/Terms-and-Concepts/virtual-networking-devices-tun-tap-and-veth-pairs-explained.html)
* [linux-virtual-networking](https://developers.redhat.com/blog/2018/10/22/introduction-to-linux-interfaces-for-virtual-networking/)
* [container-security](https://ieeexplore.ieee.org/stamp/stamp.jsp?arnumber=8693491)
* [gvisor](https://cloud.google.com/blog/products/gcp/open-sourcing-gvisor-a-sandboxed-container-runtime)
* [building-a-container-from-scratch-in-Go](https://www.youtube.com/watch?v=Utf-A4rODH8)
* [building-a-container-blog](https://www.ianlewis.org/en/container-runtimes-part-1-introduction-container-r)
* [docker-components](http://alexander.holbreich.org/docker-components-explained/)
* [containers-are-linux](https://blog.openshift.com/containers-are-linux/)
* [container-overview](https://www.youtube.com/watch?v=el7768BNUPw)
* [what-is-a-container](https://www.youtube.com/watch?v=EnJ7qX9fkcU)
* [life-of-a-packet-through-Istio](https://www.youtube.com/watch?v=cB611FtjHcQ)
