Mobis VESS
==========

Introduction
------------

The VESS is the Virtual Engine Sound System found in many KIA and Hyundai
branded vehicles. A system like VESS has been required by law for electric
vehicles (EVs) since 2020 by EU law.

This research is not aimed at disabling VESS as you can do that very simply
by just unplugging it. The work here is some reverse engineering based on the
physical inspection only.

This work was done during the Christmas holiday of 2021.

A nice future roadmap might be to control the volume and change the sound:

 1. By flashing new SPI contents directly
 2. Using the onboard serial debug port
 3. Using ISO-TP over CAN
 4. Using the OBD II port

CAN
---

We know the device broadcasts using CID: 0x5E3 (1507d), with an 8 * 0x00
broadcast data.

Many thanks to [Eric Reuter](https://www.youtube.com/watch?v=OLT1aKdpYhs) with
open source [code](https://github.com/ereuter/vess).

 - Does that make the request `5E3-8=5DB`?

Maybe using ISO-TP for multiple data packets?

 - Speed - `0x524` (1316d)
 - Gear - `0x200` (512d)

Notes
-----

Using RS485 CAN HAT:

    sudo ip link set can0 type can bitrate 500000 restart-ms 100
    sudo ip link set up can0

PCB
===

I cut the box but left the connector attached. I might want to connect this
to my car in the future and those friction-fit pins looked fragile.

Removing the conformal coating was really *fun*.

![top view](pcb-top.jpg)

![top view](pcb-bottom.jpg)

![top view](pcb-dbg.jpg)

Notable components:

U100
----

Analog `BF706CCPZ`, 400MHz 1Mb SRAM DSP.

U101
----

Macronix `MX25L3235E`, 32Mbit flash chip.

U102
----

NXP A42/3C,`TJA1042` CAN transceiver.

U103
----

JRC `NJU7291` system reset IC with watchdog timer.

U105
----

ST `FDA803D`, 1x45W class D amplifier.

U106
----

Unlabelled 5V power regulator, for audio?

- 05 → filtered 12V in?
- 04 → unknown
- 03 → `GND`
- 02 → unknown
- 01 → U102:VCC

U109
----

Unknown, marked `11S XAG`, 3V3 MCU power regulator

- 01 → 4K6 resistor then `STB`
- 04 → `U102:VCC`
- 06 → `U102:SPLIT`

U110
----

Unknown, marked `D41 X03`, 5V CAN power regulator. Note: this seems to be power phased from U109.

- 01 → `CN100:1` i.e. `VDD_EXT`
- 04 → `U102:VCC`

CN100
-----

1.27mm socket, similar to `SHR-12V-S-B`, used as debug port *presumably* for factory
programming.

 - 01, `VDD_EXT`
 - 02, `R317` → `SYS_HWRST`
 - 03, `R117` → `PB_09` → `UART0_RX`
 - 04, `R116` → `PB_08` → `UART0_TX`
 - 05, `JTG_TDO_SWO`
 - 06, `JTG_TCK_SWCLK`
 - 07, `JTG_TDI`
 - 08, `JTG_TMS_SWDIO`
 - 09, `JTG_TRST`
 - 10, `GND`
 - 11, `WDEN` [10K to GND]
 - 12, `nc`

SW100
-----

Connected to `SYS_HWRST` which triggers a CPU reset.

CN103
-----

    4  3  2 [1]
    8  7  6  5

- 01, `Battery+`
- 02, `IG1` → `D101` → MOSFET
- 03, `SWI` → Schottky diode (protected input?)
- 04, `P-CAN High` → `U102:7`
- 05, `GND`
- 06, `GND`
- 07, `IND` → `R194` (460R)
- 08, `P-CAN Low` → `U102:6`

TTY
===

Connecting a 3.3v RS-232 adaptor to `UART0_RX` and `UART0_TX` at 115200 baud
gives a repeated:

    >---------------------------------------------------------<
    >                                                         <
    >     VESS Firmware 88.0201                               <
    >                                                         <
    >---------------------------------------------------------<

...and this loops forever, unless `WDEN` is pulled to MCU VCC (3V3).

Powering the module using `Battery+` gives the same output, but also pushing
`IGN` up to 12V the additional prompt appears:

    $ tio /dev/ttyUSB0

    >---------------------------------------------------------<
    >                                                         <
    >     VESS Firmware 88.0201                               <
    >                                                         <
    >---------------------------------------------------------<
    VESS>>

HELP
----

    /************************************************************/
    HELP : Display command list
    POWERIC : Power amp IC control
    VESS : VESS Sound Output Control
    RHEOSTAT : VESS LED PWM Control
    STATUS : VESS Internal Status
    ALLFLAT : Flat EQ setting
    SOUNDTEST : 20km/h sound output
    SOUNDSETTING : display output sound setting
    GOTOUPDATER : goto reprogramming mode
    SAVETEST : save sound output setting to flash memory
    READSETTING : read sound output setting from flash memory
    EOLTEST : writing YYMMDD to flash memory
    Lib Version: 2
    File: ..\src\main.c | Line: 106 | Function: help
    Build Date: Jan 14 2019 | Build Time: 07:54:38
    /************************************************************/

STATUS
------

    ======= CAN & I/O =======
    CarRdy : 0
    GarSelDisp : 0
    WhlSpdFL_Kph : 0
    WhlSpdFR_Kph : 0
    PTOperModforECU : 7
    Avail Spd : 200.000000
    Detentout : 0
    RheostatLevel : 0
    IntTailAct : 0
    VESS Switch Input : 1
    TCU1(gear) Input : 0
    HEV_CP6(detent,rheo) Input : 0
    HCU1(ready,PTmode) Input : 0
    CGW2(tail) Input : 0

    ========== etc ==========
    VESS Switch On/Off Status: 1
    7V Det port value: 1
    18V Det port value: 1
    7V Det Status: 0
    18V Det Status: 0
    ucLEDstep: 20
    ucOpenLoadDetected: 1
    ucShortLoadDetected: 0
    ucOffsetDetected: 0
    Updater Cnt : 65535

    ====== Product Info ======
    date : 2019/08/31
    Part No. : 96390-G5650
    CAN DB ver : 0.40
    HW ver : 01.01

POWERIC
-------

    PowerIC Read/Write
    1. Write register 2. Read register 3. Unmute 4. Mute 5. Power limit
    6. PWM switching Freq/Dithering/Phase

    2

    ====Power IC====

    PowerIC Register(0x00) : 0x00
    PowerIC Register(0x01) : 0x41
    PowerIC Register(0x02) : 0x00
    PowerIC Register(0x03) : 0x1B
    PowerIC Register(0x04) : 0x00
    PowerIC Register(0x05) : 0x00
    PowerIC Register(0x06) : 0x80
    PowerIC Register(0x07) : 0x00
    PowerIC Register(0x08) : 0x21
    PowerIC Register(0x09) : 0x00
    PowerIC Register(0x0A) : 0x40
    PowerIC Register(0x0B) : 0x00
    PowerIC Register(0x0C) : 0x00
    PowerIC Register(0x0D) : 0x00
    PowerIC Register(0x0E) : 0x07

    ====Power IC====
    PowerIC Register(0x20) : 0x00
    PowerIC Register(0x21) : 0x08
    PowerIC Register(0x22) : 0x01

    5

    Input MAX voltage scale
    1.OFF, 2.15%, 3.30%, 4.40%, 5.50%, 6.60%, 7.70%, 8.80%, ESC to exit

    6

    Select PWM switching frequency at 48KHz
    1.336, 2.384, 3.432

    Select PWM clock dithering
    1.ON, 2.OFF

    Not selected

    Select PWM phase
    1.In phase, 2.Out of phase

    Not selected

Note, PowerIC Register(0x22) changes from 0x0 when muted to 0x01 when unmuted.

VESS
----

    /************************************************************/
    VESS
    Control Speed : '<' Decrease, '>' Increase
    Forward/Reverse : '?' Toggle Forward/Reverse
    Gain Slew Enable/Disable : '/' Toggle Enable/Disable
    /************************************************************/

RHEOSTAT
--------

    PWM Control
     > : Increase rheostat stage
     < : Decrease rheostat stage
     / : Automatic rheostat stage change


ALLFLAT
-------

    Speed dependent volume off

SOUNDTEST
---------

    Sound Test 20km/h
    1. Sound Generation 2. Sound Stop

    1

    Start Sound output

    2

    Stop Sound output

SOUNDSETTING
------------

    ======= Sound Setting =======

    fbegin[0]: 0.000000
    fEnd[0]: 15.000000
    fConstant[0]: 0.249870
    bCharacter[0]: 0
    bReverse[0]: 0
    NumPoint[0]: 7
    UserGain / dB[0]: 1.000000 / 31.000000
    UserSpeed[0]: 0.101563
    UserGain / dB[1]: 1.000000 / 25.000000
    UserSpeed[1]: 2.500000
    UserGain / dB[2]: 1.000000 / 19.000000
    UserSpeed[2]: 5.000000
    UserGain / dB[3]: 1.000000 / 19.000000
    UserSpeed[3]: 7.500000
    UserGain / dB[4]: 1.000000 / 19.000000
    UserSpeed[4]: 10.000000
    UserGain / dB[5]: 1.000000 / 19.000000
    UserSpeed[5]: 12.500000
    UserGain / dB[6]: 1.000000 / 0.000000
    UserSpeed[6]: 15.000000

    fbegin[1]: 10.000000
    fEnd[1]: 25.000000
    fConstant[1]: 0.249870
    bCharacter[1]: 0
    bReverse[1]: 0
    NumPoint[1]: 7
    UserGain / dB[0]: 1.000000 / 0.000000
    UserSpeed[0]: 10.000000
    UserGain / dB[1]: 1.000000 / 15.000000
    UserSpeed[1]: 12.500000
    UserGain / dB[2]: 1.000000 / 11.000000
    UserSpeed[2]: 15.000000
    UserGain / dB[3]: 1.000000 / 11.000000
    UserSpeed[3]: 17.500000
    UserGain / dB[4]: 1.000000 / 11.000000
    UserSpeed[4]: 20.000000
    UserGain / dB[5]: 1.000000 / 15.000000
    UserSpeed[5]: 22.500000
    UserGain / dB[6]: 1.000000 / 0.000000
    UserSpeed[6]: 25.000000

    fbegin[2]: 20.000000
    fEnd[2]: 35.000000
    fConstant[2]: 0.249991
    bCharacter[2]: 0
    bReverse[2]: 0
    NumPoint[2]: 7
    UserGain / dB[0]: 1.000000 / 0.000000
    UserSpeed[0]: 20.000000
    UserGain / dB[1]: 1.000000 / 0.000000
    UserSpeed[1]: 22.500000
    UserGain / dB[2]: 1.000000 / 0.000000
    UserSpeed[2]: 25.000000
    UserGain / dB[3]: 1.000000 / 0.000000
    UserSpeed[3]: 27.500000
    UserGain / dB[4]: 1.000000 / 0.000000
    UserSpeed[4]: 30.000000
    UserGain / dB[5]: 1.000000 / 0.000000
    UserSpeed[5]: 32.500000
    UserGain / dB[6]: 1.000000 / 0.000000
    UserSpeed[6]: 35.000000

    fbegin[3]: 0.000000
    fEnd[3]: 35.000000
    fConstant[3]: 0.583327
    bCharacter[3]: 1
    bReverse[3]: 0
    NumPoint[3]: 15
    UserGain / dB[0]: 1.000000 / 36.000000
    UserSpeed[0]: 0.000000
    UserGain / dB[1]: 1.000000 / 34.000000
    UserSpeed[1]: 2.500000
    UserGain / dB[2]: 1.000000 / 30.000000
    UserSpeed[2]: 5.000000
    UserGain / dB[3]: 1.000000 / 28.000000
    UserSpeed[3]: 7.500000
    UserGain / dB[4]: 1.000000 / 26.000000
    UserSpeed[4]: 10.000000
    UserGain / dB[5]: 1.000000 / 26.000000
    UserSpeed[5]: 12.500000
    UserGain / dB[6]: 1.000000 / 26.000000
    UserSpeed[6]: 15.000000
    UserGain / dB[7]: 1.000000 / 25.000000
    UserSpeed[7]: 17.500000
    UserGain / dB[8]: 1.000000 / 25.000000
    UserSpeed[8]: 20.000000
    UserGain / dB[9]: 1.000000 / 31.000000
    UserSpeed[9]: 22.500000
    UserGain / dB[10]: 1.000000 / 0.000000
    UserSpeed[10]: 25.000000
    UserGain / dB[11]: 1.000000 / 0.000000
    UserSpeed[11]: 27.500000
    UserGain / dB[12]: 1.000000 / 0.000000
    UserSpeed[12]: 30.000000
    UserGain / dB[13]: 1.000000 / 0.000000
    UserSpeed[13]: 32.500000
    UserGain / dB[14]: 1.000000 / 0.000000
    UserSpeed[14]: 35.000000

    fbegin[4]: 0.000000
    fEnd[4]: 35.000000
    fConstant[4]: 0.583327
    bCharacter[4]: 1
    bReverse[4]: 0
    NumPoint[4]: 15
    UserGain / dB[0]: 1.000000 / 35.000000
    UserSpeed[0]: 0.101563
    UserGain / dB[1]: 1.000000 / 33.000000
    UserSpeed[1]: 2.500000
    UserGain / dB[2]: 1.000000 / 31.000000
    UserSpeed[2]: 5.000000
    UserGain / dB[3]: 1.000000 / 29.000000
    UserSpeed[3]: 7.500000
    UserGain / dB[4]: 1.000000 / 27.000000
    UserSpeed[4]: 10.000000
    UserGain / dB[5]: 1.000000 / 26.000000
    UserSpeed[5]: 12.500000
    UserGain / dB[6]: 1.000000 / 25.000000
    UserSpeed[6]: 15.000000
    UserGain / dB[7]: 1.000000 / 24.000000
    UserSpeed[7]: 17.500000
    UserGain / dB[8]: 1.000000 / 23.000000
    UserSpeed[8]: 20.000000
    UserGain / dB[9]: 1.000000 / 27.000000
    UserSpeed[9]: 22.500000
    UserGain / dB[10]: 1.000000 / 0.000000
    UserSpeed[10]: 25.000000
    UserGain / dB[11]: 1.000000 / 0.000000
    UserSpeed[11]: 27.500000
    UserGain / dB[12]: 1.000000 / 0.000000
    UserSpeed[12]: 30.000000
    UserGain / dB[13]: 1.000000 / 0.000000
    UserSpeed[13]: 32.500000
    UserGain / dB[14]: 1.000000 / 0.000000
    UserSpeed[14]: 35.000000

    fbegin[5]: 0.000000
    fEnd[5]: 0.000000
    fConstant[5]: 0.000000
    bCharacter[5]: 0
    bReverse[5]: 1
    NumPoint[5]: 7
    UserGain / dB[0]: 1.000000 / 12.000000
    UserSpeed[0]: 0.000000
    UserGain / dB[1]: 1.000000 / 10.000000
    UserSpeed[1]: 35.000000
    UserGain / dB[2]: 1.000000 / 10.000000
    UserSpeed[2]: 100.000000
    UserGain / dB[3]: 1.000000 / 0.000000
    UserSpeed[3]: 0.000000
    UserGain / dB[4]: 1.000000 / 0.000000
    UserSpeed[4]: 0.000000
    UserGain / dB[5]: 1.000000 / 0.000000
    UserSpeed[5]: 0.000000
    UserGain / dB[6]: 1.000000 / 0.000000
    UserSpeed[6]: 0.000000

READSETTING
-----------

    Read
    Read end

Flash
=====

The full dump of the SPI device is [here](backup.bin).

ASCII strings are [here](strings.txt), some of which look super interesting.

    $ binwalk backup.bin
    DECIMAL       HEXADECIMAL     DESCRIPTION
    --------------------------------------------------------------------------------
    467552        0x72260         Microsoft executable, MS-DOS

There is raw audio data in the flash from 0x100000 which is similar to what is found in the Ioniq.
To hear it, load into Audacity as "raw data" as 1-channel, signed 16 bit PCM.
