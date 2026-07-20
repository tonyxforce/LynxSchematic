# Lynx HIB (Hardware Interface Board) Schematic
This repo contains a reverse engineered schematic of and info on the Lynx HIB (REV Robotics Expansion Hub, and the Expansion Hub part of the Control Hub). This was done with a multimeter; it may have some errors.

The schematic itself is in the `pcb` directory. Photos can be found in the `photos` directory. Notes on repairs can be found in the `repair notes` directory

## PCB versions
This schematic was created from a revision 2.3 Lynx HIB. This revision uses a Bosch BNO055 IMU and still includes an IMU in Expansion Hubs. Control Hubs and Expansion Hubs of this revision only differ by the inclusion of a compute board. Newer Control Hubs (sold after September 2022) seem to use a revision 2.4c or 2.5 HIB and have an Bosch BHI260AP IMU. The IMU section of the schematic has both in revisions's in it. Other sections seem to be the same aside from some component supplier changes and some additional zero ohm resistors around the compute board header. These zero ohm resistors aren't included in the schematic. I have not seen inside an Expansion Hub without an IMU (those sold after December 1, 2021), so I don't know how accurate this schematic is for them and how were changed from revision 2.3.

### References
- [Expansion Hub IMU Removal Notice](https://www.revrobotics.com/blog/expansion-hub-IMU-updates)
- [Control Hub and IMU differences](https://docs.revrobotics.com/duo-control/sensors/i2c/imu)

## Firmware

### MCU
The MCU firmware can be found on [REV's website](https://docs.revrobotics.com/duo-control/managing-the-control-system/updating-firmware/firmware-changelog).

#### Notes on replacing the MCU
- A heated plate is recomended to preheat the board before trying to remove or resolder the MCU.
- You can source the component off of Mouser Electronics, but prepare for astronomical shipping costs. If you can find a cheaper source (with 1 MOQ), pull requests are welcome.
- Try not to overheat the MCU while soldering, but I was blasting it with 400˚C air for a solid 30 minutes and it seems to still work.
- After soldering in the new MCU, give power on the battery connector (preferably with a limited current power supply), check 3.3v present on C63 and check 1.2V present on C28, C30, C34 and C55 (should all be connected in parallel). If 1.2V is missing or shorted, check soldering and/or reheat the pads. If 3.3V is missing, check for shorts.
- The 5V rail is not always on. [The buck converter IC's enable pin is tied to a GPIO on the MCU; if the MCU is not soldered, it may not start up by itself.](https://github.com/tonyxforce/LynxSchematic/photos/5v%20regulator%20enable_email%20from%20REV.png)
- I couldn't find a pinout diagram for this chip. Not even the datasheet includes it and google pretends this chip doesn't even exist and tries to redirect me to TM4C123GH6PM. If you can find it, pull requests are welcome.

#### First Upload
The first upload must be done with JTAG. It may be possible to do over UART with the bootloader, but it hasn't been tested yet. 

A method for uploading over JTAG follows:
1) If your HIB's revision is > 2.3, solder on a JTAG connector or some leads

2) Acquire a "FTDI FT2232H Mini Module" to use as a JTAG debug probe. A genuine one from FTDI has a better chance of working reliably. Alternatively, use an ST-Link from STMicroelectronics. (An ST-Link v2 was tested but a V1 should also work, it may need a config change), I recommend this approach since the FTDI module can be a bit pricey (around $30), while an ST-Link can be had for as little as $3.

3) Connect the Mini Module or the ST-Link to the HIB. A clean way of doing this is by making an adapter out of some 0.1" pitch header and half of one of the the pre terminated cables Molex sells that connect to the HIB's JTAG connector. See below for the cable's part number. Please note that this adapter cable can be purchased for cheap since [the connector is the same as on a CYD (Cheap Yellow Display)](https://forum.arduino.cc/t/cyd-peripheral-leads/1408501). I recommend soldering a 4 pin connector, since you can tap the 3.3v and GND from elsewhere. The pinout of the JTAG connector (Assuming USB port facing up):
  - VCC
  - SWCLK
  - SWDIO
  - TDI
  - TDO
  - GND

If you are using an FT2232H module:
Connect  
  - GND to GND
  - TCK to ADBUS0
  - TDI to ADBUS1
  - TDO to ADBUS2
  - TMS to ADBUS3

If you are using an ST-Link:
Connect
  - GND to GND
  - SWCLK to SWCLK
  - SWDIO to SWDIO

No other connections needed, give power to the board through the battery connector (anything more than 3.3v is sufficient, 5v is tested and working).

4) Download [OpenOCD](https://openocd.org/) (If you are on a Debian-based Linux distro, just run `sudo apt install openocd`)

5) Download the config files for the Lynx and the Mini Module/ST-Link [here](https://github.com/
DuckTapeAndAPrayer/LynxSchematic/tree/master/openocd%20configs)

6) If you're using the FTDI module, run `openocd -f ftdi_lynx.cfg -c "program <path to firmware file> reset exit"`
If you're using an ST-Link, run `openocd -f stlink_tm4c123.cfg -c "program <path to firmware file> reset exit"`

