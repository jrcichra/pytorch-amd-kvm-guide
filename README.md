# pytorch-amd-kvm-guide
How to run pytorch with AMD GPU acceleration inside KVM/QEMU. This probably works with other ML libraries such as tensorflow (except for the container portion)
Hopefully my portion is as easy as the VFIO guide, so you can focus on ML, not faster epochs.
There is a `per-boot` thing you have to do (writing a temporary byte to the pci bus). I'll try to make a script for this.

# Grab an ISO
+ I'll be using Ubuntu 20.04.1 as my ML guest OS. 

# VFIO/IOMMU PCI Passthrough
+ The arch wiki has a great guide on how to set up PCI passthrough:
+ https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF
+ Wendell's guide might be more N00B friendly: https://forum.level1techs.com/t/ubuntu-17-04-vfio-pcie-passthrough-kernel-update-4-14-rc1/119639
+ In the end, the expectation is: you should be able to boot a VM with a dedicated AMD GPU

# AMD's Ubuntu Guide
+ https://rocmdocs.amd.com/en/latest/Installation_Guide/Installation-Guide.html#ubuntu
+ Follow this guide, but step 8 will not work. Continue with Step 9, adding ROCm to your PATH

# Patching rocm-dkms
+ We need to make a slight code change. Credits goes to:@GongYiLiao: https://github.com/RadeonOpenCompute/ROCK-Kernel-Driver/issues/100#issuecomment-703012241
+ How I did this:
+ `sudo vi /usr/src/amdgpu-3.8-30/amd/amdkfd/kfd_device.c`
+ Go to line ~563 with: `kfd->pci_atomic_requested = amdgpu_amdkfd_have_atomics_support(kgd);`
+ The next line should be an `if` block based on pci_atomics. Comment out that whole block with `/* */`. See @GongYiLaio's post for a visual if you're not sure
+ We need to tell `dkms` to rebuild the rock kernel driver with our change. To do that:
+ `sudo dkms remove amdgpu -k $(uname -r)` <-- remove amdgpu
+ `sudo dkms autoinstall -k $(uname -r)`   <-- rebuild all kernel modules we have source for that aren't installed (amdgpu)

# Blacklisting amdgpu
+ We need to stop the kernel module `amdgpu` from starting at boot. We need to set a byte in the PCI after the VM is booted each time (cleared by AHCI)
+ `sudo vi /etc/modprobe.d/blacklist.conf`
+ I added `blacklist amdgpu` with a note why at the bottom of the file
+ `sudo poweroff`. A full off is good to clear out your GPU RAM

# PCI - enabling atomics
+ The patch we did to rocm-dkms was half the battle. We still need to set something on the host after every boot of this ML VM (not ideal)
+ Boot the VM. It's probably going to be 800x600 if you have a monitor plugged in. This is okay. Leave it like that
+ Log in to the guest and open a terminal for later
+ On the HOST machine, not the guest, run these commands:
+ `lspci` <- find your graphics card's ID. You should have done something similar when setting up GPU passthrough
+ Take note of your GPU PCI ID, which you passed through. Mine is `0b:00.0`
+ Check on what bits are set for this PCI ID with: `sudo lspci -s <pci-id> -xxx`
+ Note down spot 80: the first digit pair should be `00`. We need to flip this to `40`
+ To do that, run this command: `sudo setpci -v -s <pci-id> 80.b=40`
+ Make sure this worked by doing `sudo lspci -s <pci-id> -xxx` again
+ Now, either SSH into or use the terminal you logged in with on the guest, to run `sudo modprobe amdgpu`. This will load our patched amd gpu driver and it will recognize that `40` pci thing exists and that atomics are supported on your GPU
+ You'll have to do this section every time you boot up the VM (until we script it)

# ROCM/Pytorch container
+ If you don't understand docker containers, don't worry. Think of them as `chroot`'ed environments. If you don't know `chroot`, think of it like you're downloading a directory structure from someone, and you can set `/` to the top directory they sent you. This is only one aspect of docker, but esentially what we're using it for here. You're downloading a precompiled environment with userspace ROCM/pytorch support baked in. Docker doesn't solve the kernel magic above, only userspace
+ Anyway, I built in a fix for the official rocm/pytorch container for gfx803 cards (I have an RX580). If you don't have a card in this family, you can try the base docker image instead of mine: replace `jrcichra/rocm-pytorch-gfx803` with `rocm/pytorch`. If my container is borked and you need gfx803 support. Use the official container and, once launched, manually run the line specified in the `Dockerfile` here: https://github.com/jrcichra/rocm-pytorch-gfx803
+ If you don't have docker, install it with `sudo apt install docker.io`
+ Add your user to the docker group: `sudo gpasswd -a $USER docker`
+ Close out of your terminal and get back in. Validate your priviledge with `docker ps -a`. If that returns without a socket error, you're good
+ `cd` to your machine learning project directory
+ run `sudo docker run -it -v $PWD:/projects --privileged --name pytorch --device=/dev/kfd --device=/dev/dri --group-add video jrcichra/rocm-pytorch-gfx803`
+ This will mount the current directory into `/projects`. You can navigate there and try your pytorch project. It 'should' have GPU acceleration. I checked this with:
```
import torch
if torch.cuda.is_available():
    device = torch.device("cuda:0")
    print("Running on the GPU")
else:
    device = torch.device("cpu")
    print("Running on the CPU")
```
+ I get `Running on the GPU`
+ Feel free to `ctrl+d` inside this container. It will stick around. If you want to go back in at a later time, just run:
+ `docker start pytorch ; docker exec -it pytorch bash` and keep crunching models

Feel free to open an issue if you had any trouble.
