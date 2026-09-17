# ZTE T5400 stock U-Boot command set

Captured from the stock bootloader over UART on 2026-08-08.

## Bootloader

```text
U-Boot 2016.01 (Dec 03 2022 - 17:36:22 +0800)
Build: jenkins-Soft4_T5400G_TMO_CPE-4
Prompt: IPQ5018#
Console: serial@78AF000
RAM: 512 MiB
SPI-NAND: Winbond W25N02JWZEIF, 256 MiB
```

## Available commands

```text
?       - alias for 'help'
ar8xxx_dump- Dump ar8xxx registers
base    - print or set address offset
bdinfo  - print Board Info structure
bootelf - Boot from an ELF image in memory
bootipq - bootipq from flash device
bootm   - boot application image from memory
bootp   - boot image via network using BOOTP/TFTP protocol
bootvx  - Boot vxWorks from an ELF image
bootz   - boot Linux zImage image from memory
canary  - test stack canary
chpart  - change active partition
cmp     - memory compare
coninfo - print console devices and information
cp      - memory copy
crc32   - checksum calculation
dhcp    - boot image via network using DHCP/TFTP protocol
dm      - Driver model low level access
echo    - echo args to console
editenv - edit environment variable
env     - environment handling commands
erase   - erase FLASH memory
exectzt - execute TZT
exit    - exit script
false   - do nothing, unsuccessfully
fatinfo - print information about filesystem
fatload - load binary file from a dos filesystem
fatls   - list files in a directory (default /)
fatsize - determine a file's size
fatwrite- write file into a dos filesystem
fdt     - flattened device tree utility commands
flash   - flash part_name
flasherase- flerase part_name
flinfo  - print FLASH memory information
fuseipq - fuse QFPROM registers from memory
go      - start application at address 'addr'
help    - print command description/usage
i2c     - I2C sub-system
imxtract- extract a part of a multi-image
ipq5018_mdio- IPQ5018 mdio utility commands
ipq_mdio- IPQ mdio utility commands
is_sec_boot_enabled- check secure boot fuse is enabled or not
itest   - return true/false on integer compare
loop    - infinite loop on address range
md      - memory display
mii     - MII utility commands
mm      - memory modify (auto-incrementing address)
mmc     - MMC sub system
mmcinfo - display MMC info
mtdparts- define flash/nand partitions
mtest   - simple RAM read/write test
mw      - memory write (fill)
nand    - NAND sub-system
nboot   - boot from NAND device
nfs     - boot image via network using NFS protocol
nm      - memory modify (constant address)
part    - disk partition related commands
pci     - list and access PCI Configuration Space
ping    - send ICMP ECHO_REQUEST to network host
printenv- print environment variables
protect - enable or disable FLASH write protection
reset   - Perform RESET of the CPU
run     - run commands in an environment variable
runmulticore- Enable and schedule secondary cores
saveenv - save environment variables to persistent storage
secure_authenticate- authenticate the signed image
setenv  - set environment variables
setexpr - set environment variable as the result of eval expression
sf      - SPI flash sub-system
showvar - print local hushshell variables
sleep   - delay execution for some time
smeminfo- print SMEM FLASH information
source  - run script from memory
test    - minimal test like /bin/sh
tftpboot- boot image via network using TFTP protocol
tftpput - TFTP put command, for uploading files to a server
true    - do nothing, successfully
tzt     - load and run tzt
uart    - UART sub-system
ubi     - ubi commands
ubifsload- load file from an UBIFS filesystem
ubifsls - list files in a directory
ubifsmount- mount UBIFS volume
ubifsumount- unmount UBIFS volume
usb     - USB sub-system
usbboot - boot from USB device
version - print monitor, compiler and linker version
zip     - zip a memory region
```

## Notable missing commands

The stock U-Boot does not provide:

```text
iminfo
```

Therefore FIT inspection with `iminfo` is unavailable on the router.

The generated OpenWrt FIT is inspected on the build host with:

```sh
staging_dir/host/bin/mkimage -l \
  bin/targets/qualcommax/ipq50xx/openwrt-qualcommax-ipq50xx-zte_t5400-initramfs-uImage.itb
```

## Known working network configuration

Stock environment:

```text
ipaddr=192.168.0.1
serverip=192.168.0.22
```

Physical RJ45 port 4 is used by U-Boot as:

```text
eth0 up Speed :1000 Full duplex
Using eth0 device
```

Known topology:

```text
physical port 4
  -> internal IPQ5018 GEPHY
  -> MDIO0 PHY 7
  -> GMAC0
  -> U-Boot eth0
```

Confirmed working:

```text
ping 192.168.0.22
```

## Bring-up safety notes

Commands used for read-only inspection or transient RAM/network work:

```text
bdinfo
coninfo
crc32
fdt addr
fdt print
flinfo
help
is_sec_boot_enabled
md
mii info
mii read
pci
ping
printenv
smeminfo
tftpboot
tftpput
version
```

Commands which can modify RAM or transient environment state and should be
used deliberately:

```text
cp
editenv
fdt
mm
mtest
mw
nm
setenv
```

Commands which can modify persistent state or hardware and must not be used
during the initial RAM-only bring-up:

```text
chpart
erase
fatwrite
flash
flasherase
fuseipq
ipq5018_mdio write
ipq_mdio write
mii write
nand erase
nand write
protect
saveenv
sf erase
sf write
ubi write
```

This classification is intentionally conservative. The first OpenWrt boot
must not write NAND or otherwise persist changes.
