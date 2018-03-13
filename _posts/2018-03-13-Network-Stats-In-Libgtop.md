---
layout : post
title: 'Network Stats Makes Its Way to Libgtop '
category: GNOME
tags: GNOME outreachy
comments: true
exerpt_separator: <!--more--> 
---
Hey there if you are reading this then probably network stats might be of some interest to you , but still if it doesn't, just recall that while requesting this page you had your share of packets being transferred over the vast network and delivered to your system. I guess now you'd like to check out the work which has been going on in Libgtop and exploit the network stats details to your personal use.

This post is going to be a brief update about what's new in Libgtop 

<!--more-->
### Crux of the NetStats Implementation
The implementation which I've used requires intensive use of pcap handles to start a capture on your system and segregate the packets into the process they belong to.
The following part is a detailed explanation of this , so in case you feel to just skip the details jump to the [next part](#assigning-packets-to-their-respective-process).

Flow of the setup is as follows:
* Initialize `pcap handles` for different interfaces on a system .
* Start packet dispatching on each `pcap handle`
>Note: These handles have been set to capture packets in a non-blocking mode
* As we get any packet start [processing](#tcp-parsing) it.
* This capture is repeated every time period which is set in the DBus Interface.

### Assigning Packets to their Respective Processes
In my opinion this was the coolest part of the entire project which gave me the liberty to filter the packets until I'd finally atomized them. This felt like a recursive butchering of the packets without having a formal medical science qualification. So bear in mind you can flaunt that you too can operate like doctors but only on inanimate objects 😜 . Well, jokes aside coming to technical aspect -

What the packet says is , keep discarding the headers prepended to it until you reach the desired header, in our case reaching the TCP headers.
The flow of parsing was somewhat simple :

It required checking the following headers in the sequence as below-
* Linktype: Ethernet
* EtherType: 
    * IP
    * IPv6
* TCP 

### A bit of Network Stats design dosage 
Just so that you are in sync with how my implementation seeks to do  what it should do , the design is such :

We know that every process creates various connections. Any general socket detail look like

`src ip (fixed): src port (variable) - dest ip (fixed) : dest port (variable)` 

These sockets are listed in the `/proc/net/tcp`. You'd like to have a more detailed explanation about [this](https://aakp10.github.io/Outreachy-Week2).
The packet headers will give us the connection details , we'll just have to assign it to the correct process.
This means each process has a list of connections , and each connection has a list of packets which in turn has all the relevant packets.
So getting network stats for a process is as simple as summing up the packet length details in the packet list for each connection in the connection list for that process.
> Note: While summing up stale packets aren't considered.

This is what the design looks like:

![structs](public/netstats_design.png)

### TCP parsing 
The parameters passed to this callback are:
* TCP header has information like the `source port` and the `destination port` .
* pcap packet header has the length of the packet 
* The source and destination ip is stored as a member of the `packet_handle` struct .

Given that we have all the necessary details, now the packet is initialized and all its fields are assigned values using the above mentioned parameters passed to the TCP parsing callback.
Now we check which connection to put this new packet into.
For checking this we make use of a reference packet bound to each connection.
After adding the packet to the connection, in case we had to create a new connection itself then we even need to add it to the relevant process . For doing this we use the Inode to PID mapping expalined in the [earlier post](https://aakp10.github.io/Outreachy-Week2).

### Getting stats 
In every refresh cycle getting stats is as simple as just summing up packet lengths for a given process.

### Choosing interface to expose the libgtop API
Next thing of concern was how do we make this functionality available to other applications.
* #### **Daemon already in libgtop for other OSs**
Given the fact that Libgtop already had a daemon to get details which required escalated previleges although it wasn't configured to be run on Linux, but to start the packet capture we needed root permissions , which meant requirement of something like that to enable us to do so.
After discussions with Felipe and Robert it was decided that we start the daemon and access the stats over DBus. 
* #### **Using DBus**
But we encountered some issues launching the capture as a root process , hence decided to switch over to run a systemd service and launch a DBus interface in which not only did we get the stats but the invocation to the capture was also exposed to the application .
But later we'll move on to using the daemon provided by libgtop on Linux so that we are able to get historical stats too.

### API exposed through DBus

![dbus interface](public/dbus_interface.png)

* **GetStats** This returns a dictionary of PID with the corresponding bytes sent and received
* **InitCapture** 
    * This will setup the pcap handles before starting the capture and then invoke the refresh of capture every second but this is subject to change .

    ![InitCapture](public/InitCapture.png)

    * During every refresh cycle the stats are logged into a singleton GPtrArray which is used by GetStats to send the stats over DBus.

![AddToSingleton](public/AddToSingleton.png)

* **Set and Reset capture** This is just to start or stop the capture initiated by InitCapture.
On the Usage end this function is invoked based on whether the network subview is active or not.
While the network subview is active the capture is kept ongoing otherwise we break out of the capture.


### Setting up the system bus
To setup the System bus these two files had to be added to the following paths

* org.gnome.GTopNetStats.service in `/usr/share/dbus-1/system-services`

![service](public/org.gnome.GTopNetStats.service.png)

* org.gnome.GTop.NetStats.conf  in `/etc/dbus-1/system.d`

![conf](public/org.gnome.GTopNetStats.conf.png)

## Inspecting using D-Feet
Here's what we see on inspecting the interface using d-feet

![d-feet](public/d-feet.png)

**GetStats output** : `{pid:(bytes sent, bytes recv)}`

![getstats](public/getstats.png)

You might be done with reading all these implementational details but the most important thing I haven't mentioned until now is everyone whose mind has been behind helping me do all this.

(Keeping it to be in lexical order) 

I'm extremely grateful to Felipe Borges and Robert Roth for their constant support and reviews.

Felipe's design related corrections of switching implementation to say singletons and on the Libgtop end Robert Roth helping me with those quick hacks with the daemon and DBus even after his working hours and from working on weekends to finally getting things done is what makes me indebted to them . 

I've a tint of guilt for pinging you all on weekends too .

Did I just forget to mention the entire community in general , because members like Aberto and Carlos were also the ones I sought help from.

If you did reach this fag end without getting bored let me tell you that I'm yet to post the details about the Usage integration.

Feel free to check the [work](https://gitlab.gnome.org/antares/libgtop/tree/wip/antares/jhbuild_modif).

Stay tuned !🙂


















