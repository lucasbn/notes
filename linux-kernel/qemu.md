1. Ensure you have the right `.config` when compiling the kernel
	- `make menuconfig`: save and exit
	- `make kvm_guest.config`

1. Create a file system image:

TODO: Add guide here on how to create your own custom image with lots of things preinstalled

```
mkdir image && cd image
wget https://raw.githubusercontent.com/google/syzkaller/master/tools/create-image.sh -O create-image.sh
chmod +x create-image.sh
./create-image.sh
```

2. Create a `run.sh` script:

```
qemu-system-aarch64 \
  -machine virt \
  -cpu cortex-a57 \
  -smp 2 \
  -m 2048 \
  -kernel arch/arm64/boot/Image \
  -append "root=/dev/vda rw console=ttyAMA0" \
  -drive file=images/rootfs.ext4,format=raw,if=virtio \
  -nographic \
  -netdev tap,id=net0,ifname=tap0,script=no,downscript=no \
  -device virtio-net-pci,netdev=net0
```

And a `setup-network.sh` script (change interface + gateway + ip accordingly):

```
ip tuntap add dev tap0 mode tap user $USER
ip link set tap0 up

ip link add name br0 type bridge
ip link set br0 up

ip link set ens160 master br0
ip link set tap0 master br0

ip addr flush dev ens160
ip addr add 192.168.228.138/24 dev br0
ip link set br0 up
ip route add default via 192.168.228.2
```
