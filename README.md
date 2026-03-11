# ATF and u-boot for mt798x

## Forked from
Original repository: https://github.com/hanwckf/bl-mt798x

## About bl-mt798x
- https://cmi.hanwckf.top/p/mt798x-uboot-usage

![](/u-boot.gif)

## Prepare

```
sudo apt install gcc-aarch64-linux-gnu build-essential flex bison libssl-dev device-tree-compiler qemu-user-static
```

## Build
```
# UART boot (for loading via mtk_uartboot)
SOC=mt7981 BOARD=tenda_rx12l_pro_uart ./build.sh

# SPI NOR flash (for permanent installation)
SOC=mt7981 BOARD=tenda_rx12l_pro ./build.sh
```

---

## Tenda RX12L Pro

### Hardware Specifications
| Feature | Specification |
|---------|---------------|
| SoC | MediaTek MT7981B (Dual-core 1.3GHz Cortex-A53) |
| RAM | 256MB DDR3 |
| Flash | 16MB SPI NOR |
| Ethernet | 1x WAN + 3x LAN (via MT7531) |
| WiFi | Dual-band AX3000 (MT7976C) |

### Flash Layout (16MB SPI NOR)
| Partition | Offset | Size |
|-----------|--------|------|
| BL2 | 0x00000 | 256KB |
| u-boot-env | 0x40000 | 64KB |
| Factory | 0x50000 | 64KB |
| bdinfo | 0x60000 | 64KB |
| FIP | 0x70000 | 512KB |
| firmware | 0xF0000 | ~15MB |

### Build Options

#### 1. UART Boot (for mtk_uartboot)
Used for initial flashing when device has no working bootloader.
```bash
SOC=mt7981 BOARD=tenda_rx12l_pro_uart ./build.sh
```
Output: `mt7981_tenda_rx12l_pro_uart-bl2.bin`, `mt7981_tenda_rx12l_pro_uart-fip-uart.fip`

#### 2. SPI NOR Flash Boot
Used for permanent installation to flash.
```bash
SOC=mt7981 BOARD=tenda_rx12l_pro ./build.sh
```
Output: `mt7981_tenda_rx12l_pro-bl2.bin`, `mt7981_tenda_rx12l_pro-fip.fip`

### Flashing Instructions

#### Step 1: Build OpenWrt
First, build OpenWrt for Tenda RX12L Pro to get `sysupgrade.bin`.

#### Step 2: UART Boot (Initial Flash)
1. Connect UART pins (TX, RX, GND)
2. Use [mtk_uartboot](https://github.com/981213/mtk_uartboot) to load bootloader:
   ```bash
   # Load BL2 via UART
   mtk_uartboot -p /dev/ttyUSB0 -b mt7981_tenda_rx12l_pro_uart-bl2.bin
   # Load FIP via UART
   mtk_uartboot -p /dev/ttyUSB0 -f mt7981_tenda_rx12l_pro_uart-fip-uart.fip
   ```
3. U-Boot should start, then load OpenWrt initramfs via TFTP

#### Step 3: Load OpenWrt to RAM
1. Boot into U-Boot via UART (see Step 2)
2. Load OpenWrt initramfs via TFTP:
   ```
   setenv ipaddr 192.168.1.2
   setenv serverip 192.168.1.3
   tftpboot 0x46000000 openwrt-*-tenda_rx12l_pro-squashfs-initramfs.bin
   bootm 0x46000000
   ```

#### Step 4: Flash via LuCI
1. After OpenWrt boots, login to LuCI at http://192.168.1.1
2. Go to **System → Backup/Flash Firmware**
3. Under "Flash new firmware image", click **Choose File**
4. Select `openwrt-*-tenda_rx12l_pro-squashfs-sysupgrade.bin`
5. Make backup of your partitions (0-4): click **Generate archive**
6. Click **Flash image**

Alternatively, via SSH:
   ```
   scp openwrt-*-tenda_rx12l_pro-squashfs-sysupgrade.bin root@192.168.1.1:/tmp/
   ssh root@192.168.1.1 "sysupgrade -n /tmp/openwrt-*-tenda_rx12l_pro-squashfs-sysupgrade.bin"
   ```

#### Step 5: Permanent Flash (Optional)
After OpenWrt is running, you can flash the bootloader to SPI NOR:
1. Build SPI NOR version: `SOC=mt7981 BOARD=tenda_rx12l_pro ./build.sh`
2. Upload files to router
3. Flash BL2 and FIP:
   ```bash
   # Flash BL2 (256KB)
   mtd write mt7981_tenda_rx12l_pro-bl2.bin bl2
   
   # Flash FIP (512KB)
   mtd write mt7981_tenda_rx12l_pro-fip.fip fip
   ```

### Porting Sequence (for developers)

1. **Build BL2+FIP for UART boot** - temporary bootloader for initial flashing
   ```bash
   SOC=mt7981 BOARD=tenda_rx12l_pro_uart ./build.sh
   ```

2. **Build OpenWrt** - get sysupgrade.bin

3. **Flash via mtk_uartboot** - load BL2 and FIP to RAM

4. **Load OpenWrt to RAM** - via TFTP in U-Boot

5. **Flash sysupgrade.bin** - via LuCI (System → Backup/Flash Firmware)

6. **Build final BL2+FIP for flash** - for permanent installation
   ```bash
   SOC=mt7981 BOARD=tenda_rx12l_pro ./build.sh
   ```

7. **Flash to SPI NOR** - BL2 → 0x0, FIP → 0x70000
