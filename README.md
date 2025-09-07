<p align="right">
  <a href="README.pt.md">
    <img src="https://flagcdn.com/br.svg" width="20"> Português
  </a>
</p>


# Radxa X4 SR-IOV iGPU Activation and Looking Glass

This project details the process of activating **SR-IOV (Single-Root I/O Virtualization)** on the **iGPU (integrated GPU)** of the **Radxa X4** board. The goal is to create a **VF (Virtual Function)** to perform a **passthrough** of this GPU to a **Windows** virtual machine (VM). The VM is then used to run games and graphical applications, with the video output captured and displayed on the Linux host machine using the **Looking Glass** tool.

## Why this Project?

Activating SR-IOV on integrated GPUs is a technical challenge, and documentation for the Radxa X4 is scarce. This project serves as a detailed, step-by-step guide to help other users with the same setup replicate the process and leverage the graphical performance of the iGPU in a VM without needing a second dedicated graphics card. Using a GPU in a VM also helps decrease the load on the processor, making it a more practical solution for **VDI (Virtual Desktop Infrastructure)** applications.

## Key Components

  * **Radxa X4:** The motherboard used as the host.
  * **SR-IOV:** A virtualization feature that allows a single PCIe device (the GPU) to be shared among multiple VMs as if they were separate devices.
  * **Windows VM:** The virtual machine where the GPU's VF will be used.
  * **GPU Passthrough:** The process of giving a VM direct and exclusive access to a GPU's VF.
  * **Looking Glass:** A low-latency tool that captures the VM's framebuffer and displays it on the host, allowing the VM to be used as if it were a native application on the Linux desktop.

## Prerequisites

To follow this guide, you will need the following items:

  * **Radxa X4** board (with an updated BIOS that supports SR-IOV).
  * A **Linux** operating system (recommended distributions: Arch Linux or Ubuntu Server). This project was tested on a distribution called **Cachy OS**, which is based on Arch.
  * Virtualization tools like **QEMU/KVM** and **libvirt**.
  * Basic knowledge of the Linux command line.
  * The **Looking Glass** tool (compiled from source or installed via a repository).

## Installation and Configuration Guide

### 1\. BIOS Configuration

The first step is to ensure that SR-IOV is enabled in your Radxa X4's BIOS. Reboot the board and enter the BIOS. Look for and enable the following options:

  * **Intel VT-d**


Save the settings and reboot.

Additionally, for the passthrough to work correctly, the **IntelGopDriver.efi** driver file is required. The only way to obtain it is by extracting it directly from your BIOS. Use a tool like `UEFItool` to extract the driver and place it in a location accessible to your VM.

### 2\. Linux Host Configuration

1.  **Enable SR-IOV in the Kernel:**
    Edit the `/etc/default/grub` file on the `GRUB_CMDLINE_LINUX_DEFAULT` line.

    ```bash
    GRUB_CMDLINE_LINUX_DEFAULT='intel_iommu=on i915.enable_guc=3 i915.max_vfs=7 i915.enable_fbc=0 apparmor=1 security=apparmor'
    ```

    **Parameter Explanations:**

      * **`intel_iommu=on`**: Activates the **IOMMU (Input/Output Memory Management Unit)** for Intel processors, which is essential for **PCI Passthrough**.
      * **`i915.enable_guc=3`**: Enables the use of the **GuC (Graphics micro-controller)** firmware for the Intel i915 GPU, crucial for virtualization. A value of `3` activates task scheduling and firmware loading.
      * **`i915.max_vfs=7`**: Specifies that the GPU can create 7 **Virtual Functions**, allowing each one to be assigned to a different VM.
      * **`i915.enable_fbc=0`**: Disables **Frame Buffer Compression (FBC)** to prevent compatibility and stability issues during passthrough.
      * **`apparmor=1 security=apparmor`**: Enables the **AppArmor** security module in the kernel to enforce access control policies.

    Update GRUB:

    ```bash
    sudo update-grub
    ```

2.  **Verify IOMMU and VFs:**
    After rebooting, check if IOMMU is active and if the GPU has been partitioned into VFs.

    ```bash
    dmesg | grep -i iommu
    lspci -nnk
    ```

    You should see your GPU listed with the `iommu group` and the VFs that have been created.

3.  **Configure the VM in `libvirt`:**
    The `hostdev` section in the XML is the most critical part, as it configures the **passthrough** of the GPU's **VF** to the VM.

      * **`hostdev` Device**: Adds a `hostdev` device for the **passthrough** of a PCI device from the host to the VM.
      * **Device Address**: The address (`address domain="0x0000" bus="0x00" slot="0x02" function="0x2"`) is the specific address of the iGPU's **VF**, obtained using the `lspci` command.
      * **`vfio` Driver**: The **VFIO** driver is used to allow the VM secure, direct access to the device.
      * **GPU BIOS ROM (`IntelGopDriver.efi`)**: The line `rom bar="on" file="/home/${user}/Downloads/IntelGopDriver.efi"` injects the driver file extracted from your BIOS, which is essential for the VM to boot with the GPU.

-----

## VM Configuration Details (`.xml`)

This project uses an XML configuration file for the virtual machine, which contains the specific settings required for GPU passthrough and Looking Glass integration. The most important sections are:

### GPU Passthrough and BIOS Injection

The `<devices>` section defines the GPU passthrough. The `vfio` driver is used to isolate the **Virtual Function (VF)** of the GPU from the host, and the VF's address is specified.

```xml
<hostdev mode="subsystem" type="pci" managed="yes">
  <driver name="vfio"/>
  <source>
    <address domain="0x0000" bus="0x00" slot="0x02" function="0x2"/>
  </source>
  <rom bar="on" file="/home/${user}/Downloads/IntelGopDriver.efi"/>
  <address type="pci" domain="0x0000" bus="0x06" slot="0x00" function="0x0"/>
</hostdev>
```

It is crucial to note the `<rom>` line, which injects the `IntelGopDriver.efi` file into the VM. This file is extracted from your BIOS and is essential for the GPU passthrough to work correctly.

### Looking Glass Configuration

For communication between the VM and the host, `ivshmem` is configured via the QEMU command line.

```xml
<qemu:commandline>
  <qemu:arg value="-device"/>
  <qemu:arg value="ivshmem-plain,memdev=ivshmem"/>
  <qemu:arg value="-object"/>
  <qemu:arg value="memory-backend-file,id=ivshmem,share=on,mem-path=/dev/kvmfr0,size=128M"/>
</qemu:commandline>
```

This configuration creates a shared memory space (`/dev/kvmfr0`) that the Looking Glass client on the host uses to read the VM's framebuffer with low latency.

### Disabling the Virtual Video Device

To ensure the VM uses the real GPU and not the virtual one, the virtual video card is disabled.

```xml
<video>
  <model type="none"/>
</video>
```

### 3\. Running Looking Glass

1.  **On the Windows VM:**
    Install the GPU drivers and the Looking Glass client (`looking-glass-client.exe`). Make sure the VM is configured to have Looking Glass active on startup.

2.  **On the Linux Host:**
    Execute `looking-glass-client` to connect to the VM.

    ```bash
    looking-glass-client -f /dev/kvmfr0
    ```

    The client will connect to the VM and display its screen in a window on your Linux desktop, allowing you to interact with the VM as if it were a native application.

## Contribution

Contributions are always welcome\! If you find an error or have a suggestion for an improvement, feel free to open an *issue* or submit a *pull request*.

-----