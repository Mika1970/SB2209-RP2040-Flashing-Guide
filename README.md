# SB2209 RP2040 Flashing Guide

✅ Tested and working flashing method for BigTreeTech SB2209 (RP2040).  
This guide was validated after several failed attempts (DFU not working, Katapult issues) and is confirmed working with CANBUS.  

All hardware (Manta E3EZ, CB2, U2C, SB2209) was purchased from **BigTreeTech**.  
During testing, I had 3 modules:  
- The **first one** powered on only briefly (2 seconds) then switched off → unusable.  
- The **second one** burned during testing.  
- The **third one** is now working correctly using this method.  

---

## 🔑 Key Discoveries
1. The RP2040 does **not** use DFU but **BOOTSEL** mode.  
2. The most reliable BOOTSEL method: keep USB plugged, hold **BOOT**, press **RESET**, release **BOOT**.  
3. BOOTSEL device auto-unmounts after copying `.uf2` files.  
4. Automatic scripts are essential to flash quickly while `/dev/sda1` is mounted.  
5. `dmesg -w` is indispensable to monitor USB reactions.  
6. Udev rules may need reloading to see `/dev/serial/by-id/`.  
7. Katapult can be unstable → flashing Klipper directly works better.  
8. Adding `restart_method: command` causes Klipper to fail → **do not use**.  

---

## Step 1: Compile Katapult (optional)

```bash
cd ~/katapult
make menuconfig
Config:

Micro-controller Architecture: Raspberry Pi RP2040

Processor model: rp2040

Flash chip: W25Q080 with CLKDIV 2

Build deployment: 16KiB bootloader

Communication Interface: USB (USB-SERIAL)

Disable Status LED (GPIO25 used)

Disable "bootloader entry" options

bash
Copier le code
make clean
make
ls -lh ~/katapult/out/katapult.uf2
Step 2: Flash Script (automatic)
Create script ~/flash_klipper_fast.sh:

bash
Copier le code
#!/bin/bash
echo "Waiting for RP2040 BOOTSEL mode..."
echo "Put SB2209 in BOOTSEL NOW"

while [ ! -e /dev/sda1 ]; do
    sleep 0.1
done

echo "Device detected! Flashing immediately..."
sleep 0.5

MOUNT_POINT=$(mount | grep sda1 | awk '{print $3}')

if [ -z "$MOUNT_POINT" ]; then
    sudo mkdir -p /mnt/rp2
    sudo mount /dev/sda1 /mnt/rp2
    MOUNT_POINT="/mnt/rp2"
fi

sudo cp ~/klipper/out/klipper.uf2 $MOUNT_POINT/
sync
sudo umount $MOUNT_POINT
echo "File copied! Device will auto-eject..."
sleep 15
echo "Checking for Klipper..."
ls /dev/serial/by-id/
Make it executable and run:

bash
Copier le code
chmod +x ~/flash_klipper_fast.sh
~/flash_klipper_fast.sh
Step 3: Monitor with dmesg
bash
Copier le code
dmesg -w
Step 4: Flash Klipper directly (without Katapult)
bash
Copier le code
cd ~/klipper
make menuconfig
Config USB:

Architecture: Raspberry Pi RP2040

Bootloader offset: No bootloader

Flash chip: W25Q080 with CLKDIV 2

Communication: USB

bash
Copier le code
make clean
make
Step 5: Fix udev symlink
bash
Copier le code
sudo udevadm control --reload-rules
sudo udevadm trigger
sleep 3
ls -la /dev/serial/by-id/
Expected result:

text
Copier le code
usb-Klipper_rp2040_XXXX-if00 -> ../../ttyACM1
Step 6: Recompile for CAN
bash
Copier le code
cd ~/klipper
make menuconfig
Config CAN:

Architecture: Raspberry Pi RP2040

Bootloader offset: No bootloader

Flash chip: W25Q080 with CLKDIV 2

Communication: CAN bus

CAN bus interface: can0 (on gpio4/gpio5)

CAN bus speed: 1000000

bash
Copier le code
make clean
make
make flash FLASH_DEVICE=/dev/serial/by-id/usb-Klipper_rp2040_XXXX-if00
Step 7: Connect and configure CAN
bash
Copier le code
ip link show can0
sudo ip link set can0 up type can bitrate 1000000
Detect UUID:

bash
Copier le code
~/klippy-env/bin/python ~/klipper/scripts/canbus_query.py can0
Add to printer.cfg:

ini
Copier le code
[mcu EBBCan]
canbus_uuid: YOUR_UUID_HERE
Final Configuration
ini
Copier le code
[mcu EBBCan]
canbus_uuid: 40734952e69c
⚠️ Do not add restart_method: command → it will break Klipper.

Files
~/flash_klipper_fast.sh → auto flash script

~/klipper/.config → RP2040 CAN config

/dev/serial/by-id/usb-Klipper_rp2040_XXXX-if00 → USB ID

printer.cfg → [mcu EBBCan] with canbus_uuid

✅ Status
This method is confirmed working.
Tested on Manta E3EZ + CB2 + U2C + SB2209 RP2040.
Final result: stable CANBUS communication, UUID detected, Klipper operational.
