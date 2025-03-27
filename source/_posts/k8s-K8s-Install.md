---
title: '[K8s] Kubernetes Install'
date: 2024-07-08 13:30:02
tags: k8s
categories: Posts
---

# K8s Install

## 1. Standard

- 2GB or more of RAM
- 2CPUs or more

## 2. Prepare

- You need to install a container runtime into each node in the cluster so that Pods can run there.

- Certain ports are open on your machines
  - systemctl firewalld stop
- Disable swap
  - swapoff -a
  - check /etc/fstab
- Verify glibc is provide or not in machines
- Verify the MAC address and product_uuid are unique for every node
  - ip link
  - ifconfig -a
  - cat /sys/class/dmi/id/product_uuid
- Check required ports
  - nc 127.0.0.1 6443

## 3. Installing kubeadm, kubelet and kubectl

>kubelet version may never exceed the API server version.
>
>For example, the kubelet running 1.7.0 should be fully compatible with a 1.8.0 API server, but not vice versa.

#### **Red Hat-based distributions**

- Set SELinux to `permissive` mode

  - setenforce 0
  - sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

- Add the Kubernetes `yum` repository

  ```bash
  cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
  [kubernetes]
  name=Kubernetes
  baseurl=https://pkgs.k8s.io/core:/stable:/v1.29/rpm/
  enabled=1
  gpgcheck=1
  gpgkey=https://pkgs.k8s.io/core:/stable:/v1.29/rpm/repodata/repomd.xml.key
  exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
  EOF
  ```

- Install kubelet, kubeadm and kubectl, and enable kubelet to ensure it's automatically started on startup

  - yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
  - systemctl enable --now kubelet

#### **Without a package manager**

- Install CNI plugins (required for most pod network)

  ```bash
  CNI_PLUGINS_VERSION="v1.3.0"
  ARCH="amd64"
  DEST="/opt/cni/bin"
  sudo mkdir -p "$DEST"
  curl -L "https://github.com/containernetworking/plugins/releases/download/${CNI_PLUGINS_VERSION}/cni-plugins-linux-${ARCH}-${CNI_PLUGINS_VERSION}.tgz" | sudo tar -C "$DEST" -xz
  
  cat >> /etc/cni/net.d/10-containerd-net.conflist <<EOF
  {
    "cniVersion": "1.0.0",
    "name": "containerd-net",
    "plugins": [
      {
        "type": "bridge",
        "bridge": "cni0",
        "isGateway": true,
        "ipMasq": true,
        "promiscMode": true,
        "ipam": {
          "type": "host-local",
          "ranges": [
            [{
              "subnet": "10.88.0.0/16"
            }],
            [{
              "subnet": "2001:4860:4860::/64"
            }]
          ],
          "routes": [
            { "dst": "0.0.0.0/0" },
            { "dst": "::/0" }
          ]
        }
      },
      {
        "type": "portmap",
        "capabilities": {"portMappings": true}
      }
    ]
  }
  EOF
  ```

- Define the directory to download command files

  ```bash
  DOWNLOAD_DIR="/usr/local/bin"
  sudo mkdir -p "$DOWNLOAD_DIR"
  ```

- Install crictl (required for kubeadm / Kubelet Container Runtime Interface (CRI))

  ```bash
  CRICTL_VERSION="v1.28.0"
  ARCH="amd64"
  curl -L "https://github.com/kubernetes-sigs/cri-tools/releases/download/${CRICTL_VERSION}/crictl-${CRICTL_VERSION}-linux-${ARCH}.tar.gz" | sudo tar -C $DOWNLOAD_DIR -xz
  ```

- Install `kubeadm`, `kubelet`, `kubectl` and add a `kubelet` systemd service

  ```bash
  RELEASE="$(curl -sSL https://dl.k8s.io/release/stable.txt)"
  ARCH="amd64"
  cd $DOWNLOAD_DIR
  sudo curl -L --remote-name-all https://dl.k8s.io/release/${RELEASE}/bin/linux/${ARCH}/{kubeadm,kubelet}
  sudo chmod +x {kubeadm,kubelet}
  
  RELEASE_VERSION="v0.16.2"
  curl -sSL "https://raw.githubusercontent.com/kubernetes/release/${RELEASE_VERSION}/cmd/krel/templates/latest/kubelet/kubelet.service" | sed "s:/usr/bin:${DOWNLOAD_DIR}:g" | sudo tee /etc/systemd/system/kubelet.service
  sudo mkdir -p /etc/systemd/system/kubelet.service.d
  curl -sSL "https://raw.githubusercontent.com/kubernetes/release/${RELEASE_VERSION}/cmd/krel/templates/latest/kubeadm/10-kubeadm.conf" | sed "s:/usr/bin:${DOWNLOAD_DIR}:g" | sudo tee /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
  ```

  > Notes:
  >
  > Site raw.githubusercontent.com is 404 ... replace following url
  >
  > https://github.com/kubernetes/release/blob/master/cmd/krel/templates/latest/kubelet/kubelet.service
  >
  > https://github.com/kubernetes/release/blob/master/cmd/krel/templates/latest/kubeadm/10-kubeadm.conf

  ```bash
  cat >> /etc/systemd/system/kubelet.service << 'EOF'
  [Unit]
  Description=kubelet: The Kubernetes Node Agent
  Documentation=https://kubernetes.io/docs/
  Wants=network-online.target
  After=network-online.target
  
  [Service]
  ExecStart=/usr/bin/kubelet
  Restart=always
  StartLimitInterval=0
  RestartSec=10
  
  [Install]
  WantedBy=multi-user.target
  EOF
  cat >> /etc/systemd/system/kubelet.service.d/10-kubeadm.conf << 'EOF'
  # Note: This dropin only works with kubeadm and kubelet v1.11+
  [Service]
  Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
  Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml"
  # This is a file that "kubeadm init" and "kubeadm join" generates at runtime, populating the KUBELET_KUBEADM_ARGS variable dynamically
  EnvironmentFile=-/var/lib/kubelet/kubeadm-flags.env
  # This is a file that the user can use for overrides of the kubelet args as a last resort. Preferably, the user should use
  # the .NodeRegistration.KubeletExtraArgs object in the configuration files instead. KUBELET_EXTRA_ARGS should be sourced from this file.
  EnvironmentFile=-/etc/sysconfig/kubelet
  ExecStart=
  ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS
  EOF
  ```

