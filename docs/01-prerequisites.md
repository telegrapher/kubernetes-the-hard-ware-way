# Prerequisites

In this lab you will install the operating system and basic tools.

## One server

Since we plan to use KVM, the CPU should support the virtualization extensions (VT-x on Intel or VMX on AMD).

I will use for this tutorial a machine that I casually have floating around with 16 core and 256 GB of RAM.

## Operating system

Download and install [Debian Stretch](https://www.debian.org/). Since there are plenty of tutorials and great instructions on the Debian website, this is left as an exercise to the reader.

In this tutorial we'll assume a minimalist installation with only the base utilities and the SSH server to access the system. Everything else will be specified in the next steps.

## KVM virtualization

KVM and Xen are the reference hypervisors on Linux, I tend to favor KVM, but Xen wouldn't be a bad choice.

### Install KVM

As root:
```
apt install qemu-kvm libvirt-clients libvirt-daemon-system 
apt install virtinst --no-install-recommends
```
Verify the installation:
```
virsh --connect qemu:///system list --all
```

## Running Commands in Parallel with tmux

[tmux](https://github.com/tmux/tmux/wiki) can be used to run commands on multiple compute instances at the same time. Labs in this tutorial may require running the same commands across multiple compute instances, in those cases consider using tmux and splitting a window into multiple panes with `synchronize-panes` enabled to speed up the provisioning process.

> The use of tmux is optional and not required to complete this tutorial.

![tmux screenshot](images/tmux-screenshot.png)

> Enable `synchronize-panes`: `ctrl+b` then `shift :`. Then type `set synchronize-panes on` at the prompt. To disable synchronization: `set synchronize-panes off`.

Next: [Installing the Client Tools](02-client-tools.md)
