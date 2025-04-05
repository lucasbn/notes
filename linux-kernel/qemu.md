1. Create a file system image:

```
mkdir image && cd image
wget https://raw.githubusercontent.com/google/syzkaller/master/tools/create-image.sh -O create-image.sh
chmod +x create-image.sh
./create-image.sh
```

2. Create a `run.sh` script:

```
qemu-system-aarch64 \
        -m 2G \
        -smp 2 \
	-machine virt \
        -kernel $1/arch/arm64/boot/Image \
        -append "console=ttyS0 root=/dev/sda earlyprintk=serial net.ifnames=0 nokaslr" \
        -drive file=$2/bullseye.img,format=raw \
        -net user,host=10.0.2.10,hostfwd=tcp:127.0.0.1:10021-:22 \
        -net nic,model=e1000 \
        -nographic \
        -pidfile vm.pid \
        2>&1 | tee vm.log
```
