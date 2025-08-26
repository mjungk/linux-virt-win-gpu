# QUEMU/KVM Installation on Arch

Always check the official Arch documentation for QEMU for troubleshooting and to verify if there are any changes to the installation process: [QEMU - ArchWiki](https://wiki.archlinux.org/title/QEMU)

---

### Install QEMU, libvirt, and Related Packages

Install the required packages using `pacman`:

```bash
sudo pacman -S qemu-full qemu-img libvirt virt-install virt-manager virt-viewer edk2-ovmf swtpm
```

#### Package Descriptions

- **qemu-full**: Complete QEMU package, providing full system emulation and virtualization capabilities.
- **qemu-img**: Tool for creating, converting, and modifying virtual disk images.
- **libvirt**: Toolkit to manage virtualization platforms; provides a daemon and API for VM management.
- **virt-install**: Command-line tool to create new virtual machines using libvirt.
- **virt-manager**: Graphical user interface for managing virtual machines via libvirt.
- **virt-viewer**: Lightweight viewer for connecting to virtual machine graphical consoles.
- **edk2-ovmf**: UEFI firmware for virtual machines, required for UEFI boot support.
- **swtpm**: Software TPM emulator, enables TPM support for virtual machines.