# Dynamic Runner

PoC for a system that can run ephemeral containers and VMs.

```shell
python3 -m pip install -U pip
python3 -m pip install ansible ansible-lint
```

## Making VM qcow img

```shell
sudo apt install -y guestfs-tools

# Ensure source VM is off
sudo virt-sparsify -v --compress --format qcow2 /var/lib/libvirt/images/tinycore.qcow2 --convert qcow2 ./qemu-images/tinycore.qcow

```

## BRUTE

Thank this guide: https://iximiuz.com/en/posts/container-networking-is-simple/

```shell
# Create the initial bridge
sudo ip link add vbr0 type bridge
sudo ip link set vbr0 up
sudo ip addr add 172.18.0.1/24 dev vbr0
# Allow routing
sudo bash -c 'echo 1 > /proc/sys/net/ipv4/ip_forward'
# NAT (masquerade)
sudo iptables -t nat -A POSTROUTING -s 172.18.0.1/24 ! -o vbr0 -j MASQUERADE
# Warning! The above is apparently dangerous as it might not validate incoming connection??

# Base so it doesn't install into resolv.conf
sudo apt install dnsmasq-base

# Because libvirt will bind to our iface (I don't think this is needed)
echo 'except-interface=vbr0' | sudo tee /etc/dnsmasq.d/vbr0

cat <<EOF | sudo tee sudo /etc/dnsmasq.conf
listen-address=172.18.0.1
bind-interfaces

# DHCP range
dhcp-range=172.18.0.10,172.18.0.250,5m

# Set default gateway (should be done by default)
#dhcp-option=3,172.18.0.1

# Set DNS servers to announce (should be done by default)
#dhcp-option=6,172.18.0.1

EOF

#TODO: UNRELATED local dev machine issue bind-interfaces will fix it

# IF libvirt is running, nuke it
sudo killall dnsmasq
sudo systemctl restart libvirtd.service

#TODO: Create a service file
# Run this in another shell:
sudo dnsmasq -C /etc/dnsmasq.conf --no-daemon --log-queries --log-debug


# TODO: Might not have to install cni plugins? Seems
# Install CNI plugins into /opt/cni/bin
# https://github.com/containernetworking/plugins


# containernetworking-plugins has them installed? /usr/lib/cni/
#cd /tmp
#wget https://github.com/containernetworking/plugins/releases/download/v1.3.0/cni-plugins-linux-amd64-v1.3.0.tgz
#sudo mkdir -p /opt/cni/bin/
#sudo tar -C /opt/cni/bin/ -xzvf ./cni-plugins-linux-amd64-v1.3.0.tg

#sudo mkdir -p /etc/cni/net.d
#curl -qsSL https://raw.githubusercontent.com/containers/podman/main/cni/87-podman-bridge.conflist | sudo tee /etc/cni/net.d/87-podman-bridge.conflist


cat <<EOF | sudo tee /etc/cni/net.d/80-lolnet.conflist
{
  "cniVersion": "0.4.0",
  "name": "lolnet",
  "plugins": [
    {
      "type": "macvlan",
      "master": "vbr0",
      "linkInContainer": false,
      "ipam": {
        "type": "dhcp",
        "daemonSocketPath": "/run/cni/dhcp.sock",
        "request": [
          {
            "skipDefault": false
          }
        ],
        "provide": [
          {
            "option": "host-name",
            "fromArg": "K8S_POD_NAME"
          }
        ]
      }
    }
  ]
}

EOF

# Service came from `containernetworking-plugins`, which podman installed
sudo systemctl enable --now cni-dhcp.socket

# Run a podman
sudo podman run -dt --name web --network lolnet quay.io/libpod/banner
sudo podman exec web ip address show eth0


# MANUAL

sudo ip netns add netns0
sudo ip link add veth0 type veth peer name ceth0
sudo ip link set veth0 up
sudo ip link set ceth0 netns netns0
# If this errors, do it after you set an IP for ceth0
sudo ip link set veth0 master vbr0

# Enter the namespace
#sudo nsenter --net=/var/run/netns/netns0
sudo ip netns exec netns0
ip link set lo up
ip link set ceth0 up
ip addr add 172.18.0.10/24 dev ceth0
ip route add default via 172.18.0.1 dev ceth0
exit

# Test DHCP
nmap --script broadcast-dhcp-discover -e ceth0
# Test DNS
dig @172.18.0.1 google.com



sudo /opt/cni/bin/dhcp daemon -pidfile /run/cni/lol.pid  -socketpath /run/cni/lol-dhcp.sock -broadcast=true

enp0s3 10.0.2.100

```

```shell

#docker

# Create the initial bridge
sudo ip link add vbr0 type bridge
sudo ip link set vbr0 up
sudo ip addr add 172.18.0.1/24 dev vbr0
# Allow routing
sudo bash -c 'echo 1 > /proc/sys/net/ipv4/ip_forward'
# NAT (masquerade)
sudo iptables -t nat -A POSTROUTING -s 172.18.0.1/24 ! -o vbr0 -j MASQUERADE
# Warning! The above is apparently dangerous as it might not validate incoming connection??

# Base so it doesn't install into resolv.conf
sudo apt install dnsmasq-base

# Because libvirt will bind to our iface (I don't think this is needed)
echo 'except-interface=vbr0' | sudo tee /etc/dnsmasq.d/vbr0

cat <<EOF | sudo tee sudo /etc/dnsmasq.conf
listen-address=172.18.0.1
bind-interfaces
dhcp-range=172.18.0.10,172.18.0.250,2m
EOF

# There is a known issue with this plugin where containers fail to start
# if set to start automatically https://github.com/devplayer0/docker-net-dhcp/issues/23
# Might be a fix here: https://github.com/devplayer0/docker-net-dhcp/pull/34
sudo docker plugin install ghcr.io/devplayer0/docker-net-dhcp:0.1.4
sudo docker network create -d ghcr.io/devplayer0/docker-net-dhcp:0.1.4 --ipam-driver null -o bridge=vbr0 potato

sudo docker run --rm --name web --network potato

FUCK NO OFFLINE

```

libvirt network: basic bridge to vbr0

```xml
<interface type="bridge">
  <mac address="52:54:00:fb:fa:20"/>
  <source bridge="vbr0"/>
  <model type="virtio"/>
  <address type="pci" domain="0x0000" bus="0x01" slot="0x00" function="0x0"/>
</interface>
``````

Here's an attempt with ipvlan

```shell
sudo docker network create -d ipvlan --subnet=10.0.2.0/24 --gateway=10.0.2.1 -o ipvlan_mode=l2 -o parent=enp0s3 tvlan

sudo docker run --rm -d --name web --network tvlan nginx

```

sudo ip link set vbr0 promisc on


## HOLY CRAP THIS WORKED

```shell

{
    "cniVersion": "0.3.1",
    "name": "wow",
    "plugins": [{
    "type": "bridge",
    "bridge": "wowbr0",
    "isGateway": true,
    "isDefaultGateway": true,
    "ipMasq": true,
    "hairpinMode": true,
    "ipam": {"type":"dhcp"}
}]
}

```
