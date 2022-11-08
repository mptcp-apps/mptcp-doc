.. _chapter-linux:

Using Multipath TCP on recent Linux kernels
===========================================

The first version of Multipath TCP on Linux was an off-tree patch intially developed by UCLouvain researchers :cite:`mptcp-kernel`. This implementation was initially the reference implementation of Multipath TCP. It influenced the design of the protocol as new features were always tested on this implementation. You can find additional information about this implementation on `https://www.multipath-tcp.org <https://www.multipath-tcp.org>`_.

Starting with version 5.6, the official Linux kernel includes support for Multipath TCP. The set of features supported by this implementation has increased over time as shown by its `ChangeLog <https://github.com/multipath-tcp/mptcp_net-next/wiki>`_.

To avoid any interference with regular TCP, this implementation only creates a Multipath TCP connection if the application has created its ``socket`` using the ``IPPROTO_MPTCP`` protocol. Applications will probably be modified in the coming months and years to add specific support for Multipath TCP, but in the mean time, the Multipath TCP developers have created a work around to force legacy applications to use Multipath TCP with the ``mptcpize`` command which is bundled with the `mptcpd <https://intel.github.io/mptcpd/>`_ daemon. We use this approach in this section and discuss applications with native Multipath TCP support `later <native-mptcp-linux>`_.

To illustrate Multipath TCP, we use a very simple setup with a Linux client using Ubuntu 22 and a Linux server using Debian. The client uses Linux kernel version 5.15 while the server uses version 5.17. The server has a single network interface with an IPv4 and an IPv6 address. The client has both a Wi-Fi and an Ethernet interface. These two interfaces are connected to the same router that allocates IP addresses in the same subnet on both interfaces. The client has both an IPv4 and an IPv6 address.

.. todo add figure

.. spelling:word-list::
   netcat
   Netcat
   

Enabling Multipath TCP
----------------------

Multipath TCP is a feature that needs to be compiled inside the kernel. If you compile your own kernel, you can manually select Multipath TCP. 

Most users prefer to rely on already compiled Linux kernels that are included in their distribution. The following distributions support Multipath TCP:

 - CentOS starting with
 - Debian starting with 
 - Ubuntu starting with 22.04

   
You need to to install a recent kernel to benefit from Multipath TCP. On some distributions, this installation will be part of the regular upgrade. On other distributions, you will need to add it manually.

Once the kernel has been installed and your computer has rebooted, you first need to verify that Multipath TCP is enabled.

.. code:: console

	 sudo sysctl -a | grep mptcp.enabled
	 net.mptcp.enabled = 1

Here, the kernel supports Multipath TCP. If, for any reason, you want to disable Multipath TCP, you need to set this ``sysctl`` variable to ``0``.


To illustrate the basic operation of ``mptcpize``, let us first use the netcat command over the loopback interface. This is obviously not the target use case for Multipath TCP, but a nice way to perform tests.

Netcat allows to easily launch clients and servers. We start the server using: ``mptcpize run nc -l -p 12345``. This is a TCP server that listens on port ``12345``. The client connects to this server using the ``mptcpize run nc 127.0.0.1 12345`` command. The connection is established and all text lines sent by the client are printed by the server on standard output.

.. code:: console

	   # mptcpize run nc -l -p 12345
	   Simple test

There are several ways to check that Multipath TCP is used for this connection. First, the ``ss`` command provides information about the status of the different sockets.

.. spelling:word-list::

   snd_una
   rcv
   snd
   una
   
.. code:: console

	  ss -iaM
	  State    Recv-Q    Send-Q       Local Address:Port        Peer Address:Port    Process
	  ESTAB    0         0                127.0.0.1:12345          127.0.0.1:52854    
	           subflows_max:2 remote_key token:5bba80d9 write_seq:2266a099179e2476 snd_una:2266a099179e2476 rcv_nxt:de9999038d0a29a2
	  ESTAB    0         0                127.0.0.1:52854          127.0.0.1:12345    
	           subflows_max:2 remote_key token:c1f12b87 write_seq:de9999038d0a29a2 snd_una:de9999038d0a29a2 rcv_nxt:2266a099179e2476

