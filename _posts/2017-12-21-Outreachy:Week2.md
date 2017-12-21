---
layout : post
title: 'Outreachy Week 2: Getting Connection Details for Network Processes '
category: GNOME
tags: GNOME outreachy
comments: true
exerpt_separator: <!--more--> 
---

This blog post summarizes my progress until the second week of Outreachy. 
Mainly over these two weeks I've worked on fetching the following details which will eventually help to associate packets with their corresponding processes .   
<!--more-->

I've made extensive use of `/proc ` file system to fetch these details.

### Fetching a List of Sockets on different interfaces

The virtual files present in `/proc/net/` have details about our system's network configuration and `/proc/net/tcp` and `/proc/net/udp` in particular have details about the sockets which have been created to transfer data with the respective tranport protocol .
Appropriate data structures have been used to store the socket entries of these files after parsing the necessary details:
{% highlight c %}
/*
*HEADER OF /proc/net/tcp
*sl local_address rem_address t tx_queue rx_queue
*tr tm->when retrnsmt uid timeout inode    
**/
int matches=sscanf(buf,"%*X: %64[0-9A-Fa-f]:%X %64[0-9A-Fa-f]:%X %*X"
                        "%*X:%*X %*X:%*X %*X %*d %*d %ld %*512s\n",
                        temp_local_addr,
                        &(temp_socket->local_port),
                        temp_rem_addr,
                        &(temp_socket->rem_port),
                        &(temp_socket->inode));
{% endhighlight %} 

### Mapping Socket Inode to its PID 

Although `/proc/net/tcp` and `/proc/net/udp` do have the socket's inode but they don't have the `PID` information required to map the packets to processes. Therefore a traversal of the `/proc` dir was done to map the inode with the process to which it belonged to.  

### Sample Output from the Test 
For those interested to have a look at the details which have been fetched through these steps, here is a sample output from a test file :
{% highlight c %}
/*TCP SOCKET DETAILS*/
proc_name:jekyll inode: 535408 pid: 5536 local_addr: 127.0.0.1 rem_addr: 0.0.0.0 local_port: 4000 rem_port: 0 sa_family: 2 

proc_name:mysqld inode: 24445 pid: 1142 local_addr: 127.0.0.1 rem_addr: 0.0.0.0 local_port: 3306 rem_port: 0 sa_family: 2 

proc_name:systemd-resolve inode: 24220 pid: 1135 local_addr: 0.0.0.0 rem_addr: 0.0.0.0 local_port: 5355 rem_port: 0 sa_family: 2 

proc_name:master inode: 24471 pid: 1567 local_addr: 0.0.0.0 rem_addr: 0.0.0.0 local_port: 25 rem_port: 0 sa_family: 2 

proc_name:firefox inode: 693964 pid: 2515 local_addr: 192.168.43.126 rem_addr: 216.58.199.162 local_port: 45994 rem_port: 443 sa_family: 2 

{% endhighlight %}
Next thing in line which I’m working on is the packet capture on different interfaces.  
Feel free to check out my [work](https://gitlab.gnome.org/antares/libgtop/tree/wip/netsockets).    
Stay tuned !