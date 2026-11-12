# Modbus Sniffing & Manipulation Lab

## 1. Overview

This lab demonstrates how to capture, analyze, and manipulate unencrypted industrial control system (ICS) network traffic using **OpenPLC Runtime 4**, **mbpoll**, and **Wireshark**. The OpenPLC runtime executes the **simple conveyor belt** PLC program located at [https://github.com/alexkitaa/simple-conveyor](https://github.com/alexkitaa/simple-conveyor), providing realistic process variables, coils, and holding registers to interact with and analyze.

## 2. Environment & Topology

The lab environment consists of **two separate VirtualBox virtual machines** connected on the same virtual network segment:

* **VM 1 – Modbus Server / Slave (OpenPLC Runtime):** `172.16.10.25:502`
* **VM 2 – Modbus Client / Attacker (`mbpoll` + Wireshark):** `172.16.10.5`
* **Protocol:** Modbus TCP

<img width="1408" height="768" alt="network diagram" src="https://github.com/user-attachments/assets/7fa68c71-5764-4fa4-8523-d8516ed1f275" />

> **Packet Capture Available:** A recorded Wireshark packet capture (`.pcap`) of the lab traffic is included in the [PCAP](pcap/) folder of this repository for offline inspection and analysis.

## 3. Wireshark Capture Filter

To isolate the traffic between the target PLC and the testing machine, apply the following display filter in Wireshark:

```wireshark
ip.addr == 172.16.10.5 && ip.addr == 172.16.10.25 && modbus
```
## 4. Lab Test Cases & Execution

### Test Case A: Reading Holding Registers (`FC 03`)
* **Command:**
  ```bash
  mbpoll -m tcp -a 1 -p 502 -t 4 -r 1 -c 10 -1 172.16.10.25
  ```
* **Description:** Reads 10 holding registers starting at address 1 (offset `0x0000`) from Slave ID 1.
* **Wireshark Payload:** Look for Function Code `0x03`, starting address `0x0000`, and quantity `0x000A`. Verify the byte count and returned register values in big-endian format.

### Test Case B: Reading Coils (`FC 01`)
* **Command:**
  ```bash
  mbpoll -m tcp -a 1 -p 502 -t 0 -r 1 -c 10 -1 172.16.10.25
  ```
* **Description:** Reads 10 discrete coils starting at address 1 from Slave ID 1.
* **Wireshark Payload:** Look for Function Code `0x01`, starting address `0x0000`, and quantity `0x000A`.

### Test Case C: Writing a Single Coil (`FC 05`)
* **Command:**
  ```bash
  mbpoll -m tcp -a 1 -p 502 -t 0 -r 1 -1 172.16.10.25 1
  ```
* **Description:** Forces Coil at address 1 (offset `0x0000`) to `ON` (`1`) on Slave ID 1.
* **Wireshark Payload:** Look for Function Code `0x05`, Output Address `0x0000`, and Value `0xFF00` (ON). Confirm the OpenPLC state changes accordingly.

### Test Case D: Writing to Holding Registers (`FC 16` / `0x10`)
* **Command:**
  ```bash
  mbpoll -m tcp -a 1 -p 502 -t 4 -r 1 -1 172.16.10.25 35
  ```
* **Description:** Writes value `35` to the holding register starting at offset 0.
* **Wireshark Payload:** Look for Function Code `0x10`, Register Address `0x0000`, quantity, and hex value encoding.

### Test Case E: Reading Holding Registers After Write (`FC 03`)
* **Command:**
  ```bash
  mbpoll -m tcp -a 1 -p 502 -t 4 -r 1 -c 10 -1 172.16.10.25
  ```
* **Description:** Reads back the 10 holding registers to verify that the value `35` was successfully written and updated in the PLC memory map.

<img width="1362" height="603" alt="modbus_manipulation" src="https://github.com/user-attachments/assets/f826132f-32e5-414b-9716-c844d140611f" />


## 5. Security Findings & Takeaways

1. **Cleartext Exposure:** Because Modbus TCP operates without encryption or authentication, anyone on the network segment can completely inspect sensor readings and actuator states.
2. **Lack of Access Control:** OpenPLC accepted write/force commands from an external IP (`172.16.10.5`) without checking permissions, allowing unauthorized process interference.