ss provides several useful information to debug a Multipath TCP connection. The first column indicates that the connection is in the Established state, which means that it can currently transfer data. It also indicates the length of the Send and Receive queues at the TCP level and the four-tuple that identifies the connection. The next line provides Multipath TCP information with the maximum number of subflows which can be attached to the connection, the token assigned by the remote host and the write_seq, snd_una and rcv_next parameters of the sate machine. The next two lines provide information about the other direction of the connection.

It is also possible to capture packets on the loopback interface to verify that Multipath TCP is used. The output below provides the first collected packets:

.. code:: console

   tcpdump: verbose output suppressed, use -v[v]... for full protocol decode listening on lo, link-type EN10MB (Ethernet), snapshot length 262144 bytes
   18:43:42.676396 IP 127.0.0.1.52854 > 127.0.0.1.12345: Flags [S], seq 904893125, win 65495, options [mss 65495,sackOK,TS val 4026038040 ecr 0,nop,wscale 7,mptcp capable v1], length 0
   18:43:42.676426 IP 127.0.0.1.12345 > 127.0.0.1.52854: Flags [S.], seq 1804351310, ack 904893126, win 65483, options [mss 65495,sackOK,TS val 4026038040 ecr 4026038040,nop,wscale 7,mptcp capable v1 {0x45edb502d861e7b1}], length 0
   18:43:42.676472 IP 127.0.0.1.52854 > 127.0.0.1.12345: Flags [.], ack 1, win 512, options [nop,nop,TS val 4026038040 ecr 4026038040,mptcp capable v1 {0xdbb760db1d55e07b,0x45edb502d861e7b1}], length 0
   18:44:59.519697 IP 127.0.0.1.52854 > 127.0.0.1.12345: Flags [P.], seq 1:13, ack 1, win 512, options [nop,nop,TS val 4026114884 ecr 4026038040,mptcp capable v1 {0xdbb760db1d55e07b,0x45edb502d861e7b1},nop,nop], length 12
   18:44:59.519755 IP 127.0.0.1.12345 > 127.0.0.1.52854: Flags [.], ack 13, win 512, options [nop,nop,TS val 4026114884 ecr 4026114884,mptcp dss ack 16040019788386937262], length 0

		   
The first packet is the ``SYN`` that includes the ``MP_CAPABLE`` option. The server replies with the ``SYN+ACK`` with the ``MP_CAPABLE`` containing the server key. The client returns the third ``ACK`` with the ``MP_CAPABLE`` and the two keys. As the server did not send any data, the ``MP_CAPABLE`` option is sent again in the packet containing the ``Simple test`` string. This packet also contains the ``DSS`` option. The server replies with an acknowledgment that carries the ``DSS`` option.  	   


We can now use the netcat application to explore the operation of Multipath TCP over the Internet. Let us start with a very simple example.

.. code:: console

   mptcpize run nc serverv4 12345
   hello
	  

The netcat process listens on port 12345 on the server. This results in the following Multipath TCP connection :

.. code:: console

   
   09:05:23.695876 IP host-78-129-5-171.dynamic.voo.be.41510 > serverv4.12345: Flags [S], seq 3525674027, win 64240, options [mss 1460,sackOK,TS val 2619832768 ecr 0,nop,wscale 7,mptcp capable v1], length 0
   09:05:23.696076 IP serverv4.12345 > host-78-129-5-171.dynamic.voo.be.41510: Flags [S.], seq 1745741580, ack 3525674028, win 65160, options [mss 1460,sackOK,TS val 3069340264 ecr 2619832768,nop,wscale 7,mptcp capable v1 {0x82aa42ef4245f0d0}], length 0
   09:05:23.707909 IP host-78-129-5-171.dynamic.voo.be.41510 > serverv4.12345: Flags [.], ack 1, win 502, options [nop,nop,TS val 2619832783 ecr 3069340264,mptcp capable v1 {0x9dc8e3972e3d9f25,0x82aa42ef4245f0d0}], length 0
   09:05:30.776312 IP host-78-129-5-171.dynamic.voo.be.41510 > serverv4.12345: Flags [P.], seq 1:7, ack 1, win 502, options [nop,nop,TS val 2619839851 ecr 3069340264,mptcp capable v1 {0x9dc8e3972e3d9f25,0x82aa42ef4245f0d0},nop,nop], length 6
   09:05:30.776484 IP serverv4.12345 > host-78-129-5-171.dynamic.voo.be.41510: Flags [.], ack 7, win 510, options [nop,nop,TS val 3069347345 ecr 2619839851,mptcp dss ack 1561335003985645838], length 0


