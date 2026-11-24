
# Developing OpenWrt support for a new router

This describes my process of investigating a new router and developing initial OpenWrt support for it. It covers hardware identification, firmware analysis, UART access, bootloader investigation, testing existing OpenWrt images and developing a device-specific configuration. The aim is to get OpenWrt running on the router without immediately modifying its permanent firmware or flash storage.

---

## 1. Hardware and software requirements

### 1.1 Hardware

The following hardware is useful when developing OpenWrt support:

- USB-to-UART adapter with selectable voltage levels (1.8V / 3.3V).
- Small Dupont jumper wires.
- Soldering iron (if the router does not have accessible serial pins).
- Multimeter for identifying serial pins and checking voltage levels.
- Cheap router that you don't mind potentially breaking if something goes wrong.
- TFTP server for transferring test firmware to the router. This could be an existing OpenWrt router.

### 1.2 Software

- Linux machine or virtual machine running Debian.
- OpenWrt build environment.
- Git for tracking changes to the OpenWrt source.
- Serial terminal software, such as picocom, minicom, screen or PuTTY.
- Firmware analysis tools, particularly binwalk.

---

## 2. Research and selecting a router

Start by looking for a cheap router on eBay or another second-hand marketplace. Before purchasing, try to establish as much as possible about the hardware.

### 2.1 Research the router

Useful sources include:

- **FCC ID database:** Look for internal photographs of the router, which can reveal the board layout, chip markings and serial connections.
- **WikiDevi:** Check whether the router has already been documented.
- **TechInfoDepot:** Look for hardware specifications and information about the internal components.
- **Manufacturer's website:** Look for firmware downloads and GPL source code.

### 2.2 Check OpenWrt compatibility

Make sure the router uses a System-on-Chip (SoC) that is already supported by OpenWrt. The most promising candidates are generally routers based on Qualcomm or MediaTek platforms, although other supported platforms may also be suitable. Ideally, find an existing OpenWrt-supported router using the same SoC and similar hardware. This will provide a starting point for the device tree, kernel configuration and other platform-specific settings.

---

## 3. Preliminary firmware investigation

Before the router arrives, you can start investigating the manufacturer's firmware if it is available.

### 3.1 Download and analyse the stock firmware

Download the manufacturer's firmware and analyse it using binwalk. Try to extract the firmware and identify its contents. Things to look for include:

- Flattened Device Tree (FDT) files, which can potentially be converted into DTS source files.
- Kernel images.
- Root filesystem images.
- Firmware files required by the Wi-Fi hardware.
- Partition information and other hardware-specific configuration.

The extracted files may provide valuable information later when creating the OpenWrt device tree.

### 3.2 Prepare the build environment

Set up your Debian build machine using the official OpenWrt build instructions. Download the OpenWrt source code and make sure you can successfully build an image for an existing router using the same SoC family. This confirms that your build environment works before you start making hardware-specific changes.

---

## 4. Initial investigation of the router

Once the router arrives, the first priority is to investigate its existing firmware and hardware without making any modifications.

### 4.1 Test the router in its original state

Power on the router and connect it to your PC. Do not connect it to the internet, as this avoids the possibility of automatic firmware updates.

Check the following:

- Is there a working management webpage?
- Can you log in using the default credentials? (A factory reset may be required first.)
- Are there any options to enable SSH or Telnet access?

If SSH or Telnet is available, log in and investigate the running system. Useful commands include:

```bash
dmesg
cat /proc/cpuinfo
cat /proc/meminfo
cat /proc/partitions
cat /proc/mtd
fw_printenv
```

Save the output of each command to text files for later reference. These files provide useful information about the processor, memory, flash partitions, boot process and environment variables.

### 4.2 Open the router and inspect the PCB

If there is no way to access the command line, the next step is to open the router.

- Look for screws hidden underneath rubber feet, labels or stickers.
- Check the FCC ID photographs for clues about screw locations.
- Use guitar picks or plastic spudgers to release the plastic clips.

Before removing the PCB, take photographs of the board in the enclosure. This provides a reference for putting everything back together. Remove any metal shielding cages and take detailed photographs of the larger components. Check both sides of the PCB. Write down all the chip codes and markings, then research them online. The most important components to identify are:

| Component | Why it matters |
|---|---|
| SoC | Determines the OpenWrt platform and kernel support |
| NAND / NOR / eMMC | Identifies the flash storage type |
| RAM | Determines available memory |
| Ethernet switch | Determines network hardware configuration |
| Wi-Fi radio chips | Identifies wireless hardware and firmware requirements |
| Buttons and LEDs | Needed for additional device functionality |

Also look for anything resembling a serial header. This is often a row of four pins or through-holes on the PCB.

---

## 5. Identifying and connecting the UART interface

### 5.1 Identify the serial pins

With the router switched off, use a multimeter in continuity mode to identify the ground pin. The VCC pin sometimes has a small triangle marking next to it, although this is not guaranteed. 
Switch the multimeter to voltage measurement mode. Power on the router and measure the voltage on the suspected serial pins. Common UART logic levels are 1.8V or 3.3V. Identifying the correct voltage is important before connecting the USB-to-UART adapter. Make sure the UART adapter uses the same logic voltage as the router. Using an incompatible voltage can damage the router's components.

