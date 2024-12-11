# ZTE F6005 GPON ONT Modding Project

## Overview
This repository documents my **independent research and progress** in modding the ZTE F6005 GPON ONT and his *father* CIG G-97CP. The goal is to unlock advanced features, customize functionality, and explore the firmware and hardware of this type of GPON devices.

## Objectives
- **Understand the internal workings** including bootloader, kernel, and root filesystem.
- **Unlock advanced features** such as serial port access, configuration changes, and compatibility with ISPs.
- **Develop tools and methods** to modify the firmware and hardware for educational and experimental purposes.

## Current Progress
1. Successfully dumped the NAND from the device using a CH341A programmer.
2. Analyzed the firmware using tools such as Binwalk, Firmware-Mod-Kit, and Jefferson.
3. Investigated and compared components with similar ONTs (e.g., Nokia G-010G-T and, of course, CIG G-97CP).
4. Replaced and tested bootloaders and root filesystems with partially successful results.
5. Currently working on integrating a custom kernel to resolve boot issues and laser driver compatibility.

## Tools Used
- **Binwalk**: For analyzing and extracting firmware components.
- **Firmware-Mod-Kit**: For unpacking and modifying firmware.
- **Flashrom**: For reading and writing the NAND flash memory.
- **CH341A Programmer**: For direct hardware access to the flash memory.
- **Tera Term / PuTTY**: For serial port communication.

## Challenges
- **Firmware Compatibility**: Creating a functional firmware that integrates correctly with the device's hardware and optical transceiver.
- **Hardware Differences**: Managing discrepancies between components on the ZTE F6005 and similar devices like the Nokia G-010G-T.

## Disclaimer
This project is for educational and experimental purposes only. Use the information and tools here responsibly and at your own risk. To be updated...
