# T730-I350 Rig

Technical notes for a home automation build based on the HP T730.

## Hardware

I evaluated several HP thin clients before selecting the HP T730. The main alternatives were the HP T620 Plus and several newer HP thin-client models. The T730 offered the best balance of expandability, availability, and long-term practicality. Newer models were less attractive because they are often harder to source, more expensive, and frequently lack a PCIe slot for an additional network adapter.

The T620 Plus offers better mechanical and thermal characteristics, but it is based on an older chipset. Some revisions also include a mini PCIe slot and an additional mSATA connector. Community reports suggest that it supports only IOMMU v1, which can limit PCI passthrough scenarios. By comparison, the T730's M.2 Key E slot is more useful for this build, and the T620 Plus's legacy ports add little practical value. For those reasons, the T730 was selected.

For networking, the target was a four-port Gigabit Ethernet adapter with SR-IOV support. The Intel I350-T4 is a strong candidate, but genuine Intel cards are relatively uncommon and expensive on the secondary market. Dell OEM variants are easier to source and still receive firmware updates, although SR-IOV may be disabled by default on some firmware versions. Based on those considerations, a Dell I350-T4 was selected.

For Wi-Fi, there does not appear to be a widely available 802.11ac/ax adapter with reliable AP-mode support on Debian or FreeBSD. Many Intel adapters also rely on CNVi and are therefore unsuitable for this setup. Atheros-based modules remain the primary confirmed option for AP mode, but they are difficult to source and are often limited to Wi-Fi AC and Bluetooth 4.0. An external MikroTik access point is therefore the most practical choice.

For image detection, the plan is to use a Google Coral Edge TPU. Google offers USB, mini PCIe, and M.2 variants. The M.2 dual-module version appears to offer the best price-to-performance ratio, but the T730 may not support PCIe bifurcation, which could limit the device to a single active core. Until that module becomes available, the existing USB accelerator will be used.

## Firmware

The first maintenance task is to update firmware.

The T730 BIOS can be updated either from Windows 10 installed on the internal drive or directly from the BIOS by using a USB flash drive. HP's workflow for creating flash media changes over time, so the most reliable approach is to prepare the USB drive on a separate Windows machine running in UEFI mode. The installer fails in legacy-mode virtual machines because it checks for UEFI support before allowing USB media creation.

The TPM can also be upgraded from version 1.2 to 2.0. The T730 uses Infineon SLB9660/SLB9665 chips, and Infineon documentation indicates that this upgrade path is supported. In practice, a working firmware package may come from a different HP model, provided it is compatible with the same chip family.

The key constraint is that TPM updates are distributed as delta packages, so each step must move from the currently installed version to a compatible newer version. If the target package does not accept the current firmware directly, an intermediate version is required. In practice, the process begins by updating TPM 1.2 to the latest firmware version supported by the machine and then flashing a compatible TPM 1.2-to-2.0 migration package by following the TPM 2.0 update procedure for this thin client. Additional TPM 2.0 firmware upgrades may then be possible if the appropriate delta package is available.

Relevant SoftPaqs:

- [TPM 1.2 update](https://ftp.hp.com/pub/softpaq/sp82001-82500/sp82133.exe)
- [Compatible TPM 1.2 to 2.0 binary](https://ftp.hp.com/pub/softpaq/sp87501-88000/sp87753.exe)
- [Instructions and scripts](https://ftp.hp.com/pub/softpaq/sp84001-84500/sp84096.exe)
- [BCU tool](https://ftp.hp.com/pub/softpaq/sp143501-144000/sp143621.exe)

The procedure appears to be Windows-based, although parts of it may be automatable under Linux and some BCU-related steps may be possible from the BIOS. I have not verified whether the TPM 1.2 update can be incorporated into the same workflow, because that updater was distributed as a standalone executable and was not analyzed further.

Dell's I350 firmware can be updated directly from Windows. Dell also provides a Linux updater, but it does not appear to work reliably on Debian-based systems because NIC discovery fails. After updating to the latest firmware, the adapter no longer exposed SR-IOV capability. Further investigation showed that SR-IOV was enabled in firmware 15.0.27 but disabled by default in firmware 20.0.16. Intel provides administrative tools for enabling SR-IOV, but those tools do not appear to support OEM cards. As a workaround, SR-IOV can be enabled directly in EEPROM with `ethtool`.

References:

- [Dell firmware 15.0.27 release notes](https://dl.dell.com/FOLDER01772935M/1/FW_release_notes.txt?uid=251ecd72-7594-468f-c8c6-8c94ceb404c9&fn=FW_release_notes.txt)
- [Dell firmware 20.0.16 release notes](https://dl.dell.com/FOLDER07032138M/1/fw_release.txt?uid=0d3a9d66-4e22-4f61-39eb-1a80ded24894&fn=fw_release.txt)
- [Intel administrative tools for Intel network adapters](https://www.intel.com/content/www/us/en/download/2593/administrative-tools-for-intel-network-adapters.html)
- [vfio-users guide](https://listman.redhat.com/archives/vfio-users/2017-August/msg00086.html)

## Operating System Configuration

The Proxmox PCI passthrough guide covers the required IOMMU and SR-IOV configuration: [PCI Passthrough](https://pve.proxmox.com/wiki/PCI_Passthrough)