This is a Multipath TCP connection since it includes the Multipath TCP options, but the client does not create an additional subflow and the server does not announce its other addresses. This is the expected behavior since these operations are controlled by the path manager. On Linux, the Multipath TCP path manager can be configured using the `ip-mptcp <https://man7.org/linux/man-pages/man8/ip-mptcp.8.html>`_ command. This command can be used to configure different parameters that are associated to an IP address. The path manager associates a numeric identifier to each IP address or endpoint. The ``ip mptcp endpoint show`` command lists the identifiers of the active IP addresses on the host. Here is an example of the output of this command on our client:

.. code:: console

   ip mptcp endpoint show
   fe80::3934:7572:b1ff:b555 id 1 dev wlp3s0 
   192.168.0.43 id 2 dev wlp3s0 
   fe80::5642:39bd:3390:43d3 id 3 dev enp2s0 
   192.168.0.37 id 4 dev enp2s0 
   2a02:2788:10c4:123:3d66:f590:d891:8fb3 id 5 dev wlp3s0 
   2a02:2788:10c4:123:6636:10c6:692b:18cc id 6 dev enp2s0 
   2a02:2788:10c4:123:2a09:5ec7:9b99:4a97 id 7 dev enp2s0 


The two ``fe80`` addresses are the IPv6 link local addresses configured on the Ethernet (``enp2s0``) and Wi-Fi (``wlp3s0``) interfaces of our client host. There are three flags which can be associated with each endpoint identifier:

 - ``subflow``. When this flag is set, the path manager will try to create a subflow over this interface when a Multipath TCP is created or the interface becomes active while there was an ongoing Multipath TCP connection. This flag is mainly useful for clients.
 - ``signal``. When this flag is set, the path manager will announce the address of the endpoint over any Multipath TCP connection created using other addresses. This flag can be used on clients or servers. It is mainly useful on servers that have multiple addresses.
 - ``backup``. This flag can be combined with the two other flags. When combined with the ``subflow`` flag, it indicates that a backup subflow will be created. When combined with the ``signal`` flag, it indicates that the address will be advertised as a backup address.

On our client host, we can configure the Wi-Fi interface as a backup interface that creates subflows as follows :

.. code:: console

   sudo ip mptcp endpoint del id 2	  
   sudo ip mptcp endpoint add 192.168.0.43 dev wlp3s0 subflow backup
   sudo ip mptcp endpoint show
   fe80::3934:7572:b1ff:b555 id 1 dev wlp3s0 
   fe80::5642:39bd:3390:43d3 id 3 dev enp2s0 
   192.168.0.37 id 4 dev enp2s0 
   2a02:2788:10c4:123:3d66:f590:d891:8fb3 id 5 dev wlp3s0 
   2a02:2788:10c4:123:6636:10c6:692b:18cc id 6 dev enp2s0 
   2a02:2788:10c4:123:2a09:5ec7:9b99:4a97 id 7 dev enp2s0 
   192.168.0.43 id 8 subflow backup dev wlp3s0 

We had to first remove the configuration for this endpoint because a default one was already active. Then we added the new parameters and verified them.
   
The path manager also has some limits which can be configured using the ``ip mptcp limits`` command. Two limits can be set.

 - ``ip mptcp limits set subflow n`` where ``n`` is an integer. This restricts the Multipath TCP connection to use up to ``n`` different subflows. Servers should protect themselves by setting this limit to a few subflows. Most use cases would work well with 2 or 4 subflows.
 - ``ip mptcp limits set add_addr_accepted n`` where ``n`` is an integer. This parameter limits the number of addresses that are learned over each Multipath TCP connection. This parameter could be used to protect the Multipath TCP implementation against attacks where two many addresses are advertised. Most use cases would work with 4 accepted addresses.

These parameters control the path manager, but before creating Multipath TCP subflows over different paths, we need to configure the IP routing table of our client host. Our client host has two network interfaces: Wi-Fi and Ethernet. By default, Linux prefers the Ethernet interface to Wi-Fi. The two interfaces are configured as follows :

