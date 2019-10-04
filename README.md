# Naza M Hacking #

##The Hardware##
The FC is based on the LPC1768, an Atmel At88SC3216 crypto memory and several sensors that are irrelevant to this post. On the PCB are programming pads that connect to the LPC1768 pins that are required for in system programming (ISP). With some probing it was possible to determine which pad is connected to which LPC1768 pin.

During boot it was observed that the UART is active (115200 8N1), when running the 'lite' version of the FC firmware the message “active security ok!” along with some other strings were captured. If the AT88SC is disabled (shorting SDA and SCL pins) the message changes to “active security failed!”.

## Boot Sequence ##
By holding P2.10 low while releasing reset the LPC1768 boot ROM enters ISP mode, in the samples I have examined code readout protection (CRP) is set to level 2. CRP2 allows a very limited set of ISP commands, principally CRP2 allows us to erase the LPC1768's flash memory.

In normal operation the LPC1768's boot ROM checks for a valid image in flash, if one is found then control of the CPU passes to user code at flash offset 0, this is the DJI bootloader. Normally the DJI bootloader checks for the presence of an application (a simple blank check on sector 10) and if this succeeds then control of the CPU is passed to the application.

If the application is not present the bootloader starts up its USB interface as a virtual comm port (VCP) and waits commands from the host computer. By linking the F1 & F2 servo connectors we can force the bootloader into recovery mode, the application presence check is skipped and the bootloader starts up the VCP.

## Updating The Application Firmware ##
Updates are downloaded and applied by the PC software DJI Assistant, the PC software performs a check on the currently installed version of the FC application and makes the decision whether to fetch and apply an update.

Using Wireshark it is possible to capture and extract the downloaded binary. When the DJI bootloader is in recovery mode only 3 commands are necessary to download the firmware update and it is relatively straightforward to write a tool to apply an update repeatedly.

The update downloaded from the DJI servers is encrypted and downloaded 'as-is' to the FC in 256 byte chunks, each chunk is acknowledged by the FC. The FC decrypts each chunk and copies it to flash starting at sector 10.

An interesting timing leak was revealed here, the LPC1768 has asymmetric flash sectors. The first 16 sectors are 4KB in size, the remainder are 32KB. The FC typically takes 7ms to process a 256 byte chunk and send an acknowledgement. Some chunks take much longer (70ms) suggesting that the next flash sector is being erased. By comparing the acknowledgement times, amount of data transferred and the flash sector sizes it is possible to deduce that the update is being programmed into flash sectors starting at sector 10. This was subsequently confirmed by reverse engineering the DJI bootloader.

## Obtaining The Update Key ##
I attached wires to the programming pads discussed above and then put the FC into recovery mode, after transferring 4KB of update data I pulled P2.10 low and reset the LPC1768. Using the PC application “Flash Magic” I erased the whole flash and the CRP setting, I was then able to dump the LPC1768's RAM. Searching the RAM dump I identified the AES inverse S-Box and close to it is the update key.

Decrypting the application update using openssl's AES-128-CBC mode yielded a binary file that was mostly correct but had 16 bytes of incorrectly decrypted data every 256 bytes. Further experimentation shows that the application is encrypted in 256 byte chunks with the same key and IV. A simple Python script allows me to encrypt & decrypt the application update

Key: 7F0B9A5026674ADA0BB64F27E6D8C8A6

IV: 00000000000000000000000000000000

Next I performed a simple test to determine if the FC application is authenticated by the bootloader. I modified the “active security ok!” string, re-encrypted it, put the FC into recover mode and using my own tool I was able to update the FC with the modified update. At next boot the modified string was sent by the UART.

The same key/iv pair can decrypt Naza M, Naza M V2 and Phanton 2 firmwares.

## DJI Bootloader ##
Next I patched the application to dump the whole flash to UART, from this dump the DJI bootloader can be extracted. Analysis of the DJI bootloader reveals that it contains the 'secret seed' needed to perform authentication and encrypted data transfers with the AT88SC3216.

## Naza M ##

The secret seed is 'hidden' in the bootloader binary, 32 bit words at flash addresses 0x1188, 0x12FC, 0x1A78 and 0x1C54 are collected into one 16 byte array and AES-128-CBC decrypted.

Key: 82314e66e1d1f513b653d2c6937f3972

IV: 00000000000000000000000000000000

Of the 16 bytes decrypted only 8 are used as the secret seed (SS) the other 8 (xx) are discarded, see below

Decrypted data: SS xx SS xx SS xx SS xx SS xx SS xx SS xx SS xx

Secret seed: SS SS SS SS SS SS SS SS

NOTE: The secret seed is unique to each device, this means that is you use Flash Magic to erase the flash and dump RAM you WILL lose the secret seed and render the device unusable. There is no way to recover the secret seed, a new AT88SC must be programmed and fitted to the device. You have been warned!

## Device Version ##
A lot of data is read from the AT88SC by the application code during bootup, one field is identified as the “version”. If the read fails an attempt is made to write a default value. The defaults that I have found in the FC firmwares so far are:

Naza M – 0x14010000

Naza M V2 – 0x05040506

Phantom 2 – 0x1D03000E

Later in the FC application bootup process a check is made on the MSB of the “version” against a list of allowed versions. If the check fails the FC does not work.

Allowed versions:

Phantom2: 0x05, 0x06, 0x09, 0x14, 0x1D

Naza M: 0x05, 0x06, 0x09, 0x14

Naza M V2 4.02: 0x05, 0x06, 0x09, 0x14

Naza M V2 4.06: 0x05, 0x06, 0x09

Testing reveals that Naza M V2 4.02 firmware will run on the Naza M but 4.02 Naza M V2 firmware will not. When the “version” check in the Naza M V2 4.06 firmware is patched the firmware runs.

## Feature Licensing ##
DJI Naza Assistant for the Naza M and Naza M V2 allow the user to enter a string of 32 characters, DJI refer to this as a serial number. The FC maintains a counter that is initially set to 30, if the user enters an invalid serial number the counter is decremented. When the counter reaches 0 the FC will no longer perform a number of important functions and the user must contact DJI customer services. The DJI assistant application hints at the true purpose of this serial number when it states that you may be asked to enter a new serial number if you purchase additional features.

The serial number is the MD5 hash of a license file, the license file is text based, must be exactly 30 characters long and is of the general form:

YYYY000000000000f4XXXXXXXXXX50

Where:

Y – May be be '1' or '0', setting it to '1' enables the feature. Only the 4 left hand bits can be set in a valid license file

XXXXXXXXXX – The 10 digit serial number of the FC, for example, 1000000123

On the Linux console a valid “serial number” with no feature flags can be generated for my example FC:

$ printf 0000000000000000f4100000012350 | md5sum

64881E8A19F80149B8798A5BB6DBCD80

or, with all four feature flags set:

$ printf 1111000000000000f4100000012350 | md5sum

536DD1A511603E82DE47CAE206EEC0BA

When the user sends a new “serial number” the FC generates all 16 possible license files, calculates the MD5 hashes and compares each one to user's “serial number”. If the “serial number” matches then it is stored in the AT88SC. If it does not match then a counter, also stored in the AT88SC, is decremented.

This brute force search is also performed on the “serial number” stored in the AT88SC during the FC bootup sequence.

The exact use of these feature flags is unknown, however, it should be noted that these flags are one of many things (attempt counter, allowed device version etc) that are checked during FC bootup. It is possible that some illegal flag combinations could render the FC unusable.

Update:
I just checked my Phantom 2 Vision+ FC and the license file hash is the same as the Naza M