- Install and Set Up kubectl on Linux

  - Install kubectl binary with curl on Linux

    1. Download the latest release with the command

       ```bash
       curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
       
       ## View stable version
       curl -L -s https://dl.k8s.io/release/stable.txt
       ```

    2. Validate the binary (optional)

       ```bash
       ## Download the kubectl checksum file
       curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"
       
       ## Validate the kubectl binary against the checksum file
       echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check
       ```

    3. Install kubectl

       ```bash
       sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
       ```

    4. Test to ensure the version you installed is up-to-date

       ```bash
       kubectl version --client
       ## Or use this for detailed view of version
       kubectl version --client --output=yaml
       ```

  - Install using native package management

    1. Add the Kubernetes `yum` repository. If you want to use Kubernetes version different than v1.29, replace v1.29 with the desired minor version in the command below.

       ```bash
       # This overwrites any existing configuration in /etc/yum.repos.d/kubernetes.repo
       cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
       [kubernetes]
       name=Kubernetes
       baseurl=https://pkgs.k8s.io/core:/stable:/v1.29/rpm/
       enabled=1
       gpgcheck=1
       gpgkey=https://pkgs.k8s.io/core:/stable:/v1.29/rpm/repodata/repomd.xml.key
       EOF
       ```

    2. Install kubectl using `yum`

       ```bash
       sudo yum install -y kubectl
       ```

- Enable and start `kubelet`

  - ```bash
    systemctl enable --now kubelet
    ```

> Container runtimes
>
> `kubeadm init` output error
>
> ` [ERROR CRI]: container runtime is not running`
>
> https://github.com/containerd/containerd/blob/main/docs/getting-started.md
>
> https://github.com/containerd/containerd/blob/main/containerd.service
>
> yum -y install socat conntrack-tools

## 4. Creating a cluster with kubeadm

```bash
kubeadm init   --apiserver-advertise-address=192.168.175.133   --image-repository registry.aliyuncs.com/google_containers   --kubernetes-version v1.29.0   --service-cidr=10.1.0.0/16   --pod-network-cidr=10.244.0.0/16


mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

## if you are the root user, you can run:
  export KUBECONFIG=/etc/kubernetes/admin.conf
```

### Installing a Pod network add-on

> https://github.com/containerd/containerd/blob/main/script/setup/install-cni

```bash
#!/usr/bin/env bash

#   Copyright The containerd Authors.

#   Licensed under the Apache License, Version 2.0 (the "License");
#   you may not use this file except in compliance with the License.
#   You may obtain a copy of the License at

#       http://www.apache.org/licenses/LICENSE-2.0

#   Unless required by applicable law or agreed to in writing, software
#   distributed under the License is distributed on an "AS IS" BASIS,
#   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
#   See the License for the specific language governing permissions and
#   limitations under the License.

#
# Builds and installs cni plugins to /opt/cni/bin,
# and create basic cni config in /etc/cni/net.d.
# The commit defined in go.mod
#
set -eu -o pipefail

CNI_COMMIT=${1:-$(go list -f "{{.Version}}" -m github.com/containernetworking/plugins)}
CNI_DIR=${DESTDIR:=''}/opt/cni
CNI_CONFIG_DIR=${DESTDIR}/etc/cni/net.d
: "${CNI_REPO:=https://github.com/containernetworking/plugins.git}"

# e2e and Cirrus will fail with "sudo: command not found"
SUDO=''
if (( $EUID != 0 )); then
    SUDO='sudo'
fi

TMPROOT=$(mktemp -d)
git clone "${CNI_REPO}" "${TMPROOT}"/plugins
pushd "${TMPROOT}"/plugins
git checkout "$CNI_COMMIT"
./build_linux.sh
$SUDO mkdir -p $CNI_DIR
$SUDO cp -r ./bin $CNI_DIR
$SUDO mkdir -p $CNI_CONFIG_DIR
$SUDO cat << EOF | $SUDO tee $CNI_CONFIG_DIR/10-containerd-net.conflist
{
  "cniVersion": "1.0.0",
  "name": "containerd-net",
  "plugins": [
    {
      "type": "bridge",
      "bridge": "cni0",
      "isGateway": true,
      "ipMasq": true,
      "promiscMode": true,
      "ipam": {
        "type": "host-local",
        "ranges": [
          [{
            "subnet": "10.88.0.0/16"
          }],
          [{
            "subnet": "2001:4860:4860::/64"
          }]
        ],
        "routes": [
          { "dst": "0.0.0.0/0" },
          { "dst": "::/0" }
        ]
      }
    },
    {
      "type": "portmap",
      "capabilities": {"portMappings": true}
    }
  ]
}
EOF

popd
rm -fR "${TMPROOT}"
```

### Control plane node isolation

```bash
## By default, your cluster will not schedule Pods on the control plane nodes for security reasons. If you want to be able to schedule Pods on the control plane nodes, for example for a single machine Kubernetes cluster, run:
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