7) Remove power from the board and reapply it. It should start working normally. (LED Blinking)

This flashing process has been tested with OpenOCD v0.12.0 on Debian Linux.

#### Subsequent uploads
If firmware has be uploaded before it can be upgraded the same way or with the bootloader by using REV Hardware Client, the FtcRobotController app, or [DuckUpdate](https://github.com/DuckTapeAndAPrayer/DuckLynx/tree/master/DuckUpdate). [TI's LMFlashProgrammer](https://www.ti.com/tool/LMFLASHPROGRAMMER) will also work if the MCU is restarted into the bootloader. This can be done with DuckUpdate or with [REV's node-expansion-hub-ftdi](https://github.com/REVrobotics/node-expansion-hub-ftdi).

### FT230XQ
There is no firmware for this device, but it needs to have its config updated. Its config holds the serial number for the HIB, so when writing the same config to many devices be careful to not overwrite the serial number field. As well, it is connected to the reset and bootloader entry pins of the MCU and by default it will drive these pins to ground which will hold the MCU in reset forever. To fix this the CBUS pin modes of need to be changed to GPIO. The easiest way to do this is with [FTDI's FT_Prog](https://ftdichip.com/utilities/) utility, but it also can be done with [libftdi](https://www.intra2net.com/en/developer/libftdi/index.php) or [FTDI's d2xx](https://ftdichip.com/drivers/d2xx-drivers/) library. To update the modes with FT_Prog, connect the device to a computer running the program, edit `Hardware Specific` -> `CBUS Signals` to all be `GPIO`, and save the new config to the device.

### BNO055 IMU
The BNO055 IMU has built in firmware from Bosch. It can be updated, but it isn't intended to be replaced by user firmware. If the IMU has to be replaced whatever firmware that ships on the IMU from the factory will be fine. 

### BHI260AP IMU
The BHI260AP IMU has some built in libraries and a bootloader stored in ROM, but the user has to supply firmware. It is either supplied by the host device or it is read from an external SPI flash. On the Lynx HIB an external W25Q64JWSSIQ SPI Flash is used. The flash contains the same data across devices, so it probably doesn't have any device specific calibration in it and if the flash has to be replaced the data from one HIB's flash can be copied to another's

## Components
- A large chunk of the passives are in a 0603 package
- Aux Shunt resistors are 20 mΩ (milliohm) in a 2512 package. P/N - Panasonic ERJ-M1WSF20MU

### ICs
| Description | Part Number | Additional Info |
| ----------- | ----------- | -------- |
| Main MCU | Texas Instruments TM4C123GH6PGEI7 |
| Motor controller | STMicroelectronics VNH5050AE |
| USB to UART | FTDI FT230XQ |
| RS485 transceiver | STMicroelectronics ST3485EB |
| Compute board bus transceiver | Texas Instruments SN74LVC8T245 |
| Op Amp for shunts | STMicroelectronics LMV321RILT | For AUX Shunts. Labeled K176. SOT 23-5 |
| 5V Buck converter driver | Texas Instruments TPS54527 | 
| 3.3V Buck converter driver | Texas Instruments TPS562209 |
| Adjustable shunt voltage regulator | Diodes Incorporated TL431ASA | For AREF. U17. labeled AAXXX |
| High side current monitor | Diodes Incorporated ZXCT1010E5TA | U25 |
| USB OTG ESD Diodes | Nexperia PUSBM5V5X4-TL | U19 |
| IMU | Bosch BNO055 | Labeled 701. Revision == 2.3 |
| IMU | Bosch BHI260AP | Revisions > 2.3 |
| 1.8V LDO | MCP1700T-1802E/TT | Revisions > 2.3 |
| Geomagnetic Sensor | Bosch BMM150 | Revisions > 2.3 |
| SPI flash | Winbond W25Q64JWSSIQ | Revisions > 2.3. For IMU |

### Diodes
| Description | Part Number | Additional Info |
| ----------- | ----------- | -------- |
| Status LED | Lite-On LTST-G683GEBW | D1 |
| Zener diode | onsemi 1SMB5931BT3G | For motor drives. D4 |
| Tiny ESD diode | Nexperia PESD15VS1UL | D5 |
| ESD protection diode array | Comchip CPDV5-5V0U(-HF?) | Labeled E5U. SOT-353 |
| ESD protection diode array | Comchip CPDV5-3V3UP(-HF?) | Labeled E3V3. SOT-353 |

### Transistors
Many of these have labels with more characters listed, but those parts are date codes.
| Description | Part Number | Additional Info |
| ----------- | ----------- | -------- |
| Big N channel MOSFET | onsemi FDD9409-F085 | Q29. For reverse current protection |
| NPN Transistor | ROHM DTC143XKA | Labeled 43. LED Driver |
| N channel MOSFET | Nexperia BSH103 | Labeled WJ3. Q15, Q19 |
| P channel MOSFET | Toshiba SSM3J328R | Labeled KFH. 5V servo power enable. Q20 |
| P channel MOSFET | Alpha and Omega AO3407A | Labeled X73L. Used throughout the board. Q22, Q23, Q14, Q27 |
| N channel MOSFET | Diodes Incorporated BSS138 | Labeled K38. Q28. Revision == 2.3 |
| N channel MOSFET | onsemi BSS138K | Labeled SK. Q28. Revision > 2.3 |
| N channel MOSFET | onsemi BSS138 | Labeled SS. Q30 & Q31. Only on revision > 2.3 |
| N channel MOSFET | Alpha and Omega AO3434A | Labeled Y4. Different unknown P/N and labeled N3 on revision == 2.3. Used for GPIO and servo ports |

### Connectors
| Description | Part Number | Additional Info |
| ----------- | ----------- | -------- |
| XT30 Male | Amass XT30UPB-M |
| XT30 Female | Amass XT30UPB-F |
| Mini USB B | MUSBS5FBM1RA | This part number exists nowhere on the internet. It is Leoco product series 0850 and P/N 0850BFBD111. The F could be J or K as the gold plating thickness is unknown. TE Connectivity P/N 1734035-1 seems to have the same pad dimensions, so it may work |
| JTAG Connector Female | Molex 53398-0671 | Connector not included on revision > 2.3. Same part should fit on SPI connector |
| JTAG Connector Male Cable | Molex 15134-0605 | Not on Lynx HIB, but helpful for connecting a debugger |
| All shrouded external connectors | JST PH |
| All motor connectors | JST VH |

### Fuses
All from Bel Power? Bel Fuse? They are green.
| Description | Part Number | Additional Info |
| ----------- | ----------- | -------- |
| 2A hold current | 0ZCJ0200FF2C | Labeled b2. Used on servo ports |
| 1.5A hold current | 0ZCH0150FF2E | Labeled bS. Used for USB port |
| 1A hold current | 0ZCJ0100FF2E | Labeled b1. Used for GPIO ports |
| 0.5A hold current | 0ZCK0050FF2E | Labeled bM. Used for I2C, analog, and encoder ports |

### Other
| Description | Part Number | Additional Info |
| ----------- | ----------- | -------- |
| USB Common Mode Choke | ACM-0603-900-T | L2 |

### Mystery Components
- D2 - Different from revision 2.3 to revision > 2.3
- D3 - Maybe ESD Protection Diode LittelFuse SMF17A
- All the capacitors
- L1
- Y2 - 16 MHz main oscillator
- Y1 - revision 2.3 IMU oscillator. 32.768 KHz

## Notes about the board
The board seems to have more than two layers, so the creation of a schematic by observation wasn't possible.

Make sure to not touch the 3.3v switching converter circuitry while the board is powered, since it can throw off the voltage reference and can make the USB converter IC blow up in a nice bunch of sparks (which has happened)

### Compute board header (J18)
The header that connects to the compute board (if equipped).

Pin numbering:
```
 1 - - 40
 2 - - 39
 3 - - 38
 :     :
 :     :
20 - - 21
```

The pinout of the connector on the compute board can be found on page 16 of the [96Boards CE Specification Version 1.0](https://raw.githubusercontent.com/96boards/documentation/master/Specifications/96Boards-CE-Specification.pdf).
  - The compute board of the Control Hub is based off this specification, see [gm0](https://gm0.org/en/latest/docs/software/adv-control-system/control-system-internals.html#control-hub) for details

### Bus transceivers
A is Compute Board Header - 1.8V  
B is System - 3.3V  
Eight channels

### MCU connections
Overall temp sensor - in MCU  
3.3V voltage sense - doesn't exist

### FTDI FT230XQ
PB1 (Bootconfig enter bootloader) FT230X Pin 11 (CBUS1) and TP3 (NPRG)  
RST FT230X Pin 12 (CBUS0) and TP2 (NRST)  
PJ7 FT230X Pin 5 (CBUS2)  
PK5 FT230X Pin 16 (RTS) and Compute Board Header 36

### Digital GPIO
- GPIOs are set as open drain pins and have pull up resistors  
- Output and input are from the perspective of the lynx  

### Analog GPIO
VCCA and GNDA are just connected to VCC and GND  
VREFA- is connected to GND  
VREFA+ is connected to a circuit that makes 2.994V. This is derived from the TL431A datasheet by using the formula `Vout = Vref * (1 + (R1 / R2))`, which in the case of the HIB is `2.994 = 2.495 * (1 + (2000 / 10000))`

### 5V rail
Voltage monitoring AIN23 (PP0) - Through resistor divider with two 10K resistors to divide voltage by 2.  

### Servo
Servo 5V enable PC6 - Has pull down  

### Battery
Voltage - connected to AIN22 (PP1) via a 1/6 resistor divider with 10K and 2K resistors  
Current - Uses a high side current monitor IC with a 0.003 ohm shunt and 3K Rout resistor into AIN13 (PD2)  

### External oscillator
Called main oscillator by the datasheet  
16 MHz  
