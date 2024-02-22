# ESP32 UART and Flash Hack 

In this lab, students will learn how to ethically hack the flash memory of a ESP32 through it's UART port.

There are three serial ports on the original ESP32. UART0 is often used for programming and communication with devices outside of the ESP32 while the other two UART ports are unused. The Micro-USB connector of our IoT kit is connected to UART0 through a USB-UART bridge chip. If there is no protection of the UART0 interface, a hacker can access the flash through UART0. The access to UART can be protected by a password. That is, a user must enter the correct password to upload firmware and communicate with the IoT kit. 

If the UART ports and flash of a device are not protected, various exploits are possible.

## 1. Build and run the WiFi station firmware

In this lab, we will work on the [WiFi station example](https://github.com/espressif/esp-idf/tree/master/examples/wifi/getting_started/station) in our environment located at */home/iot/esp/esp-idf/examples/wifi/getting_started/station/*. This ESP-IDF project has been included in this repository for the students convince. On our Ubuntu VM, through VS Code, we can build this project, which connects the IoT kit to a wireless router (often called AP), and flash the firmware onto the IoT kit.

### Open the WiFi station example project
Within VS Code, open the tab *File* -> *Open Folder* -> Navigate to */home/iot/esp/esp-idf/examples/wifi/getting_started/station/* -> *Open*

<img src="https://user-images.githubusercontent.com/69218457/223019335-9143af50-390c-4526-a18d-9cd49f96c984.png" width=640>

### Configure WiFi

Once the WiFi station example project is opened, within VS Code, click the *ESP-IDF menuconfig* icon as shown in the screenshot below, then use the search bar to find the WiFi options. Configure the *WiFi SSID* and *WiFi Password* then *Save* as shown in the screenshot below.

<img src="Imgs/WiFi-Station.PNG" width=640>

### Build, flash and monitor

Please refer to the screenshot above, build the project, flash device and monitor device.
1. Build Project. *Show Running Tasks* to see the progress while building the project.
2. Flash device. Choose the right USB port, e.g., /dev/ttyACM0.
3. Monitor device. You shall see the ESP32 device is connected to the WiFi as the screenshot shown below.

<img src="Imgs/WiFi-Station-Connected.PNG" width=640>

## 2. Retrieve partition table
We now can perform the ethical hacking of the IoT kit, we will try to obtain the WiFi credentials embedded in the firmware and even modify the firmware. This has to be done within a terminal. We will use the Linux terminal to do it.

### Start ESP-IDF terminal
**Close VS Code**. Otherwise, there will be errors?

Open a Linux terminal. Within the terminal, run the following command to set environment variables for all the ESP-IDF tools.
```sh
. $HOME/esp/esp-idf/export.sh
```

### Read flash

Please refer to [the use of esptool.py](https://docs.espressif.com/projects/esptool/en/latest/esp32/) for more information. The following command will retrieve the partition table of the IoT kit flash in the binary format:
```
esptool.py read_flash 0x8000 0xc00 ptable.img
```
where 0x8000 is the start address of the partition table and 0xc00 is the length of the partition table. The binary partition table is saved in ptable.img. 

**Note**: Some esp-idf versions have a bug with the Python tool *esptool.py* while our VM is configured right. They will need the following shebang line added to the start of the code at /home/iot/esp/esp-idf/components/esptool_py/esptool/esptool.py if the shebang line is missing. Any text editor can be used to add this line, one example is *nano*.
```
#!/usr/bin/env python
```

### Print out partition table

Please refer to [the use of gen_esp32part.py](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-guides/partition-tables.html) for more information. The following command will print out the partition table of our IoT kit in the CSV (comma-separated values) format. The partition table shows how the flash memory of the ESP32 is partitioned. 
```sh
gen_esp32part.py ptable.img
```

## 3. Search firmware for sensitive info
### Retrieve firmware
The following command retrieves the entire flash memory of the device although students can also refer to the partition table and print out only the occupied part of the flash.
```sh
esptool.py read_flash 0 0x400000 flash_contents.bin
```
The first numeric value 0 is the starting address and 0x400000 is the length of the flash to copy which is the total size of the ESP32-WROOM-32 surface-mount module board's flash that our IoT kit uses. The entire flash in the binary format will be saved in the flash_contents.bin file. 

### View firmware and search for sensitive info
Students can use a hex editor (e.g. wxhexeditor) to search the WiFi credentials in the flash dump. 

**wxhexeditor is already installed in our Ubuntu VM**. You don't have to reinstall the program if it is already present. However, if needed, you can use the following commands to install and set up wxhexeditor.
```sh
sudo apt-get install wxhexeditor                      #Install wxhexeditor  and then run wxHexEditor
sudo ln -s /usr/bin/wxHexEditor /usr/bin/wxhexeditor  #Create a symbolic to use the lowercase command wxhexeditor
```

This can be launched from the application page of Ubuntu, or the *Terminal*. The following is how to launch wxhexeditor and open a file "flash_contents.bin" from the *Terminal*.
```sh
wxhexeditor flash_contents.bin
```

The following screenshot shows how to use *Edit* -> *Find* within wxhexeditor to search for some text.

<img src="Imgs/wxhexeditor.PNG" width=640>


**Notice**: When you run wxhexeditor, you may get an error when using the *find* utility as shown below. Click *Continue*, otherwise the program will exit.
<img src="Imgs/WXhex-Error.png" width = 640>

## 4. Change the firmware
### Option 0
The hex editor (e.g. wxhexeditor) can be used to change the flash dump. The changed flash dump can be flashed back into the IoT kit. Another different firmware may be written to the device as well. The esptool.py program can be used to write the modified firmware back to the ESP32.


```sh
# Write to device
esptool.py write_flash 0 flash_contents_all_changed.bin
```

It should be noted that the ESP32 utilizes a *checksum hash* to verify the factory APP partition. Unless this value is changed, the bootloader will panic on startup, and fail to run the changed APP partition. It is possible for us to modify the bootloader if *secure-boot* is not enabled.

### Option 1
The option is locating the *checksum hash*, modifying it, and writing the modified firmware to the device. 

### Option 2
The option involves configuring the project bootloader to not verify the APP partition using menuconfig. This is accessed through the gear button at the bottom of the VS Code page, or through the command ```idf.py menuconfig``` in a terminal.  

```sh
# Bootloader Config -> Skip image validation always
idf.py menuconfig 
```

After this, we can re-flash the entire firmware, and re-extract the binary, modify it, and write it with the command below.

```sh
# Extract Firmware
esptool.py read_flash 0 0x40000 flash_contents_all.bin

# Write to device
esptool.py write_flash 0 flash_contents_all_changed.bin
```

<!--
Alternatively, we can re-flash *only* the bootloader with the modifications made by our new configuration. For this we would want to dump data starting from the ```0x9000``` address offset, as we want to write everything *but* the bootloader.

An example process is provided.
1. Extract Firmware (Excluding the bootloader)
    ```sh
    esptool.py read_flash 0x9000 0x31000 flash_contents_no_boot.bin
    ```
2. compile new bootloader
    ```sh
    idf.py bootloader
    ```
3. Flash bootloader
    ```sh
    idf.py bootloader-flash
    ```
4. Write the modified firmware
    ```sh
    esptool.py write_flash 0x9000 flash_contents_no_boot_modified.bin
    ```
-->
# Notes

[HxD](https://mh-nexus.de/en/hxd/) is a better freeware hex editor and disk editor for Windows than wxhexeditor.