**Important:** Do not assume the pinout based on physical markings alone. Verify the connections and voltage levels before attaching the adapter.

### 5.2 Connect the UART adapter

Connect the following:

| Router | USB-to-UART adapter |
|---|---|
| GND | GND |
| TX | RX |
| RX | TX |

**Do not connect the VCC pin.**

Start a serial terminal using a baud rate of 115200. For example:

```bash
picocom -b 115200 /dev/ttyUSB0
```

Power on the router and check whether boot messages appear.

If there is no output:

1. Power off the router.
2. Swap the RX and TX wires.
3. Power it on again.

If the output is garbled, try different baud rates.

### 5.3 Investigate the bootloader

Try to interrupt U-Boot during startup. This is commonly done by pressing Ctrl+C or another key combination displayed on the screen. If you manage to interrupt the boot process, start by examining the bootloader environment by typing:

```bash
printenv
```

Save the output to a text file. Next, display the available commands by typing:

```bash
?
```

Ideally, the bootloader will provide commands such as:

- `tftpboot`
- `usbboot`
- Various hardware information commands.

Run any available `info` commands and save their output. These can reveal important details about the hardware.

### 5.4 Capture the stock boot log

In U-Boot, type:

```bash
reset
```
 
Allow the router to boot normally, capture the complete boot log and save it to a text file. If you are lucky, the router may eventually provide a shell prompt, allowing you to investigate the running system as described earlier. If the command line is password-protected, you may be able to use standard default credentials, but access is not always possible.

---

## 6. Testing existing OpenWrt initramfs images

At this stage, you should have a reasonable understanding of the router's hardware. The next objective is to find an existing OpenWrt image that can boot on the device. The goal is not yet to install OpenWrt permanently. Instead, you want to find an image that runs entirely from RAM.

### 6.1 Find similar supported devices

Look through the OpenWrt Table of Hardware for devices using:

- The same SoC.
- Similar RAM configuration.
- Similar flash storage.
- Similar Ethernet switch hardware.

Download the corresponding kernel-initramfs images from a selection of similar devices and add these images to your TFTP server.

### 6.2 Test the initramfs images

Reboot the router and interrupt U-Boot. Use the bootloader's TFTP functionality to load one of the candidate initramfs images into RAM. Try the different images until you find one that successfully boots. An initramfs image runs entirely from RAM and does not permanently modify the router's flash storage. This makes it a relatively safe way to test hardware compatibility.

Ideally, you want to find an image that:

- Boots successfully.
- Provides at least one working Ethernet port.
- Correctly recognises the flash storage.

Don't worry about Wi-Fi at this stage. It is not an initial priority. Save the complete boot log from each successful test. The boot log is particularly useful because it shows which hardware components are detected and which are not working.

**Note:** An initramfs image from a similar device is not necessarily compatible with your router. Check the hardware configuration carefully and avoid commands that write to flash during initial testing.

---

## 7. Developing an initial device-specific OpenWrt image

Once you have found an initramfs image that boots successfully, you can start developing a proper OpenWrt configuration for your router. At this point, you should have collected:

- Stock firmware files, if available.
- Hardware photographs and chip identification.
- Stock boot logs.
- U-Boot environment variables.
- Information about the flash partitions.
- Results from testing existing initramfs images.
- Boot logs from any successful initramfs tests.

### 7.1 Investigate similar devices in the OpenWrt source

Look through the OpenWrt Git history for recently added devices with similar hardware. This is particularly useful because you can see how other developers implemented support for a new device and which files they needed to add or modify. Pay attention to:

- Device Tree Source (DTS) files.
- Device definitions.
- Image build configuration.
- Kernel configuration.
- Device-specific scripts.
- Flash partition definitions.

On your build machine, create a new Git branch for your router. This ensures that all your changes are tracked separately from the original OpenWrt source.

For example:

```bash
git checkout -b add-new-router
```

### 7.2 Create an initial DTS file

A good starting point is to copy the DTS file from one of the existing initramfs images that successfully booted. Gradually modify it to match the actual hardware of your router.  Use the information collected during the preliminary investigation to adjust the different sections in the dts file. Look at dts files of other devices for examples. Avoid changing too many things at once. Make small adjustments, build a new image and test it on the router. This makes it much easier to identify which changes work and what causes problems.

---

## 8. Useful resources

The following resources are useful throughout the development process:

