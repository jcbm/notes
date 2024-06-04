# Chapter 2 - What is a socket?

Everything in Unix is a file.\
A *file descriptor* is simply an integer associated with an open file - *file* meaning network connection, a FIFO, a pipe, a terminal, a real on-the-disk file, or just about anything else.\
When you communicate with another program over the Internet you do it through a file descriptor.

When you call to the socket() system routine, a socket descriptor is returned through which you communicate using 
- send() 
- recv() 

socket calls.

The normal file descriptor methods read() and write() could also be used, but send/recv offer greater control over data transmission.

Many types of sockets:  DARPA Internet addresses (Internet Sockets), path names on a local node (Unix Sockets), CCITT X.25 addresses (X.25 Sockets), etc

## 2.1 Two Types of Internet Sockets

- **Stream**
-- SOCK_STREAM
-- reliable two-way connected communication streams
--- Two items transmitted through the socket in the order “1, 2” will arrive in the order “1, 2”
--Error free
- **Datagram** ("connectionless socket")
-- SOCK_DGRAM

Applications that need things to arrive in a fixed order (e.g. characters in the order in which they were typed) use stream sockets - telnet, ssh, HTTP, etc\
Stream sockets achieve a high level of data transmission quality using TCP.\
TCP makes sure your data arrives sequentially and error-free.\
The IP part of TCP/IP deals primarily with Internet routing - generally not responsible for data integrity.


Datagram sockets also use IP for routing, but use UDP instead of TCP.\
Called *Connectionless* because you don’t have to maintain an open connection as you do with stream sockets.
You build a packet, slap an IP header on it with destination information, and send it out. No connection needed.

Generally used when
- a TCP stack is unavailable
- a few dropped packets here and there is tolerable 

tftp (trivial file transfer protocol, a little brother to FTP), dhcpcd (a DHCP client), multiplayer games,streaming audio, video conferencing, etc.

tftp and similar programs have their own protocol on top of UDP.\
In the tftp protocol, for each packet that gets sent, the recipient has to send back a acknowledging packet (“ACK”).\
If the sender of the original packet gets no reply in, say, five seconds, he re-transmits the packet until he finally gets an ACK.\
Very important when implementing reliable SOCK_DGRAM applications.

For unreliable applications (games, audio, or video) you simply ignore dropped packets, or try to cleverly compensate for them.\
Mainly used for performance - way faster to fire-and-forget than it is to keep track of what has arrived safely and make sure it’s in order.\
If you’re sending chat messages, use TCP.\
If you’re sending 40 positional updates per second of the players in the world, it doesn’t matter so much if one or two get dropped, UDP is a good choice.

## 2.2 Low level Nonsense and Network Theory

Data encapsulation:\
As a packet is created, it gets wrapped (“encapsulated”) in a header (and rarely a footer) by the first protocol (say, the TFTP protocol), then the whole thing (TFTP header included) is encapsulated again by the next protocol (say, UDP), then again by the next (IP), then again by the final protocol on the hardware (physical) layer (say, Ethernet).

When another computer receives the packet
- hardware strips the Ethernet header
- the kernel strips the IP and UDP headers
- the TFTP program strips the TFTP header and finally has the data

The Layered Network Model (aka “ISO/OSI”) describes a system of network functionality that has many advantages over other models.\
For instance, you can write sockets programs that are exactly the same without caring how the data is physically transmitted (serial, thin Ethernet, AUI, whatever) because programs on lower levels deal with it for you.\
The actual network hardware and topology is transparent to the socket programmer.
- Application
-- where users interact with the network.
- Presentation
- Session
- Transport
- Network
- Data Link
- Physical
-- the hardware (serial, Ethernet, etc.)

 A layered model more consistent with Unix might be:
- Application Layer (telnet, ftp, etc.)
- Host-to-Host Transport Layer (TCP, UDP)
- Internet Layer (IP and routing)
- Network Access Layer (Ethernet, wi-fi, etc)

Evident how these layers correspond to the encapsulation of the original data.

All you have to do for stream sockets is send() the data out.\
All you have to do for datagram sockets is encapsulate the packet in the method of your choosing and sendto() it out.\
The kernel builds the Transport Layer and Internet Layer on for you and the hardware does the Network Access Layer. 