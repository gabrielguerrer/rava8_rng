## PCB and components

The first step is fabricating the RAVA's printed circuit board (PCB). 
Several companies provide PCB manufacturing services, accepting a Gerber file as input.
The Gerber file in the RAVA's repository contains the circuit board details, including top and bottom copper layers, solder masks, silk screen information, and drill data.

Concerning the circuit components, there are two options
* The user may choose to acquire and manually solder them to the PCB
* The user may ask the PCB manufacturing company to sell and solder the components using the CPL and BOM files available in the repository

**Important:** Soldering the microcontroller (MCU) to the PCB before programming its fuses will require the user to include and solder pin headers to the circuit's ICSP/SPI headers. For more details, see the [Microcontroller](https://github.com/gabrielguerrer/rng_rava/wiki/RAVA-%E2%80%90-Assembling-and-Inspecting#microcontroller) section below.

The CPL file contains the position and orientation of each component within the assembly.  
Meanwhile, the BOM file provides access to the manufacturer's distinctive ID code for the components utilized in the circuit.  

To illustrate, let's consider the RAVA circuit v1.0.  
This implementation, depicted below, employs [these components](https://github.com/gabrielguerrer/rng_rava/blob/main/v1.0/rng_rava_top_bom.csv), which can be visualized in the [circuit schematics](https://github.com/gabrielguerrer/rng_rava/blob/main/v1.0/rng_rava_schematics.png).

![RAVA v1.0](https://github.com/gabrielguerrer/rng_rava/blob/main/v1.0/rng_rava_photo.png)

### Manual soldering

Should users opt for manual soldering of the components, special attention must be given to those that require a specific orientation.
In all cases, the PCB stencil shows features matching the components' markings, thereby revealing the correct placement.
More specifically

* The MCU, comparator, and operational amplifiers are straightforward. The stencil features circles and lines that align with the ICs' clear markings
* Zener and Schottky's diodes require more attention due to their ICs' subtle line markings. Given the visibility issue, these components may require magnifying lenses to ensure accurate placement

The MCU is the most challenging component to solder due to its minute connectors.
Initial concerns about the proximity of each leg and the risk of short circuits might arise. However, various techniques exist to overcome this problem, such as employing flux paste to facilitate the solder distribution and removing excesses using a desoldering braid.

For additional insights, search for "SMD soldering tutorial" to find guidance on effective soldering methods.

### Before powering up

Before powering the circuit for the first time, the user must evaluate any potential short circuits.

This assessment can be conducted by measuring the resistance difference between the GND and 5V headers with a multimeter tool.
A reading of 0 Ohms indicates a short circuit, thereby prompting a reevaluation of the circuit connections.

## Microcontroller

The ATmega32u4 MCU is shipped with the CKDIV8 fuse programmed, causing it to operate at 2 MHz while utilizing a 16 MHz oscillator crystal. 
To achieve the intended 16MHz clock frequency, the fuse setting must be modified.
This adjustment cannot be accomplished via the USB interface. 
Instead, it requires connecting the MCU's SPI interface to an AVR programming tool.

There is a variety of available AVR programming tools. Two simple, budget-friendly, and readily available options are
* [Arduino as ISP](https://docs.arduino.cc/built-in-examples/arduino-isp/ArduinoISP)
* [USBasp](https://www.fischl.de/usbasp/)

Equipped with a programming tool, users have two viable choices for connecting it to the MCU
* Employing a TQFP-44 Programmer Adapter to reprogram the MCU before soldering it to the RAVA's PCB
* Soldering the MCU to the RAVA's PCB and then using the exposed ISP headers to reprogram the MCU

The photo below shows both options connected to an USBASP programmer. 

![MCU SPI to USBASP](https://github.com/gabrielguerrer/rng_rava/blob/main/images/wiki_mcu_spi_usbasp.jpg)

### AVRDUDE

To configure the fuses of Microchip's AVR MCUs, we rely on [AVRDUDE](https://github.com/avrdudes/avrdude/), an open-source software designed to manipulate on-chip memories. 

On a Debian based Linux OS, AVRDUDE can be installed with 
```
sudo apt install avrdude
````

Depending on your distribution, you may need to grant permission to utilize the USB port. For instance, in Ubuntu, this can be accomplished by including your user in the dialout group with 
```
sudo usermod -a -G dialout $USER
```

On a MacOs, start by installing the package management system [Homebrew](https://brew.sh/), which also installs the Xcode command line tools (`xcode-select`). Then, get AVRDUDE with 
```
brew install avrdude
```

For installing ARDUDE on a Windows OS, download the [last release](https://github.com/avrdudes/avrdude/releases).  
Next, download and install [Zadig](https://zadig.akeo.ie/).
Within the Zadig application, the driver configuration proceeds as
* In the Options menu, ensure that "List all Devices" is checked
* From the dropdown list, choose "USBasp"
* Select the "libusbK" driver
* Finally, click on the "Install Driver" button to complete the installation

In all OS, to check the connection of the programming tool with the MCU, type:

USBasp 
```
avrdude -v -c usbasp -p m32u4
```

Arduino as ISP (where the -P flag designates the port to which the Arduino is connected)
```
avrdude -v -c avrisp -p m32u4 -P /dev/ttyACM0 -b 19200
```

In a Windows system, replace `avrdude` with `averdude.exe` and adjust the -P argument to the appropriate COM port.

For more information in the AVRDUDE parameters, see the [user manual](https://www.nongnu.org/avrdude/user-manual/avrdude_3.html).

These are the base commands which will be repeated and extended in the forthcoming steps.  
Should any errors arise, begin the troubleshooting process by confirming whether your operating system identifies the programming device. If this detection is successful, then the issue likely resides within the RAVA circuit. In such case, the user should review and validate all connections.


### Bootloader

The bootloader is the program that enables firmware updates through the device's USB interface.

Since the MCU is shipped from the factory with LB1 and LB2 lock bits enabled, the fuses are effectively locked, rendering them unmodifiable except through the `-e` (erase memory) command. 

To install the default bootloader, download it from the [manufacturer's website](https://www.microchip.com/en-us/product/atmega32u4#document-table). For more information about the DFU bootloader, refer to its [datasheet](https://ww1.microchip.com/downloads/en/DeviceDoc/doc7618.pdf). Then, upload it with the following command
```
avrdude -v -c usbasp -p m32u4 -e -U flash:w:/path_to_the/ATMega32U4-usbdevice_dfu-1_0_0.hex
```

To initiate the bootloader code, users are required to execute a hardware reset on the RAVA device. This procedure is detailed in a subsequent section that provides instructions on how to upload the firmware onto the device.


### Fuses

After installing the bootloader, the fuses and lock bits should be programmed  with the following command
```
avrdude -v -c usbasp -p m32u4 -U lfuse:w:0xff:m -U hfuse:w:0xd9:m -U efuse:w:0xf3:m -U lock:w:0xce:m
```

The employed fuse values correspond to configuration choices which can be visualized through the use of [this fuse calculator](https://eleccelerator.com/fusecalc/fusecalc.php?chip=atmega32u4&LOW=FF&HIGH=D9&EXTENDED=F3&LOCKBIT=CE)

![Fuse options](https://github.com/gabrielguerrer/rng_rava/blob/main/images/wiki_mcu_fuses.png)

In summary, the specified fuse bit configuration states that
* The MCU will operate within a clock derived from a Low Power Crystal Oscillator of 16MHz (CKSEL and CKDIV8)
* External SPI programmer can be used to interact with the device (SPIEN)
* 4KB reserved for bootloader section (BOOTSZ)
* Triggers Brown-out Detection (BOD) if Vcc < 2.6V (BODLEVEL) 
* Bootloader is accessible only via an external reset (HWBE and BOOTRST)

The lockbit specifies
* LB_MODE_2: External SPI programmers can read Flash/EEPROM/fuses, but cannot write or modify them. A chip erase is required first, which clears the lock bits
* BLB0_MODE_1: Bootloader has unrestricted read and write access to the application section
* BLB0_MODE_3: Application cannot read or write the bootloader section

For more information in the ATmega32u4 fuse parameters, see the the [ATmega32u4 datasheet](https://ww1.microchip.com/downloads/en/DeviceDoc/Atmel-7766-8-bit-AVR-ATmega16U4-32U4_Datasheet.pdf).


## MCU meets OS

Given that the user possesses a RAVA circuit with the MCU correctly configured with the appropriate fuses and bootloader, the test at this stage involves establishing a connection between the RAVA and the user's computer via the USB port, followed by verifying if the operating system detects the device.

On Linux, execute the command `sudo dmesg` and inspect the output for information similar to
```
[ xxxx.xxxxxx] usb x-x: Product: ATm32U4DFU
[ xxxx.xxxxxx] usb x-x: Manufacturer: ATMEL
```

On macOS, access the System Report and go to Hardware -> USB to find the listing for the ATm32U4DFU device.

On Windows, open the Device Manager, where the ATm32U4DFU device should be listed.


## Firmware

The randomness characteristics of the RAVA device can only be tested after installing its firmware. Without the firmware, the circuit does not generate the PWM signal used by the boost converter to produce the high voltage that stimulates the reverse Zener diodes into the avalanche noise.

To install the RAVA's firmware, see the [RAVA Firmware - Installation](https://github.com/gabrielguerrer/rng_rava_firmware/wiki/RAVA-Firmware-%E2%80%90-Installation) wiki section of the project [rng_rava_firmware](https://github.com/gabrielguerrer/rng_rava_firmware/).

For those seeking a deeper comprehension of the firmware's inner workings, refer to the wiki section titled [RAVA Firmware ‐ How it works](https://github.com/gabrielguerrer/rng_rava_firmware/wiki/RAVA-Firmware-%E2%80%90-How-it-works).


## Voltage inspection

Assuming a powered-up RAVA circuit is running the proper firmware, the next important step is to check if the noise sources operate accordingly.
This auditing task should be accomplished before testing the circuit's random byte outputs, as it can unveil issues that might otherwise remain unnoticed.

While a comprehensive assessment necessitates an oscilloscope, we initiate by employing a multimeter to showcase what can be achieved with basic instrumentation. 

### Multimeter

A multimeter can detect if the boost converter circuitry produces the high voltage required to generate the avalanche noise.
When using 24V Zeners, the voltage measurement between the GND and BV headers should register approximately 25.5V.
Any deviation from this value indicate a failure to meet the entropy generation requirements.

If a failure scenario arises, the user should measure the BV signal frequency using an oscilloscope or other equally capable tool.
If it deviates from the anticipated 46.9KHz, the user should re-upload the firmware and conduct another evaluation.
In cases where the frequency is correct, the user should revisit the soldering connections of the boost converter components. 

Moreover, the multimeter can check if the VCC, 5V, and 2.5V header measurements are within their expected voltage ranges.
The VCC reveals the voltage provided by the USB port, while 5V includes a voltage drop introduced by the fuse, which limits the circuit current to 500mA.
Under typical circumstances, VCC reads 5.00V, while the 5V header indicates 4.83V and the 2.5V header results in half of the previous measurement rated at 2.42V. 

### Oscilloscope

The oscilloscope's main application is to evaluate the voltage characteristics of the avalanche and differential noise channels.

Considering Semtechs's MM3Z24 24V Zeners, the **avalanche noise** measured in all four Ai headers should mirror the pattern in the figure below.
The user should look for a randomly changing signal with the following properties
* Centered at the voltage reading of the 2.5V header
* Average peak-to-peak magnitude of approximately 800mV
* Mean frequency of approximately 2.2MHz  
  (The value of 3.1MHz shown in the figure corresponds to an instant value. For better accuracy, look for a trend plot feature in the oscilloscope that calculates the mean over a time window)

![Avalanche Noise DC](https://github.com/gabrielguerrer/rng_rava/blob/main/images/wiki_inspect_avalanche_dc.jpg)

> Avalanche Noise, DC coupling: The oscilloscope is configured with horizontal dotted lines representing 1V increments and vertical dotted lines depicting the interval of 1 us.

It is also possible to configure the oscilloscope to AC decouple the signal, allowing for a more detailed observation of the avalanche breakdown, as illustrated below.

![Avalanche Noise AC](https://github.com/gabrielguerrer/rng_rava/blob/main/images/wiki_inspect_avalanche_ac.jpg)

> Avalanche Noise, AC coupling: The oscilloscope is configured with horizontal dotted lines representing 200mV increments and vertical dotted lines depicting the interval of 1 us.

A given avalanche channel Ai is deemed inactive when it exhibits a signal centered at 2.5V, accompanied by a minor peak-to-peak amplitude.

Troubleshooting Avalanche Noise
* All four channels are inactive: Verifying whether the boost converter supplies a BV voltage exceeding 25V. If not, inspect the boost converter's connections and confirm whether the device operates with the appropriate firmware.
* Single channel failure: Evaluate the corresponding Zener diode orientation, then examine its connection to the operational amplifier.

Meanwhile, the **differential noise** exhibited by the two CMPi headers should resemble the characteristics depicted below. 
The user should look for a randomly changing signal with the following properties
* Digital signal with two possible states, 0V and 5V
* Mean frequency of approximately 3.1MHz  

![Differential Noise](https://github.com/gabrielguerrer/rng_rava/blob/main/images/wiki_inspect_differential_dc.jpg)

> Differential Noise, DC coupling: The oscilloscope is configured with horizontal dotted lines representing 2V increments and vertical dotted lines depicting the interval of 1 us.

Troubleshooting Avalanche Noise
* Check the corresponding operation amplifier connections to the comparator IC.

A common issue is to find one inactive avalanche channel, let's say A1.
Then, as CMP1 compares the operational A2 channel with a constant signal centered at 2.5V, it generates digital pulses linked to the avalanche breakdown events within A2.
Although the resultant differential channel might exhibit adequate entropy to pass various statistical tests, it is not functioning in its intended differential mode. This circumstance leads to losing its inherent resilience against external environmental factors.

## Next steps

By installing the firmware and ensuring the correct operation of the avalanche and differential noise channels, the user is ready to
* Install the [RAVA driver](https://github.com/gabrielguerrer/rng_rava_driver_py) and test the basic functionality
* Collect random bytes and perform statistical tests using the [diagnostics tool](https://github.com/gabrielguerrer/rng_rava_diagnostics_py)