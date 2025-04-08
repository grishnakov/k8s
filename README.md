# Setting up k8s

### Fix for booting into Linux installer (insert into grub)

```
radeon.modeset=0
```

## Configuring firewall

### For all nodes, but dependent on application

```
sudo ufw allow ssh
sudo ufw allow 20
sudo ufw allow 80
sudo ufw allow 443
```

### For control plane node

```
sudo ufw allow 6443
sudo ufw allow 2379
sudo ufw allow 2380
sudo ufw allow 10250
sudo ufw allow 10259
sudo ufw allow 10257
```

For worker nodes:

```
sudo ufw allow 10256
sudo ufw allow 10250
sudo ufw allow 30000:32767/tcp
```

```sudo vim /etc/fstab``` comment out the line with swap (last line in the case of boss)

```sysctl net.ipv4.ip_forward``` needs to output 1.

# sysctl params required by setup, params persist across reboots

```
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF
```

# Apply sysctl params without reboot

`sudo sysctl --system`
