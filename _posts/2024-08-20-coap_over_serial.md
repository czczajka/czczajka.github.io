---
layout: post
title:  "CoAP over serial in IoT"
date:   2024-08-20 13:00:00 +0100
categories: jekyll update
---

## Introduction
CoAP (Constrained Application Protocol) is a lightweight protocol designed for resource-constrained devices. Although it typically operates over UDP, there are instances where serial communication is useful, particularly in embedded systems or for prototyping. This article explores how to implement CoAP over serial communication in Go, with reference to a complete implementation in the repository.

### Setup details
 - Server: Raspberry Pi 4
 - Client: macBook


## Setting up CoAP over serial
To run CoAP over a serial connection, it’s necessary to adapt the default CoAP libraries to work with serial communication. The key steps are configuring the serial port, sending CoAP messages, and handling responses. Below is a high-level overview of the process.

### Server side
The server listens for incoming CoAP requests on the serial port. A serial connection is established using the tarm/serial library, and CoAP messages are read and processed. Here’s a simplified setup for the server:
```
c := &serial.Config{Name: "/dev/ttyGS0", Baud: 9600}
serialConn, _ := serial.OpenPort(c)

for {
    buf := make([]byte, 1024)
    n, _ := serialConn.Read(buf)
    req := pool.NewMessage(context.Background())
    req.UnmarshalWithDecoder(coder.DefaultCoder, buf[:n])

    // Process CoAP request and send response
    handleRequest(serialConn, req)
}
```

### Client side
The client sends CoAP requests to the server over the serial connection. The serial port is similarly configured using tarm/serial. A CoAP message is marshaled and written to the serial port:
```
c := &serial.Config{Name: "/dev/tty.usbmodem1201", Baud: 9600}
s, _ := serial.OpenPort(c)

req := pool.NewMessage(context.Background())
req.SetPath("/a")
data, _ := req.MarshalWithEncoder(coder.DefaultCoder)
s.Write(data)

n, _ := s.Read(readBuf)
msg := pool.NewMessage(context.Background())
msg.UnmarshalWithDecoder(coder.DefaultCoder, readBuf[:n])
```

### Handling Messages on the server side
#### Standard CoAP over UDP
In a typical CoAP server setup using UDP, handling messages is straightforward. You simply need to define your request handler and start the server using the listenAndServe function. Here’s a quick overview:
```
func main() {
	r := mux.NewRouter()
	r.Use(loggingMiddleware)
	r.Handle("/a", mux.HandlerFunc(handleA))
	r.Handle("/b", mux.HandlerFunc(handleB))

	log.Fatal(coap.ListenAndServe("udp", ":5688", r))
}
```
In this standard CoAP server, mux.NewRouter().Handle registers a handler for the "/a" path, and ListenAndServe starts the server on port 5683, processing incoming CoAP requests with minimal setup.

#### Customizing for CoAP over Serial
When using CoAP over a serial connection, the standard network-based routing provided by ListenAndServe isn’t available. While it's possible to use parts of the CoAP library for handling messages, I decided to write my own routing logic to better understand and demonstrate how CoAP over serial works in a more hands-on manner.

##### Custom Router & ServeCOAP
The custom router maps CoAP paths to handlers, and the ServeCOAP function directs the incoming CoAP messages from the serial connection to the correct handler.
```
type Router struct {
    routes map[string]HandlerFunc
}

func (r *Router) Handle(path string, handler HandlerFunc) {
    r.routes[path] = handler
}

func (r *Router) ServeCOAP(conn *serial.Port, req *pool.Message) {
    path, _ := req.Path()
    if handler, ok := r.routes[path]; ok {
        handler(conn, req)
    }
}
```

In this setup, the ServeCOAP function decodes the CoAP message read from the serial port and sends it to the corresponding handler. Each handler processes the request and responds via the same serial connection. Example handler implementation has been shown below:
```
func handleRequest(conn *serial.Port, req *pool.Message) {
    resp := pool.NewMessage(context.Background())
    resp.SetCode(codes.Content)
    resp.SetBody(bytes.NewReader([]byte("Hello World")))
    data, _ := resp.MarshalWithEncoder(coder.DefaultCoder)
    conn.Write(data)
}
```
By implementing this custom logic, I’ve gained deeper insight into how CoAP routing works and can showcase how to manage message flows over serial communication. This approach also offers flexibility when working with non-standard communication methods like serial ports, giving more control over how messages are handled.

## Complete Code Example Repository
For the complete implementation of CoAP over serial, including message handling, routing, and response logic, you can explore the full code in the repository linked below. The repository provides detailed instructions and examples, allowing you to experiment with CoAP over serial communication between a client and server.

Inside the repo, you’ll find both the client and server code, making it easy to set up and test the communication on your own devices. With this setup, you can exchange messages via CoAP, simulating how embedded devices might interact over a serial connection.

To get started, follow the instructions in the README.md file, where all steps for configuring your Raspberry Pi and setting up the serial communication are explained.
[coap over serial repo](https://github.com/czczajka/coap-over-serial)

## Notes
### Enable USB Serial Mode on Raspberry Pi
To set up your Raspberry Pi as a USB serial device, you need to modify two configuration files:
```
# On Raspberry Pi 4 
# Add to the file: /boot/config.txt
dtoverlay=dwc2

# Add to the file: /boot/cmdline.txt
modules-load=dwc2,g_serial

```


## Wrapping Up: CoAP Over Serial – A Simple but Powerful Tool
CoAP over serial communication provides a versatile way to connect and control IoT devices in environments where network connectivity might not be readily available. By adapting the CoAP protocol to work over serial ports, you open up a wide range of possibilities, from local debugging to low-level device interactions in embedded systems.

In this example, I’ve shown how to bridge the gap between serial communication and CoAP, making it possible to easily exchange messages between devices over a simple serial link. Whether you’re working on a small-scale IoT prototype or building out an industrial system, using CoAP over serial can streamline your development process.

But this is just the beginning! By adding security layers like DTLS (which we'll explore in the next part of this series), you can transform this simple communication method into a secure and reliable protocol for even more critical applications.
