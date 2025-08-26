<h1> linux-virt-win-gpu </h1>

A concise, hands-on guide for setting up a Windows virtual machine (VM) on a Linux host using QEMU/KVM with GPU passthrough.

This guide focuses on the complete setup, including VM configuration and related software, to enable seamless integration between your Linux host and the VM.

<h2> Table of Contents </h2>

- [Prerequisites](#prerequisites)
- [Checking Virtualization Support](#checking-virtualization-support)
- [QEMU/KVM Installation](#qemukvm-installation)
  - [Install the necessary packages](#install-the-necessary-packages)
  - [Enabling the libvirt Daemon](#enabling-the-libvirt-daemon)
    - [Enable Monolithic Daemon (not recommended)](#enable-monolithic-daemon-not-recommended)
    - [Use Modular Daemons (recommended)](#use-modular-daemons-recommended)
- [Enabling Nested Virtualization (Optional)](#enabling-nested-virtualization-optional)
  - [For the Current Session](#for-the-current-session)
  - [Persistent Nested Virtualization](#persistent-nested-virtualization)
- [IOMMU \& VFIO Binding](#iommu--vfio-binding)
  - [Enabling VFIO in supergfxctl](#enabling-vfio-in-supergfxctl)
- [VM Configuration and Installation](#vm-configuration-and-installation)
- [Looking Glass Installation and Configuration](#looking-glass-installation-and-configuration)
  - [Ensuring a Display for Looking Glass](#ensuring-a-display-for-looking-glass)
- [Additional VM Configuration Improvements](#additional-vm-configuration-improvements)
  - [CPU Pinning](#cpu-pinning)
  - [Hardware Info (SMBIOS)](#hardware-info-smbios)
  - [CPU Configuration Options](#cpu-configuration-options)
  - [I/O Threads and Dedicated vCPUs for I/O](#io-threads-and-dedicated-vcpus-for-io)
- [Troubleshooting](#troubleshooting)
  - [Windows “Unsupported CPU” Bluescreen After Update](#windows-unsupported-cpu-bluescreen-after-update)

## Prerequisites

Before you begin, ensure the following requirements are met:

- **Virtualization** is enabled in your BIOS/UEFI settings.
- **IOMMU** (Input-Output Memory Management Unit) is enabled in BIOS/UEFI.
- You have **two GPUs** (e.g., one integrated GPU and one dedicated GPU).
- Your kernel includes the **KVM module**.
- Your hardware supports IOMMU

_Note: GPU passthrough is technically possible with only one dedicated GPU, but this requires stopping the graphical session on the host system, which is generally impractical._

## Checking Virtualization Support

To verify that your system supports virtualization, run:

```bash
lscpu | grep -i virtualization
```

> Note: The word "virtualization" may appear translated, depending on your system language.

## QEMU/KVM Installation

Thanks to [@tatumroaquin's Guide](https://gist.github.com/tatumroaquin/c6464e1ccaef40fd098a4f31db61ab22) for providing an excellent resource on kernel virtualization setup and installation. Refer to that guide for further details or additional topics that may be relevant to your environment.

### Install the necessary packages

[Arch Guide](QUEMU_KVM_INSTALL_ARCH.md)  
[Fedora Guide](QUEMU_KVM_INSTALL_ARCH.md)

### Enabling the libvirt Daemon

Check the [libvirt documentation](https://libvirt.org/daemons.html) in case the steps for enabling the daemon have changed.

#### Enable Monolithic Daemon (not recommended)

```bash
sudo systemctl enable libvirtd.service
```

#### Use Modular Daemons (recommended)

Refer to the section "Switching to modular daemons" in the libvirt documentation for up-to-date instructions.

## Enabling Nested Virtualization (Optional)

Nested virtualization allows you to run virtual machines inside other virtual machines. This is optional and only needed if you plan to use nested VMs. The following steps are from the [offical Fedora Docs](https://docs.fedoraproject.org/en-US/quick-docs/using-nested-virtualization-in-kvm)

### For the Current Session

With the following commands nested virtualization is enabled until the host is rebooted.

**Intel:**
```bash
sudo modprobe -r kvm_intel
sudo modprobe kvm_intel nested=1
```

**AMD:**
```bash
sudo modprobe -r kvm_amd
sudo modprobe kvm_amd nested=1
```

### Persistent Nested Virtualization

To enable nested virtualization permanently add the following line to the `/etc/modprobe.d/kvm.conf`.

**Intel:**
```bash
options kvm_intel nested=1
```

**AMD:**
```bash
options kvm_amd nested=1
```

## IOMMU & VFIO Binding

To pass a GPU through to a virtual machine, it must be bound to **VFIO**. For this to work, the kernel modules for your GPU driver must **not** attach to the GPU. Instead, VFIO must claim your GPU first.

_For a permanent VFIO binding, refer to the [Arch Wiki guide on setting up IOMMU](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF#Setting_up_IOMMU)._

An quick approach, which also allows switching between using the GPU in your VM or on your Linux host, is to use **supergfxctl**.

Both KDE Plasma and GNOME have extensions that communicate with the `supergfxd` daemon, making it easy to switch between modes.

> **Note:** In my experience, switching from *Hybrid* mode (GPU used by Linux) to VFIO is not always stable. It typically requires logging out, and in some cases, the NVIDIA driver kernel module fails to unload. This can cause the daemon to hang in an unkillable state, preventing system shutdown and requiring a forced restart. In practice, a full reboot is often more reliable.  
> If you need to switch frequently, a more stable and robust solution (or a configuration adjustment)  should be evaluated. For occasional use, setting VFIO once via this tool is also a viable option.

You can find build and installation instructions in the [supergfxctl repository](https://gitlab.com/asus-linux/supergfxctl).  
_Arch Linux users can install it from the AUR._

### Enabling VFIO in supergfxctl

After installation, create or edit `/etc/supergfxd.conf` and set the following properties:

```json
{
  "mode": "Hybrid",
  "vfio_enable": false,
  "vfio_save": true
}
```

In my configuration, the default mode is set to "Hybrid", with VFIO binding not being preserved across reboots. Other configuration options can be found in the [source code](https://gitlab.com/asus-linux/supergfxctl/-/blob/main/src/config.rs?ref_type=heads). While they are documented to some extent in the code, fine-tuning them may help address the stability issues mentioned above.

This tool makes it straightforward to set up VFIO or switch between modes. However, if switching is unnecessary, a permanent VFIO binding is generally the cleaner approach. For frequent switching, a more stable long-term solution or configuration refinement should be evaluated.

## VM Configuration and Installation

Follow [the guide for the initial VM setup and installation](VM_CONFIG_INSTALL.md).

## Looking Glass Installation and Configuration

> **Disclaimer:** The **Looking Glass Host** version on Windows **must match** the **Looking Glass Client** version on Linux.  
> To avoid compatibility issues, it is recommended to first update the Windows host version if a newer release is available before installing or updating the Linux client.

**Looking Glass** is a low-latency KVM frame relay solution that allows you to view and interact with a Windows virtual machine’s display directly from your Linux host without a physical monitor connected to the VM’s GPU.

---

<h3> 1. Download and Install on Windows (Host) </h3>
1. Visit the official [Looking Glass Downloads](https://looking-glass.io/downloads) page.
2. Download the latest **Windows Host** release (matching the version you will use on Linux).
3. Extract the archive and run the installer.
4. Ensure the host application is set to start with Windows under services.msc.

---

<h3> 2. Install on Linux (Client) </h3>

**Arch Linux (AUR)**  
- Install the stable package (not the `-git` version) to avoid frequent updates requiring a matching bleeding-edge Windows host build:  
  ```bash
  yay -S looking-glass-client
  ```

**Other Distributions**  
- Follow the official [Looking Glass Build Instructions](https://looking-glass.io/docs/B7/install_client/) to compile from source.

---

<h3> 3. Client Configuration on Linux </h3>
The Linux client can be configured via `/etc/looking-glass-client.ini`.  
Full configuration options are documented here: [Looking Glass Client Configuration](https://looking-glass.io/docs/stable/usage/).

Example configuration:

```ini
[win]
noScreensaver=yes
jitRender=yes
maximize=yes

[egl]
doubleBuffer=yes

[opengl]
vsync=yes

[input]
autoCapture=yes
grabKeyboardOnFocus=yes
mouseSens=3
rawMouse=yes
```

---

<h3> 4. Install the Required Kernel Module </h3>
Looking Glass now uses the **`kvmfr`** kernel module for shared memory.  
The older method of creating a shared memory file manually is **no longer recommended**.  
Follow the official documentation for installation: [KVMFR Module Setup](https://looking-glass.io/docs/stable/ivshmem_kvmfr).

---

<h3> 5. QEMU XML Configuration Adjustments </h3>

Compare for possible changes with the [offical documentation](https://looking-glass.io/docs/stable/ivshmem_kvmfr/#libvirt).

**Namespace Requirement**  
To use `<qemu:commandline>` in libvirt XML, ensure the `<domain>` element (first line) includes the QEMU namespace:  
```xml
<domain type='kvm' xmlns:qemu='http://libvirt.org/schemas/domain/qemu/1.0'>
```

**Command Line Arguments for kvmfr** 

Add inside `<devices>`:

```xml
<qemu:commandline>
  <qemu:arg value="-device"/>
  <qemu:arg value="{'driver':'ivshmem-plain','id':'shmem0','memdev':'looking-glass'}"/>
  <qemu:arg value="-object"/>
  <qemu:arg value="{'qom-type':'memory-backend-file','id':'looking-glass','mem-path':'/dev/kvmfr0','size':134217728,'share':true}"/>
</qemu:commandline>
```

**Setting the `kvmfr0` Size**  
- The shared memory size is defined in the QEMU XML (see below) in **bytes**.  
- For example, `134217728` bytes = **128 MiB**.  
- Adjust this value if you need more frame buffer space (e.g., higher resolutions or multiple monitors).

More details about the calculcation of the required size is described [here](https://looking-glass.io/docs/stable/ivshmem_kvmfr/#libvirt)

**Virtio Keyboard and Mouse for Low-Latency Input**  
Still inside `<devices>`, add:

```xml
<input type="keyboard" bus="virtio">
  <address type="pci" domain="0x0000" bus="0x07" slot="0x00" function="0x0"/>
</input>

<input type="mouse" bus="virtio">
  <address type="pci" domain="0x0000" bus="0x08" slot="0x00" function="0x0"/>
</input>
```

These provide **direct, low-latency input devices** to the VM, bypassing some of the limitations of USB passthrough and ensuring smooth control when using Looking Glass.


**Disable Memory Ballooning**  
```xml
<memballoon model="none"/>
```

---

### Ensuring a Display for Looking Glass

Looking Glass requires that the Windows VM’s GPU has an **active display** connected — without it, the host OS may not output a usable signal.

**Options to provide a display signal:**

1. **HDMI Dummy Plug**  
   - Plug a small HDMI dummy adapter into the GPU’s output.  
   - This tricks Windows into thinking a monitor is connected, allowing Looking Glass to capture the display even without a physical screen.

2. **Dual-Cable Fallback to Your Main Monitor**  
   - Connect your main monitor to the VM’s GPU with a **second HDMI (or DisplayPort) cable** in addition to its usual connection to your Linux host GPU.  
   - **Benefits:**
     - You can easily **fall back** to direct monitor output if you accidentally update the Linux client and it becomes incompatible with the Windows host.  
       In this case, simply switch your monitor’s input source to the VM’s GPU — all input devices will still work because Looking Glass uses **SPICE** for input, so you can control the VM and update the host without needing a working Looking Glass session.
     - Ensures Windows always detects the correct resolution and refresh rate (Hz) for optimal performance.

3. **Virtual Display Driver**  
   - Instead of physical adapters, you can install a virtual display device in Windows.  
   - This creates a “dummy” monitor entirely in software, with customizable resolution and refresh rate.  
   - See the [Virtual Display Driver project on GitHub](https://github.com/VirtualDrivers/Virtual-Display-Driver) for installation and usage instructions.

---

> **Disclaimer:** Regardless of which option you choose, it’s often best to configure Windows so that the connected display (dummy or physical) is the **only active monitor**. This prevents applications or dialogs from spawning on an unseen display.

> **Pro Tip:** When using Looking Glass, enable **input capture** so that all keyboard shortcuts are passed directly to the VM.  
> This allows you to use Windows’ built-in shortcut **`Win + Alt + Arrow Keys`** to move application windows between monitors — even if one of those monitors is not currently visible.

## Additional VM Configuration Improvements

### CPU Pinning
CPU pinning assigns specific physical CPU cores (vCPUs) from the host to the VM. This can reduce context switching, improve cache locality, and deliver more consistent performance — especially for gaming or real‑time workloads.

It is recommended to watch or read a proper guide for a more in depth explanation it would be too much to cover it here.

Example:
```xml
<vcpu placement="static">4</vcpu>
<cputune>
  <vcpupin vcpu="0" cpuset="2"/>
  <vcpupin vcpu="1" cpuset="6"/>
  <vcpupin vcpu="2" cpuset="3"/>
  <vcpupin vcpu="3" cpuset="7"/>
</cputune>
```

### Hardware Info (SMBIOS)
Adding realistic hardware information to the VM can help Windows retain activation, especially if you are booting from an existing installation.

Example:
```xml
<sysinfo type="smbios">
  <bios>
    <entry name="vendor">American Megatrends Inc.</entry>
    <entry name="version">5302</entry>
  </bios>
  <system>
    <entry name="manufacturer">ASUSTeK COMPUTER INC.</entry>
    <entry name="product">ROG STRIX B450-F GAMING</entry>
    <entry name="version">Rev 1.xx</entry>
    <entry name="serial">1234567890</entry>
  </system>
  <baseBoard>
    <entry name="manufacturer">ASUSTeK COMPUTER INC.</entry>
    <entry name="product">ROG STRIX B450-F GAMING</entry>
    <entry name="version">Rev 1.xx</entry>
    <entry name="serial">1234567890</entry>
  </baseBoard>
  <chassis>
    <entry name="manufacturer">ASUSTeK COMPUTER INC.</entry>
    <entry name="version">1.0</entry>
    <entry name="serial">1234567890</entry>
    <entry name="asset">Default string</entry>
    <entry name="sku">SKU12345</entry>
  </chassis>
</sysinfo>
```

You can retrieve your real hardware info from the host using:

```bash
sudo dmidecode -t bios -t system -t baseboard -t chassis
```

### CPU Configuration Options

```xml
<cpu mode="host-passthrough" check="none" migratable="on">
  <topology sockets="X" dies="X" clusters="1" cores="X" threads="X"/>
  <cache mode="passthrough"/>
  <feature policy="disable" name="hypervisor"/>
  <feature policy="require" name="invtsc"/>
  <feature policy="require" name="topoext"/>
</cpu>
```

- cache mode="passthrough" — Passes through host CPU cache info to the guest.
- feature policy="disable" name="hypervisor" — Hides the “hypervisor” CPUID bit to reduce VM detection. (Might be needed for UNSUPPORTED_CPU bluescreen. See _Troubleshooting section_)
- feature policy="require" name="invtsc" — Ensures invariant TSC (stable time source) is available.
- feature policy="require" name="topoext" — AMD-specific topology extensions for better CPU scheduling.

### I/O Threads and Dedicated vCPUs for I/O

Assigning dedicated I/O threads can improve responsiveness:

```xml
<iothreads>1</iothreads>
<cputune>
  <iothreadpin iothread="1" cpuset="8"/>
</cputune>
```

- iothreads — Number of I/O threads for the VM.
- iothreadpin — Pins an I/O thread to a specific host CPU core.

## Troubleshooting

### Windows “Unsupported CPU” Bluescreen After Update
  
After a Windows update, the VM fails to boot and shows a **blue screen** with the message `UNSUPPORTED_PROCESSOR` even though the physical CPU is officially supported by Windows 11.

**Solutions:**

1. **Switch CPU Model**  
    
    Change from `host-passthrough` to a model like `EPYC-Rome-v1` in your VM XML:  
   
   ```xml
   <cpu mode="custom" match="exact" check="none">
     <model fallback="allow">EPYC-Rome-v1</model>
     ...
   </cpu>
   ```

   _Your can change this also via the UI to see a list of available CPU models._

2. **Keep Host-Passtrough but disable the hypervisor flag (better option)**  
   
   ```xml
    <cpu mode="host-passthrough" check="none" migratable="on">
        <topology sockets="1" dies="1" clusters="1" cores="6" threads="2"/>
        <feature policy="disable" name="hypervisor"/>
        ...
    </cpu>
   ```