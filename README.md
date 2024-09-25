# LoRaWAN system repository

Keeping track of changes/updates for reports.

## System High Level View

### Component

![high-level-system-overview](https://github.com/user-attachments/assets/fe2ef5cc-c6d2-4814-a8f4-31da2c9e2775)

### Sequence of Normal operation

![flowchart-system](https://github.com/user-attachments/assets/bb2dec51-1d31-49cf-9a15-4e10a0b1d0a3)

### Other Sequence between Gateway and Network server

`PUSH_DATA`: This packet type is used by the gateway mainly to forward the RF packets 
received, and associated metadata, to the server.

`PULL_DATA`: This packet type is used by the gateway to poll data from the server.
- Data exchange is initialized by the gateway because it might be impossible for the server to send packets to the gateway if the gateway is behind a NAT.
- When the gateway initialize the exchange, the network route towards the server will open and will allow for packets to flow both directions.
- The gateway must periodically send PULL_DATA packets to be sure the network route stays open for the server to be used at any time.

For more details checkout the [packet forwader protocol](https://github.com/Lora-net/packet_forwarder/blob/master/PROTOCOL.TXT) by Semtech

![flowchart-gateway-network](https://github.com/user-attachments/assets/6a2ffdba-2304-47cc-87b2-b98031d23510)

## LoRaWAN Architecture

### LoRaWAN Overview
#### Introduction
LoRaWAN networks typically are laid out in a star-of-stars topology in which gateways relay messages between end-devices and a central network server at the backend. Gateways are connected to the network server via standard IP connections while end devices use single-hop LoRa or FSK communication to one or many gateways.

All communication is generally bi-directional, although uplink communication from an end device to the network server is expected to be the predominant traffic. Communication between end-devices and gateways is spread out on different frequency channels and data rates. The selection of the data rate is a trade-off between communication range and message duration, communications with different data rates do not interfere with each other. LoRa data rates range from 0.3 kbps to 50 kbps.

To maximize both battery life of the end-devices and overall network capacity, the LoRa network infrastructure can manage the data rate and RF output for each end-device individually by 16 means of an adaptive data rate (ADR) scheme.
#### Implementation Specific
In this particular LoRaWAN system, some parts are not implemented as specified in LoRaWAN specification as well as the encryption scheme used for Message Authentication Code (MAC):
- Only uplink communication is available for end devices, althought the firmware for the gateway can receive downlink message from the server.
- The LoRaWAN package currently does not implement FOpts bytes - which means end device will not send MAC commands.
- The encryption scheme used for MAC is [Asconmacav12](https://github.com/ascon/ascon_collection) instead of AES-CMAC [RFC 4493].
- The LoRa network infrastructure does not manage the data rate and RF output for each end-device, the end device selects it's desired RF output and data rate.

### Application Server
The Application Server processes application-specific data messages received from end devices. It consists of a front end to display sensor information and data, a database for storage and accessing data.

The front end uses [ReactJS](https://react.dev/learn), [Bootstrap](https://getbootstrap.com/docs/5.3/getting-started/introduction/) and various libraries to deliver a better UI for the user. [Firebase](https://firebase.google.com/docs?_gl=1*r6o2wb*_up*MQ..&gclid=Cj0KCQjwo8S3BhDeARIsAFRmkOM6rhYCSer1fpWr1jg4m0INjS480yrLrGMxlxHO8eSu5HexHxQ3ZgEaAuaYEALw_wcB&gclsrc=aw.ds) is used as the database as it offers many benefits and wide range of support for many platform, tools and user needs.

### UDP Network Server
The Network Server manages gateways, end-devices, applications, and users in the entire LoRaWAN network.

For this system, the statments below are true for Network Server:
- Establishing secure 128-bit AES connections for the transport of messages between end-devices and the Application Server (end-to-end security).
- Validating the authenticity of end devices and integrity of messages.
- Device address checking.
- Providing acknowledgements of uplink data messages.
- Forwarding uplink application payloads to the appropriate application servers
- Routing uplink application payloads to the appropriate Application Server

### Sensor Devices
End devices are wirelessly connected to the LoRaWAN network through gateways using LoRa RF modulation. In this system, the devices are transmitting data in the AS923 (AS2) frequency band as specified in Vietnam's regulations which range from 920 MHz - 923 MHz.

The sensor device usually consist of two main parts: the sensor to get data from the environment and the microcontroller which is used to process the data from the sensor and package the LoRaWAN protocol frame to be transmitted using LoRa RF modulation. Most of the time the microcontroller will be put in Sleep mode. This helps prolong the battery life and only process data when it's necessary.

Currently the microcontroller in used is STM32F103C8T6, as for the sensor it can be any kind depends on the applications or problems we're trying to solve.

## LoRaWAN message format
All LoRa uplink and downlink messages carry a PHY payload (Payload) starting with a single-octet MAC header (MHDR), followed by a MAC payload (MACPayload), and ending with a 4-octet message integrity code (MIC).

**Note:** CRC is only available on uplink messages.

![lora-package drawio](https://github.com/user-attachments/assets/63aea58b-bdca-4299-bc04-4584786a2dcb)

## Ascon
The Ascon family is a set of lightweight cryptographic algorithms designed for efficiency, especially in resource-constrained environments such as IoT devices. It includes authenticated encryption and hashing algorithms. Ascon was selected in 2023 as the NIST lightweight cryptography standard.

It consists of:
- Authenticated encryption schemes with associated data (AEAD)
- Hash functions (HASH) and extendible output functions (XOF)
- Pseudo-random functions (PRF) and message authentication codes (MAC)

### Ascon MAC (Message Authentication Code) for LoRaWAN package
The Ascon AEAD algorithm offer both encryption and message authentication. In the process of generating a ciphertext, Ascon use a built-in mechanism that generates a Message Authentication Code (MAC) to ensure that the message has not been changed during transmission. The MAC is part of the authentication process in Ascon’s AEAD mode.

With the Ascon AEAD algorithm, the outputs are ciphertext and a tag. But with the standalone Ascon MAC algorithm, the output is a 128-bit tag used to authenticate the message.

Leveraging this MAC algorithm, we can replace it with the traditional AES-CMAC used in LoRaWAN package frame.
But does this algorithm provide any benefits over the old one ? Is it faster in term of calculation ? Does it cost a lot of memory space ?
- We will take a look at these questions in another section.

Currently this LoRaWAN system employ the [Asconmacav12 reference implementation](https://github.com/ascon/ascon-c/tree/main/crypto_auth/asconmacav12/ref) (self-contained, portable and very fast).

The message integrity code (MIC) is calculated over all the fields in the message using Ascon algorithm.

**Pseudocode**
```
msg = MHDR | FHDR | FPort | FRMPayload
mac = asconmacav12(NwkSKey, B | msg)
MIC = mac[0..3]

Where B:
B = [0x49, 0x00, 0x00, 0x00, 0x00, Dir, DevAddr[4], FCntUp/Down[4], 0x00, len(msg)]
```
This will look familiar to you if you have implemented AES-CMAC before for LoRaWAN and you're correct ! Only the message authentication function changed, in this case it's `asconmacav12` instead of `aes128_cmac`

### Ascon MAC vs AES-MAC

- Placeholder