.. code:: console

   ip -4 addr show
   1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
       inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
   2: enp2s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
       inet 192.168.0.37/24 brd 192.168.0.255 scope global dynamic noprefixroute enp2s0
       valid_lft 75697sec preferred_lft 75697sec
   3: wlp3s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
       inet 192.168.0.43/24 brd 192.168.0.255 scope global dynamic noprefixroute wlp3s0
       valid_lft 75696sec preferred_lft 75696sec

By default, Linux creates the two following default routes.


.. code:: console
	  
   route -n
   Kernel IP routing table
   Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
   0.0.0.0         192.168.0.1     0.0.0.0         UG    100    0        0 enp2s0
   0.0.0.0         192.168.0.1     0.0.0.0         UG    600    0        0 wlp3s0

   
We need to configure the routing tables to be able to use the two interfaces simultaneously. For this, we need to ensure that packets with source address ``192.168.0.37`` are sent over the ``enp2s0`` interface while packets with source address ``192.168.0.43`` are sent over the ``wlp3s0`` interface. This can be achieved using two different routing tables.

.. spelling:word-list::

   ip

.. code:: console

   # create the two routing tables	  
   ip rule add from 192.168.0.37 table 1
   ip rule add from 192.168.0.43 table 2

   # Configure routing table 1 for enp2s0
   ip route add 192.168.0.0/24 dev enp2s0 scope link table 1
   ip route add default via 192.168.0.1 dev enp2s0 table 1

   # Configure routing table 2 for wlp3s0
   ip route add 192.168.0.0/24 dev wlp3s0 scope link table 2
   ip route add default via 192.168.0.1 dev wlp3s0 table 2

   # Configure a default route to regular internet
   ip route add default scope global nexthop via 192.168.0.1 dev enp2s0

   
We can check the routing tables using the ip command.

.. code:: console


   ip rule show
   0:	from all lookup local
   32764:	from 192.168.0.43 lookup 2
   32765:	from 192.168.0.37 lookup 1
   32766:	from all lookup main
   32767:	from all lookup default

   ip route
   default via 192.168.0.1 dev enp2s0 
   default via 192.168.0.1 dev enp2s0 proto dhcp metric 100 
   default via 192.168.0.1 dev wlp3s0 proto dhcp metric 600 
   169.254.0.0/16 dev wlp3s0 scope link metric 1000 
   192.168.0.0/24 dev enp2s0 proto kernel scope link src 192.168.0.37 metric 100 
   192.168.0.0/24 dev wlp3s0 proto kernel scope link src 192.168.0.43 metric 600 

   ip route show table 1
   default via 192.168.0.1 dev enp2s0 
   192.168.0.0/24 dev enp2s0 scope link 

   ip route show table 2
   default via 192.168.0.1 dev wlp3s0 
   192.168.0.0/24 dev wlp3s0 scope link 


We can verify that the two routing tables are correct using ``nc`` by forcing it to use a specific source address.

.. code:: console

   echo -e "GET / HTTP/1.0\r\n" | nc -4 -s 192.168.0.37 test.multipath-tcp.org 80
   HTTP/1.0 200 OK
   Content-Type: text/html
   ETag: "4215149735"
   Last-Modified: Tue, 05 Jul 2022 16:11:47 GMT
   Content-Length: 389
   Connection: close
   Date: Wed, 06 Jul 2022 11:34:24 GMT
   Server: lighttpd/1.4.59
   
   <!DOCTYPE html>
   <html>
   <head>
   <title>Welcome to test.multipath-tcp.org!</title>
   <style>
   body {
   width: 35em;
   margin: 0 auto;
   font-family: Tahoma, Verdana, Arial, sans-serif;
   }
   </style>
   </head>
   <body>
   <h1>Welcome to test.multipath-tcp.org !</h1>
   <p>This web server runs Multipath TCP v1</p>
   
   <p><em>Thank you for using Multipath TCP.</em></p>
   </body>
   </html>
	  

You should get the same result when using the second interface, IP address ``192.168.0.43`` in our example.

.. code:: console

   echo -e "GET / HTTP/1.0\r\n" | nc -4 -s 192.168.0.43 test.multipath-tcp.org 80
   
   
The next step is to verify that Multipath TCP is working correctly and that two subflows are created. For this, we'll use the ``-i`` parameter of ``nc`` to add a delay between the two lines of the HTTP GET. We will leverage this delay to check that MPTCP is correctly working using ``ss`` or ``tcpdump`` 

