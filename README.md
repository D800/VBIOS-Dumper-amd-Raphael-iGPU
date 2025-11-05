# VBIOS Dumper for AMD Raphael iGPU

Utility for extracting VBIOS from AMD Ryzen 7000 series (Raphael) integrated graphics through ACPI VFCT table for GPU passthrough in Proxmox/QEMU.

## Description

This program extracts VBIOS from the VFCT (VBIOS Function Control Table) in the system ACPI firmware. This method works even when the GPU is already bound to the `vfio-pci` driver, unlike methods that read from `/sys/bus/pci/devices/.../rom`.

### Why VBIOS is needed

VBIOS is required for proper AMD iGPU initialization in virtual machines during GPU passthrough. Without the correct VBIOS file, the guest system cannot use the graphics card.

## Supported Processors

- AMD Ryzen 7000 series (Raphael): 7950X, 7900X, 7800X3D, 7700X, 7600X, etc.
- AMD Ryzen 8000G series (Phoenix/Hawk Point)
- Other AMD processors with integrated RDNA2/RDNA3 graphics


## Installation and Usage

### Compilation

```bash
gcc -o vbios vbios.c
```


### Execution

```bash
sudo ./vbios
```

```
The program will create a file named `vbios_<VendorID>_<DeviceID>.bin`, for example `vbios_1002_164e.bin` for Ryzen 7800X3D.
```


### Verify Output

```bash
# Check file size (typically 128-512 KB for iGPU)
ls -lh vbios_*.bin

# Verify ROM validity (first bytes should contain signature)
hexdump -C vbios_*.bin | head -2
```


## Proxmox Installation

### Copy ROM Files

```bash
# Copy extracted VBIOS
mv vbios_1002_164e.bin /usr/share/kvm/raphael_vbios.bin

# Download AMDGopDriver for audio device
cd /usr/share/kvm/
wget https://github.com/isc30/ryzen-gpu-passthrough-proxmox/raw/main/AMDGopDriver.rom
```


### VM Configuration

Edit `/etc/pve/qemu-server/<VMID>.conf`:

```
hostpci0: 0000:18:00.0,pcie=1,x-vga=1,romfile=raphael_vbios.bin
hostpci1: 0000:18:00.1,pcie=1,romfile=AMDGopDriver.rom
```


### Configure vfio

```bash
echo "options vfio_iommu_type1 allow_unsafe_interrupts=1" >> /etc/modprobe.d/vfio.conf
update-initramfs -u -k all
reboot
```


## VFCT Method Advantages

- Works when GPU is already bound to `vfio-pci`
- No driver unbinding required
- Extracts original VBIOS loaded by UEFI
- Universal for all AMD GPUs with VFCT support


## Requirements

- Linux system with access to `/sys/firmware/acpi/tables/VFCT`
- Root privileges for reading ACPI tables
- Motherboard with UEFI and VFCT table (standard for modern AM5 systems)
- GCC for compilation


## Technical Details

The program parses the VFCT table structure containing:

- ACPI header with table length
- Offset to VBIOS images (`VBIOSImageOffset`)
- Headers for each GPU with PCI coordinates, VendorID/DeviceID
- Binary VBIOS images for each device


## Troubleshooting

If `/sys/firmware/acpi/tables/VFCT` is unavailable, ensure that:

- System is booted in UEFI mode (not Legacy BIOS)
- Integrated graphics is enabled in BIOS
- Table is not corrupted (check `dmesg | grep -i vfct`)


## Credits

Method based on code from [ryzen-gpu-passthrough-proxmox](https://github.com/isc30/ryzen-gpu-passthrough-proxmox) repository.
