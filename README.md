# sd3_klipper_configs


## Purpose and Background
This repository holds all the configs and documentation needed in order to correctly configure Klipper on a Solidoodle 3 with some modifications. There's a lot of good information and resources online, but the websites may have broken links or some of the hardware needed to get the configuration up is no longer avialble to purchase. I want to document this project on GitHub as well as provide my configuration once the setup is completed. Hopefully this can be used for others, if they are having difficulty with getting a solidoodle up and running.

I recieved a solidoodle 3 as a donation. The previous owner mentioned that hot end was broken, but everything else was running fine. 
- X Y and Z motors worked. 
- Heated bed worked
- hot end heated
- extruder was cracked. 
- Controller is running Marlin V1, so should probably update that. Also, with a new hot end and extruder, I'd need a way to flash new configurations anyways. The board Seems to have a HID bootloader, but I was unable flash new firmware with it. 

Since this was my first ever 3D printer, I did a lot of research into deciding what I wanted to do for this project. After some planning I have decided my course of action.

## Project Goals
1. Replace extruder and hot end assembly. Decided to go with E3D Titan and V6.
2. Replace extruder motor
3. Flash a new bootloader, and Reflash controller with klipper, and run klippy/etc. on Raspberry PI 5. This idea seems to make a lot of sense to me. The board is over 10 years old, so offloading as much calculation off the controller makes a lot of sense. Maybe it will postpone an inevitable controller swap
4. Tune Klipper to get ready to print
5. Add a PWM fan for part cooling. Header is not soldered on, so need to do a little soldering.

## Hardware and Equipment
### Printer: Solidoodle 3, Controller Solidoodle Printrboard Rev. E
One important bit of information is that the Solidoodle Printrboard Rev. E is slighly different compared to the other Printrboards out in the wild. This link gave me a lot of clues on how to flash new firmware for the controller: http://wiki.solidoodle.com/update-firmware
also, the schematic for the board comes in handy: https://github.com/laris/Solidoodle_Motherboard_REV_E

|Category|Description|Comments|
|---|---|---|
|MCU |Atmel AVR AT90USB1286|Data sheet: https://ww1.microchip.com/downloads/en/DeviceDoc/doc7593.pdf|
|Bootloader| HID Bootloader|see lincomatics post for additional available bootloaders https://blog.lincomatic.com/bootloaders-for-at90usb1286/
|JTAG ISP| 6-pin ports available|Need to find a programmer online|
|Motor Drivers|Allegro A4982|https://www.allegromicro.com/en/products/motor-drivers/brush-dc-motor-drivers/a4982|
|PWM Fan| Connector JP17| No Molex connectors are populated on the board for this yet. Should add one in to install a part cooling fan

### Equipment needed for the project
|Project Goal|Part|Comments|
|---|---|---|
|1| E3D Titan Extruder| was looking at the Titan Aero, but since it is about 50mm shorter, I didnt think it was going to be tall enough to clear the extruder mount assembly over the X-axis
|1| E3D V6| seems reliable
|1| Bracket plate for Titan and motor| You can find these off THingiverse. (will post link of the one I used Once i find it)|
|1| Connectors| need some molex and JHP connectors for the rewiring
|2| E3D Compact but Powerful Motor| not neccesary, but just opted to update the motor
|3|JTAG Programmer|Needed to get one since HID bootloader doesnt seem to work. In fact, I can just flash the firmware through the ISP without needing to re-install a bootloader. Any AVR programmer will do, but I found this one for pretty cheap at Pololu: https://www.pololu.com/product/3172|
|3| Raspberry Pi| Any will rev 3+ will work for running Klipper

## Goal 1. Replace Parts
Follow the instructions off the E3D website
E3D Titan:https://e3d-online.zendesk.com/hc/en-us/sections/6157667943069-Titan

E3D V6: https://e3d-online.zendesk.com/hc/en-us/sections/6157539579037-V6

E3D Compact but powerful Motor: new cables are needed. refer to the datasheet for the wiring: https://e3d-online.zendesk.com/hc/en-us/article_attachments/360016616377 as well as the driver datasheet to wire up the other end.

## Goal 2: Reflash Controller
1. On Raspberry Pi, install Klipper. I used Kiauh: https://github.com/dw-0/kiauh 
2. Klipper does have installation in structions on their site https://github.com/Klipper3d/klipper, but let me paraphrase specifically for the Solidoodle 3.
``` bash
cd ~/klipper/
make menuconfig
```

when the menuconfig opens, select the at90usb1286.
```
Micro-controller Architecture (Atmega AVR)
Processor model (at90usb1286)
```
Press Q and Y to Quit and save the configuration

compile the source code:
```bash
make
```

after compiling, the output should specifify the location of the hex file. remember this file as this is the hex file that we will flash to the controller

3. Its now time to reflash the controller. But before doing anything its a good idea to check fuses and back up flash and eeprom. The instructions here are based off using a Pololu USB AVR as I was unfortunately able to get the HID bootloader working.

Check the fuses. fuses should look like this:
|Fuses|Value|Comments|
|---|---|---|
|lfuse|0xDE||
|hfuse|0x9B or 0xDB|use 0x9B to enable JTAG programming, 0xDB to disable and open up some additional IOs|
|efuse|0xF0||

If fuses arent set correctly you write it in. Liconmatic's bootloader guide talks about it in detail. Be carefule when setting the fuses as it could brick the device if not done properly.

Backup Flash
```
avrdude -c stk500v2 -p at90usb1286 -P /dev/ttyACM1 -U flash:r:firmware.hex:i
```

Backup EEPROM
```
avrdude -c stk500v2 -p at90usb1286 -P /dev/ttyACM1 -U eeprom:r:eeprom.hex:i
```

Rewrite HID Bootloader (you can choose HID, DFU or CDC on preference. refer to Lincomatic's blog post)
```
avrdude -c stk500v2 -p at90usb1286 -P /dev/ttyACM1 -U flash:w:BootloaderHID.hex:i
```

Flash klipper.elf.hex
```
avrdude -c stk500v2 -p at90usb1286 -P /dev/ttyACM1 -U flash:w:klipper.elf.hex:i
```