.. code:: console

   echo -e "GET / HTTP/1.0\r\n" | mptcpize run nc -4 -i 5 test.multipath-tcp.org 80

   
We can observe the creation of the connection and the subflow using both ``ss`` and ``tcpdump``. ``ss`` shows that there are two subflows towards ``test.multipath-tcp.org``.

.. code:: console

   ss -4 -iatM  dst test.multipath-tcp.org 
   Netid  State  Recv-Q  Send-Q         Local Address:Port      Peer Address:Port  Process                                                                         
   tcp    ESTAB  0       0        192.168.0.43%wlp3s0:34801     5.196.67.207:http  
	 cubic wscale:7,7 rto:220 rtt:17.439/8.719 mss:1448 pmtu:1500 rcvmss:536 advmss:1448 cwnd:10 bytes_acked:1 segs_out:2 segs_in:2 send 6.64Mbps lastsnd:1776 lastrcv:1776 lastack:1764 pacing_rate 13.3Mbps delivered:1 rcv_space:14480 rcv_ssthresh:64088 minrtt:17.439
   tcp    ESTAB  0       0               192.168.0.37:47672     5.196.67.207:http  
	 cubic wscale:7,7 rto:216 rtt:14/5.405 mss:1448 pmtu:1500 rcvmss:536 advmss:1448 cwnd:10 bytes_sent:16 bytes_acked:17 segs_out:3 segs_in:3 data_segs_out:1 send 8.27Mbps lastsnd:1808 lastrcv:1808 lastack:1792 pacing_rate 16.5Mbps delivery_rate 790kbps delivered:2 busy:16ms rcv_space:14480 rcv_ssthresh:64088 minrtt:13.905
   mptcp  ESTAB  0       0               192.168.0.37:47672     5.196.67.207:http  
	 subflows:1 subflows_max:8 remote_key token:e1e3cdeb write_seq:1045ecfa3f05f4ea snd_una:1045ecfa3f05f4ea rcv_nxt:d0568f430363c9aa


The line starting with ``mptcp`` indicates that the Multipath TCP connection above has one additional subflow.

The ``tcpdump`` output reveals precisely which packets have been sent over each network interface.

