A tap device is a virtual network interface.

A network interface is a component that provides a mechanism for sending and
receiving packets from the network. An ethernet card or Wi-Fi chip is a network
interface. It consists of things like IP addresses, a MAC address and IP routing
rules. You can plug an ethernet cable directly into an ethernet card. 

When you want to send data across the network, you can bind a socket to the 
network interface linux exposes that is associated with this ethernet card. Data 
written to this socket will eventually make its way to this ethernet cable. The
kernel will wrap your data in L2 / L3 headers that correspond to this network
interface.

I think a good way to understand TAP devices is in the context of virtual
machines. You may want to create a virtual machine and connect it to the same
LAN that the host runs on and assign it its own IP from the subnet. You can do
this by creating a TAP device and attaching it to your physical ethernet card.
This is very much like adding a completely new physical ethernet card, except
you can't choose to connect different ethernet cables to the TAP device (because
it isn't real, it's virtual). It shares the ethernet cable with the physical
ethernet card but can be assigned a separate MAC and set of IP addresses.

When you want to send data using a TAP device, you are responsible for doing the
work that the kernel normally does: constructing ethernet headers (and therefore
headers for protocols at higher layers). The TAP device effectively gives you 
raw access to the ethernet cable.

This has a very important implication: you can't use the _all_ of the standard
socket syscalls to send data via a TAP interface. You _can_ use an AF_RAW or
AF_PACKET socket, but not for example a AF_INET socket. However, the easiest way
to understand how to send data via a TAP interface is with the `open()`,
`read()` and `write()` syscalls:

```
// First, open a file corresponding to the tap  interface
int fd = open("/dev/net/tap0", O_RDWR);

// Construct an ethernet frame and write it to the file descriptor
char frame[1500];
write(fd, frame, len);
```

The kernel then checks that the ethernet frame is valid and directs it to the
ethernet cable.

This is how a VM can connect to the host LAN as if it were a separate host
itself - the kernel networking stack in the VM will construct the ethernet frame
and send it to something like QEMU which will write it to the tap fd.

This is not extensive and maybe even slightly incorrect - maybe I'll do more of
a kernel internals deep dive to figure out exactly how this all works.