- [OpenWrt documentation](https://openwrt.org/docs/start)
- [OpenWrt Table of Hardware](https://openwrt.org/toh/start)
- [OpenWrt Firmware Selector](https://firmware-selector.openwrt.org/)
- [OpenWrt Git repository](https://git.openwrt.org/openwrt/openwrt.git)
- [FCC ID search](https://www.fcc.gov/oet/ea/fccid)
- [WikiDevi](https://wikidevi.wi-cat.ru/)
- [TechInfoDepot](https://techinfodepot.shoutwiki.com/)

---


# Using AI (Claude) to develop OpenWrt support for a new router

The following is a workflow I've used to get started with adding OpenWrt support for a new router, using Claude (free tier) as an assistant.

---

## 1. Set up the project and provide context

Start a new project in Claude and upload all the relevant files and information you've collected about the router. This might include:

- Hardware specifications and chipset information.
- Extracted stock DTS file.
- Boot logs and console output.
- Existing DTS files or information from similar supported devices.
- Any other technical documentation you've managed to find.

Give Claude a clear description of the project and establish some ground rules. For example:

> Add OpenWrt support for a new router. Follow OpenWrt coding and style conventions. Keep answers short and provide only a couple of commands or tests at a time. I have a local build machine with Git already configured, but I'm not very experienced with Git, so guide me through the process slowly, using small, manageable steps.

---

## 2. Investigate the available information and create a plan

Ask Claude to analyse the uploaded information, identify what is already known and what is missing, and propose a development plan. The initial goal should be to get a working initramfs image with basic Ethernet and flash support. A working initramfs allows you to boot OpenWrt without modifying the router's existing firmware. Once Ethernet is working, you can use it to make a complete backup of the stock firmware and transfer that backup to your PC for further investigation.

---

## 3. Investigate the stock firmware

Once you have a firmware backup, use tools such as binwalk and other firmware analysis utilities to investigate its contents. Ideally, try to extract the original FDT/DTB or DTS file from the stock firmware. This can provide valuable information about the router's hardware configuration. If you manage to extract an original DTS, provide it to Claude and ask it to compare the information with your current OpenWrt DTS and suggest improvements. Stock firmware DTS files are often based on older kernel versions and may use outdated syntax or conventions. Claude is generally quite capable of translating these into the format and conventions used by current OpenWrt sources.

---

## 4. Work incrementally

Avoid making too many changes at once. Break the development into small, manageable tasks and tackle them individually. For each change:

1. Ask Claude to explain the proposed change before implementing it.
2. Make the change and build a new image.
3. Boot the image on the router using TFTP.
4. Test the functionality and collect relevant boot logs or other output.
5. Share the results with Claude and decide on the next step.

When you get something to work, ask Claude to produce a concise progress summary covering:

- What has been implemented and tested.
- Which files have been changed.
- What is known to work.
- Any remaining problems or limitations.
- Suggested next steps.

Save this summary and add it to the project context. Then start a fresh chat for the next task, using the summary and relevant files as context. Keeping chats relatively short helps prevent Claude from losing track of the original objectives or going off on tangents. It also helps conserve the limited context and token allowance available on the free tier.

---

## 5. Use Git to track your changes

Keep track of all modifications on your local OpenWrt build machine using Git. Since you're working incrementally, make sure you understand which files have changed and why. Ask Claude for guidance with Git commands when necessary, taking things one step at a time. Regularly review your changes so that you can identify and revert problematic modifications without losing previously working functionality. Keep rebuilding and testing new images throughout the process.

---

## 6. Prioritise hardware support

Work through the hardware components in a sensible order. A suggested priority is:

1. **Ethernet:** Get network connectivity working so you can transfer files and firmware backups.
2. **Flash:** Identify and configure the flash device and its partitions.
3. **USB:** Get USB ports and connected devices working.
4. **Buttons:** Configure reset and other physical buttons.
5. **Wi-Fi:** Get the wireless hardware working.
6. **LEDs:** Configure status and activity indicators.
7. **Installation and sysupgrade:** Implement the permanent installation and firmware upgrade procedures.

Keep the initial focus on Ethernet and flash. These are particularly important because they provide the means to back up and recover the original firmware. Leave permanent flashing and sysupgrade functionality until the other components are working reliably.

**Important:** Before attempting any permanent flash operations, make absolutely sure you have a complete and verified backup of the stock firmware and a reliable method of restoring it. Ideally, have a recovery method that does not depend on the router's normal firmware or boot process.

---

## 7. Final review and preparation for submission

Once the router is working and the required functionality has been implemented, ask Claude to perform a final review of all the changed files on your local build machine.

Have it check for:

- Compliance with current OpenWrt coding and style conventions.
- Correct DTS syntax and hardware definitions.
- Unnecessary or redundant changes.
- Correct partition definitions and firmware handling.
- Any missing files or configuration requirements.
- Potential problems with installation and sysupgrade.

Once everything has been reviewed and tested, ask Claude to help combine the changes into a single, well-organised Git commit. The commit should have an appropriate message and include the relevant hardware specifications, supported features, installation instructions and recovery/restore instructions. Finally, push the changes to your GitHub account and open a pull request against the appropriate OpenWrt repository.

---

## 8. Follow up on the pull request

Keep an eye on the pull request for comments and review feedback. OpenWrt developers may request additional information, changes or testing. Work through their feedback, make the necessary adjustments and update the pull request. Be prepared to repeat the build-and-test cycle as required until the submission is ready to be merged.