.. code:: console

   sudo tcpdump -n -i any host test.multipath-tcp.org and port 80tcpdump: data link type LINUX_SLL2
   tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
   listening on any, link-type LINUX_SLL2 (Linux cooked v2), snapshot length 262144 bytes
   13:43:26.620667 enp2s0 Out IP 192.168.0.37.47672 > 5.196.67.207.80: Flags [S], seq 3585891423, win 64240, options [mss 1460,sackOK,TS val 3993892549 ecr 0,nop,wscale 7,mptcp capable v1], length 0
   13:43:26.634537 enp2s0 In  IP 5.196.67.207.80 > 192.168.0.37.47672: Flags [S.], seq 1788691420, ack 3585891424, win 65160, options [mss 1460,sackOK,TS val 1030255900 ecr 3993892549,nop,wscale 7,mptcp capable v1 {0x54f04ad5bd2d9f42}], length 0
   13:43:26.634609 enp2s0 Out IP 192.168.0.37.47672 > 5.196.67.207.80: Flags [.], ack 1, win 502, options [nop,nop,TS val 3993892563 ecr 1030255900,mptcp capable v1 {0xff2ec3a2f6151881,0x54f04ad5bd2d9f42}], length 0
   13:43:26.634718 enp2s0 Out IP 192.168.0.37.47672 > 5.196.67.207.80: Flags [P.], seq 1:17, ack 1, win 502, options [nop,nop,TS val 3993892563 ecr 1030255900,mptcp capable v1 {0xff2ec3a2f6151881,0x54f04ad5bd2d9f42},nop,nop], length 16: HTTP: GET / HTTP/1.0
   13:43:26.649351 enp2s0 In  IP 5.196.67.207.80 > 192.168.0.37.47672: Flags [.], ack 17, win 509, options [nop,nop,TS val 1030255916 ecr 3993892563,mptcp dss ack 1172603837543216362], length 0
   13:43:26.649351 enp2s0 In  IP 5.196.67.207.80 > 192.168.0.37.47672: Flags [.], ack 17, win 509, options [nop,nop,TS val 1030255916 ecr 3993892563,mptcp dss ack 1172603837543216362], length 0
   13:43:26.649498 wlp3s0 Out IP 192.168.0.43.34801 > 5.196.67.207.80: Flags [S], seq 2321572505, win 64240, options [mss 1460,sackOK,TS val 1218002018 ecr 0,nop,wscale 7,mptcp join id 8 token 0xeef7df2f nonce 0xc0d346f6], length 0
   13:43:26.666895 wlp3s0 In  IP 5.196.67.207.80 > 192.168.0.43.34801: Flags [S.], seq 1973196884, ack 2321572506, win 65160, options [mss 1460,sackOK,TS val 1030255931 ecr 1218002018,nop,wscale 7,mptcp join id 0 hmac 0xc7489cf7056428b4 nonce 0xa54f9af], length 0
   13:43:26.666966 wlp3s0 Out IP 192.168.0.43.34801 > 5.196.67.207.80: Flags [.], ack 1, win 502, options [nop,nop,TS val 1218002035 ecr 1030255931,mptcp join hmac 0xb4e6a41bf5861313df7f5f454966998ad7e698a4], length 0
   13:43:26.677776 wlp3s0 In  IP 5.196.67.207.80 > 192.168.0.43.34801: Flags [.], ack 1, win 510, options [nop,nop,TS val 1030255944 ecr 1218002035,mptcp dss ack 1172603837543216362], length 0
   13:43:31.635023 enp2s0 Out IP 192.168.0.37.47672 > 5.196.67.207.80: Flags [P.], seq 17:18, ack 1, win 502, options [nop,nop,TS val 3993897563 ecr 1030255916,mptcp dss ack 56871338 seq 1172603837543216362 subseq 17 len 1,nop,nop], length 1: HTTP
   13:43:31.646703 enp2s0 In  IP 5.196.67.207.80 > 192.168.0.37.47672: Flags [.], ack 18, win 509, options [nop,nop,TS val 1030260913 ecr 3993897563,mptcp dss ack 1172603837543216363], length 0
   13:43:31.647276 enp2s0 In  IP 5.196.67.207.80 > 192.168.0.37.47672: Flags [P.], seq 1:602, ack 18, win 509, options [nop,nop,TS val 1030260914 ecr 3993897563,mptcp dss ack 1172603837543216363 seq 15012343925868579242 subseq 1 len 601,nop,nop], length 601: HTTP: HTTP/1.0 200 OK
   13:43:31.647300 enp2s0 Out IP 192.168.0.37.47672 > 5.196.67.207.80: Flags [.], ack 602, win 501, options [nop,nop,TS val 3993897576 ecr 1030260914,mptcp dss ack 15012343925868579843], length 0
   13:43:31.647276 enp2s0 In  IP 5.196.67.207.80 > 192.168.0.37.47672: Flags [.], ack 18, win 509, options [nop,nop,TS val 1030260914 ecr 3993897563,mptcp dss fin ack 1172603837543216363 seq 15012343925868579843 subseq 0 len 1,nop,nop], length 0
   13:43:31.647321 enp2s0 Out IP 192.168.0.37.47672 > 5.196.67.207.80: Flags [.], ack 602, win 501, options [nop,nop,TS val 3993897576 ecr 1030260914,mptcp dss ack 15012343925868579844], length 0
   13:43:31.647330 wlp3s0 Out IP 192.168.0.43.34801 > 5.196.67.207.80: Flags [.], ack 1, win 502, options [nop,nop,TS val 1218002046 ecr 1030255944,mptcp dss ack 15012343925868579844], length 0
   13:43:31.648565 wlp3s0 In  IP 5.196.67.207.80 > 192.168.0.43.34801: Flags [.], ack 1, win 510, options [nop,nop,TS val 1030255944 ecr 1218002035,mptcp dss fin ack 1172603837543216363 seq 15012343925868579843 subseq 0 len 1,nop,nop], length 0
   13:43:36.635392 enp2s0 Out IP 192.168.0.37.47672 > 5.196.67.207.80: Flags [.], ack 602, win 501, options [nop,nop,TS val 3993897576 ecr 1030260914,mptcp dss fin ack 15012343925868579844 seq 1172603837543216363 subseq 0 len 1,nop,nop], length 0
   13:43:36.635416 wlp3s0 Out IP 192.168.0.43.34801 > 5.196.67.207.80: Flags [.], ack 1, win 502, options [nop,nop,TS val 1218007017 ecr 1030255944,mptcp dss fin ack 15012343925868579844 seq 1172603837543216363 subseq 0 len 1,nop,nop], length 0
   13:43:36.636468 enp2s0 Out IP 192.168.0.37.47672 > 5.196.67.207.80: Flags [.], ack 602, win 501, options [nop,nop,TS val 3993897576 ecr 1030260914,mptcp dss fin ack 15012343925868579844 seq 1172603837543216363 subseq 0 len 1,nop,nop], length 0
   13:43:36.636482 wlp3s0 Out IP 192.168.0.43.34801 > 5.196.67.207.80: Flags [.], ack 1, win 502, options [nop,nop,TS val 1218007017 ecr 1030255944,mptcp dss fin ack 15012343925868579844 seq 1172603837543216363 subseq 0 len 1,nop,nop], length 0
   13:43:36.640425 enp2s0 Out IP 192.168.0.37.47672 > 5.196.67.207.80: Flags [.], ack 602, win 501, options [nop,nop,TS val 3993897576 ecr 1030260914,mptcp dss fin ack 15012343925868579844 seq 1172603837543216363 subseq 0 len 1,nop,nop], length 0
   13:43:36.640431 wlp3s0 Out IP 192.168.0.43.34801 > 5.196.67.207.80: Flags [.], ack 1, win 502, options [nop,nop,TS val 1218007017 ecr 1030255944,mptcp dss fin ack 15012343925868579844 seq 1172603837543216363 subseq 0 len 1,nop,nop], length 0
   13:43:36.645605 enp2s0 In  IP 5.196.67.207.80 > 192.168.0.37.47672: Flags [.], ack 18, win 509, options [nop,nop,TS val 1030265912 ecr 3993897576,mptcp dss ack 1172603837543216364], length 0
   13:43:36.645659 enp2s0 Out IP 192.168.0.37.47672 > 5.196.67.207.80: Flags [F.], seq 18, ack 602, win 501, options [nop,nop,TS val 3993902574 ecr 1030265912,mptcp dss ack 15012343925868579844], length 0
   13:43:36.645674 wlp3s0 Out IP 192.168.0.43.34801 > 5.196.67.207.80: Flags [F.], seq 1, ack 1, win 502, options [nop,nop,TS val 1218012014 ecr 1030255944,mptcp dss ack 15012343925868579844], length 0
   13:43:36.646315 wlp3s0 In  IP 5.196.67.207.80 > 192.168.0.43.34801: Flags [.], ack 1, win 510, options [nop,nop,TS val 1030260930 ecr 1218002046,mptcp dss ack 1172603837543216364], length 0
   13:43:36.647699 enp2s0 In  IP 5.196.67.207.80 > 192.168.0.37.47672: Flags [F.], seq 602, ack 18, win 509, options [nop,nop,TS val 1030265912 ecr 3993897576,mptcp dss ack 1172603837543216364], length 0
   13:43:36.647718 enp2s0 Out IP 192.168.0.37.47672 > 5.196.67.207.80: Flags [.], ack 603, win 501, options [nop,nop,TS val 3993902576 ecr 1030265912,mptcp dss ack 15012343925868579844], length 0
   13:43:36.648629 wlp3s0 In  IP 5.196.67.207.80 > 192.168.0.43.34801: Flags [F.], seq 1, ack 1, win 510, options [nop,nop,TS val 1030265912 ecr 1218002046,mptcp dss ack 1172603837543216364], length 0
   13:43:36.648649 wlp3s0 Out IP 192.168.0.43.34801 > 5.196.67.207.80: Flags [.], ack 2, win 502, options [nop,nop,TS val 1218012017 ecr 1030265912,mptcp dss ack 15012343925868579844], length 0
   13:43:36.657040 enp2s0 In  IP 5.196.67.207.80 > 192.168.0.37.47672: Flags [.], ack 19, win 509, options [nop,nop,TS val 1030265923 ecr 3993902574,mptcp dss ack 1172603837543216364], length 0
   13:43:36.662211 wlp3s0 In  IP 5.196.67.207.80 > 192.168.0.43.34801: Flags [.], ack 2, win 510, options [nop,nop,TS val 1030265928 ecr 1218012014,mptcp dss ack 1172603837543216364], length 0

	 

