---
sidebar_position: 7
title: RKE2 cluster with SELinux enabled
---

# Intro
`SELinux`, or Security-Enhanced Linux, is a security framework for Linux operating systems that provides an additional layer of access control and mandatory access controls (`MAC`) beyond the traditional discretionary access controls (`DAC`). Developed by the National Security Agency (`NSA`) in collaboration with the open-source community, `SELinux` aims to enforce fine-grained control over processes, files, and system resources to enhance the overall security of the system.

> In this guide, we'll show you how to deploy an RKE2 cluster with SELinux enabled and enforcing.

## Prerequisites

* 1 VM. Hint [SLE Micro on OSX on Apple Silicon (UTM)](https://suse-edge.github.io/docs/quickstart/slemicro-utm-aarch64) or [SLE Micro on X86_64 on libvirt (virt-install)](https://suse-edge.github.io/docs/quickstart/slemicro-virt-install-x86_64) can be used as the base platform for validation here, but these instructions should work on any SLE Micro based system.
  - The VM should meet the [RKE2 requirements](https://docs.rke2.io/install/requirements#linuxwindows).

## Installation

### Installation for x86-64 architecture

Once we've got the VM started and running, let's prepare the config to enable SELinux mode in the RKE2 configuration file:

```shell
mkdir -p /etc/rancher/rke2 && echo "selinux: true" >> /etc/rancher/rke2/config.yaml
```

Install RKE2 cluster

```shell
curl -sfL https://get.rke2.io | INSTALL_RKE2_CHANNEL=stable INSTALL_RKE2_METHOD=rpm RKE2_SELINUX=true sh -

# Enable and Start RKE2
systemctl enable rke2-server.service
```

Now, the VM should be rebooted for the transactional-update to finish properly:

```shell
reboot
```

### Installation on arm64 architecture

As there are no RPM builds for RKE2 on `arm64` architecture, the `tarball` method will be used and the `rke2-selinux` policy will be installed manually.

#### Install rke2-selinux

The first thing that will be installed is the [rke2-selinux](https://github.com/rancher/rke2-selinux) policy.

Let's connect to the VM and run:

```shell
cat >> install-selinux.sh << 'END'
#!/bin/bash

# Install rpm-testing.rancher.io repository key and the rke2-selinux package
mkdir -p /var/lib/rpm-state
curl -o /root/public.key https://rpm-testing.rancher.io/public.key
curl -L -o /root/rke2-selinux.rpm https://github.com/rancher/rke2-selinux/releases/download/v0.15.testing.1/rke2-selinux-0.15-1.slemicro.noarch.rpm
rpmkeys --import /root/public.key

# Install RKE2 with SELinux
zypper install -y /root/rke2-selinux.rpm
END

chmod +x install-selinux.sh && transactional-update run /root/install-selinux.sh
```

Now, the VM should be rebooted for the transactional-update to finish properly:

```shell
reboot
```

After restarting the VM, we can verify that the policy was successfully installed as follows:

```shell
rpm -qa | grep rke2
```

#### Install RKE2

As a second step, an RKE2 cluster will be installed, which will use the policy installed in the previous section.

As the `rke2-policy` was installed manually on the VM, some of its paths may not be created correctly, so the following commands will ensure that all the paths are fine.


Let's connect to the VM and run:
```shell
mkdir -p /var/lib/cni
mkdir -p /opt/cni
mkdir -p /var/lib/kubelet/pods
mkdir -p /var/lib/rancher/rke2/agent/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots
mkdir -p /var/lib/rancher/rke2/data
mkdir -p /var/run/flannel
mkdir -p /var/run/k3s
restorecon -R -i /etc/systemd/system/rke2.service
restorecon -R -i /usr/lib/systemd/system/rke2.service
restorecon -R /var/lib/cni
restorecon -R /opt/cni
restorecon -R /var/lib/kubelet
restorecon -R /var/lib/rancher
restorecon -R /var/run/k3s
restorecon -R /var/run/flannel
```

It's time for the RKE2 cluster to be installed but before that, RKE2 must be running Selinux mode:

```shell
mkdir -p /etc/rancher/rke2 && echo "selinux: true" >> /etc/rancher/rke2/config.yaml
```

Install RKE2 Using Install Script

```shell
curl -sfL https://get.rke2.io | INSTALL_RKE2_EXEC="server" RKE2_SELINUX=true INSTALL_RKE2_VERSION=v1.27.3+rke2r1 sh -

# Enable and Start RKE2
systemctl enable --now rke2-server.service
```

> **NOTE:** RKE2 version `1.27` is the first that supports `arm64` architecture and it is still an experimental feature.

### Get the Kubeconfig

To use the Kubeconfig outside of the node, the following commands can be used:

```shell
# Replace <node-ip> with the actual ip
export NODE_IP=<node-ip>

sudo scp ${NODE_IP}:/etc/rancher/rke2/rke2.yaml ~/.kube/config && sed -i '' "s/127.0.0.1/${NODE_IP}/g" ~/.kube/config && chmod 600 ~/.kube/config
```

### Verify the setup

Check SELinux status:
```shell
sestatus
```
The output should be similar to this one:
```
SELinux status:                 enabled
SELinuxfs mount:                /sys/fs/selinux
SELinux root directory:         /etc/selinux
Loaded policy name:             targeted
Current mode:                   enforcing
Mode from config file:          enforcing
Policy MLS status:              enabled
Policy deny_unknown status:     allowed
Memory protection checking:     requested (insecure)
Max kernel policy version:      33
```

Check that all pods are in Running state:
```shell
kubectl get pod -A
```
The output should be similar to this one:
```
NAMESPACE     NAME                                                    READY   STATUS      RESTARTS       AGE
kube-system   cloud-controller-manager-slemicro                       1/1     Running     0 (2m3s ago)   3d5h
kube-system   etcd-slemicro                                           1/1     Running     0 (2m9s ago)   3d5h
kube-system   kube-apiserver-slemicro                                 1/1     Running     0 (2m9s ago)   3d5h
kube-system   kube-controller-manager-slemicro                        1/1     Running     0 (2m7s ago)   3d5h
(2m9s ago)   3d5h
...
```
