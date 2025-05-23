# Setting up k8s

### Fix for booting into Linux installer (insert into grub) for old radeon cards (I think)(fixes issue of linux installer not loading)

```bash
radeon.modeset=0
```

## Disable swap

```
sudo vim /etc/fstab
```

comment out the line with swap (last line in the case of boss)

## Enable IPv4 packet forwarding

### sysctl params required by setup, params persist across reboots

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
```

and add overlay and br_netfilter to current running environment os that they load on booting

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

Apply sysctl params without reboot

```bash
sudo sysctl --system
```

Verify everything is running:

```bash
lsmod | grep br_netfilter
lsmod | grep overlay
```

and

```bash
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
```

The values should be set to 1

## Configure cgroup v2 to be the only cgroup (ensure you use systemd as your cgroup driver)

Verify systemd version:

```bash
systemd --version
```

this should output version 240+

```bash
sudo vim  /etc/default/grub
```

Add `cgroup_no_v1=all` after the `GRUB_CMDLINE_LINUX` in the grub file.

```bash
sudo update-grub
```

```bash
sudo reboot
```

Do

```bash
cat /proc/cmdline | grep cgroup_no_v1=all
```

After reboot to verify. If you see `cgroup_no_v1=all` in the output, it means cgroup v2 is active

## Install runc

### Install required dependencies

#### Install golang

golang needs to be version 1.23 (as of commit) check the go.mod file of the [runc repository](https://github.com/opencontainers/runc/blob/main/go.mod) to verify which version of golang is needed

```bash
# Download latest Go
curl -s https://go.dev/VERSION?m=text | \
  xargs -I {} curl -LO https://go.dev/dl/{}.linux-amd64.tar.gz
```

```bash
# Remove any previous Go installation
sudo rm -rf /usr/local/go
```

```bash
# Extract the new version
sudo tar -C /usr/local -xzf go*.linux-amd64.tar.gz
```

```bash
# Add Go to your PATH (if not already)
echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.bashrc
source ~/.bashrc
```

```bash
# Remove tar.gz file
rm go*.tar.gz
```

#### Install other dependencies

```bash
sudo apt update && sudo apt install -y make gcc linux-libc-dev libseccomp-dev pkg-config git
```

### Installing runc

```bash
git clone https://github.com/opencontainers/runc
cd runc
```

run the make command

```bash
make
sudo make install
```

**If there is an error, first check if it was installed, it is possible that the make command built everything already**
To check:

```bash
which runc
runc --version
```

For maintaining/other issues reference [https://github.com/opencontainers/runc?tab=readme-ov-file](https://github.com/opencontainers/runc?tab=readme-ov-file)

## Install cni plugins (container network plugins)

Get current tag name from github and set appropriate environment variable:

```bash
export ARCH=$([ "$(uname -m)" = "aarch64" ] && echo "arm64" || echo "amd64")
export CNI_PLUGIN_VERSION=$(curl -s \
  https://api.github.com/repos/containernetworking/plugins/releases/latest \
  | grep '"tag_name":' \
  | sed -E 's/.*"([^"]+)".*/\1/')
```

Download latest release:

```bash
curl -fsSL -o cni-plugins.tgz \
  "https://github.com/containernetworking/plugins/releases/download/${CNI_PLUGIN_VERSION}/cni-plugins-linux-${ARCH_CNI}-${CNI_PLUGIN_VERSION}.tgz"
```

Extract and install the binary:

```bash
sudo mkdir -p /opt/cni/bin
sudo tar -C /opt/cni/bin -xzvf cni-plugins.tgz
sudo chown root:root /opt/cni/bin/*
sudo chmod +x /opt/cni/bin/*
```

Verify installation worked correctly:

```bash
ls /opt/cni/bin
```

You should see plugins like `bridge`, `host-local`, `ipvlan`, `loopback`, etc.

## Install required packages for setup of container runtime (containerd)

Download and unzip the latest tar.gz:

```bash
CRD_VERSION=$(curl -s https://api.github.com/repos/containerd/containerd/releases/latest \
            | grep '"tag_name":' \
            | sed -E 's/.*"v?([^"]+)".*/\1/')
# 3. Download the corresponding tarball
wget https://github.com/containerd/containerd/releases/download/v${CRD_VERSION}/containerd-${CRD_VERSION}-linux-${ARCH}.tar.gz
# 4. Extract into /usr/local
sudo tar -C /usr/local -xzvf containerd-${CRD_VERSION}-linux-${ARCH}.tar.gz
```

Create needed folder for systemd

```bash
sudo mkdir -p /usr/local/lib/systemd/system/
```

containerd.service file:

```bash
sudo bash -c 'cat <<EOF > /usr/local/lib/systemd/system/containerd.service
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
After=network.target dbus.service

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
EOF'
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now containerd
```

Verify:

```bash
sudo systemctl status containerd.service
```

Should be `active(running)`

#### Creating the config

```bash
sudo mkdir /etc/containerd
```

```bash
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
```

OLD VERSION

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
```

```bash
sudo apt-get install -y containerd.io
```

### Containerd config instructions

 [https://github.com/containerd/containerd/blob/main/docs/cri/config.md](https://github.com/containerd/containerd/blob/main/docs/cri/config.md)

Should be located in `/etc/containerd/config.toml`

### Create the correct folder and default config file

```bash
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
```

### Modify the config file for systemd to be the cgroup driver

```bash
vim /etc/containerd/config.toml
```

In the section containing

```
      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
      ...
```

Modify the value for `SystemCgroup` from `false` to `true`

```
            SystemdCgroup = true
```

#### Doable in one command

```bash
sudo sed -i 's/            SystemdCgroup = false/            SystemdCgroup = true/' /etc/containerd/config.toml
```

### Restart containerd with the new config

```bash
sudo systemctl restart containerd
```

(some info take from this [blog](https://www.nocentino.com/posts/2021-12-27-installing-and-configuring-containerd-as-a-kubernetes-container-runtime/))

## Configuring firewall

### For all nodes, but dependent on application

```bash
sudo ufw allow ssh
sudo ufw allow 22
sudo ufw allow 80
sudo ufw allow 443
```

### For control plane node

```bash
sudo ufw allow 6443
sudo ufw allow 2379
sudo ufw allow 2380
sudo ufw allow 10250
sudo ufw allow 10259
sudo ufw allow 10257
```

For worker nodes:

```bash
sudo ufw allow 10256
sudo ufw allow 10250
sudo ufw allow 30000:32767/tcp
```

After creating all rules, enable and reload the firewall:

```bash
sudo ufw enable
sudo ufw reload
```

## Kubelet config

kubelet should be configured to use systemd:

```
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
cgroupDriver: "systemd"
```

## Kubeadm config

File in this repo [kubeadm-config.yaml](./kubeadm-config.yaml)