If your host is dual stack, you also need to do the same configuration for IPv6 as well. Our test host uses the following IPv6 addresses.

.. code:: console

   ip -6 addr show
   1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 state UNKNOWN qlen 1000
       inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
   2: enp2s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 state UP qlen 1000
       inet6 2a02:2788:10c4:123:f468:1851:9a9f:7d44/64 scope global temporary dynamic 
       valid_lft 592298sec preferred_lft 73422sec
       inet6 2a02:2788:10c4:123:6636:10c6:692b:18cc/64 scope global dynamic mngtmpaddr noprefixroute 
       valid_lft 1209600sec preferred_lft 604800sec
       inet6 fe80::5642:39bd:3390:43d3/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
   3: wlp3s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 state UP qlen 1000
       inet6 2a02:2788:10c4:123:3d66:f590:d891:8fb3/64 scope global dynamic noprefixroute 
       valid_lft 1209600sec preferred_lft 604800sec
       inet6 fe80::3934:7572:b1ff:b555/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever


We thus had to configure the following IPv6 routing tables. This is similar to the commands used to configure the IPv4 routing tables.

.. code:: console

   ip -6 rule add from 2a02:2788:10c4:123:6636:10c6:692b:18cc table 1
   ip -6 rule add from 2a02:2788:10c4:123:3d66:f590:d891:8fb3 table 2
   ip route add 2a02:2788:10c4:123::/64 dev enp2s0 scope link table 1
   ip route add 2a02:2788:10c4:123::/64 dev wlp3s0 scope link table 2
   ip route add default via fe80::10:18ff:fe07:fc33 dev enp2s0 table 1 
   ip route add default via fe80::10:18ff:fe07:fc33 dev wlp3s0 table 2 
   ip route add default scope global nexthop via fe80::10:18ff:fe07:fc33 dev enp2s0 


