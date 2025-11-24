#
# INSTALL Fedora CoreOS
-----------------------
https://discussion.fedoraproject.org/t/beginners-guide-to-fedora-coreos/143037
<br>
<br>
## Wipe all disks from the live USB before reinstalling:

```bash
 
  # Wipe the boot disk
  sudo wipefs -af /dev/nvme0n1
  sudo sgdisk --zap-all /dev/nvme0n1

  # Wipe the containers disk
  sudo wipefs -af /dev/nvme1n1
  sudo sgdisk --zap-all /dev/nvme1n1

  # Verify both are clean
  lsblk
```


## How to install Fedora CoreOS

```bash
sudo coreos-installer install /dev/nvme0n1 \
    --ignition-url http://172.22.3.132:8000/fcos.ign \
    --insecure-ignition
```


## Making things easier

```bash
alias butane='sudo docker run --rm --interactive --security-opt label=disable --volume "${PWD}:/pwd" --workdir /pwd quay.io/coreos/butane:release'
```

```bash
rpm-ostree install firewalld
systemctl reboot
```

## Update firewall rules for PiHole
```bash
sudo firewall-cmd --zone=FedoraServer --add-port=80/tcp
sudo firewall-cmd --zone=FedoraServer --add-port=443/tcp
sudo firewall-cmd --zone=FedoraServer --add-port=53/tcp
sudo firewall-cmd --zone=FedoraServer --add-port=53/udp
sudo firewall-cmd --zone=FedoraServer --add-port=67/udp
sudo firewall-cmd --permanent --zone=FedoraServer --add-port=53/udp
sudo firewall-cmd --permanent --zone=FedoraServer --add-port=53/tcp
sudo firewall-cmd --permanent --zone=FedoraServer --add-port=443/tcp
sudo firewall-cmd --permanent --zone=FedoraServer --add-port=67/udp
sudo firewall-cmd --permanent --zone=FedoraServer --add-port=80/tcp
```

## Update static to addresses in case not already set
```bash
sudo nmcli device modify enp9s0f0 ipv4.addresses 172.22.3.60/24
sudo nmcli device modify enp9s0f0 ipv4.gateway 172.22.3.1
sudo nmcli device modify enp9s0f0 ipv4.dns "8.8.8.8 8.8.4.4"

sudo nmcli device modify enp9s0f1 ipv4.addresses 172.22.3.61/24
sudo nmcli device modify enp9s0f1 ipv4.gateway 172.22.3.1
sudo nmcli device modify enp9s0f1 ipv4.dns "8.8.8.8 8.8.4.4"

sudo systemctl disable systemd-resolved
sudo systemctl stop systemd-resolved
sudo unlink /etc/resolv.conf
sudo systemctl restart NetworkManager
```

## Remove docker daemon (so when we say "docker" we mean podman)

```bash
sudo rpm-ostree override remove moby-engine docker-cli moby-filesystem && \
  sudo rpm-ostree install podman-docker
```

## Setup PiHole

```bash
sudo podman volume create pihole_pihole
sudo podman volume create pihole_dnsmasq
sudo podman pull docker.io/pihole/pihole

sudo podman run --replace --name=pihole \
  --hostname=pihole \
  --cap-add=NET_ADMIN \
  --dns=127.0.0.1 \
  --dns=1.1.1.1 \
  -e TZ=America/Miami \
  -e SERVERIP=172.22.3.60 \
  -e DNS1=1.1.1.1 \
  -e DNS2=1.0.0.1 \
  -e DNSSEC=true \
  -e DNSMASQ_LISTENING=all \
  -e CONDITIONAL_FORWARDING=true \
  -e CONDITIONAL_FORWARDING_IP=172.22.3.1 \
  -e CONDITIONAL_FORWARDING_DOMAIN=lan \
  -e TEMPERATUREUNIT=c \
  -v pihole_pihole:/etc/pihole:Z \
  -v pihole_dnsmasq:/etc/dnsmasq.d:Z \
  -p 443:443/tcp \
  -p 67:67/udp \
  -p 53:53/tcp \
  -p 53:53/udp \
  docker.io/pihole/pihole
  
 
sudo podman exec -it pihole pihole setpassword 'Pit0yFlauta2112!'  

## Starting PiHole container automatically

```bash
  sudo mkdir -p /etc/containers/systemd
  sudo nano /etc/containers/systemd/pihole.container


[Unit]
Description=Pi-hole DNS
After=network-online.target
Wants=network-online.target

