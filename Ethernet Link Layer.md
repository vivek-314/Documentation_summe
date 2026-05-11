# Ethernet link layer 

### What us Ethernet?
It is a wired technology used to connect various devices within local network.
Eg. We can connect various computers via ethernet for fast transfer of data.
It defines how data is formatted transmitted and received between devices connected through cables such as twisted pair and fiber optic cables.

Ethernet operated at 
  - Physical layer and 
  - Data link layer
of Open Systems Interconnection (OSI) model which is a standardize 7 stage model use for networking functions, allowing diverse communication systems to communicate.

Daily Devices whicih use ethernet are computers, routers, FPGA boards etc.

## Why ethernet is used
  -Because it provides
    -High speed coomunicationa and data transfer.
    -Low cost
    -standardize data transfer protocols
    -Easy scalable
  - Main use of ethernet was seen in
    -Internet communication
    -Data centers
    -Industrial automation
    -FPGA based networking system

Here is a image giving a small brief of OSI model for refernece.
![OSI Model](OSI-7-layers.jpg)

## What Ethernet link layer does?

It is responsible for
  - Framing data
  - MAC addressing
  - Error detection
  - Media access control

Data at this layer is refered as **Frame**

### **Ethernet Fream Structure**
![Frame Strcture](framestructure.png)

Let's us look at frame structure one by one.

## Preamble
It is a 7 bit alternating pattern of 1's and 0's [1010101..] which gives a heads up to the reciving end about incoming data. It allows the sender and reciver to establish bit synchronization. It allows the receiver to lock onto the data stream before the actual frame begins.

## SFD - Start of frame delimiter 
It is a 1 bit field set to 10101011. It indicates the next bits are actual start of the frame after the preamble ends.

[ ## MAC address
It is a local address with all devices. A 48 bit hardware address assigned to all network devices.
Represented by 12 hexadecimal. 
Eg. 00:1B:2C:5D:3E:2A ]