Remember that if you want to create subflows using IPv6 addresses, you also need to configure the stack with ``ip mptcp endpoint add`` as you did for the IPv4 addresses.

.. note::

   The current versions of the Linux kernel only use one address family at a time. If a connection was created using IPv4, then only IPv4 addresses will be used to create new subflows. Future versions of the kernel will allow to mix IPv4 and IPv6 subflows.


   


Analyzing the output of ss
--------------------------

.. spelling:word-list::

   ss 

.. include:: nstat-intro.rst

.. include:: nstat-mptcp.rst


.. _native-apps-linux:	     

Native Multipath TCP applications on Linux
------------------------------------------

On recent Linux kernels, Multipath TCP is enabled on a per-socket basis by passing ``IP_PROTO_MPTCP`` as the third parameter of the ``socket`` system call that creates the socket. This implies that existing applications that use TCP need to be changed to support Multipath TCP.

The `mptcp-hello <https://github.com/mptcp-apps/mptcp-hello>`_ project on `GitHub <https://github.com>`_ provides simple examples showing how to enable Multipath TCP on a TCP application in the following programming languages :

 - `C <https://github.com/mptcp-apps/mptcp-hello/blob/main/c/README.md>`_
 - `Python <https://github.com/mptcp-apps/mptcp-hello/blob/main/python/README.md>`_
 - `Perl <https://github.com/mptcp-apps/mptcp-hello/blob/main/perl/README.md>`_
 - `Rust <https://github.com/mptcp-apps/mptcp-hello/blob/main/rust/README.md>`_

Patches have been proposed to add Multipath TCP support to the following applications :

 - `iperf3 <https://github.com/mptcp-apps/iperf>`_
 - `openssh <https://github.com/mptcp-apps/openssh-portable/tree/mptcp_support>`_
 - `wrk <https://github.com/mptcp-apps/wrk/tree/mptcp>`_
 - `curl <https://github.com/mptcp-apps/curl/tree/mptcp-clean>`_
 - `httping <https://github.com/mptcp-apps/HTTPing/tree/mptcp>`_
 - `openvpn <https://github.com/mptcp-apps/openvpn/tree/mptcp>`_
   
In addition some specific applications are developed with Multipath TCP support :

 - `Minimal Multipath TCP capable FTP client in python <https://github.com/vanyingenzi/mftpclient>`_
