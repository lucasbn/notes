1. Creating a custom filesystem image

First, you'll need to install some dependencies:

```
sudo apt update
sudo apt install debootstrap qemu-user-static binfmt-support systemd-container=
```

Create a directory to house the filesystem:

```
mkdir -p rootfs-arm64
```

Use `debootstrap` to create a minimal Ubuntu filesystem:

```
sudo debootstrap --arch=arm64 --foreign jammy ~/rootfs-arm64 http://ports.ubuntu.com
```

Copy the `qemu-aarch64-static` binary to the rootfs so that we can run ARM64
binaries when we chroot into it:

```
sudo cp /usr/bin/qemu-aarch64-static ~/rootfs-arm64/usr/bin/
```

Now chroot into the filesystem directory:

```
sudo chroot ~/rootfs-arm64 /usr/bin/qemu-aarch64-static /bin/bash
```

Run `debootstrap`:

```
/debootstrap/debootstrap --second-stage
```

Configure the hostname and root password:

```
echo "ubuntu-qemu" > /etc/hostname
echo "root:root" | chpasswd
```

Install whichever packages you'd like:

```
apt update
apt install openssh-server sudo net-tools iproute2 systemd vim curl wget python3 golang git build-essential clang software-properties-common -y
add-apt-repository universe
apt update
apt install clang
```

Enable SSH root login:

```
sed -i 's/^#PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config
```

To exit the chroot, just type:

```
exit
```

Now create an empty disk image with however much space you'd like (e.g 2GB) and
format it as ext4:

```
dd if=/dev/zero of=rootfs.ext4 bs=1M count=2048
mkfs.ext4 rootfs.ext4
```

Mount the image, copy the root filesystem into it and then unmount it:

```
mkdir /mnt/tmpmnt
sudo mount -o loop rootfs.ext4 /mnt/tmpmnt
sudo cp -a ~/rootfs-arm64/* /mnt/tmpmnt/
sudo umount /mnt/tmpmnt
```

The `rootfs.ext4` image can now be used by qemu (see below).


2. Setting up the networking in QEMU

You can add the QEMU machine to your host network by running the following
script:

```
# Replace this with the identifier of you main network interface
main_interface=ens160

# Replace this with the current IP address of you main interface
ip_address=192.168.228.138

# Run `ip route` and replace this with your current default gateway
default_gateway=192.168.228.2

ip tuntap add dev tap0 mode tap user $USER
ip link set tap0 up

ip link add name br0 type bridge
ip link set br0 up

ip link set $main_interface master br0
ip link set tap0 master br0

ip addr flush dev $main_interface
ip addr add $ip_address/24 dev br0
ip link set br0 up
ip route add default via $default_gateway
```

3. Create a `run.sh` script:

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

```
#!/bin/bash

# Check for --gdb flag
EXTRA_QEMU_ARGS=""
if [[ "$1" == "--gdb" ]]; then
    EXTRA_QEMU_ARGS="-s -S"
fi

qemu-system-x86_64 \
	-m 2G \
	-smp 2 \
	$EXTRA_QEMU_ARGS \
	-kernel arch/x86/boot/bzImage \
        -append "console=ttyS0 root=/dev/sda rw earlyprintk=serial net.ifnames=0 nokaslr" \
        -drive file=image/rootfs.ext4,format=raw \
  	-netdev user,id=net0,hostfwd=tcp::2222-:22 \
  	-device virtio-net-pci,netdev=net0 \
	-nographic \
	-enable-kvm \
        -pidfile vm.pid \
        2>&1 | tee vm.log
```


4. When the kernel loads:

```
systemctl enable systemd-networkd
systemctl start systemd-networkd
mkdir -p /etc/systemd/network
cat <<EOF > /etc/systemd/network/20-eth0.network
[Match]
Name=eth0

[Network]
DHCP=yes
EOF
```

Then reboot and the `ip addr` should show that eth0 is up and has an address