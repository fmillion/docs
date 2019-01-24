# Running Hyper-V inside of a KVM/QEMU virtual machine

This guide explains how to get Microsoft's Hyper-V hypervisor to successfully run inside a QEMU container running with KVM.

## Common problems

If you are using a virtualization framework that uses KVM (this includes Proxmox, oVirt, and plain qemu), and you have tried to run a Windows virtual machine with Hyper-V enabled, you will often find that your efforts are unsuccessful. Commonly experienced issues that this guide will help you resolve are:

* Hyper-V just doesn't work. Trying to boot a VM throws errors.
* Hyper-V seems to work, but networking stops working - your Hyper-V host VM cannot communicate over the network at all.
* After enabling Hyper-V, your VM enters a boot loop - it crashes immediately upon trying to boot and throws you into Automatic Repair.

The settings outlined in this guide will resolve these and possibly other issues that keep you from getting Hyper-V running inside KVM.

## HOWTO

### Requirements

In order to be able able to do this at all, your host system's CPU must support SLAT (second-level address translation), which Intel calls EPT (enhanced page tables). On Intel, any processor in the Westmere series (Xeon L|E|X56xx series) and later should have the necessary support for EPT, with the exception of some low-end consumer CPUs in the Celeron and Pentium lines. For AMD, Phenom II or later chips should be supported.

### Make sure nested KVM is enabled

In order to run *any* hypervisor operating system you must ensure that nested virtualization is enabled in your host operating system. This will be similar on any Linux OS, including Proxmox and oVirt. 
