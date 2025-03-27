---
title: '[K8s] Prepare Environment'
date: 2024-07-08 13:26:38
tags: k8s
categories: Posts
---

# Prepare Environment

## Containerd Install

> Container Runtimes
>
> Kubernetes 1.29 requires that you use a runtime that conforms with the [Container Runtime Interface](https://kubernetes.io/docs/concepts/overview/components/#container-runtime) (CRI).

###  1. Installing containerd

```bash
wget https://github.com/containerd/containerd/releases/download/v1.7.11/containerd-1.7.11-linux-amd64.tar.gz

tar -xvf containerd-1.6.26-linux-amd64.tar.gz -C /usr/local/
```

> https://github.com/containerd/containerd/blob/main/containerd.service

```bash
cat >> /etc/systemd/system/containerd.service << 'EOF'
# Copyright The containerd Authors.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target local-fs.target

[Service]
ExecStartPre=-/sbin/modprobe overlay
ExecStart=/usr/local/bin/containerd

Type=notify
Delegate=yes
KillMode=process
Restart=always
RestartSec=5

# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNPROC=infinity
LimitCORE=infinity

# Comment TasksMax if your systemd version does not supports it.
# Only systemd 226 and above support this version.
TasksMax=infinity
OOMScoreAdjust=-999

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now containerd
systemctl status containerd
```

### 2. Installing runc

```bash
wget https://github.com/opencontainers/runc/releases/download/v1.1.11/runc.amd64

install -m 755 runc.amd64 /usr/local/sbin/runc
```

### 3. Installing CNI plugins

```bash
wget https://github.com/containernetworking/plugins/releases/download/v1.4.0/cni-plugins-linux-amd64-v1.4.0.tgz

mkdir -p /opt/cni/bin
tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.1.1.tgz
```

### Default config

> containerd uses a configuration file located in `/etc/containerd/config.toml` for specifying daemon level options. A sample configuration file can be found [here](https://github.com/containerd/containerd/blob/main/docs/man/containerd-config.toml.5.md).
>
> The default configuration can be generated via `containerd config default > /etc/containerd/config.toml`.

```
version = 2

root = "/var/lib/containerd"
state = "/run/containerd"
oom_score = 0
imports = ["/etc/containerd/runtime_*.toml", "./debug.toml"]

[grpc]
  address = "/run/containerd/containerd.sock"
  uid = 0
  gid = 0

[debug]
  address = "/run/containerd/debug.sock"
  uid = 0
  gid = 0
  level = "info"

[metrics]
  address = ""
  grpc_histogram = false

[cgroup]
  path = ""

[plugins]
  [plugins."io.containerd.monitor.v1.cgroups"]
    no_prometheus = false
  [plugins."io.containerd.service.v1.diff-service"]
    default = ["walking"]
  [plugins."io.containerd.gc.v1.scheduler"]
    pause_threshold = 0.02
    deletion_threshold = 0
    mutation_threshold = 100
    schedule_delay = 0
    startup_delay = "100ms"
  [plugins."io.containerd.runtime.v2.task"]
    platforms = ["linux/amd64"]
    sched_core = true
  [plugins."io.containerd.service.v1.tasks-service"]
    blockio_config_file = ""
    rdt_config_file = ""
```

## Install and configure prerequisites

```bash
## Forwarding IPv4 and letting iptables see bridged traffic
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

## sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system

## Verify that the br_netfilter, overlay modules are loaded by running the following commands:
lsmod | grep br_netfilter
lsmod | grep overlay

## Verify that the net.bridge.bridge-nf-call-iptables, net.bridge.bridge-nf-call-ip6tables, and net.ipv4.ip_forward system variables are set to 1 in your sysctl config by running the following command:
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
```

## Cgroup drivers

- cgroupfs

  - The `cgroupfs` driver is **not** recommended when [systemd](https://www.freedesktop.org/wiki/Software/systemd/) is the init system because systemd expects a single cgroup manager on the system. Additionally, if you use [cgroup v2](https://kubernetes.io/docs/concepts/architecture/cgroups), use the `systemd` cgroup driver instead of `cgroupfs`.

- systemd

  - To set `systemd` as the cgroup driver, edit the [`KubeletConfiguration`](https://kubernetes.io/docs/tasks/administer-cluster/kubelet-config-file/) option of `cgroupDriver` and set it to `systemd`. For example:

    ```yaml
    apiVersion: kubelet.config.k8s.io/v1beta1
    kind: KubeletConfiguration
    ...
    cgroupDriver: systemd
    ```

  > Note: Starting with v1.22 and later, when creating a cluster with kubeadm, if the user does not set the cgroupDriver field under KubeletConfiguration, kubeadm defaults it to systemd

##  Container runtimes

> In this step once you've created a valid `config.toml` configuration file.
>
> You can find this file under the path `/etc/containerd/config.toml`.
>
> On Linux the default CRI socket for containerd is `/run/containerd/containerd.sock`.

### Configuring the systemd cgroup driver

> The `systemd` cgroup driver is recommended if you use [cgroup v2](https://kubernetes.io/docs/concepts/architecture/cgroups).

```bash
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  ...
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
    
sudo sed -i "s#SystemdCgroup = false#SystemdCgroup = true#g" /etc/containerd/config.toml
### If you apply this change, make sure to restart containerd:
sudo systemctl restart containerd
```

###  Overriding the sandbox (pause) image

In your [containerd config](https://github.com/containerd/containerd/blob/main/docs/cri/config.md) you can overwrite the sandbox image by setting the following config:

```
[plugins."io.containerd.grpc.v1.cri"]
  sandbox_image = "registry.k8s.io/pause:3.2"
```

```bash
[plugins."io.containerd.grpc.v1.cri".registry.mirrors]
### add following config in China   [plugins."io.containerd.grpc.v1.cri".registry.mirrors."k8s.gcr.io"]
      endpoint = ["registry.cn-hangzhou.aliyuncs.com/google_containers"]

### This is important if you live in China.
sudo sed -i "s#registry.k8s.io/pause#registry.cn-hangzhou.aliyuncs.com/google_containers/pause#g" /etc/containerd/config.toml
sudo systemctl restart containerd
```

```bash
ctr images pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.22.3
ctr images pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.22.3
ctr images pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.22.3
ctr images pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.22.3
ctr images pull registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.5
ctr images pull registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.5.0-0
ctr images pull registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:v1.8.4

ctr images tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.22.3 k8s.gcr.io/kube-apiserver:v1.22.3
ctr images tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.22.3 k8s.gcr.io/kube-controller-manager:v1.22.3
ctr images tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.22.3 k8s.gcr.io/kube-scheduler:v1.22.3
ctr images tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.22.3 k8s.gcr.io/kube-proxy:v1.22.3
ctr images tag registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.5 k8s.gcr.io/pause:3.5
ctr images tag registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.5.0-0 k8s.gcr.io/etcd:3.5.0-0
ctr images tag registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:v1.8.4 k8s.gcr.io/coredns/coredns:v1.8.4
```

