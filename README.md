# ESP32 UART and Flash Hack 

There are three serial ports on the original ESP32. UART0 is often used for programming and communication with outside of ESP32 while the other two UART ports are unused. The Micro-USB connector of our IoT kit is connected to UART0 through a USB-UART bridge chip. If there is no protection of UART, a hacker can access the flash through UART. The access to UART can be protected by a password. That is, a user must enter the correct password to upload firmware and communicate with the IoT kit. 

Please read the introduction to the [WiFi station example](https://github.com/espressif/esp-idf/tree/master/examples/wifi/getting_started/station) at GitHub. On the Ubuntu VM, through VS Code, we can build the /home/iot/esp/esp-idf/examples/wifi/getting_started/station/ example, which connects the IoT kit to a wireless router (often called AP too), and flash the firmware onto the IoT kit.


We now can perform ethical hack of the IoT kit, try to obtain the WiFi credentials embedded in the firmware and even change the firmware. 

## Retrieve partition table
Please refer to [the use of esptool.py](https://github.com/espressif/esptool). The following command will retrieve the partition table of the IoT kit flash in the binary format:
```
esptool.py read_flash 0x8000 0xc00 ptable.img
```
where 0x8000 is the start address of the partition table and 0xc00 is the length of the partition table. The binary partition table is saved in ptable.img. 

Please refer to [the use of gen_esp32part.py](https://docs.espressif.com/projects/esp-idf/en/v3.0-rc1/api-guides/partition-tables.html). The following command will print out the partition table of our IoT kit in the CSV (comma-separated values) format. The partition table shows how the flash is partitioned. 
```
gen_esp32part.py ptable.img
```

## Retrieve firmware
The following command retrieves the whole flash content although the student can also refer to the partition table and print out only the occupied part of the flash.
```
esptool.py read_flash 0 0x400000 flash_contents.bin
```
where 0 is the starting address and 0x400000 is the length of the flash of the ESP32-WROOM-32 surface-mount module board that our IoT kit uses. The whole flash in the binary format is saved in flash_contents.bin. 

## Use hex editor to view the firmware and search for sensitive info
Students can use a hex editor (e.g. wxhexeditor) to search the WiFi credentials in the flash dump. The following commands show how to install and configure wxhexeditor.
```
sudo apt-get install wxhexeditor                      #Install wxhexeditor  and then run wxHexEditor
sudo ln -s /usr/bin/wxHexEditor /usr/bin/wxhexeditor  #Create a symbolic to use the lowercase command wxhexeditor
```

## Change the firmware
The hex editor (e.g. wxhexeditor) can be used to change the flash dump. The changed flash dump can be flash back to the IoT kit. Another firmware may be written. esptool.py can be used to write the changed firmware back to the ESP32.
```
esptool.py write_flash 0 flash_contents_good.bin
```

# Notes

[HxD](https://mh-nexus.de/en/hxd/) is a better freeware hex editor and disk editor for Windows than wxhexeditor.
