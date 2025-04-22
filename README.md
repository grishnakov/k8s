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

## Install cni plugins

## Install required packages for setup of container runtime (containerd)

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
sudo ufw allow 20
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
