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