[Container]
Image=docker.io/pihole/pihole
ContainerName=pihole
HostName=pihole
AddCapability=NET_ADMIN
# host is handling time sync
# AddCapability=CAP_SYS_TIME
DNS=127.0.0.1
DNS=1.1.1.1
Environment=TZ=America/Miami
Environment=SERVERIP=172.22.3.60
Environment=DNS1=1.1.1.1
Environment=DNS2=1.0.0.1
Environment=FTLCONF_dns_dnssec=true
Environment=FTLCONF_dns_listeningMode=all
Environment=FTLCONF_dns_ipv6=false
Environment=FTLCONF_webserver_port=443s
Environment=CONDITIONAL_FORWARDING=true
Environment=CONDITIONAL_FORWARDING_IP=172.22.3.1
Environment=CONDITIONAL_FORWARDING_DOMAIN=lan
Environment=TEMPERATUREUNIT=c
Volume=pihole_pihole:/etc/pihole:Z
Volume=pihole_dnsmasq:/etc/dnsmasq.d:Z
PublishPort=9090:443/tcp
PublishPort=67:67/udp
PublishPort=53:53/tcp
PublishPort=53:53/udp

[Service]
Restart=always
TimeoutStartSec=300

[Install]
WantedBy=multi-user.target default.target
```

### Then reload and start:

```bash
  sudo systemctl daemon-reload
  sudo systemctl start pihole
```

### Check status
```bash
  sudo systemctl status pihole
```

### View logs
```bash
  sudo journalctl -u pihole -f
```

### Restart
```bash
  sudo systemctl restart pihole
```


# Prepare CoreOS for AI loads

## Install AMD PRO ROCM

```bash
sudo nano /etc/yum.repos.d/rocm.repo 

[rocm]
name=ROCm 7.1.0 repository
baseurl=https://repo.radeon.com/rocm/el10/7.1/main
enabled=1
priority=50
gpgcheck=0
gpgkey=https://repo.radeon.com/rocm/rocm.gpg.key

[amdgraphics]
name=AMD Graphics 7.1.0 repository
baseurl=https://repo.radeon.com/graphics/7.1/el/10/main/x86_64/
enabled=1
priority=50
gpgcheck=0
gpgkey=https://repo.radeon.com/rocm/rocm.gpg.key
```

## NOTE: gpgcheck=0 because of regression in FCOS 43

```bash
sudo rpm-ostree install rocm rocm-developer-tools
sudo reboot
```


## Checking ROCM installation


### Basic level

```bash
rocm-smi
rocminfo | grep -E "Name:|Marketing Name:|gfx"

amd-smi version
cat /opt/rocm/.info/version

hipconfig --full

rocminfo
clinfo
rocm-smi --showuse

lsmod|grep amdgpu
dmesg | grep -i amdgpu
```

### Deep Dive

```bash
git clone https://github.com/ROCm/HIP-Examples.git
cd HIP-Examples/vectorAdd
make
./vectoradd
```

```bash
sudo rocm-bandwidth-test
```


### Test PyTorch with ROCm (newer cards)

```bash
sudo podman run -it --device=/dev/kfd --device=/dev/dri --group-add video --group-add render --security-opt label=disable \
docker.io/rocm/pytorch:latest python3 -c "
import torch
import sys

print(f'PyTorch version: {torch.__version__}')
print(f'ROCm available: {torch.cuda.is_available()}')
print(f'Device count: {torch.cuda.device_count()}')
print(f'Device name: {torch.cuda.get_device_name(0)}')
sys.stdout.flush()

try:
    print('Creating tensor on GPU...')
    sys.stdout.flush()
    x = torch.randn(1000, 1000, device='cuda')
    print('Tensor created')
    sys.stdout.flush()
    y = torch.matmul(x, x)
    print('Matrix multiplication test: PASSED')
except Exception as e:
    print(f'ERROR: {e}')
sys.stdout.flush()
"
```

See how to use ROCm with PyTorch:
https://github.com/ROCmSoftwarePlatform/pytorch

See how to use ROCm with TensorFlow:
https://github.com/ROCmSoftwarePlatform/tensorflow-upstream

See how to use ROCm with ONNX:
https://github.com/ROCmSoftwarePlatform/onnx-mlir

```bash
podman run -it --device=/dev/kfd --device=/dev/dri docker.io/rocm/pytorch:latest
```

### Test PyTorch with ROCm (legacy Radeon VII)

sudo podman run -it --device=/dev/kfd --device=/dev/dri --group-add video --group-add render --security-opt label=disable \
  docker.io/rocm/pytorch:rocm5.7_ubuntu22.04_py3.10_pytorch_2.0.1 python3 -c "
import torch
import sys

print(f'PyTorch version: {torch.__version__}')
print(f'ROCm available: {torch.cuda.is_available()}')
print(f'Device count: {torch.cuda.device_count()}')
print(f'Device name: {torch.cuda.get_device_name(0)}')
sys.stdout.flush()

try:
    print('Creating tensor on GPU...')
    sys.stdout.flush()
    x = torch.randn(1000, 1000, device='cuda')
    print('Tensor created')
    sys.stdout.flush()
    y = torch.matmul(x, x)
    print('Matrix multiplication test: PASSED')
except Exception as e:
    print(f'ERROR: {e}')
sys.stdout.flush()
"
