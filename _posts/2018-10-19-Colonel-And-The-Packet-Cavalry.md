---
layout : post
title: 'Colonel And The Packet Cavalry'
category: Networks
tags: Networks TCP/IP
comments: true
exerpt_separator: <!--more--> 
---
Returning after a long break this post gleans through the kernel space jugglery with the network packets. 

<!--more-->

I'm not trying to reinvent any wheel with regards to the network stack. The standard TCP/IP stack continues to be the base, perhaps, this time our beloved kernel comes out with revealing its trickery.

## sk_buff
This is a complex struct which is used throughout the network stack wherever buffers are used inside the kernel. Mainly because it is overprovisioned for being compliant with varied range of protocols. It has fields to point to the header of every subsequent layer as a packet traverses down the network stack. As intended it will store the packet's data and do the necessary book keeping for keeping a track of its associated process.

# Sender
## Application

Any process trying to interact with the network stack starts with initializing a socket belonging to a particular family (which determines the underlying transport layer features available to it.)
Now that the application has a uniquely identifiable socket, any data or message to be sent from the application is written to the socket via its descriptor.  

### Transport Layer Interface 

Any of the `write` or `send` function calls translate to `sock_sendmsg` system call irrespective of the family of the socket.  
`sock_sendmsg` will perform some checks like:
* userspace buffer is readable
* obtain the struct sock* using the socket fd
* generate relevant socket control messages having UID , PID, GID information all applicable to the process which tries to perform a write to the socket buffer.  

Next in line, depending on whether the socket belongs to `SOCK_STREAM` or `SOCK_DGRAM` or any other family a transport layer protocol specific system call is invoked, say `tcp_sendmsg`.

## Transport Layer
The transport layer protocol's initial setup before initiating a communication between the hosts are performed such as a TCP handshake or setting up other state parameters, etc.   
Following this comes the main task of copying the user space data into a sk_buff instance which is nothing but the buffer we usually associate with a socket. It is ensured that if a sk_buff exists and has space to accomodate more data from the userspace buffer it is written to the same.
It is this layer where the (reliable) data transfer of the messages is accounted, which involves adding special fields into the transport layer header of the packet like `SRC_PORT`, `DEST_PORT`, `SEQ_NO`, `ACK` etc.

## Network Layer
The transport layer segment gets encapsulated by an IP header with `SRC_IP` and `DEST_IP`. This layer is also capable of handling routing protocols. If the packet is to be transferred to a local address it is delivered to a higher layer otherwise it travels down to the Link Layer.

## Link Layer
The usual transfer of the datagram to the device is performed by this layer. This layer performs queueing discipline implementation for which it is referred to as the **queueing layer**. It'll queue the datagram (to be precise the `sk_buff` instance) into the device driver's queue.
This queue provides the asynchronous transmission of the datagrams as neither the NIC nor the IP layer has to wait for the other counterpart to be available to service its request of packet transmission or reception.

## Physical Layer
Lastly, the packet data after being marshalled will be sent out on the physical channel. This act of transmitting onto the physical channel involves invoking `ioctl` calls to interact with the NIC.

**It's cleanup time: `sk_buff` is finally freed.**

# Receiver

## Physical and Link Layer
When the packets arrive they are copied to the rx_ring into the kernel using DMA.  
Again `sk_buff` is to be initialized for this packet data. Following which an interrupt is raised tp the processor. But as the interrupt processing has to be minimal the `ISR` initiates a `softirq` which takes the packet processing further.
Afterwards, the multiplexing of the packets is performed and the `sk_buff` instance's data after all the headers have been removed is copied to the userspace buffer. Finally, this results into `sk_buff` being freed.




