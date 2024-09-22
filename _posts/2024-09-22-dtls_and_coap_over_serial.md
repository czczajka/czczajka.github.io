---
layout: post
title:  "Securing CoAP communication over serial using DTLS"
date:   2024-09-22 13:00:00 +0100
categories: jekyll update
---

## Introduction
fter implementing CoAP over a serial connection, the next step is to add encryption and authentication using DTLS (Datagram Transport Layer Security). This ensures that the communication between the client and server is secure, preventing unauthorized access and ensuring message confidentiality. In this article, we’ll focus on how DTLS fits into the CoAP-over-serial implementation, particularly on handling DTLS-specific issues like MTU sizes and buffer management.


## Handling DTLS Over Serial
The key to securing communication over serial is to add a DTLS layer on top of the serial connection. This layer is responsible for encrypting and authenticating messages using certificates.

When using DTLS, the server and client exchange certificates during the handshake process to authenticate each other. On top of that, DTLS secures all CoAP messages, providing confidentiality and data integrity.

## The Problem with MTU and DTLS
One of the common issues when using DTLS over a serial connection is handling the MTU (Maximum Transmission Unit). DTLS adds extra overhead in terms of security headers, which increases the total size of the packets.

### Why MTU Matters

Serial communication differs from network-based communication because the serial port buffer size is fixed, and exceeding it can cause packet loss or fragmentation. When the MTU is set too high, the DTLS overhead can cause the packet to exceed the buffer size, leading to handshake failures or incomplete message transfers.

In practice, lowering the MTU to around 1024 bytes ensures that packets fit within the buffer, even with the DTLS overhead.
```
const (
    MTU                = 1024
    SERIAL_BUFFER_SIZE = 2048
)
```
This configuration provides a safe balance where the DTLS security headers and CoAP message fit into the buffer without causing errors.

## Custom SerialPacketConn for DTLS
To enable DTLS to function over a serial connection, a custom interface SerialPacketConn is required. This interface mimics packet-based communication (like UDP) over a serial connection.
```
func NewSerialPacketConn(serialPort *serial.Port) *SerialPacketConn {
    return &SerialPacketConn{
        serialPort: serialPort,
        readBuffer: make([]byte, SERIAL_BUFFER_SIZE),
    }
}

func (s *SerialPacketConn) ReadFrom(p []byte) (n int, addr net.Addr, err error) {
    n, err = s.serialPort.Read(p)
    addr = &net.UDPAddr{IP: net.IPv4zero, Port: 0} // Dummy address
    return n, addr, err
}
```
This interface ensures that DTLS can work as expected over the serial connection by simulating packet-based transmission, handling reads and writes through the serial port while providing dummy network addresses for compatibility with DTLS libraries.

## Complete Code Example Repository
The full implementation of CoAP over serial with DTLS, including server and client code, is available in the repository. You can clone the repository to explore the code in detail, test it on your own setup, and adjust the serial paths according to your hardware.

The repository includes all the necessary files to test the handshake, message exchange, and secure communication using DTLS and CoAP over a serial connection.
[dtls serial repo](https://github.com/czczajka/coap-over-serial)

## Conclusion
Securing CoAP communication over serial using DTLS provides encryption and authentication, ensuring the integrity and confidentiality of messages. By addressing key challenges like MTU size and buffer management, we ensure that the communication between the client and server remains reliable and secure, even in resource-constrained environments.
