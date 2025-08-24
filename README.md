# ebpf-netfilter-redirect
Linux Kernel customizations that allow sandboxed eBPF programs attached to the traffic control module to access Netfilter’s libnetfilter_queue features and reinject ingress packets or requeue egress packets returned from a userspace application.

This patch of the OpenWrt 6.6.73 Linux Kernel is made publicly available in order to meet the GPLv2 license requirements for a custom Linux kernel used in commercial applications. By incorporating these changes, we ensure that our modifications and their distribution comply with the terms of the license, preserving the free and open-source nature of the kernel for all users.

## Patch Summary

When a packet is encountered that requires userspace inspection, a ‘tc’ attached eBPF program can optionally redirect it instead of allowing it to continue as normal. In cases of eBPF redirection a kernel customization is applied that performs a redirect to one of two (custom) Netfilter hooks. By default eBPF programs can only redirect to other interfaces. Thus, this patch opens a new pathway for eBPF programs to forward packets to userspace via Netfilter.

Placement of the new Netfilter hooks and their associated reinjection functions allow eBPF programs to forward selected packets to userspace via a kernel module leveraging libnetfilter_queue. Packets that are allowed by userspace can be reinjected (ingress packets) or requeued (egress packets) by the kernel module so they can complete their original transit path. The most common use case for this patch is to allow userspace applications to inspect new connections while operating “out of view” of other system components. 

In order to make this patch useful some other components are required.

### A Sandboxed eBPF program:

The program must be attached to an interface and return either TC_ACT_REDIRECT or TC_ACT_OK. There are ample guides demonstrating how to build and attach these programs. When TC_ACT_REDIRECT is returned, the packet will be redirected to one of the custom Netfilter hooks. The eBPF program can be as complex as needed but must follow eBPF constraints.  The ‘tc’ utility is used to attach the eBPF to any desired interface.

### A Kernel Module:

The patch introduces two new Netfilter hooks: 
- NF_INET_REDIRECT_INGRESS
- NF_INET_REDIRECT_EGRESS

The kernel module is tasked with handling packets redirected from eBPF via the custom hooks. Within the kernel module, libnetfilter_queue features are leveraged to copy packets to userspace. The kernel maintains a handle to the packet while a userspace application makes a determination. Netfilter kernel modules are fairly boilerplate and many examples are available.

### Kernel options:
The most common general purpose distros likely include all required kernel features for Netfilter and eBPF. However, embedded systems like OpenWrt will certainly need to have some kernel features enabled and the kernel compiled.

### A userspace application:
When inserted, the kernel module will configure queues that allow packet copies to be sent to userspace. Thus, a userspace application must listen on any configured queues and return either NF_ACCEPT or NF_DROP.
