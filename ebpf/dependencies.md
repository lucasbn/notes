### `fatal error: 'bpf/bpf_helpers.h' file not found`

This means that you don't have the BPF Linux headers installed on your machine.
These headers define the interface between user space programs and the kernel.
This includes declarations of functions, constants, data structures and macros.

_Usually_, you can install these with:

```
apt install linux-headers-$(uname -r)
```

However, if you are running a custom linux kernel then you won't be able to do
this because the headers for your kernel will (probably) be slightly different
and these won't be in an apt package.

You can see which headers you have installed by inspecting:

```
root@ubuntu-qemu:~# ls /usr/include/linux/ | grep bpf
bpf.h
bpf_common.h
bpf_perf_event.h
```

Here, we do not have a `bpf/` directory or a `bpf_helpers.h`. To install those,
you should first run (inside your linux source code directory):

```
make -j$(nproc) INSTALL_HDR_PATH=headers headers_install
```