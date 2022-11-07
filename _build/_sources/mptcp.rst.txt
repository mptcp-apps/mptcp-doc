.. meta::
   :github_url: github.com/obonaventure/mmtp-book

Multipath TCP
*************


Multipath TCP :cite:`rfc8684` is an extension to the TCP protocol :cite:p:`rfc793` that was described in chapter :ref:`chapter-tcp`. We start with an overview of Multipath TCP. Then we explain how a Multipath TCP connection can be established. Then we analyze how data is exchanged over different paths and explain the multipath congestion control schemes. Finally, we explain how Multipath TCP connections can be terminated.


.. _mptcp-overview:

A brief overview of Multipath TCP
=================================

The main design objective for Multipath TCP :cite:`rfc6824` was to enable hosts to exchange the packets that belong to a single TCP connection over different network paths. Several definitions are possible for a network path. Considering a TCP connection between a client and a server, a network path can be defined as the succession of the links and routers that create a path between the client and the server. For example, in :numref:`fig-simple-network`, there are many paths between the client host `C` and the server `S`, e.g. :math:`C \rightarrow R1 \rightarrow R2 \rightarrow R4 \rightarrow S` and :math:`C \rightarrow R1 \rightarrow R3 \rightarrow R4 \rightarrow S`, but also :math:`C \rightarrow R1 \rightarrow R3 \rightarrow R5 \rightarrow R4 \rightarrow S` or even :math:`C \rightarrow R1 \rightarrow R2 \rightarrow R4 \rightarrow R3 \rightarrow R5 \rightarrow R4 \rightarrow S`.   

.. _fig-simple-network:
.. tikz:: A simple network providing multiple paths between :math:`C` and :math:`S`
   :libs: positioning, matrix, arrows, math

   \tikzset{router/.style = {rectangle, draw, text centered, minimum height=2em}, }
   \tikzset{host/.style = {circle, draw, text centered, minimum height=2em}, }
   \node[host] (C) {C};
   \node[router, right of=C] (R1) {R1};
   \node[router, right=of R1] (R3) {R3};
   \node[router, right=of R3] (R5) {R5};
   \node[router, below=of R1] (R2) {R2};
   \node[router, below=of R3] (R4) {R4};
   \node[host, right of=R4] (S) {S};

   \path[draw,thick]
   (C) edge (R1)
   (R1) edge (R2)
   (R3) edge (R1)
   (R2) edge (R4)
   (R4) edge (R3)
   (R4) edge (R5)
   (R3) edge (R5)
   (R4) edge (S);


   
During the first discussions on Multipath TCP within the IETF, there was a debate on the types of paths that Multipath TCP could use in IP networks. Although networks provide a wide range of paths between a source and a destination, it is not necessarily simple to use all these paths in a pure IP network. Looking a :numref:`fig-simple-network` and assuming that all links have the same IGP weight, packets sent by `C` will follow one of the two shortest paths, i.e. :math:`C \rightarrow R1 \rightarrow R2 \rightarrow R4 \rightarrow S` or :math:`C \rightarrow R1 \rightarrow R3 \rightarrow R4 \rightarrow S`. Since routers usually use hash-based load-balancing :cite:`rfc2992` to distribute packets over equal cost paths, all the packets from a given connection will follow either the first or the second shortest path. In most networks, the path followed by a TCP connection will only change if there are link or router failures on this particular path.

When Multipath TCP was designed, the IETF did not want to design techniques to enable the transport layer to specify the paths that packets should follow. They opted for a very conservative definition of the paths that Multipath TCP can use :cite:`rfc6182`. Multipath TCP assumes that the endpoints of a TCP connection are identified by their IP addresses. If two hosts want to exchange packets over different paths, then at least one of them must have two or more IP addresses. This covers two very important use cases:

 - mobile devices like the smartphones that have a cellular and a Wi-Fi network interface each identified by its own IP address
 - dual-stack hosts that have both an IPv4 and an IPv6 address


In this document, we will often use smartphones to illustrate Multipath TCP client hosts. This corresponds to a widely deployed use case that simplifies many of the examples, but is not the only possible deployment.


.. note:: Using non-equal cost paths with Multipath TCP
	  
   When Multipath TCP was designed, there was no standardized solution that enabled a host to control the path followed by its packets inside a network. This is slowly changing. First, the IETF has adopted the Segment Routing architecture :cite:`rfc8402`. This architecture is a modern version of source routing which can be used in MPLS and IPv6 networks. In particular, using the IPv6 Segment Routing Header :cite:`rfc8754`, a host can decide the path that its packets will follow inside the network. This opens new possibilities for Multipath TCP. Some of these possibilities are explored by the Path Aware Networking Research Group of the Internet Research Task Force.

A second important design question for the Multipath TCP designers was how use two or more paths for a single connection ? As an example, let us consider a smartphone that interacts with a server. This smartphone has two different IP addresses: one over its Wi-Fi interface and one over its cellular interface. A naive way to use these two networks would be to operate as shown :numref:`fig-mptcp-naive`. The smartphone would initiate a TCP connection over its Wi-Fi interface as shown in blue in :numref:`fig-mptcp-naive`. This handshake creates a connection and thus some shared state between the smartphone and the server. Given this state, could the smartphone simply sent the next date over the cellular interface (shown in red in :numref:`fig-mptcp-naive`) ?

.. _fig-mptcp-naive:
.. tikz:: A naive approach to create a Multipath TCP connection 
   :libs: positioning, matrix, arrows, math

   \tikzmath{\c1=1; \c2=1.5; \s1=8; \s2=8.5; \max=7; }
   
   \tikzstyle{arrow} = [thick,->,>=stealth]
   \tikzset{state/.style={rectangle, dashed, draw, fill=white} }
   \node [black, fill=white] at (\c1,\max) {Smartphone};
   \node [black, fill=white] at (\s1,\max) {Server};
   
   \draw[blue,thick,->] (\c1,\max-0.5) -- (\c1,0.5);
   \draw[red,thick,->] (\c2,\max-0.5) -- (\c2,0.5);
   \draw[black,thick,->] (\s1,\max-0.5) -- (\s1,0.5);
   
   \tikzmath{\y=\max-1;}
   
   \draw[blue,thick, ->] (\c1,\y) -- (\s1,\y-1) node [midway, align=left, fill=white] {SYN\small{[seq=x]}};
   \draw[blue,thick, ->] (\s1,\y-1) -- (\c1,\y-2) node [midway, align=left, fill=white] {SYN+ACK\small{[seq=y,ack=x+1]}};
   \draw[blue,thick, ->] (\c1,\y-2.1) -- (\s1,\y-3) node [midway, align=left, fill=white] {ACK\small{[seq=x+1,ack=y+1]}};
   \draw[blue,thick, ->] (\c1,\y-3) -- (\s1,\y-4) node [midway, align=left, fill=white] {Data\small{[seq=x+1]}};
   \draw[red,thick, ->] (\c2,\y-4) -- (\s1,\y-5) node [midway, align=left, fill=white] {Data\small{[seq=x+2]}};
   


Unfortunately, this utilization of the two paths between the smartphone and the server poses different problems. First, the server must be able to accept the packet sent by the smartphone over the cellular interface and associate it with the connection created over the Wi-Fi interface. However, the packets sent over the cellular interface use a different source address than those sent over the Wi-Fi interface. When the server receives such a packet, how can it be associated with an existing connection ? If the server blindingly accept this packet from another address than the one used during the handshake, then there are obvious security risks. By sending a single packet, an attacker could inject data inside an existing connection. Furthermore, he could cause a denial of service attack by sending a spoofed packet in an existing connection that requests the server to send a large volume of data to the spoofed address. Furthermore, a middlebox such as a firewall on the cellular path between the smartphone and the server could block the packet because it does not belong to a TCP connection created on the cellular path.


To cope with this problem, the Multipath TCP designers opted for an architecture where a Multipath TCP connection combines several TCP connections that are called subflows over the different paths. In the above example, the smartphone would first create a connection over the Wi-Fi interface. It would later initiate a TCP connection over its cellular interface and use Multipath TCP to link it to the connection created over the Wi-Fi interface.

A Multipath TCP connection starts with a three-way handshake like a regular TCP connection. As with all TCP extensions, the client uses an option in the ``SYN`` to indicate its willingness to use the multipath extensions. The server confirms that it agrees to use this extension by sending the same option in the ``SYN+ACK``.  This is illustrated in :numref:`fig-mptcp-capable-join` where the client sends a ``SYN`` with the ``MPC`` option to negotiate a Multipath TCP connection with a server. If the server replies with the same option, the handshake succeeds and creates the first subflow belonging to this Multipath TCP connection. The client and the server can send data over this connection as over any TCP connection. To use a second path, the client (or the server), must initiate another TCP handshake over the new path. The ``SYN`` sent over this second path uses the ``MPJ`` option to indicate that this is an additional subflow that must be linked to an existing Multipath TCP connection. This is illustrated in :numref:`fig-mptcp-capable-join`.
   

.. _fig-mptcp-capable-join:
.. tikz:: A Multipath TCP connection with two subflows
   :libs: positioning, matrix, arrows, math

   \tikzmath{\c1=1; \c2=1.5; \s1=8; \s2=8.5; \max=10; }
   
   \tikzstyle{arrow} = [thick,->,>=stealth]
   \tikzset{state/.style={rectangle, dashed, draw, fill=white} }
   \node [black, fill=white] at (\c1,\max) {Smartphone};
   \node [black, fill=white] at (\s1,\max) {Server};
   \node [blue, fill=white] at (\c1, \max-0.5) {$IP_{\alpha}$};
   \node [red, fill=white] at (\c2, \max-0.5) {$IP_{\beta}$};
   \node [black, fill=white] at (\s1, \max-0.5) {$IP_{S}$};
   \draw[blue,thick,->] (\c1,\max-0.75) -- (\c1,0.75);
   \draw[red,thick,->] (\c2,\max-0.75) -- (\c2,0.75);
   \draw[black,thick,->] (\s1,\max-0.75) -- (\s1,0.75);
   
   \tikzmath{\y=\max-1;}
   
   \draw[blue,thick, ->] (\c1,\y) -- (\s1,\y-1) node [midway, align=center, fill=white] {SYN\small{[seq=x]}\\\small{MPC}};
   \draw[blue,thick, ->] (\s1,\y-1) -- (\c1,\y-2) node [midway, align=center, fill=white] {SYN+ACK\small{[seq=y,ack=x+1]}\\\small{MPC}};
   \draw[blue,thick, ->] (\c1,\y-2.1) -- (\s1,\y-3) node [midway, align=center, fill=white] {ACK\small{[seq=x+1,ack=y+1]}};
   \draw[blue,thick, ->] (\c1,\y-3) -- (\s1,\y-4) node [midway, align=center, fill=white] {Data\small{[seq=x+1]}};
   \draw[red,thick, ->] (\c2,\y-4) -- (\s1,\y-5) node [midway, align=center, fill=white] {SYN\small{[seq=p]}\\\small{MPJ}};
   \draw[red,thick, ->] (\s1,\y-5) -- (\c2,\y-6) node [midway, align=center, fill=white] {SYN+ACK\small{[seq=q,ack=p+1]}\\\small{MPJ}};
   \draw[red,thick, ->] (\c2,\y-6) -- (\s1,\y-7) node [midway, align=center, fill=white] {ACK\small{[seq=p+1,ack=q+1]}};
   \draw[red,thick, ->] (\c2,\y-7) -- (\s1,\y-8) node [midway, align=center, fill=white] {Data\small{[seq=p+1]}};   


These two three-way handshakes create two TCP connections called subflows in the Multipath TCP terminology. It is useful to analyze how these two connections are identified on the server. A host identifies a TCP connection using four identifiers that are present in all the packets of this connection:

 - the local IP address
 - the remote IP address
 - the local port
 - the remote port

Assume that the client uses IP address :math:`IP_{\alpha}` on its Wi-Fi interface and :math:`IP_{\beta}` on its cellular interface and that :math:`p` is the port used by the server. If the client used port :math:`p_1` to create the initial subflow, then the identifier of this subflow on the server is :math:`<IP_{S},IP_{\alpha},p,p_{1}>`. Similarly, the second subflow is identified by the :math:`<IP_{S},IP_{\beta},p,p_{2}>` tuple on the server. Note that these two connection identifiers differ by at least one IP address as specified in :cite:`rfc6182`.

A server usually manages a large number of simultaneous connections. Furthermore, a client may establish several connections with the same server. To associate a new subflow with an existing Multipath TCP connection, a server must be able to link an incoming ``SYN`` with the corresponding Multipath TCP connection. For this, the client must include an identifier of the associated Multipath TCP connection in its ``MPJ`` option. This identifier must unambiguously identify the corresponding Multipath TCP connection on the server.

A first possible identifier is the four tuple that identifies the initial subflow, i.e. :math:`<IP_{S},IP_{\alpha},p,p_{1}>`. If the server received this identifier in the ``MPJ`` option, it could link the new subflow to the previous one. Unfortunately, this solution does not work in today's Internet. The main concern comes from the middleboxes such as NATs and transparent proxies. To illustrate the problem, consider a simple NAT, such as the one used on most home Wi-Fi access points. :numref:`fig-nat-interference` illustrates a TCP handshake in such an environment. 
   

.. _fig-nat-interference:
.. tikz:: Network Address Translation interferes with TCP 
   :libs: positioning, matrix, arrows, math

   \tikzstyle{arrow} = [thick,->,>=stealth]
   \tikzmath{\c1=1; \c2=1.5; \s1=8; \s2=8.5; \max=5; \nat=4.5;}
   
   
   \node [red, fill=white,align=center] at (\nat,\max) {NAT \\$IP_{N}$};
   \node [black, fill=white,align=center] at (\c1,\max) {Smartphone \\ $IP_{p}$};
   \node [black, fill=white,align=center] at (\s1,\max) {Server \\$IP_{S}$};

   
   \draw[black,thick,->] (\c1,\max-0.5) -- (\c1,0.5);
   \draw[black,thick,->] (\s1,\max-0.5) -- (\s1,0.5);
   \draw[red,dashed,very thick,-] (\nat,\max-0.5) -- (\nat,0.5);
   
   
   \tikzmath{\y=\max-1;}
   
   \draw[blue,thick, ->] (\c1,\y) -- (\nat,\y-0.5) node [midway, align=center, fill=white] {$IP_{P}\rightarrow IP_{S}$\\SYN};
   \draw[blue,thick, ->] (\nat,\y-0.5) -- (\s1,\y-1) node [midway, align=center, fill=white] {$IP_{N}\rightarrow IP_{S}$\\SYN};   
   \draw[blue,thick, ->] (\s1,\y-1.5) -- (\nat,\y-2) node [midway, align=center, fill=white] {$IP_{S}\rightarrow IP_{N}$\\SYN+ACK};
   \draw[blue,thick, ->] (\nat,\y-2) -- (\c1,\y-2.5) node [midway, align=center, fill=white] {$IP_{S}\rightarrow IP_{P}$\\SYN+ACK};   
   \draw[blue,thick, ->] (\c1,\y-3) -- (\nat,\y-3.5) node [midway, align=center, fill=white] {$IP_{P}\rightarrow IP_{S}$\\ACK};
   \draw[blue,thick, ->] (\nat,\y-3.5) -- (\s1,\y-4) node [midway, align=center, fill=white] {$IP_{N}\rightarrow IP_{S}$\\ACK};

The smartphone uses a private IP address, :math:`IP_{P}` and the NAT uses a public address :math:`IP_{N}`. If we assume that the NAT only changes the client's IP address, then the connection is identified by the :math:`<IP_{P},IP_{S},p,p_{1}>` tuple on the smartphone and :math:`<IP_{S},IP_{N},p,p_{1}>` on the server. Note some NATs also change the client port. If the smartphone places its local connection identifier inside an ``MPJ`` option, the server might not be able to recognize the corresponding connection in the ``SYN`` packets that it received.
   

To cope with this problem, Multipath TCP uses a local identifier, called `token` in the Multipath TCP specification, to identify each Multipath TCP connection. The client assigns its token when it initiates a new Multipath TCP connection. A server assigns its token when it accepts a new Multipath TCP connection. These two tokens are chosen independently by the client and the server. For security reasons, these tokens should be random. The ``MPJ`` option contains the token assigned by the remote host. This is illustrated in :numref:`fig-mptcp-capable-join-token`. The server assigns token `456` to the Multipath TCP connection created as the first subflow. It informs the smartphone by sending this token in its ``MPC`` option in the ``SYN+ACK``. When the client creates the second subflow, it includes its token in the ``MPJ`` option of its ``SYN``.

.. _fig-mptcp-capable-join-token:
.. tikz:: The tokens exchanged during the handshake allow to associate subsequent subflows to existing Multipath TCP connections
   :libs: positioning, matrix, arrows, math

   \tikzmath{\c1=1; \c2=1.5; \s1=8; \s2=8.5; \max=10; }
   
   \tikzstyle{arrow} = [thick,->,>=stealth]
   \tikzset{state/.style={rectangle, dashed, draw, fill=white} }
   \node [black, fill=white] at (\c1,\max) {Smartphone};
   \node [black, fill=white] at (\s1,\max) {Server};
   
   \draw[blue,thick,->] (\c1,\max-0.5) -- (\c1,0.5);
   \draw[red,thick,->] (\c2,\max-0.5) -- (\c2,0.5);
   \draw[black,thick,->] (\s1,\max-0.5) -- (\s1,0.5);
   
   \tikzmath{\y=\max-1;}
   
   \draw[blue,thick, ->] (\c1,\y) -- (\s1,\y-1) node [midway, align=center, fill=white] {SYN\small{[seq=x]}\\\small{MPC[token=123]}};
   \draw[blue,thick, ->] (\s1,\y-1) -- (\c1,\y-2) node [midway, align=center, fill=white] {SYN+ACK\small{[seq=y,ack=x+1]}\\\small{MPC[token=456]}};
   \draw[blue,thick, ->] (\c1,\y-2.1) -- (\s1,\y-3) node [midway, align=center, fill=white] {ACK\small{[seq=x+1,ack=y+1]}};
   \draw[blue,thick, ->] (\c1,\y-3) -- (\s1,\y-4) node [midway, align=center, fill=white] {Data\small{[seq=x+1]}};
   \draw[red,thick, ->] (\c2,\y-4) -- (\s1,\y-5) node [midway, align=center, fill=white] {SYN\small{[seq=p]}\\\small{MPJ[token=456]}};
   \draw[red,thick, ->] (\s1,\y-5) -- (\c2,\y-6) node [midway, align=center, fill=white] {SYN+ACK\small{[seq=q,ack=p+1]}\\\small{MPJ[\ldots]}};
   \draw[red,thick, ->] (\c2,\y-6) -- (\s1,\y-7) node [midway, align=center, fill=white] {ACK\small{[seq=p+1,ack=q+1]}};



.. note:: Multipath TCP in datacenters   
   
   The Multipath TCP architecture :cite:`rfc6182` assumes that at least one of the communicating hosts use different IP addresses to identify the different paths used by a Multipath TCP connection. In practice, this architectural requirement is not always enforced by Multipath TCP implementations. A Multipath TCP implementation can combine different subflows into one Multipath TCP connection provided that each subflow is identified by a different four-tuple. Two subflows between two communicating hosts can differ in their client-selected ports. This solution has been chosen when Multipath TCP was proposed to mitigate congestion in datacenter networks :cite:`Raiciu_Datacenter:2011`.

   Several designs exist for datacenter networks, but the fat-tree architecture shown in :numref:`fig-fat-tree` is a very popular one.	  

   .. _fig-fat-tree:
   .. tikz:: A simple datacenter network
      :libs: positioning, matrix, arrows, math

       \begin{tikzpicture}[node distance=4cm]
       \tikzset{router/.style = {rectangle, draw, text centered, minimum height=2em}, }
       \tikzset{host/.style = {circle, draw, text centered, minimum height=2em}, }
       \node[router] (C1) {C1};
       \node[router, right= 6cm of C1] (C2) {C2};
       \node[router, below left=1cm of C1] (A1) {A1};
       \node[router, below right= 1cm of C1] (A2) {A2};
       \node[router, below left= 1cm of C2] (A3) {A3};
       \node[router, below right= 1cm of C2] (A4) {A4};
       \node[router, below= 1cm of A1] (E1) {E1};
       \node[router, below= 1cm of A2] (E2) {E2};
       \node[router, below= 1cm of A3] (E3) {E3};
       \node[router, below= 1cm of A4] (E4) {E4};
       \node[host, below left= 0.5cm of E1] (P1) {$\alpha$};
       \node[host, below right= 0.5cm of E1] (P2) {$\beta$};
       \node[host, below left= 0.5cm of E2] (P3) {$\gamma$};
       \node[host, below right= 0.5cm of E2] (P4) {$\delta$};
       \node[host, below left= 0.5cm of E3] (P5) {$\kappa$};
       \node[host, below right= 0.5cm  of E3] (P6) {$\nu$};
       \node[host, below left= 0.5cm of E4] (P7) {$\mu$};
       \node[host, below right= 0.5cm of E4] (P8) {$\pi$};
       \path[draw,thick]
       (P1) edge (E1)
       (P2) edge (E1)
       (P3) edge (E2)
       (P4) edge (E2)
       (P5) edge (E3)
       (P6) edge (E3)
       (P7) edge (E4)
       (P8) edge (E4)
       (E1) edge (A1)
       (E1) edge (A2)
       (E2) edge (A1)
       (E2) edge (A2)
       (E3) edge (A3)
       (E3) edge (A4)
       (E4) edge (A3)
       (E4) edge (A4)
       (A1) edge (C1)
       (A1) edge (C2)
       (A2) edge (C1)
       (A2) edge (C2)
       (A3) edge (C1)
       (A3) edge (C2)
       (A4) edge (C1)
       (A4) edge (C2);

       \end{tikzpicture}


       
   This network topology exposes a large number of equal cost paths between the servers that are shown using circles in :numref:`fig-fat-tree`. For example, consider the paths between the :math:`\alpha` and :math:`\pi` hosts. The paths start at :math:`E1`. This router can reach :math:`E4` and :math:`\pi` via :math:`A1` or :math:`A2`. Each of these two aggregation routers can reach :math:`\pi` via one of the two core routers. These two routers can then balance the flows via both :math:`A3` and :math:`A4`. There are :math:`2^{4}=16` different paths between :math:`\alpha` and :math:`\pi` in this very small network. If each of these routers balance the incoming packets using a hash function :cite:`rfc2992` that takes as input their source and destination addresses and ports, then the subflows of a Multipath TCP connection that use different client problems will be spread evenly across the network topology.  Raiciu et al. provide simulations and measurements showing the benefits of using Multipath TCP in datacenters :cite:`Raiciu_Datacenter:2011`.


..  explain architecture and show that an MPTCP connection manages several subflows    

Once a Multipath TCP connection and the additional subflows have been established, we can use them to exchange data. An important point to remember is that a Multipath TCP connection provides a bidirectional bytestream service like a regular TCP connection. This service does not change even if Multipath TCP uses different subflows to carry the data between the sender and the receiver. As an example, consider a sender that sends ``ABCD`` one byte at a time over a Multipath TCP connection composed of two subflows. A naive approach to send these bytes would be to simply placed them in different TCP segments. This is illustrated in :numref:`fig-mptcp-data-naive` where we assume that the two TCP subflows have already been established.
    
.. _fig-mptcp-data-naive:
.. tikz:: A naive approach to send data over a Multipath TCP connection 
   :libs: positioning, matrix, arrows, math

   \tikzmath{\c1=1; \c2=1.5; \s1=8; \s2=8.5; \max=10; }
   
   \tikzstyle{arrow} = [thick,->,>=stealth]
   \tikzset{state/.style={rectangle, dashed, draw, fill=white} }
   \node [black, fill=white] at (\c1,\max) {Smartphone};
   \node [black, fill=white] at (\s1,\max) {Server};
   
   \draw[blue,thick,->] (\c1,\max-0.5) -- (\c1,0.5);
   \draw[red,thick,->] (\c2,\max-0.5) -- (\c2,0.5);
   \draw[black,thick,->] (\s1,\max-0.5) -- (\s1,0.5);
   
   \tikzmath{\y=\max-1;}
   
   \draw[blue,thick, ->] (\c1,\y) -- (\s1,\y-1) node [midway, align=center, fill=white] {DATA\small{[seq=x,A]}};
   \draw[black,thick, ->] (\s1,\y-1) -- (\s1+4,\y-1) node [midway, align=center, fill=white] {DATA.ind(A)};
   \draw[blue,thick, ->] (\s1,\y-1) -- (\c1,\y-2) node [midway, align=center, fill=white] {ACK\small{[ack=x+1]}};
   \draw[red,thick, ->] (\c2,\y-2) -- (\s1,\y-3) node [midway, align=center, fill=white] {DATA\small{[seq=p,B]}};
   \draw[black,thick, ->] (\s1,\y-3) -- (\s1+4,\y-3) node [midway, align=center, fill=white] {DATA.ind(B)};
   \draw[red,thick, ->] (\s1,\y-3) -- (\c2,\y-4) node [midway, align=center, fill=white] {ACK\small{[ack=p+1]}};
   \draw[blue,thick, ->] (\c1,\y-4) -- (\s1,\y-5) node [midway, align=center, fill=white] {DATA\small{[seq=x+1,C]}};
      \draw[black,thick, ->] (\s1,\y-5) -- (\s1+4,\y-5) node [midway, align=center, fill=white] {DATA.ind(C)};
   \draw[blue,thick, ->] (\s1,\y-5) -- (\c1,\y-6) node [midway, align=center, fill=white] {ACK\small{[ack=x+2]}};
   \draw[red,thick, ->] (\c2,\y-6) -- (\s1,\y-7) node [midway, align=center, fill=white] {DATA\small{[seq=p+1,D]}};
      \draw[black,thick, ->] (\s1,\y-7) -- (\s1+4,\y-7) node [midway, align=center, fill=white] {DATA.ind(D)};
   \draw[red,thick, ->] (\s1,\y-7) -- (\c2,\y-8) node [midway, align=center, fill=white] {ACK\small{[acl=p+2]}};

In this example, the Smartphone slowly sends data in sequence. The server receives the data in sequence over the two subflows and the server could simply deliver the data as soon as it arrives over each subflow. This is illustrated with the ``DATA.ind(...)`` primitives that represent the delivery of the data to the server application. However, consider now that the first packet sent on the red subflow is lost and is retransmitted together with the fourth byte as shown in :numref:`fig-mptcp-data-naive-2`.


.. _fig-mptcp-data-naive-2:
.. tikz:: A naive approach to send data over a Multipath TCP connection 
   :libs: positioning, matrix, arrows.meta, math

   \tikzmath{\c1=1; \c2=1.5; \s1=8; \s2=8.5; \max=10; }
   
   \tikzstyle{arrow} = [thick,->,>=stealth]
   \tikzset{state/.style={rectangle, dashed, draw, fill=white} }
   \node [black, fill=white] at (\c1,\max) {Smartphone};
   \node [black, fill=white] at (\s1,\max) {Server};
   
   \draw[blue,thick,->] (\c1,\max-0.5) -- (\c1,0.5);
   \draw[red,thick,->] (\c2,\max-0.5) -- (\c2,0.5);
   \draw[black,thick,->] (\s1,\max-0.5) -- (\s1,0.5);
   
   \tikzmath{\y=\max-1;}
   
   \draw[blue,thick, ->] (\c1,\y) -- (\s1,\y-1) node [midway, align=center, fill=white] {DATA\small{[seq=x,A]}};
   \draw[black,thick, ->] (\s1,\y-1) -- (\s1+4,\y-1) node [midway, align=center, fill=white] {DATA.ind(A)};
   \draw[blue,thick, ->] (\s1,\y-1) -- (\c1,\y-2) node [midway, align=center, fill=white] {ACK\small{[ack=x+1]}};
   \draw[red,thick, -Circle] (\c2,\y-2) -- (\s1-1,\y-2.8) node [midway, align=center, fill=white] {DATA\small{[seq=p,bseq=1,B]}};

   \draw[blue,thick, ->] (\c1,\y-4) -- (\s1,\y-5) node [midway, align=center, fill=white] {DATA\small{[seq=x+1,C]}};
   \draw[black,thick, ->] (\s1,\y-5) -- (\s1+4,\y-5) node [midway, align=center, fill=white] {DATA.ind(C) ????};
   \draw[blue,thick, ->] (\s1,\y-5) -- (\c1,\y-6) node [midway, align=center, fill=white] {ACK\small{[ack=x+2]}};
   \draw[red,thick, ->] (\c2,\y-6) -- (\s1,\y-7) node [midway, align=center, fill=white] {DATA\small{[seq=p,BD]}};
   \draw[black,thick, ->] (\s1,\y-7) -- (\s1+4,\y-7) node [midway, align=center, fill=white] {DATA.ind(BD) ????};
   \draw[red,thick, ->] (\s1,\y-7) -- (\c2,\y-8) node [midway, align=center, fill=white] {ACK\small{[acl=p+2]}};


In :numref:`fig-mptcp-data-naive-2`, it is clear that the server cannot simply deliver the data as soon as it receives it to its application. If the server behaves this way, it will deliver ``ACBD`` to its application instead of the ``ABCD`` bytestream send by the smartphone. To cope with the reordering of the data sent over the different subflows, Multipath TCP includes bytestream-level data sequence numbers that enable it to preserve the ordering of the data sent over the bytestream. This is illustrated in :numref:`fig-mptcp-data-seq` with the bytestream-level sequence number shown as ``bseq``. We will detail later how this sequence number is exactly transported by Multipath TCP.

.. _fig-mptcp-data-seq:
.. tikz:: A naive approach to send data over a Multipath TCP connection 
   :libs: positioning, matrix, arrows.meta, math

   \tikzmath{\c1=1; \c2=1.5; \s1=8; \s2=8.5; \max=10; }
   
   \tikzstyle{arrow} = [thick,->,>=stealth]
   \tikzset{state/.style={rectangle, dashed, draw, fill=white} }
   \node [black, fill=white] at (\c1,\max) {Smartphone};
   \node [black, fill=white] at (\s1,\max) {Server};
   
   \draw[blue,thick,->] (\c1,\max-0.5) -- (\c1,0.5);
   \draw[red,thick,->] (\c2,\max-0.5) -- (\c2,0.5);
   \draw[black,thick,->] (\s1,\max-0.5) -- (\s1,0.5);
   
   \tikzmath{\y=\max-1;}
   
   \draw[blue,thick, ->] (\c1,\y) -- (\s1,\y-1) node [midway, align=center, fill=white] {DATA\small{[seq=x,bseq=0,A]}};
   \draw[black,thick, ->] (\s1,\y-1) -- (\s1+4,\y-1) node [midway, align=center, fill=white] {DATA.ind(A)};
   \draw[blue,thick, ->] (\s1,\y-1) -- (\c1,\y-2) node [midway, align=center, fill=white] {ACK\small{[ack=x+1]}};
   \draw[red,thick, -Circle] (\c2,\y-2) -- (\s1-1,\y-2.8) node [midway, align=center, fill=white] {DATA\small{[seq=p,bseq=1,B]}};

   \draw[blue,thick, ->] (\c1,\y-4) -- (\s1,\y-5) node [midway, align=center, fill=white] {DATA\small{[seq=x+1,bseq=2,C]}};

   \draw[blue,thick, ->] (\s1,\y-5) -- (\c1,\y-6) node [midway, align=center, fill=white] {ACK\small{[ack=x+2]}};
   \draw[red,thick, ->] (\c2,\y-5.5) -- (\s1,\y-6.5) node [midway, align=center, fill=white] {DATA\small{[seq=p,bseq=1,BC]}};
   \draw[black,thick, ->] (\s1,\y-6.5) -- (\s1+4,\y-6.5) node [midway, align=center, fill=white] {DATA.ind(BC)};   
   \draw[red,thick, ->] (\c2,\y-6) -- (\s1,\y-7) node [midway, align=center, fill=white] {DATA\small{[seq=p,bseq=3,D]}};
   \draw[black,thick, ->] (\s1,\y-7) -- (\s1+4,\y-7) node [midway, align=center, fill=white] {DATA.ind(D)};   
   \draw[red,thick, ->] (\s1,\y-7) -- (\c2,\y-8) node [midway, align=center, fill=white] {ACK\small{[acl=p+2]}};

   
Thanks to the bytestream sequence number, the server can reorder the data received over the different subflows and preserve the ordering in the bytestream.


.. _mptcp-initial-handshake:

Creating a Multipath TCP connection
===================================

Before delving into the details of how a Multipath TCP connection is created, let use first analyze the main requirements of this establishment and how they can be met without considering all the protocol details. During the three-way handshake, TCP hosts agree to establishment a connection, select the initial sequence number in each direction and negotiate the utilization of TCP extensions. In addition to these objectives, the handshake used by Multipath TCP also allows the communicating hosts to:

 - agree to use the Multipath TCP extension
 - exchange the tokens used to identify the connection
 - agree on initial bytestream sequence numbers



To meet the first objective, the client simply needs to send a Multipath TCP option (``MPO``) in its ``SYN``. If the server supports Multipath TCP, it will respond with a ``SYNC+AC`` that carries this option.

To meet the second objective, the simplest solution is reserve some space, e.g. 64 bits, in the ``MPO`` option to encode the token chosen by the host that sends the ``SYN`` or ``SYN+ACK``. With this approach, each host can autonomously select the token that it uses to identify each Multipath TCP connection. To meet the third objective, the simplest solution is also to place the initial sequence number in the ``MPO`` option. :numref:`fig-tcp-handshake-mpo` illustrates a handshake using the ``MPO`` option. 


.. _fig-tcp-handshake-mpo:
.. tikz:: Opening a Multipath TCP connection with a MPO option
   :libs: positioning, matrix, arrows, math

   \tikzmath{\c1=1;\c2=1.5; \s1=8; \s2=8.5; \max=11; }
   
   \tikzstyle{arrow} = [thick,->,>=stealth]
   \tikzset{state/.style={rectangle, dashed, draw, fill=white} }
   \node [black, fill=white] at (\c1,\max) {Client};
   \node [black, fill=white] at (\s1,\max) {Server};
   
   \draw[blue,very thick,->] (\c1,\max-0.5) -- (\c1,0.5);
   \draw[blue,very thick,->] (\s1,\max-0.5) -- (\s1,0.5);
   \draw[red,thick,->] (\c2,\max-0.5) -- (\c2,0.5);
	  
   \tikzmath{\y=\max-1;}
   
   \draw[blue,thick, ->] (\c1,\y) -- (\s1,\y-1) node [midway, align=left, fill=white] {SYN\small{[seq=x]}\\\small{MPO[$Client_{token}$,$Client_{bseq}$]}};
   \draw[blue,thick, ->] (\s1,\y-1) -- (\c1,\y-2) node [midway, align=left, fill=white] {SYN+ACK\small{[seq=y,ack=x+1]}\\\small{MPO[$Server_{token}$,$Server_{bseq}$]}};
   \draw[blue,thick, ->] (\c1,\y-2.1) -- (\s1,\y-3) node [midway, align=left, fill=white] {ACK\small{[seq=x+1,ack=y+1]}};

   \draw[red,thick, ->] (\c2,\y-4) -- (\s1,\y-5) node [midway, align=center, fill=white] {SYN\small{[seq=p]}\\\small{MP\_Join[token=$Server_{token}$]}};
   \draw[red,thick, ->] (\s1,\y-5) -- (\c2,\y-6) node [midway, align=center, fill=white] {SYN+ACK\small{[seq=q,ack=p+1]}\\\small{MP\_Join[\ldots]}};
   \draw[red,thick, ->] (\c2,\y-6) -- (\s1,\y-7) node [midway, align=center, fill=white] {ACK\small{[seq=p+1,ack=q+1]}};
   
The Multipath TCP working group was worried about the risk of attacks with this approach. When the smartphone creates an additional subflow, it includes the token allocated by the server inside the ``MP_JOIN`` option. This token serves two different purposes. First, it identifies the relevant Multipath TCP connection on the server. Second, it also "authenticates" that the ``SYN`` also originates from this client. Authenticating the client is a key concern from a security viewpoint. The main risk is that an on-path attacker who has observed the token in the ``MP_JOIN`` option can reuse it to create additional subflows from any other source. To cope with this problem, Multipath TCP relies on a shared secret that the client and the server exchange during the initial handshake. The client proposes one halve of the secret and the server the other halve. This is illustrated in :numref:`fig-tcp-handshake-mpo-secret`. The client proposes its part of the shared secret in the ``SYN`` (:math:`Client_{secret}`). The server replies with its part of the secret in the ``SYN+ACK``.
   

.. _fig-tcp-handshake-mpo-secret:
.. tikz:: Creating a Multipath TCP connection with a MPO option
   :libs: positioning, matrix, arrows, math

   \tikzmath{\c1=1;\c2=1.5; \s1=8; \s2=8.5; \max=9; }
   
   \tikzstyle{arrow} = [thick,->,>=stealth]
   \tikzset{state/.style={rectangle, dashed, draw, fill=white} }
   \node [black, fill=white] at (\c1,\max) {Client};
   \node [black, fill=white] at (\s1,\max) {Server};
   
   \draw[blue,very thick,->] (\c1,\max-0.5) -- (\c1,0.5);
   \draw[blue,very thick,->] (\s1,\max-0.5) -- (\s1,0.5);
   \draw[red,thick,->] (\c2,\max-0.5) -- (\c2,0.5);
	  
   \tikzmath{\y=\max-1;}
   
   \draw[blue,thick, ->] (\c1,\y) -- (\s1,\y-1) node [midway, align=left, fill=white] {SYN\small{[seq=x]}\\\small{MPO[$Client_{token}$,$Client_{bseq}$,$Client_{secret}$]}};
   \draw[blue,thick, ->] (\s1,\y-1) -- (\c1,\y-2) node [midway, align=left, fill=white] {SYN+ACK\small{[seq=y,ack=x+1]}\\\small{MPO[$Server_{token}$,$Server_{bseq}$,$Server_{secret}$]}};
   \draw[blue,thick, ->] (\c1,\y-2.1) -- (\s1,\y-3) node [midway, align=left, fill=white] {ACK\small{[seq=x+1,ack=y+1]}};

   \draw[red,thick, ->] (\c2,\y-4) -- (\s1,\y-5) node [midway, align=center, fill=white] {SYN\small{[seq=p]}\\\small{MP\_Join[$Server_{token}$,$Client_{random}$]}};
   \draw[red,thick, ->] (\s1,\y-5) -- (\c2,\y-6) node [midway, align=center, fill=white] {SYN+ACK\small{[seq=q,ack=p+1]}\\\small{MP\_Join[$Server_{random}$,HMAC1]}};
   \draw[red,thick, ->] (\c2,\y-6) -- (\s1,\y-7) node [midway, align=center, fill=white] {ACK\small{[seq=p+1,ack=q+1]}\\\small{MP\_Join[HMAC2]}};

   \node[black,fill=white,align=right] at (\c1,0) {HMAC1=HMAC(key=$Server_{secret}$||$Client_{Secret}$,\\msg=$Server_{random}||Client_{random}$)};
   \node[black,fill=white,align=right] at (\c1,-1)  {HMAC2=HMAC(key=$Client_{secret}$||$Server_{Secret}$,\\msg=$Client_{random}||Server_{random}$)};
   
   
Using these two components of the shared secret, the client and the server must be able to authenticate the additional subflows without revealing the shared secret to an attacker who is able to capture packets on the path of the additional subflow. Multipath TCP requires each host to perform a HMAC :cite:`rfc2104` of a random number to confirm their knowledge of the shared secret. This is illustrated in the second part of :numref:`fig-tcp-handshake-mpo-secret`. To create the additional subflow, the client send a ``SYN`` with the ``MP_JOIN`` option containing the :math:`Server_{token}` and a random nonce, :math:`Client_{random}`. The server confirms the establishment of the subflow by sending a ``SYN+ACK`` containing the HMAC computed using the :math:`Client_{random}` and the :math:`Client_{secret}` and :math:`Server_{secret}` input. Thanks to this HMAC computation, the server can reveal that it knows :math:`Client_{secret}` and :math:`Server_{secret}` without explicitly sending them. The server also places a random number, :math:`Server_{random}` in the ``MP_JOIN`` option of the ``SYN+ACK``. The client computes a HMAC and returns it in the third ``ACK``. With these two HMACs, the client and the server can authenticate the establishment of the additional subflow without revealing the shared secret.


.. note:: The security of Multipath TCP depends on the security of the initial handshake

   The ability of correctly authenticate the addition of new subflows to a Multipath TCP connection depends on the secrecy of the :math:`Client_{secret}` and :math:`Server_{secret}` exchanged in the ``SYN`` and ``SYN+ACK`` of the initial handshake. An on-path attacker which is able to capture this initial handshake has all the information required to attach a new subflow to this Multipath TCP connection at any time. Multipath does not include the strong cryptographic techniques (besides HMAC) that would have been required to completely secure the establishment the protocol and the establishment of additional subflows in particular. This threat was considered acceptable for Multipath TCP :cite:`rfc6181` because an attacker who can capture the packets of a single path TCP connection can also inject data inside this connection. To be fully secure Multipath TCP would need to rely on cryptographic techniques that are similar to those used in Transport Layer Security :cite:`rfc8446`. 
   

The solution described above meets the requirements of the Internet Engineering Task Force. From a security viewpoint, the :math:`Client_{secret}`, :math:`Server_{secret}` and the random nonces should be as large as possible to prevent attacks where their values are simply guessed. Unfortunately, since Multipath TCP uses TCP options to exchange all this information, we need to ensure that it fits inside the extended header of a TCP ``SYN``. The TCP specification :cite:`rfc793` reserves up to 40 bytes to place the TCP options in a ``SYN``. Today's TCP stacks already consume 4 bytes for the ``MSS`` option :cite:`rfc793`, 3 for the ``Window Scale`` option :cite:`rfc1323`, 2 for ``SACK Permitted`` :cite:`rfc2018` and 10 for the timestamp option :cite:`rfc1323`. This leaves only 20 bytes to encode a Multipath TCP option that must contain an initial sequence number, a token and a secret. Multipath TCP solves this problem by deriving these three values from a single field encoded in a TCP option. Let us now analyze the Multipath TCP handshake in more details.

.. _mptcp-initial-mptcp-handshake:

The Multipath TCP handshake
---------------------------

	  
A Multipath TCP connection starts with a three-way handshake like a regular TCP connection. To indicate that it wishes to use Multipath TCP, the client adds the ``MP_CAPABLE`` option to the ``SYN`` segment. In the ``SYN`` segment, this option only contains some flags and occupies 4 bytes. The server replies with a ``SYN+ACK`` segment than contains an ``MP_CAPABLE`` option including a server generated 64 bits random key that will be used to authenticate connections over different paths. The client concludes the handshake by sending an ``MP_CAPABLE`` option in the ``ACK`` segment containing the random keys chosen by the client and the server.

.. _fig-tcp-handshake-mptcp:
.. tikz:: Negotiating the utilization of Multipath TCP during the three-way handshake
   :libs: positioning, matrix, arrows, math

   \tikzmath{\c1=1;\c2=1.5; \s1=8; \s2=8.5; \max=6; }
   
   \tikzstyle{arrow} = [thick,->,>=stealth]
   \tikzset{state/.style={rectangle, dashed, draw, fill=white} }
   \node [black, fill=white] at (\c1,\max) {Client};
   \node [black, fill=white] at (\s1,\max) {Server};
   
   \draw[black,very thick,->] (\c1,\max-0.5) -- (\c1,0.5);
   \draw[black,very thick,->] (\s1,\max-0.5) -- (\s1,0.5);
   
   \tikzmath{\y=\max-1;}
   
   \draw[blue,thick, ->] (\c1,\y) -- (\s1,\y-1) node [midway, align=left, fill=white] {SYN\small{[seq=x]}\\\small{MPC[flags]}};
   \draw[blue,thick, ->] (\s1,\y-1) -- (\c1,\y-2) node [midway, align=left, fill=white] {SYN+ACK\small{[seq=y,ack=x+1]}\\\small{MPC[flags,$Server_{key}$]}};
   \draw[blue,thick, ->] (\c1,\y-2.1) -- (\s1,\y-3) node [midway, align=left, fill=white] {ACK\small{[seq=x+1,ack=y+1]}\\\small{MPC[flags,$Client_{key}$,$Server_{key}$]}};


   
.. note:: Multipath TCP version 0
   
   The first version of Multipath TCP used a slightly different handshake :cite:p:`rfc6824`. The ``MP_CAPABLE`` option sent by the client contains the 64 bits key chosen by the client. The ``SYN+ACK`` segment contains an ``MP_CAPABLE`` option with 64 bits key chosen by the server. The client echoes the client and server keys in the third ``ACK`` of the handshake. 

          
   .. _fig-tcp-handshake-mptcp-v0:
   .. tikz:: Negotiating the utilization of Multipath TCP version 0
      :libs: positioning, matrix, arrows, math

      \tikzmath{\c1=1;\c2=1.5; \s1=8; \s2=8.5; \max=6; }
   
      \tikzstyle{arrow} = [thick,->,>=stealth]
      \tikzset{state/.style={rectangle, dashed, draw, fill=white} }
      \node [black, fill=white] at (\c1,\max) {Client};
      \node [black, fill=white] at (\s1,\max) {Server};
   
      \draw[black,very thick,->] (\c1,\max-0.5) -- (\c1,0.5);
      \draw[black,very thick,->] (\s1,\max-0.5) -- (\s1,0.5);
   
      \tikzmath{\y=\max-1;}
   
      \draw[blue,thick, ->] (\c1,\y) -- (\s1,\y-1) node [midway, align=left, fill=white] {SYN\small{[seq=x]}\\\small{MPC[flags,$Client_{key}$]}};
      \draw[blue,thick, ->] (\s1,\y-1) -- (\c1,\y-2) node [midway, align=left, fill=white] {SYN+ACK\small{[seq=y,ack=x+1]}\\\small{MPC[flags,$Server_{key}$]}};
      \draw[blue,thick, ->] (\c1,\y-2.1) -- (\s1,\y-3) node [midway, align=left, fill=white] {ACK\small{[seq=x+1,ack=y+1]}\\\small{MPC[flags,$Client_{key}$,$Server_{key}$]}};


The 64 bits random keys chosen by the client and the server play three different roles in Multipath TCP. Their first role is to identify the Multipath TCP connection to which an additional connection must be attached. Since a Multipath TCP connection can combine several TCP connections, Multipath TCP cannot use the IP addresses and port numbers to identify a TCP connection. Multipath TCP uses a specific identifier that is called a token. For technical reasons, this token is derived from the 64 bits key as the most significant 32 bits of the SHA-256 :cite:p:`rfc6234` hash of the key. The second role of the 64 bits keys is to authenticate the establishment of additional connections as we will see shortly. Finally, the keys are also used to compute random initial sequence numbers.

The main benefit of Multipath TCP is that a Multipath TCP connection can combine different TCP connections that potentially use different paths. Starting from now on, we will consider a client with two network interfaces and a server with one network interface. This could for example correspond to a client application running on a smartphone that interacts with a server. We explore more complex scenarios later.

.. In the figures below, the blue arrows correspond to the segments sent over the first interface while the red arrows represent the segments sent over the second interface. In practice, these "interfaces" do not need to be physical interfaces. For example, the red arrows could correspond to IPv6 while the blue arrows correspond to IPv4.

We can know explain how a Multipath TCP connection can combine different TCP connections. According to the Multipath TCP specification, these connections are called subflows :cite:p:`rfc8684`. We also adopt this terminology in this document. :numref:`fig-mptcp-join` shows a Multipath TCP that combines two subflows. To establish the Multipath TCP connection, the client initiates the initial subflow by using the ``MP_CAPABLE`` option during the three-way handshake. At the end of the initial handshake, the client and the server have exchanged their keys. Based on their keys, they have both computed the token that the remote host uses to identify the Multipath TCP connection.

To attach a second subflow to this Multipath TCP connection, the client needs to create it. For this, it starts a three-way handshake with the server by sending a ``SYN`` segment containing the ``MP_JOIN`` option. This option indicates that the client uses Multipath TCP and wishes to attach this new connection to an existing Multipath TCP connection. The ``MP_JOIN`` option contains two important fields:

 - the token that the server uses to identify the Multipath TCP connection
 - a random nonce

The client has derived the token from the key announced by the server in the ``MP_CAPABLE`` option of the ``SYN+ACK`` segment on the initial subflow. Thanks to this token, the server knows to which Multipath TCP connection the new subflow needs to be attached.

.. todo:: discuss security concerns

The server uses the random nonce sent by the client and its own random nonce to prove its knowledge of the keys exchanged during the initial handshake. The server computes :math:`HMAC(Key=(Server_{key}||Client_{key}), Msg=(nonce_{Server}||nonce_{Client}))`, where ``||`` denotes the concatenation operation. It then returns the high order 64 bits of this HMAC in the ``MP_JOIN`` option of the ``SYN+ACK`` segment together with its 32 bits nonce. The client computes :math:`HMAC(Key=(Client_{key}||Server_{key}), Msg=(nonce_{Client}||nonce_{Server}))` and sends the 160 bits HMAC in the ``ACK`` segment. 
            

.. _fig-mptcp-join:
.. tikz:: A client creates a second subflow by creating a TCP connection with the ``MP_JOIN`` option
   :libs: positioning, matrix, arrows, math

   \tikzmath{\c1=1;\c2=1.5; \s1=8; \s2=8.5; \max=9; }
   
   \tikzstyle{arrow} = [thick,->,>=stealth]
   \tikzset{state/.style={rectangle, dashed, draw, fill=white} }
   \node [black, fill=white] at (\c1,\max) {Client};
   \node [black, fill=white] at (\s1,\max) {Server};
   
   \draw[blue,very thick,->] (\c1,\max-0.5) -- (\c1,0.5);
   \draw[black,very thick,->] (\s1,\max-0.5) -- (\s1,0.5);
   \draw[red,very thick,->] (\c2,\max-0.5) -- (\c2,0.5);
   
   
   \tikzmath{\y=\max-1;}
   
   \draw[blue,thick, ->] (\c1,\y) -- (\s1,\y-1) node [midway, align=left, fill=white] {SYN\small{[seq=x]}\\\small{MPC[flags]}};
   \draw[blue,thick, ->] (\s1,\y-1) -- (\c1,\y-2) node [midway, align=left, fill=white] {SYN+ACK\small{[seq=y,ack=x+1]}\\\small{MPC[flags,$S_{key}$]}};
   \draw[blue,thick, ->] (\c1,\y-2.1) -- (\s1,\y-3) node [midway, align=left, fill=white] {ACK\small{[seq=x+1,ack=y+1]}\\\small{MPC[flags,$C_{key}$,$S_{key}$]}};

   
   \tikzmath{\y=\max-4.5;}
   
   \draw[red,thick, ->] (\c2,\y) -- (\s1,\y-1) node [midway, align=left, fill=white] {SYN\small{[seq=x]}\\\small{MP\_JOIN[$S_{token}$,$nonce_{C}$]}};
   \draw[red,thick, ->] (\s1,\y-1) -- (\c2,\y-2) node [midway, align=left, fill=white] {SYN+ACK\small{[seq=y,ack=x+1]}\\\small{MP\_JOIN[$HMAC_{S}$,$nonce_{S}$]}};
   \draw[red,thick, ->] (\c2,\y-2.1) -- (\s1,\y-3) node [midway, align=left, fill=white] {ACK\small{[seq=x+1,ack=y+1]}\\\small{MP\_JOIN[$HMAC_{C}$]}};



.. note:: Generating Multipath TCP keys


   From a security viewpoint, the keys that Multipath TCP hosts exchange in the ``MP_CAPABLE`` option should be completely random to prevent them from being guessed by attackers. However, since the token is derived from the key, it cannot be completely random. A host will typically generate a random key and hash it into a token to verify that it does not correspond to an existing connection. On clients, with a few tens of connections, this is not a concern, but on servers, the delay to generate random keys increases with the number of established Multipath TCP connections :cite:`Raiciu_Hard:2012`. This does not prevent servers from supporting large numbers of Multipath TCP connections :cite:`keukeleire2020increasing`.
   
A Multipath TCP connection combines a number of subflows which can change during the connection lifetime. It starts with an initial subflow, but this subflow may terminate before the connection. A Multipath TCP connection is a pair of states that are maintained on the client and the server.
	  
The above figure shows how a client adds a subflow to an existing Multipath TCP connection. This is the most common way of adding subflows to a connection. According to the specification, a server could also add subflows to a Multipath TCP connection. For this, the server needs to be able to determine the client addresses. This is the role of the address subflow management parts of Multipath TCP.

.. _mptcp-addr-management:

Address and subflow management
==============================

Each Internet host has one address per network interface. A smartphone with active Wi-Fi and cellular interfaces has two network addresses. With the advent of IPv6, a large fraction of the hosts are dual-stack and have both an IPv4 and an IPv6 address for each network interface. Multipath TCP specifies options that allow a host to advertise all its addresses to the other host. Given the limited size of the TCP header, these options cannot be exchanged during the handshake. They are typically attached to packets that carry acknowledgments.

Each host maintains a list of its active addresses and associates a numeric identifier to each address. To advertise an address, the host simple adds the Multipath TCP ``ADD_ADDR`` option to one outgoing packet. This option contains four main fields:

 - the IPv4 or IPv6 address of the host
 - the numeric identifier of the address
 - an optional port number
 - a truncated HMAC to authenticate the address advertisement


The IP address is the main information contained in the ``ADD_ADDR`` option. The identifier allows the host to advertise the list of all its addresses one option at a time. The port number allows to indicate that the hosts listens to another port number than the one used for the subflow where the option is sent. This can be useful if a client wishes to accept subflows initiated by the server or if a server uses another port to listen for additional subflows. The HMAC is the 64 bits truncation of :math:`HMAC(Key=(Server_{key}||Client_{key}), Msg=(Address identifier||IP address|| port))` when the server advertises an address and :math:`HMAC(Key=(Client_{key}||Server_{key}), Msg=(Address identifier||IP address|| port))` for an address advertised by the client. The HMAC allows to prevent attacks where an attacker sends spoofed packets containing an ``ADD_ADDR`` option.

In addition to these four fields, the ``ADD_ADDR`` option contains an ``Echo`` bit. The ``ADD_ADDR`` option is usually sent inside a TCP acknowledgment. A host can easily send an acknowledgment even if it did not recently receive data. Unfortunately, TCP acknowledgments are, by design, unreliable. As TCP uses cumulative acknowledgments, the loss of an acknowledgment is compensated by the next acknowledgment. This is true for the acknowledgment number, but not for the options that were contained in the loss packet. The first version of Multipath TCP did not try to deal with the loss of ``ADD_ADDR`` options. The current version relies on the ``Echo``. A host advertises an address by sending its ``ADD_ADDR`` option with the ``Echo`` bit set to ``0``. To confirm the reception of this address, the peer simply replies with an acknowledgment containing the same option but with its ``Echo`` bit set to one. A host that sent an ``ADD_ADDR`` option needs to retransmit it if it does not receive it back. This is illustrated in :numref:`fig-mptcp-addaddr`.

.. _fig-mptcp-addaddr:
.. tikz:: Thanks to the Echo bit, a Multipath TCP host can retransmit lost ADD_ADDR options. 
   :libs: positioning, matrix, arrows.meta, math

   \tikzmath{\c1=1;\c2=1.5; \s1=8; \s2=8.5; \max=6; }
   
   \tikzstyle{arrow} = [thick,->,>=stealth]
   \tikzset{state/.style={rectangle, dashed, draw, fill=white} }
   \node [black, fill=white] at (\c1,\max) {Client};
   \node [black, fill=white] at (\s1,\max) {Server};
   
   \draw[blue,very thick,->] (\c1,\max-0.5) -- (\c1,0.5);
   \draw[black,very thick,->] (\s1,\max-0.5) -- (\s1,0.5);
   \draw[red,very thick,->] (\c2,\max-0.5) -- (\c2,0.5);
   
   
   \tikzmath{\y=\max-1;}
   
   \draw[blue,thick, -Circle] (\c1,\y) -- (\s1,\y-1) node [midway, align=left, fill=white] {ACK\small{[ADD\_ADDR(E=0,Id=1,1.2.3.4:10)]}};
  
   \draw[blue,thick, ->] (\c1,\y-3) -- (\s1,\y-4) node [midway, align=left, fill=white] {ACK\small{[ADD\_ADDR(E=0,Id=1,1.2.3.4:10)]}};

   \draw[blue,thick, ->] (\s1,\y-4) -- (\c1,\y-5) node [midway, align=left, fill=white] {ACK\small{[ADD\_ADDR(E=1,Id=1,1.2.3.4:10)]}};


Thanks to the ``ADD_ADDR`` option, a host can advertise all its addresses at the beginning of a Multipath TCP connection. Since the option can be sent at any time, a mobile host that learns a new address, e.g. a smartphone attached to a new Wi-Fi network, can advertise it immediately. This makes Multipath TCP agile on mobile hosts. A host may also stop being able to use an IP address. This occurs when a mobile hosts goes away from a wireless network. In this case, the host should inform its peer about the loss of the corresponding address. This is the role of the ``REMOVE_ADDR`` option that contains the numeric identifier of the removed address. In contrast with the ``ADD_ADDR`` option, the ``REMOVE_ADDR`` option is not authenticated using a truncated HMAC. The protocol specification suggests that when a host receives a ``REMOVE_ADDR`` option, it should first check whether it is currently used by an active subflow. If no, the address can be removed. If yes, it should send a TCP Keepalive on this subflow to verify whether the address still works. If it does not receive a response to its keepalive, the address can be removed and the associated subflow is reset. Otherwise, the ``REMOVE_ADDR`` option is ignored.   
   
Multipath TCP hosts use the ``ADD_ADDR`` and ``REMOVE_ADDR`` options to maintain the list of addresses used by their peer. However, this is not the only source of information that Multipath TCP uses. A Multipath TCP hosts also learns the source addresses of the established subflows. The first addresses are those used for the initial subflow. The client remembers the server's address as address ``0`` on this Multipath TCP connection. The server does the same with the client address. When the client creates a new subflow, it places the numeric identifier of the source address of this subflow in the ``MP_JOIN`` option. This enables the server to learn additional addresses and their associated numeric identifiers. This is illustrated in :numref:`fig-mptcp-addr-management`. The server first learns that the client is reachable via the address used for the initial subflow (:math:`IP_{A}`). The identifier of this address is :math:`0`. Then, the server learns that the client is also reachable through IP address :math:`IP_{B}`. Thanks to the identifier contained in the ``MP_JOIN`` option, the server also learns the identifier (:math:`2`) of this address. Then, the server learns the third address ($IP_{C}$) using the ``ADD_ADDR`` option.


.. _fig-mptcp-addr-management:
.. tikz:: A Multipath TCP hosts remembers the addresses used by its peer
   :libs: positioning, matrix, arrows.meta, math

   \tikzmath{\c1=1;\c2=1.5; \s1=8; \s2=8.5; \max=12; }
   
   \tikzstyle{arrow} = [thick,->,>=stealth]
   \tikzset{state/.style={rectangle, dashed, draw, fill=white} }
   \node [black, fill=white] at (\c1,\max+0.5) {Client};
   \node [black, fill=white] at (\s1,\max+0.5) {Server};
   
   \draw[blue,very thick,->] (\c1,\max-0.5) -- (\c1,0.5);
   \draw[black,very thick,->] (\s1,\max-0.5) -- (\s1,0.5);
   \draw[red,very thick,->] (\c2,\max-0.5) -- (\c2,0.5);
   \node [blue, fill=white] at (\c1-0.5,\max) {$IP_{A}$};
   \node [red, fill=white] at (\c2,\max) {$IP_{B}$};
   
   \tikzmath{\y=\max-1;}
   
   \draw[blue,thick, ->] (\c1,\y) -- (\s1,\y-1) node [midway, align=left, fill=white] {SYN \small{MPC}};
   \draw[blue,thick, ->] (\s1,\y-1) -- (\c1,\y-2) node [midway, align=left, fill=white] {SYN+ACK \small{MPC}};
   \draw[blue,thick, ->] (\c1,\y-2.1) -- (\s1,\y-3) node [midway, align=left, fill=white] {ACK \small{MPC}};
   \node [black,fill=white,align=left] at (\s1+1,\y-3.7) {Client addrs: $0:IP_{A}$};
   
   \tikzmath{\y=\max-4.5;}
   
   \draw[red,thick, ->] (\c2,\y) -- (\s1,\y-1) node [midway, align=left, fill=white] {SYN \small{MP\_JOIN[id=2]}};
   \draw[red,thick, ->] (\s1,\y-1) -- (\c2,\y-2) node [midway, align=left, fill=white] {SYN+ACK \small{MP\_JOIN}};
   \draw[red,thick, ->] (\c2,\y-2.1) -- (\s1,\y-3) node [midway, align=left, fill=white] {ACK \small{MP\_JOIN}};
   
   \node [black,fill=white,align=left] at (\s1+1,\y-3.7) {Client addrs: $0:IP_{A}$,$2:IP_{B}$};

   \draw[blue,thick, ->] (\c1,\y-4) -- (\s1,\y-5) node [midway, align=left, fill=white] {ACK \small{ADD\_ADDR[id=1,$IP_{C}$]}};

   \node [black,fill=white] at (\s1+1,\y-5.7) {Client addrs: $0:IP_{A}$,$1:IP_{C}$,$2:IP_{B}$};

   
   
.. note:: Is the ``ADD_ADDR`` option required on all Multipath TCP hosts ?

   The previous section has explained how Multipath TCP hosts learn the addresses of their peers by using the ``ADD_ADDR`` and ``REMOVE_ADDR`` options. These options are important for a server that has multiple addresses (e.g. an IPv4 and an IPv6 address) and wants to advertise them to its clients. On the other hand, servers rarely create subflows and thus they do not really need to learn the client addresses. In fact, Apple's implementation of Multipath TCP on the iPhones does not use the ``ADD_ADDR`` option. iPhones simply create subflows over the cellular and Wi-Fi interfaces as when needed and the server relies on the ``MP_JOIN`` option to validate these subflows. It is interesting to note that the ``REMOVE_ADDR`` option remains useful even if the ``ADD_ADDR`` option is not used. Consider a smartphone that has created an initial subflow over its Wi-Fi interface and a second subflow over the cellular one. If the smartphone looses its Wi-Fi interface, it can send a ``REMOVE_ADDR`` option over the subflow that uses the cellular interface to inform the server that it cannot be reached anymore through its Wi-Fi interface. 

   
.. _mptcp-data-transfer:

Data transfer
=============

Thanks to the ``MP_CAPABLE`` and ``MP_JOIN`` option, Multipath TCP hosts can associate one of more subflows to a Multipath TCP connection. Each host can send and receive data on any of the established subflows. As these subflows follow different paths, packets experience different delays. To preserve the in-order bytestream, the receiver must be able to reorder the data received over the different subflows.

A simple approach to perform this reordering would be to rely on the TCP sequence number that is included in the TCP header. This approach is illustrated in :numref:`fig-mptcp-dss-naive`. The client creates two subflows and uses the same initial sequence numbers on the different subflows. The server also selects the same initial sequence numbers. The client then sends three bytes: ``A`` over the initial subflow, ``B`` over the second subflow and ``C`` over the initial one. Each byte has its own sequence number and the receiver can reorder them. However, note that sequence number ``x+2`` is not sent over the initial subflow. Furthermore, sequence numbers ``x+1`` and ``x+3`` are not sent over the second subflow. 

.. _fig-mptcp-dss-naive:
.. tikz:: A naive approach to exchange data over different subflows
   :libs: positioning, matrix, arrows, math

   \tikzmath{\c1=1;\c2=1.5; \s1=8; \s2=8.5; \max=12; }
   
   \tikzstyle{arrow} = [thick,->,>=stealth]
   \tikzset{state/.style={rectangle, dashed, draw, fill=white} }
   \node [black, fill=white] at (\c1,\max+0.5) {Client};
   \node [black, fill=white] at (\s1,\max+0.5) {Server};
   
   \draw[blue,very thick,->] (\c1,\max-0.5) -- (\c1,0.5);
   \draw[black,very thick,->] (\s1,\max-0.5) -- (\s1,0.5);
   \draw[red,very thick,->] (\c2,\max-0.5) -- (\c2,0.5);

   
   \tikzmath{\y=\max-1;}
   
   \draw[blue,thick, ->] (\c1,\y) -- (\s1,\y-1) node [midway, align=left, fill=white] {SYN[seq=x] \small{MPC}};
   \draw[blue,thick, ->] (\s1,\y-1) -- (\c1,\y-2) node [midway, align=left, fill=white] {SYN+ACK[seq=y,ack=x+1] \small{MPC}};
   \draw[blue,thick, ->] (\c1,\y-2.1) -- (\s1,\y-3) node [midway, align=left, fill=white] {ACK[ack=y+1] \small{MPC}};
   
   \tikzmath{\y=\max-4.5;}
   
   \draw[red,thick, ->] (\c2,\y) -- (\s1,\y-1) node [midway, align=left, fill=white] {SYN[seq=x] \small{MP\_JOIN[id=2]}};
   \draw[red,thick, ->] (\s1,\y-1) -- (\c2,\y-2) node [midway, align=left, fill=white] {SYN+ACK[seq=y,ack=x+1] \small{MP\_JOIN}};
   \draw[red,thick, ->] (\c2,\y-2.1) -- (\s1,\y-3) node [midway, align=left, fill=white] {ACK[ack=y+1] \small{MP\_JOIN}};
   

   \draw[blue,thick, ->] (\c1,\y-4) -- (\s1,\y-5) node [midway, align=left, fill=white] {[seq=x+1] "A"};
   \draw[red,thick, ->] (\c2,\y-4.5) -- (\s1,\y-5.5) node [midway, align=left, fill=white] {[seq=x+2] "B"};
   \draw[blue,thick, ->] (\c1,\y-5) -- (\s1,\y-6) node [midway, align=left, fill=white] {[seq=x+3] "C"};   


Unfortunately, this simple approach suffers from several problems. First, it assumes that the client and the server use the same initial sequence numbers. On the client side, this might be feasible, but on the server side, this would prohibit the utilization of techniques such as SYN cookies that are important to protect from denial of service attacks. Another concern is that there will be gaps in the sequence numbers that are used over each path. These gaps might cause problems with middleboxes such as firewalls. The same problem applies for the acknowledgments. Although TCP supports selective acknowledgments :rfc:`2018`, these were not designed to support a large number of gaps.


Multipath TCP solves these problems by using a second level of sequence numbers that are encoded inside TCP options. Conceptually, Multipath TCP associates a data sequence number to the first byte of the payload of each TCP packet. Each Multipath TCP packet carries two different sequence numbers. The first is the sequence number that is included in the TCP header and is called the subflow sequence number. This sequence number plays the same role as in a regular TCP connection. It enables the receiver to reorder the received packets on a given subflow and detect losses. The data sequence number corresponds to the bytestream. It indicates the position of the first byte of the payload of the TCP packet in the bytestream. This data sequence number is used by the receiver to reorder the data received over different subflows and detect losses at this level. Multipath TCP also uses acknowledgments to confirm the reception of data. At the subflow level these are regular TCP acknowledgments (or selective acknowledgments if this extension is active). At the Multipath TCP connection level, the receiver always returns a data acknowledgment that contains the next expected in-sequence data sequence number. This is illustrated in :numref:`fig-mptcp-dss-concept`. 


The client sends the first byte of the bytestream over the initial subflow. This byte is sent in a TCP packet whose sequence number is ``x+1``. It carries a Multipath TCP option that contains the data sequence number, i.e. ``0`` since this is the first byte of the bytestream. The server returns an acknowledgment that indicates that the ``x+2`` is the next expected sequence number over the initial subflow. This TCP ACK also contains a Multipath TCP option that indicates that ``1`` is the next expected data sequence number. The sends the second byte over the second subflow. For this, it sends a packet whose sequence number is set to ``w+1``, i.e. the first sequence number over this subflow. This packet contains a Multipath TCP option that indicates that this is the second byte (data sequence set to ``1``) of the bytestream. The server confirms the reception of this packet with an acknowledgment.

.. _fig-mptcp-dss-concept:
.. tikz:: Multipath TCP relies on data sequence numbers and acknowledgments
   :libs: positioning, matrix, arrows, math

   \tikzmath{\c1=1;\c2=1.5; \s1=8; \s2=8.5; \max=17; }
   
   \tikzstyle{arrow} = [thick,->,>=stealth]
   \tikzset{state/.style={rectangle, dashed, draw, fill=white} }
   \node [black, fill=white] at (\c1,\max+0.5) {Client};
   \node [black, fill=white] at (\s1,\max+0.5) {Server};
   
   \draw[blue,very thick,->] (\c1,\max-0.5) -- (\c1,0.5);
   \draw[black,very thick,->] (\s1,\max-0.5) -- (\s1,0.5);
   \draw[red,very thick,->] (\c2,\max-0.5) -- (\c2,0.5);

   
   \tikzmath{\y=\max-1;}
   
   \draw[blue,thick, ->] (\c1,\y) -- (\s1,\y-1) node [midway, align=left, fill=white] {SYN[seq=x] \small{MPC}};
   \draw[blue,thick, ->] (\s1,\y-1) -- (\c1,\y-2) node [midway, align=left, fill=white] {SYN+ACK[seq=y,ack=x+1] \small{MPC}};
   \draw[blue,thick, ->] (\c1,\y-2.1) -- (\s1,\y-3) node [midway, align=left, fill=white] {ACK[ack=y+1] \small{MPC}};
   
   \tikzmath{\y=\max-4.5;}
   
   \draw[red,thick, ->] (\c2,\y) -- (\s1,\y-1) node [midway, align=left, fill=white] {SYN[seq=w] \small{MP\_JOIN[id=2]}};
   \draw[red,thick, ->] (\s1,\y-1) -- (\c2,\y-2) node [midway, align=left, fill=white] {SYN+ACK[seq=z,ack=w+1] \small{MP\_JOIN}};
   \draw[red,thick, ->] (\c2,\y-2.1) -- (\s1,\y-3) node [midway, align=left, fill=white] {ACK[ack=z+1] \small{MP\_JOIN}};
   

   \draw[blue,thick, ->] (\c1,\y-4) -- (\s1,\y-5) node [midway, align=left, fill=white] {[seq=x+1]\small{DS[s=0]} "A"};
   \draw[blue,thick, ->] (\s1,\y-5) -- (\c1,\y-6) node [midway, align=left, fill=white] {ACK [ack=x+2]\small{DS[a=1]}};
   \draw[red,thick, ->] (\c2,\y-6) -- (\s1,\y-7) node [midway, align=left, fill=white] {[seq=w+1]\small{DS[s=1]} "B"};
      \draw[red,thick, ->] (\s1,\y-7) -- (\c2,\y-8) node [midway, align=left, fill=white] {ACK [ack=w+2]\small{DS[a=2]}};
   \draw[blue,thick, ->] (\c1,\y-8.5) -- (\s1,\y-9.5) node [midway, align=left, fill=white] {[seq=x+2]\small{DS[s=2]} "C"};   
   \draw[blue,thick, ->] (\s1,\y-9.5) -- (\c1,\y-10.5) node [midway, align=left, fill=white] {ACK [ack=x+2]\small{DS[a=3]}};   


:numref:`fig-mptcp-dss-concept2` shows a slightly different example where the first data packet sent by the client is lost. When the server receives the second byte of the bytestream on the second subflow, it acknowledges it at the subflow level (``ack=w+2``) but not at the connection level since the previous byte of the bytestream is missing. The server stores the received byte in the reordering buffer associated with the connection. When the server receives the second packet sent over the initial subflow, it stores it in the buffer associated with the initial subflow. Since it has neither received the byte that has sequence number ``x+1`` on the initial subflow, it cannot update its acknowledgment number. It could send a selective acknowledgment if these were enabled on the connection. The retransmission of the first data packet sent over the initial subflow fills the buffer associated to this subflow. The server can thus update the subflow level acknowledgment number (``ack=x+2``). The data received in order can now be passed to the connection-level buffer. The data at this level is also in-sequence and the server returns a data acknowledgment indicating that the next data sequence number it expects is ``3``. The three bytes ``ABC`` are delivered in sequence to the server application. 
   
.. _fig-mptcp-dss-concept2:
.. tikz:: Multipath TCP copes with packet losses 
   :libs: positioning, matrix, arrows, arrows.meta, math

   \tikzmath{\c1=1;\c2=1.5; \s1=8; \s2=8.5; \max=9; }
   
   \tikzstyle{arrow} = [thick,->,>=stealth]
   \tikzset{state/.style={rectangle, dashed, draw, fill=white} }
   \node [black, fill=white] at (\c1,\max+0.5) {Client};
   \node [black, fill=white] at (\s1,\max+0.5) {Server};
   
   \draw[blue,very thick,->] (\c1,\max-0.5) -- (\c1,0.5);
   \draw[black,very thick,->] (\s1,\max-0.5) -- (\s1,0.5);
   \draw[red,very thick,->] (\c2,\max-0.5) -- (\c2,0.5);

   
   \tikzmath{\y=\max-1;}
   

   \draw[blue,thick, -Circle] (\c1,\y) -- (\s1,\y-1) node [midway, align=left, fill=white] {[seq=x+1]\small{DS[s=0]} "A"};

   \draw[red,thick, ->] (\c2,\y-2) -- (\s1,\y-3) node [midway, align=left, fill=white] {[seq=w+1]\small{DS[s=1]} "B"};
      \draw[red,thick, ->] (\s1,\y-3) -- (\c2,\y-4) node [midway, align=left, fill=white] {ACK [ack=w+2]\small{DS[a=0]}};
   \draw[blue,thick, ->] (\c1,\y-4) -- (\s1,\y-5) node [midway, align=left, fill=white] {[seq=x+2]\small{DS[s=2]} "C"};   
   \draw[blue,thick, ->] (\s1,\y-5) -- (\c1,\y-6) node [midway, align=left, fill=white] {ACK [ack=x+1]\small{DS[a=0]}};   
   \draw[blue,thick, ->] (\c1,\y-6) -- (\s1,\y-7) node [midway, align=left, fill=white] {[seq=x+1]\small{DS[s=0]} "A"};
   \draw[blue,thick, ->] (\s1,\y-7) -- (\c1,\y-8) node [midway, align=left, fill=white] {ACK [ack=x+2]\small{DS[a=3]}};   

The example of :numref:`fig-mptcp-dss-concept2` showed how Multipath TCP copes with packet losses. These are frequent events on a TCP connection. A Multipath TCP only needs to cope with the loss of an entire subflow. Consider the same example as above, but the initial subflow was established over a Wi-Fi interface that stops shortly after the reception of the acknowledgment for the second data packet. The client detects the problem and sends a ``REMOVE_ADDR`` over the second subflow. It also retransmits the first packet that had not been acknowledged, but this time over the second subflow. 

.. _fig-mptcp-dss-concept3:
.. tikz:: Multipath TCP copes with subflow failures 
   :libs: positioning, matrix, arrows.meta, math

   \tikzmath{\c1=1;\c2=1.5; \s1=8; \s2=8.5; \max=11; }
   
   \tikzstyle{arrow} = [thick,->,>=stealth]
   \tikzset{state/.style={rectangle, dashed, draw, fill=white} }
   \node [black, fill=white] at (\c1,\max+0.5) {Client};
   \node [black, fill=white] at (\s1,\max+0.5) {Server};
   
   \draw[blue,very thick,->] (\c1,\max-0.5) -- (\c1,0.5);
   \draw[black,very thick,->] (\s1,\max-0.5) -- (\s1,0.5);
   \draw[red,very thick,->] (\c2,\max-0.5) -- (\c2,0.5);

   
   \tikzmath{\y=\max-1;}
   

   \draw[blue,thick, -Circle] (\c1,\y) -- (\s1,\y-1) node [midway, align=left, fill=white] {[seq=x+1]\small{DS[s=0]} "A"};

   \draw[red,thick, ->] (\c2,\y-2) -- (\s1,\y-3) node [midway, align=left, fill=white] {[seq=w+1]\small{DS[s=1]} "B"};
   \draw[red,thick, ->] (\s1,\y-3) -- (\c2,\y-4) node [midway, align=left, fill=white] {ACK [ack=w+2]\small{DS[a=0]}};
   \draw[blue,thick, ->] (\c1,\y-4) -- (\s1,\y-5) node [midway, align=left, fill=white] {[seq=x+2]\small{DS[s=2]} "C"};   
   \draw[blue,thick, ->] (\s1,\y-5) -- (\c1,\y-6) node [midway, align=left, fill=white] {ACK [ack=x+1]\small{DS[a=0]}};

   \draw[red,thick, ->] (\c2,\y-6.5) -- (\s1,\y-7.5) node [midway, align=left, fill=white] {\small{REMODE\_ADDR[id=0]}};
	  
   \draw[red,thick, ->] (\c2,\y-7) -- (\s1,\y-8) node [midway, align=left, fill=white] {[seq=w+2]\small{DS[s=0]} "A"};
   \draw[red,thick, ->] (\s1,\y-8) -- (\c2,\y-9) node [midway, align=left, fill=white] {ACK [ack=w+3]\small{DS[a=3]}};   


Conceptually, a Multipath TCP implementation can be viewed as composed of a set of queues. On the sender side, the bytestream is pushed in a queue that keeps the data until it has been acknowledged at the connection level. A packet scheduler extracts blocks of data from this queue and places them with the associated date sequence numbers in the per-subflow queues that represent the sending buffers associated to each subflow. TCP uses these per-subflow queues to send the data and perform the retransmission when required. On the receiver side, there is one queue associated with each subflow. This queue corresponds to the TCP receive buffer. TCP uses this queue to reorder the received data based on their TCP sequence numbers, but does not deal with the data sequence numbers that are contained in TCP options. Once data is in-order in a subflow receive buffer, it goes in the connection-level reorder queue that uses the data sequence numbers contained in TCP options to recover the bytestream. Multipath TCP creates the data sequence acknowledgments from the data contained in this buffer. Once data is in-sequence inside this buffer, it is passed to the application through a ``recv`` system call.   

.. tikz:: Architecture of a Multipath TCP implementation
   :libs: positioning, matrix, arrows.meta, math

   \node (A) at (0,0)  {	  	  
   figure from slide
   };

   
.. todo: explain windows and flow control


.. _mptcp-congestion:
   
Congestion control
==================

.. todo:: explain basic idea and the problem of having 

.. Why we need coupled congestion control
	  
LIA
---
:cite:`Wischik_Design:2011` and :cite:`rfc6356`

OLIA
----
:cite:`Khalili_MPTCP:2013`


BALIA
-----

:cite:`peng2014multipath`

MPCC
----

:cite:`gilad2020mpcc`


.. _mptcp-release:
      
Connection release
==================

.. todo:: keepalive and end of a connection

A TCP connection starts with a three-way handshake and ends with either the exchange of ``FIN`` packets to gracefully terminate the connection or when one of the hosts sends a ``RST`` packet. The main benefit of the graceful termination is that both hosts receive the confirmation that all the data that they have sent over the connection has been correctly received. Multipath also supports a graceful termination of the connection. As in regular TCP, this graceful termination is implemented by using a flag that indicates the end of the bytestream. This flag is included in the Data Sequence Number option.

:numref:`fig-mptcp-close` illustrates a graceful Multipath TCP connection release. We assume that the connection has two active subflows. The client sends ``XYZ`` over the initial subflow. Since this is the last byte sent over the bytestream, it adds the ``DATA_FIN`` flag to the data sequence option. This flag consumes one data sequence number as the ``FIN`` flag in the TCP header. The server returns an acknowledgment that confirms the reception of the three bytes at the subflow level (``ack=x+3``). At the connection level, four sequence numbers are acknowledged (``a=y+4``) since the ``DATA_FIN`` flag consumes one sequence number. The server decides to close its bytestream by sending its last byte, ``M``, over the second subflow with the ``DATA_FIN`` flag set. At this point, the Multipath TCP has been gracefully closed. No data will be exchanged over the different subflows. The client and/or the server can terminate the subflows by using packets with either the ``FIN`` or the ``RST`` flag in the TCP header.


.. _fig-mptcp-close:
.. tikz:: Graceful termination of a Multipath TCP connection 
   :libs: positioning, matrix, arrows, arrows.meta, math

   \tikzmath{\c1=1;\c2=1.5; \s1=8; \s2=8.5; \max=8; }
   
   \tikzstyle{arrow} = [thick,->,>=stealth]
   \tikzset{state/.style={rectangle, dashed, draw, fill=white} }
   \node [black, fill=white] at (\c1,\max+0.5) {Client};
   \node [black, fill=white] at (\s1,\max+0.5) {Server};
   
   \draw[blue,very thick,->] (\c1,\max-0.5) -- (\c1,0.5);
   \draw[black,very thick,->] (\s1,\max-0.5) -- (\s1,0.5);
   \draw[red,very thick,->] (\c2,\max-0.5) -- (\c2,0.5);

   
   \tikzmath{\y=\max-1;}

   \draw[blue,dashed,<->] (\c1,\y) -- (\s1,\y) node [midway, fill=white] {Initial subflow};    
   \draw[red,dashed,<->] (\c2,\y-0.5) -- (\s1,\y-0.5) node [midway, fill=white] {Second subflow};
    
   \draw[blue,thick, ->] (\c1,\y-2) -- (\s1,\y-3) node [midway, align=left, fill=white] {[seq=x]\small{DS[s=y,DATA\_FIN]} "XYZ"};
   \draw[blue,thick, ->] (\s1,\y-3) -- (\c1,\y-4) node [midway, align=left, fill=white] {ACK [ack=x+3]\small{DS[a=y+4]}};
   \draw[red,thick, ->] (\s1,\y-4.5) -- (\c2,\y-5.5) node [midway, align=left, fill=white] {[seq=w]\small{DS[s=z,DATA\_FIN]} "M"};
   \draw[red,thick, ->] (\c2,\y-5.5) -- (\s1,\y-6.5) node [midway, align=left, fill=white] {ACK [ack=w+1]\small{DS[a=z+2]}};


The main drawback of exchanging ``DATA_FINs`` to terminate a Multipath TCP is that this takes time. Busy servers might not be willing to spend a long time waiting for the exchange of all these packets if the application already guarantees the correct delivery of the data. A regular TCP server would send a ``RST`` packet to quickly terminate such a connection. However, such ``RST`` packets can lead to denial of service attacks :cite:`rfc5961`. A regular TCP receiver mitigates these attacks by checking the sequence number of the ``RST`` packet :cite:`rfc5691`. The Multipath TCP designers did not consider this approach to be safe since an attacker who is able to observe the packets on one path could send a ``RST`` packet that would terminate all the subflows used by the connection.

To still allow a host to quickly terminate a Multipath TCP connection, Multipath TCP must be able to verify the validity of a packet that terminates a connection. For this, Multipath TCP defines the ``FAST_CLOSE`` option that includes a 64 bits security key. These keys are exchanged during the initial handshake and included in the state associated to a Multipath TCP connection. To quickly close a connection, a host simply needs to send the key of the remote host in a ``FAST_CLOSE`` option sent over one of the active subflows. The Multipath TCP specification defines two different methods to use the ``FAST_CLOSE`` option.

The first solution is to send the ``FAST_CLOSE`` option inside an ``ACK``. Upon reception of such a packet, a host sends a ``RST`` over all active subflows. This is illustrated in :numref:`fig-mptcp-fastclose-a`.


.. _fig-mptcp-fastclose-a:
.. tikz:: Abrupt release of a Multipath TCP connection by sending FAST_CLOSE inside an ACK
   :libs: positioning, matrix, arrows, arrows.meta, math

   \tikzmath{\c1=1;\c2=1.5; \s1=8; \s2=8.5; \max=6; }
   
   \tikzstyle{arrow} = [thick,->,>=stealth]
   \tikzset{state/.style={rectangle, dashed, draw, fill=white} }
   \node [black, fill=white] at (\c1,\max+0.5) {Client};
   \node [black, fill=white] at (\s1,\max+0.5) {Server};
   
   \draw[blue,very thick,->] (\c1,\max-0.5) -- (\c1,0.5);
   \draw[black,very thick,->] (\s1,\max-0.5) -- (\s1,0.5);
   \draw[red,very thick,->] (\c2,\max-0.5) -- (\c2,0.5);

   
   \tikzmath{\y=\max-1;}

   \draw[blue,dashed,<->] (\c1,\y) -- (\s1,\y) node [midway, fill=white] {Initial subflow};    
   \draw[red,dashed,<->] (\c2,\y-0.5) -- (\s1,\y-0.5) node [midway, fill=white] {Second subflow};
   
   \draw[blue,thick, ->] (\c1,\y-2) -- (\s1,\y-3) node [midway, align=left, fill=white] {ACK,FAST\_CLOSE[$Server_{key}$] };
   \draw[blue,thick, ->] (\s1,\y-3) -- (\c1,\y-4) node [midway, align=left, fill=white] {RST};
   \draw[red,thick, ->] (\s1,\y-3.5) -- (\c2,\y-4.5) node [midway, align=left, fill=white] {RST};


.. _fig-mptcp-fastclose-b:
.. tikz:: Abrupt release of a Multipath TCP connection by sending a RST with FAST_CLOSE on all subflows
   :libs: positioning, matrix, arrows, arrows.meta, math

   \tikzmath{\c1=1;\c2=1.5; \s1=8; \s2=8.5; \max=6; }
   
   \tikzstyle{arrow} = [thick,->,>=stealth]
   \tikzset{state/.style={rectangle, dashed, draw, fill=white} }
   \node [black, fill=white] at (\c1,\max+0.5) {Client};
   \node [black, fill=white] at (\s1,\max+0.5) {Server};
   
   \draw[blue,very thick,->] (\c1,\max-0.5) -- (\c1,0.5);
   \draw[black,very thick,->] (\s1,\max-0.5) -- (\s1,0.5);
   \draw[red,very thick,->] (\c2,\max-0.5) -- (\c2,0.5);

   
   \tikzmath{\y=\max-1;}

   \draw[blue,dashed,<->] (\c1,\y) -- (\s1,\y) node [midway, fill=white] {Initial subflow};    
   \draw[red,dashed,<->] (\c2,\y-0.5) -- (\s1,\y-0.5) node [midway, fill=white] {Second subflow};
   
   \draw[blue,thick, ->] (\c1,\y-2) -- (\s1,\y-3) node [midway, align=left, fill=white] {RST,FAST\_CLOSE[$Server_{key}$]};
   \draw[red,thick, ->] (\c2,\y-2.5) -- (\s1,\y-3.5) node [midway, align=left, fill=white] {RST, FAST\_CLOSE[$Server_{key}$]};

   
.. _mptcp-middlebox:   
	  
Coping with middlebox interference
==================================
	  
.. todo: classify the different types of middleboxes and their impact


..  :cite:`Honda_Extend:2011` michio extend tcp

.. tracebox :cite:`Detal_tracebox:2013`


The previous sections have explained how Multipath TCP operates at a high level. They assume a simple network that is mainly composed of hosts, switches and routers. TCP and Multipath TCP are used by the hosts. They rely on IP packets that contain the TCP segments. These packets are forwarded by IP routers and possibly switches at layer-2 before reaching their final destination. In a network that uses layered protocols, the switches only inspect the layer-2 headers, the routers only read and change the layer-2 and layer-3 headers. Neither the switches nor the routers read or modify the payload of the packets that they forward. Unfortunately, this assumption is not true on the global Internet and in enterprise networks. Besides switches and routers, these networks contains other types of equipment that process packets :cite:`sherry2012making`. These devices are usually called middleboxes because they reside in the middle of the network and process packets in different ways. A detailed survey of all the different types of middleboxes is outside the scope of this document. We discuss below some of the popular middleboxes and analyze how they have influenced the design of Multipath TCP.

Our first middlebox is a firewall. A firewall is a device that receives packets, analyzes their contents and then forwards or blocks the packet. The simplest firewalls are the stateless firewalls that accept or reject each individual packet. Such a firewall can accept packet based on the source or destination addresses or port numbers. Some firewalls also check the flags or the IP header or reassemble the received packet fragments. Others analyze the TCP header and verify the utilization of the TCP options. A firewall can be configured using a white list or a black list. A white list specifies all the packet fields that are valid and all the others are invalid. On the other hand, a black list specifies the packets that must be rejected by the firewall and all the others are accepted. Many firewalls use a small white list that defines the TCP options that the firewall accepts. This list typically includes the widely deployed options such as MSS :cite:`rfc793`, timestamps :cite:`rfc7313`, windows scale :cite:`rfc1313` and selective acknowledgments :cite:`rfc2018`. TCP options are encoded using the Kind, Length, Value format shown in :numref:`fig-mptcp-tcp-option`.

.. _fig-mptcp-tcp-option:
.. tikz:: Generic format for TCP options
   :align: center
	   

   \node (A) at (0,0) {
   \begin{bytefield}{32}
   \bitbox{8}{Kind} & \bitbox{8}{Length} & \bitbox{16}{Value ...} 
   \end{bytefield}
   };



It is interesting to explore how such a firewall reacts when it receives a packet containing a TCP option that is not part of its whitelist. There are two possibilities. Some firewalls simply drop the packet, but this blocks a connection that could be totally legitimate. Other firewalls remove the option from the TCP header. This can be done by either removing the bytes that contain the unknown TCP option, adjust the Length field of the IP header, the TCP Header length (and possibly update the padding) and update the TCP checksum. A simpler approach is to replace the bytes of the option with byte ``1``. This corresponds to the standard No-Operation TCP option :cite:`rfc793`. The advantage of this approach is that the firewall only has to recompute the TCP checksum and does not need to adjust the packet length and move data.

The removal of TCP options by firewalls has influenced the design of Multipath TCP. Multipath TCP uses TCP options to exchange different types of information. The information carried in a ``SYN`` is not the same as the one exchanged in data packets. The selective acknowledgments TCP extension :cite:`rfc2018` defines two different options: a two bytes long ``SACK permitted`` that is used inside ``SYN`` and a variable length ``SACK`` option that carries the selective acknowledgments during the data transfer. The first versions of Multipath TCP used a similar approach with different TCP options kinds. However, the Multipath TCP designers feared that some firewalls could accept some of the Multipath TCP options and drop the others. For example, the Multipath TCP option used in the ``SYN`` could pass a firewall that would later drop the options used in data packets. It would have been very difficult for a Multipath TCP implementation to deal with all the corner cases that could happen since Multipath TCP :cite:`rfc8684` currently defines 9 different options. To prevent such problems, Multipath TCP uses a single TCP option kind and each Multipath TCP option contains a subtype field. This increases the length of the Multipath TCP options, but minimizes the risk of middlebox interference.

.. _fig-mptcp-tcp-option2:
.. tikz:: The generic format for Multipath TCP options

   \node (A) at (0,0)  {
   \begin{bytefield}{32}
   \bitheader{0-31}\\
   \bitbox{8}{Kind} & \bitbox{8}{Length} & \bitbox{4}{\tiny{Subtype}} & \bitbox[ltr]{12}{}\\
   \bitbox[blr]{32}{Sub-type specific data}
   \end{bytefield}
   };

   

Before looking at other middleboxes, it is interesting to analyze how a router forwards an IP packet that contains a TCP segment. Consider a router that receives a packet such as the one shown in :numref:`fig-mptcp-ip4tcp`. When a router forwards such a packet, it will read the IP header and may modify the fields highlighted in red:

 - the Differentiated Services Codepoint (DSCP)
 - the Explicit Congestion Notification flags (the CE bit)
 - decrement the Time to Live
 - update the IP header checksum
   
A router will never change any other field of the IP header and will not read the packet payload.

.. _fig-mptcp-ip4tcp: 
.. tikz:: Fields of an IPv4 packet carrying a TCP segment which can be modified by a router

   \node (A) at (0,0)  {
   \definecolor{lightred}{rgb}{1,0.7,0.71}
   \begin{bytefield}{32}
   \bitheader{0-31} \\
   \bitbox{4}{Ver} &  \bitbox{4}{IHL} & \bitbox{6}[bgcolor=lightred]{DSCP} & \bitbox{2}[bgcolor=lightred]{\tiny ECN} & \bitbox{16}{Length}  \\
   \bitbox{16}{Identification} & \bitbox{3}{\tiny Flags} & \bitbox{13}{Offset} \\
   \bitbox{8}[bgcolor=lightred]{TTL} & \bitbox{8}{Protocol} & \bitbox{16}[bgcolor=lightred]{IP Checksum} \\
   \bitbox{32}{Source Address} \\
   \bitbox{32}{Destination Address} \\
   \bitbox{16}{Source Port} &  \bitbox{16}{Destination Port} \\
   \bitbox{32}{Sequence number} \\
   \bitbox{32}{Acknowledgment number } \\   
   \bitbox{4}{Offset} & \bitbox{6}{Res} & \bitbox{1}{\tiny U\\R\\G} & \bitbox{1}{\tiny A\\C\\K} & \bitbox{1}{\tiny P\\S\\H} & \bitbox{1}{\tiny R\\S\\T} & \bitbox{1}{\tiny S\\Y\\N} & \bitbox{1}{\tiny F\\I\\N} & \bitbox{16}{Window} \\
   \bitbox{16}{TCP Checksum} &  \bitbox{16}{Urgent Pointer} \\
   \end{bytefield}
   };

Today, most TCP stacks set the Don't Fragment flag when sending TCP packets. This implies that IPv4 routers will not fragment the packet. Even if a router fragments an IPv4 packet, this is transparent for the TCP stack since the IP stack on the receiver will reassemble the packet before passing its contents to TCP.

Unfortunately, deployed networks also contain Network Address Translators (NAT) :cite:`rfc3022`. We consider three different types of NATs because they interfere in different ways with TCP extensions such as Multipath TCP. A NAT is usually located at the boundary between a private network and the Internet. The hosts of the private network use private IP addresses :cite:`rfc1918` and the NAT is configured with a pool of public addresses. When the NAT receives an IP packet from a host in the private network, its maps the source IP address to a public one and rewrites the packet before forwarding it to the public Internet. When the NAT receives a packet from the Internet, it checks if there is a mapping for the packet's destination address. If so, the destination address is translated and the packet is forwarded to the private host. 


.. _fig-mptcp-ip4tcp-nat: 
.. tikz:: Fields of an IPv4 packet carrying a TCP segment which can be modified by a simple NAT

   \node (A) at (0,0)  {
   \definecolor{lightred}{rgb}{1,0.7,0.71}
   \begin{bytefield}{32}
   \bitheader{0-31}\\
   \bitbox{4}{Ver} &  \bitbox{4}{IHL} & \bitbox{6}[bgcolor=lightred]{DSCP} & \bitbox{2}[bgcolor=lightred]{\tiny ECN} & \bitbox{16}{Length}  \\
   \bitbox{16}{Identification} & \bitbox{3}{\tiny Flags} & \bitbox{13}{Offset} \\
   \bitbox{8}[bgcolor=lightred]{TTL} & \bitbox{8}{Protocol} & \bitbox{16}[bgcolor=lightred]{IP Checksum} \\
   \bitbox{32}[bgcolor=lightred]{Source Address} \\
   \bitbox{32}[bgcolor=lightred]{Destination Address} \\
   \bitbox{16}{Source Port} &  \bitbox{16}{Destination Port} \\
   \bitbox{32}{Sequence number} \\
   \bitbox{32}{Acknowledgment number } \\   
   \bitbox{4}{Offset} & \bitbox{6}{Res} & \bitbox{1}{\tiny U\\R\\G} & \bitbox{1}{\tiny A\\C\\K} & \bitbox{1}{\tiny P\\S\\H} & \bitbox{1}{\tiny R\\S\\T} & \bitbox{1}{\tiny S\\Y\\N} & \bitbox{1}{\tiny F\\I\\N} & \bitbox{16}{Window} \\
   \bitbox{16}[bgcolor=lightred]{TCP Checksum} &  \bitbox{16}{Urgent Pointer} \\
   \end{bytefield}
   };
   
As illustrated in :numref:`fig-mptcp-ip4tcp-nat`, this NAT updates the source or destination address of the packet depending on the packet direction. This modification forces the NAT to recompute the IP checksum but also the TCP checksum since it covers the TCP packet and a pseudo header that includes the IP addresses :cite:`rfc793`. In practice, these two checksums are incrementally updated :cite:`rfc3022` and do not need to be recomputed.

Multipath TCP copes with these NATs by associating an identifier to each address that is used to create a subflow or advertise an address using the ``ADD_ADDR`` option. NAT are not aware of these identifiers and they do not modify them. The ``REMOVE_ADDR`` option only contains the identifier of the address that was removed. With this information, the receiver of the option can easily determine the affected subflows.


Most NAT deployments, in particular with IPv4, use a pool a public addresses that is much smaller than the set of private addresses that need to be mapped. These middleboxes also need to translate the source ports used by the internal hosts to map different private addresses to the same public addresses. These Network Address and Port Translators (NAPT) also modify the source or destination ports in the same way as they modify the addresses. Multipath TCP copes with simple NAPTs as with simple NATs. Unfortunately, most NATs and NAPTs also include Application-Level Gateways (ALG). ALGs were designed to enable applications such as the File Transfer Protocol (FTP) :cite:`rfc959` to be used through NATs and NAPTs. FTP and a few other protocols use IP addresses as parameters of the application-level messages that are exchanged within the bytestream. A simple FTP session is shown in :numref:`fig-mptcp-ftp`. In contrast with many application-level protocols, FTP uses several TCP connection. A FTP sessions starts with a TCP connection established by the client. This connection is called the control connection :cite:`rfc959`. This connection is used to exchange simple commands and the associated responses. The client issues commands such as ``USER`` (to indicate the client username) or ``PASS`` (to provide a password) as a single ASCII line sent over this connection. The server replies with one line that starts with a decimal number that indicates the success of the failure of the command and a textual explanation. However, this is not the only connection used between the client and the server. The client and the server also use data connections. If the client wants to retrieve a file or simply list the names of the files in a given directory, it needs to issue two successive commands. The first command (``PORT``) indicates the data connection that will be used to exchange the result of the subsequent command. The client listens on a local port and provides its IP address and port number as parameters of the ``PORT`` command. Upon reception of this command, the server establish a TCP connection towards the port specific by the client. If the client is behind a NAT, its private IP address and the local port must be translated by the NAT to support the establishment of a server-initiated connection. 



.. code-block:: console
   :name: fig-mptcp-ftp
   :caption: Simple ftp session
	  
   #ftp -4d ftp.belnet.be
   Connected to ftp-brudie.belnet.be.
   220-Welcome to the Belnet public FTP server ftp.belnet.be !

   All access is logged. 
  
   Currently used storage capacity : 38T / 100T on /ftp
   220 193.190.198.27 FTP server ready
   Name (ftp.belnet.be): anonymous
   ---> USER anonymous
   331 Anonymous login ok, send your complete email address as your password
   Password:
   ---> PASS XXXX
   230 Anonymous access granted, restrictions apply
   ---> SYST
   215 UNIX Type: L8
   Remote system type is UNIX.
   Using binary mode to transfer files.
   ftp> dir
   ---> PORT 192,168,0,37,133,67
   200 PORT command successful
   ---> LIST
   150 Opening ASCII mode data connection for file list
   lrw-r--r--   1 ftp      ftp            16 Feb 24  2021 arcolinux -> mirror/arcolinux
   drwxr-xr-x   3 ftp      ftp           101 Jan 12  2021 belnetstyle
   lrw-r--r--   1 ftp      ftp            13 Feb  1  2021 debian -> mirror/debian
   226 Transfer complete
   ftp> quit
   ---> QUIT
   221 Goodbye.


It is interesting to analyze how an ALG modifies a packet that carries such a ``PORT`` command. Let us assume that the ``PORT 192,168,0,37,133,67`` command is sent in a single TCP packet for simplicity. :numref:`fig-mptcp-ip4tcp-ftp-port` shows the contents of the packet sent by the client. :numref:`fig-mptcp-ip4tcp-ftp-port2` shows the packet after its translation by the NAT, assuming that the NAT maps IP address ``192.168.0.37`` onto address ``5.6.7.8``. The packet sent by client contains 26 bytes of payload. The IP packet is thus 66 bytes long.

  
.. _fig-mptcp-ip4tcp-ftp-port: 
.. tikz:: Packet carrying a PORT command sent by a client 

   \node (A) at (0,0)  {
   \definecolor{lightcyan}{rgb}{0.84,1,1}
   \definecolor{lightgray}{gray}{0.8}
   \definecolor{lightred}{rgb}{1,0.7,0.71}
   \begin{bytefield}{32}
   \bitheader{0-31}\\
   \bitbox{4}{Ver} &  \bitbox{4}{IHL} & \bitbox{6}{DSCP} & \bitbox{2}{\tiny ECN} & \bitbox{16}{len=66}  \\
   \bitbox{16}{Identification} & \bitbox{3}{\tiny Flags} & \bitbox{13}{Offset} \\
   \bitbox{8}{TTL} & \bitbox{8}{Protocol} & \bitbox{16}{IP Checksum} \\
   \bitbox{32}[bgcolor=lightcyan]{192.168.0.37} \\
   \bitbox{32}{193.190.198.27} \\
   \bitbox{16}[bgcolor=lightcyan]{57258} &  \bitbox{16}{21} \\
   \bitbox{32}[bgcolor=lightcyan]{seq=12300} \\
   \bitbox{32}{ack=5678} \\   
   \bitbox{4}{Offset} & \bitbox{6}{Res} & \bitbox{1}{\tiny U\\R\\G} & \bitbox{1}{\tiny A\\C\\K} & \bitbox{1}{\tiny P\\S\\H} & \bitbox{1}{\tiny R\\S\\T} & \bitbox{1}{\tiny S\\Y\\N} & \bitbox{1}{\tiny F\\I\\N} & \bitbox{16}{16384} \\
   \bitbox{16}{TCP Checksum} &  \bitbox{16}{Urgent Pointer} \\
   \bitbox[lt]{8}[bgcolor=lightcyan]{P} & \bitbox[t]{8}[bgcolor=lightcyan]{O}&  \bitbox[t]{8}[bgcolor=lightcyan]{R} & \bitbox[tr]{8}[bgcolor=lightcyan]{T} \\
   \bitbox[l]{8}[bgcolor=lightcyan]{} & \bitbox[]{8}[bgcolor=lightcyan]{1}&  \bitbox[]{8}[bgcolor=lightcyan]{9} & \bitbox[r]{8}[bgcolor=lightcyan]{2} \\
   \bitbox[l]{8}[bgcolor=lightcyan]{,} & \bitbox[]{8}[bgcolor=lightcyan]{1}&  \bitbox[]{8}[bgcolor=lightcyan]{6} & \bitbox[r]{8}[bgcolor=lightcyan]{8} \\
   \bitbox[l]{8}[bgcolor=lightcyan]{,} & \bitbox[]{8}[bgcolor=lightcyan]{0}&  \bitbox[]{8}[bgcolor=lightcyan]{,} & \bitbox[r]{8}[bgcolor=lightcyan]{3} \\
   \bitbox[l]{8}[bgcolor=lightcyan]{7} & \bitbox[]{8}[bgcolor=lightcyan]{,}&  \bitbox[]{8}[bgcolor=lightcyan]{1} & \bitbox[r]{8}[bgcolor=lightcyan]{3} \\
   \bitbox[l]{8}[bgcolor=lightcyan]{3} & \bitbox[]{8}[bgcolor=lightcyan]{,}&  \bitbox[b]{8}[bgcolor=lightcyan]{6} & \bitbox[br]{8}[bgcolor=lightcyan]{7} \\
   \bitbox[bl]{8}[bgcolor=lightcyan]{CR} & \bitbox[br]{8}[bgcolor=lightcyan]{LF} \\ 
   \end{bytefield}
   };


The ``PORT 192,168,0,37,133,67`` indicates that the client listens on IP address ``192.168.0.37`` and on port :math:`133*256+67=34115`. Let us assume that the NAT maps this IP address on address ``5.6.7.8`` and port ``34115`` on port :math:`31533=123*256+45`. In ASCII, the ``PORT`` command becomes ``PORT 5,6,7,8,123,45`` and the NAT sends the packet shown in :numref:`fig-mptcp-ip4tcp-ftp-port2`. The fields shown in red have been translated by the NAT. An important point to note contains 21 bytes of payload and not 66 as the packet sent by the client. This implies that the packet sent by the NAT contains the bytes having sequence numbers ``12300`` to ``12320`` while the original packet covered sequence numbers ``12300`` to ``12325``. The NAT will thus need to adjust the sequence number of the subsequent packets sent by the client and also the acknowledgments returned by the server.
   

.. _fig-mptcp-ip4tcp-ftp-port2: 
.. tikz:: Packet carrying a PORT command modified by the FTP ALG used by a NAT

   \node (A) at (0,0)  {
   \definecolor{lightcyan}{rgb}{0.84,1,1}
   \definecolor{lightgray}{gray}{0.8}
   \definecolor{lightred}{rgb}{1,0.7,0.71}
   \begin{bytefield}{32}
   \bitheader{0-31}\\
   \bitbox{4}{Ver} &  \bitbox{4}{IHL} & \bitbox{6}{DSCP} & \bitbox{2}{\tiny ECN} & \bitbox{16}[bgcolor=lightred]{len=61}  \\
   \bitbox{16}{Identification} & \bitbox{3}{\tiny Flags} & \bitbox{13}{Offset} \\
   \bitbox{8}[bgcolor=lightred]{TTL} & \bitbox{8}{Protocol} & \bitbox{16}[bgcolor=lightred]{IP Checksum} \\
   \bitbox{32}[bgcolor=lightred]{5.6.7.8} \\
   \bitbox{32}{193.190.198.27} \\
   \bitbox{16}[bgcolor=lightred]{57258} &  \bitbox{16}{21} \\
   \bitbox{32}[bgcolor=lightcyan]{seq=12300} \\
   \bitbox{32}{ack=5678} \\   
   \bitbox{4}{Offset} & \bitbox{6}{Res} & \bitbox{1}{\tiny U\\R\\G} & \bitbox{1}{\tiny A\\C\\K} & \bitbox{1}{\tiny P\\S\\H} & \bitbox{1}{\tiny R\\S\\T} & \bitbox{1}{\tiny S\\Y\\N} & \bitbox{1}{\tiny F\\I\\N} & \bitbox{16}{16384} \\
   \bitbox{16}[bgcolor=lightred]{TCP Checksum} &  \bitbox{16}{Urgent Pointer} \\
   \bitbox[lt]{8}{P} & \bitbox[t]{8}{O}&  \bitbox[t]{8}{R} & \bitbox[tr]{8}{T} \\
   \bitbox[l]{8}{} & \bitbox[]{8}[bgcolor=lightred]{5}&  \bitbox[]{8}[bgcolor=lightred]{,} & \bitbox[r]{8}[bgcolor=lightred]{6} \\
   \bitbox[l]{8}[bgcolor=lightred]{,} & \bitbox[]{8}[bgcolor=lightred]{7}&  \bitbox[]{8}[bgcolor=lightred]{,} & \bitbox[r]{8}[bgcolor=lightred]{8} \\
   \bitbox[l]{8}[bgcolor=lightred]{,} & \bitbox[]{8}[bgcolor=lightred]{1}&  \bitbox[]{8}[bgcolor=lightred]{2} & \bitbox[r]{8}[bgcolor=lightred]{3} \\
   \bitbox[l]{8}[bgcolor=lightred]{,} & \bitbox[b]{8}[bgcolor=lightred]{4}&  \bitbox[b]{8}[bgcolor=lightred]{5} & \bitbox[br]{8}[bgcolor=lightred]{CR} \\
   \bitbox[lbr]{8}[bgcolor=lightred]{LF} \\
   \end{bytefield}
   };      


As shown by the example above, an ALG can change bytes in the bytestream. It can also remove bytes from the bytestream and also add bytes in the bytestream. This happens notably when the ASCII representation of the public IP address of the NAT is longer than the private IP address of the internal host. This modification of the bytestream had a major impact on the design of Multipath TCP. It mainly affects the Data Sequence Number option that carries the data sequence numbers and acknowledgments. To detect modifications from ALGs and other middleboxes, this option covers a range of sequence numbers in the bytestream and includes an optional checksum that is computed by the Multipath TCP sender and checked by the receiver. If there is a mismatch between the checksum of the option and the data, the receiver stops using Multipath TCP and falls back to regular TCP to preserve the established connection. We discuss this fallback in more details later.  

Our third type of middlebox that splits or coalesces TCP packets. This is not a router that performs IPv4 fragmentation or a host that splits a large IPv6 packets in fragments. In-network fragmentation is mainly disabled in IPv4 network since modern TCP stacks set the ``DF`` flag of the IP header. Those middleboxes do not reside in the middle of the network. They are typically included in the network adapter used by servers and even client hosts. Measurement studies have shown that hosts can reach a higher throughput when sending and receiving large packets. For example, a recent study :cite:`hock2019tcp` reveals that over a 100 Gbps interface, a server was able to reach 25 Gbps with a single TCP connection using 1500 bytes packets. The same connection reached 40 Gbps by using jumbo frames, i.e. 9000 bytes packets. The jumbo frames are supported on modern Gigabit Ethernet networks but they are rarely used outside datacenters because most Internet paths still only supports 1500 bytes packets.

Modern network adapters support TCP Segmentation Offload (TSO) to improve the throughput of TCP connection are reduce the CPU load. In a nutshell, when TSO is enabled, the network adapter exposes a large maximum packet size, e.g. 16 KBytes to more, to the network stack. When the host sends such a large packet, it is automatically segmented in a sequence of small IP packets. On the receiver side, the network adapter performs the reverse operation. It coalesces small received packets into a larger one. :numref:`fig-mptcp-tso` shows a large (2 KBytes long) TCP packet. It is interesting to analyze how the key fields of this packet will be processed by TSO to segment it in the two smaller packets. 


.. _fig-mptcp-tso:
.. tikz:: A large IP packet containing TCP header and data

   \node (A) at (0,0)  {
   \definecolor{lightcyan}{rgb}{0.84,1,1}
   \definecolor{lightgray}{gray}{0.8}
   \definecolor{lightred}{rgb}{1,0.7,0.71}
   \begin{bytefield}{32}
   \bitheader{0-31}\\
   \bitbox{4}{Ver} &  \bitbox{4}{IHL} & \bitbox{6}{DSCP} & \bitbox{2}{\tiny ECN} & \bitbox{16}[bgcolor=lightred]{Length=2040}  \\
   \bitbox{16}[bgcolor=lightred]{Identification} & \bitbox{3}{\tiny Flags} & \bitbox{13}{Offset} \\
   \bitbox{8}{TTL} & \bitbox{8}{Protocol} & \bitbox{16}[bgcolor=lightred]{IP Checksum} \\
   \bitbox{32}{Source Address} \\
   \bitbox{32}{Destination Address} \\
   \bitbox{16}{Source Port} &  \bitbox{16}{Destination Port} \\
   \bitbox{32}[bgcolor=lightred]{Sequence number} \\
   \bitbox{32}{Acknowledgment number } \\
   \bitbox{4}{Offset} & \bitbox{6}{Res} & \bitbox{1}{\tiny U\\R\\G} & \bitbox{1}{\tiny A\\C\\K} & \bitbox{1}{\tiny P\\S\\H} & \bitbox{1}{\tiny R\\S\\T} & \bitbox{1}{\tiny S\\Y\\N} & \bitbox{1}{\tiny F\\I\\N} & \bitbox{16}{Window} \\
   \bitbox{16}[bgcolor=lightred]{Checksum} &  \bitbox{16}{Urgent Pointer} \\
   \wordbox{3}[bgcolor=lightred]{TCP options}\\
   \wordbox[ltr]{2}{Data}\\
   \skippedwords \\
   \wordbox[lrb]{2}{2000 bytes}\\
   \end{bytefield}
   };      


To segment the packet shown in :numref:`fig-mptcp-tso` in two smaller packets, TSO creates two ``1040`` bytes long IPv4 packets. The two small packets have a different IP Identification than the large one. TSO computes an IP checksum for each small packet. It then copies the TCP header of the large packet in both small ones, but with a fed adjustments. The sequence number of the  first small packet is the same as the large one. The sequence number of the sequence small packet is the one of the first packet increased by 1000.
Concerning the TCP options, TSO could analyze the contents of the option and handle each option in a specific manner. For example, TSO could adjust the TCP timestamp option of successive packets. In practice, measurements indicate that TSO simply copies the TCP options field of the large packet in all small packets :cite:`Honda_Extend:2011`. TSO places the first 1000 bytes of the payload of the large packet in the first small one and the last 1000 bytes in the second one. Finally, TSO needs to update the TCP checksum in all the small packets.

The receiver side of these network adapters implement Large Receive Offload (LRO). This basically coalesces the packets that were segmented by TSO. In this case, coalescing packets that carry different TCP options could be problematic since some of the TCP options would be lost in this process. Measurements with different TCP options show that LRO only coalesces packets that have exactly the same set of TCP implementations. 

   
.. tikz:: The TCP header

   \node (A) at (0,0)  {	  
   \begin{bytefield}{32}
   \bitheader{0-31}\\
   \bitbox{16}{Source Port} &  \bitbox{16}{Destination Port} \\
   \bitbox{32}{Sequence number} \\
   \bitbox{32}{Acknowledgment number } \\
   \bitbox{4}{Offset} & \bitbox{6}{Res} & \bitbox{1}{\tiny U\\R\\G} & \bitbox{1}{\tiny A\\C\\K} & \bitbox{1}{\tiny P\\S\\H} & \bitbox{1}{\tiny R\\S\\T} & \bitbox{1}{\tiny S\\Y\\N} & \bitbox{1}{\tiny F\\I\\N} & \bitbox{16}{Window} \\
   \bitbox{16}{Checksum} &  \bitbox{16}{Urgent Pointer} \\
   \bitbox[rlt]{32}{TCP options}\\
   \bitbox[rlb]{24}{} & \bitbox{8}{Padding}\\
   \end{bytefield}
   };      




The protocol details
====================
   


.. tikz:: The MP_JOIN option in a SYN packet

   \node (A) at (0,0)  {	  
   \begin{bytefield}{32}
   \bitheader{0-31}\\
   \bitbox{8}{Kind} & \bitbox{8}{Length=12} & \bitbox{4}{0x1} & \bitbox{3}{rsv} & \bitbox{1}{B} & \bitbox{8}{Address ID} \\
   \bitbox{32}{Receiver's token} \\
   \bitbox{32}{Sender's random nonce} \\
   \end{bytefield}
   };


.. tikz:: The MP_JOIN option in a SYN+ACK packet

   \node (A) at (0,0)  {	  
   \begin{bytefield}{32}
   \bitheader{0-31}\\
   \bitbox{8}{Kind} & \bitbox{8}{Length=12} & \bitbox{4}{0x1} & \bitbox{3}{rsv} & \bitbox{1}{B} & \bitbox{8}{Address ID} \\
   \wordbox{2}{Sender's truncated HMAC\\64 bits} \\
   \bitbox{32}{Sender's random nonce} \\
   \end{bytefield}
   };      

.. tikz:: The MP_JOIN option in the initiator's first ACK

   \node (A) at (0,0)  {	  
   \begin{bytefield}{32}
   \bitheader{0-31}\\
   \bitbox{8}{Kind} & \bitbox{8}{Length=12} & \bitbox{4}{0x1} & \bitbox{3}{rsv} & \bitbox{1}{B} & \bitbox{8}{Address ID} \\
   \wordbox{5}{Sender's truncated HMAC\\160 bits} \\
   \end{bytefield}
   };

   
.. tikz:: The Data Sequence Signal option 

   \node (A) at (0,0)  {	  
   \begin{bytefield}{32}
   \bitheader{0-31}\\
   \bitbox{8}{Kind} & \bitbox{8}{Length=12} & \bitbox{4}{0x2} & \bitbox{7}{reserved} & \bitbox{1}{\tiny{F}} & \bitbox{1}{\tiny{m}} & \bitbox{1}{\tiny{M}} & \bitbox{1}{\tiny{a}} & \bitbox{1}{\tiny{A}} \\
   \bitbox{32}{Data ACK (4 or 8 bytes)}\\
   \bitbox{32}{Data Sequence Number (4 or 8 bytes)}\\
   \bitbox{32}{Subflow Sequence Number (4 or 8 bytes)}\\
   \bitbox{16}{Data-Level Length} & \bitbox{16}{Checksum} \\
   \end{bytefield}
   };


.. tikz:: The MP_PRIO option 

   \node (A) at (0,0)  {
   \begin{bytefield}{32}
   \bitbox{8}{Kind} & \bitbox{8}{Length} & \bitbox{4}{0x5} & \bitbox{3}{rsv} & \bitbox{1}{B} \\
   \end{bytefield}
   };      


.. tikz:: The ADD_ADDR option 

   \node (A) at (0,0)  {	  
   \begin{bytefield}{32}
   \bitheader{0-31}\\
   \bitbox{8}{Kind} & \bitbox{8}{Length} & \bitbox{4}{0x3} & \bitbox{3}{rsv} & \bitbox{1}{E} & \bitbox{8}{Address ID} \\
   \bitbox{32}{IP address (16 bytes for IPv6)}\\
   \bitbox{16}{Port (2 bytes, opt.)} & \bitbox[lrt]{16}{} \\
   \bitbox[lr]{32}{Truncated HMAC (64 bits if E=0)}\\
   \bitbox[lbr]{16}{} && \bitbox[lt]{16}{}\\
   \end{bytefield}
   };      

   
.. tikz:: The REMOVE_ADDR option 

   \node (A) at (0,0)  {	  
   \begin{bytefield}{32}
   \bitbox{8}{Kind} & \bitbox{8}{Length=3+n} & \bitbox{4}{0x4} & \bitbox{4}{rsv} & \bitbox{8}{Address ID}\bitbox[]{2}{...} \\
   \end{bytefield}
   };         
   

.. tikz:: The MP_CAPABLE option in a SYN packet

   \node (A) at (0,0)  {
   \begin{bytefield}{32}
   \bitheader{0-31}\\
   \bitbox{8}{Kind}  
   \bitbox{8}{Length} & \bitbox{4}{0x0} & \bitbox{4}{Ver} & \bitbox{1}{\tiny{A}} &  \bitbox{1}{\tiny{B}}  &  \bitbox{1}{\tiny{C}}  &  \bitbox{1}{\tiny{D}}  &  \bitbox{1}{\tiny{E}}  &  \bitbox{1}{\tiny{F}}  &  \bitbox{1}{\tiny{G}}  &  \bitbox{1}{H} \\
   \end{bytefield}
   };


.. tikz:: The MP_CAPABLE option in SYN+ACK packet

   \node (A) at (0,0)  {	  
   \begin{bytefield}{32}
   \bitheader{0-31}\\
   \bitbox{8}{Kind}  
   \bitbox{8}{Length} & \bitbox{4}{0x0} & \bitbox{4}{Ver} & \bitbox{1}{\tiny{A}} &  \bitbox{1}{\tiny{B}}  &  \bitbox{1}{\tiny{C}}  &  \bitbox{1}{\tiny{D}}  &  \bitbox{1}{\tiny{E}}  &  \bitbox{1}{\tiny{F}}  &  \bitbox{1}{\tiny{G}}  &  \bitbox{1}{H} \\
   \wordbox{2}{Sender's key (key) (64 bits)} \\
   \end{bytefield}
   };

.. tikz:: The MP_CAPABLE option in initiator's first ACK (without data)

   \node (A) at (0,0)  {	  
   \begin{bytefield}{32}
   \bitheader{0-31}\\
   \bitbox{8}{Kind}  
   \bitbox{8}{Length} & \bitbox{4}{0x0} & \bitbox{4}{Ver} & \bitbox{1}{\tiny{A}} &  \bitbox{1}{\tiny{B}}  &  \bitbox{1}{\tiny{C}}  &  \bitbox{1}{\tiny{D}}  &  \bitbox{1}{\tiny{E}}  &  \bitbox{1}{\tiny{F}}  &  \bitbox{1}{\tiny{G}}  &  \bitbox{1}{H} \\
   \wordbox{2}{Sender's key (key) (64 bits)} \\
   \wordbox{2}{Receiver's key (key) (64 bits)} \\
   \end{bytefield}
   };   
   

.. tikz:: The MP_CAPABLE option in initiator's first ACK (with data)

   \node (A) at (0,0)  {	  
   \begin{bytefield}{32}
   \bitheader{0-31}\\
   \bitbox{8}{Kind}  
   \bitbox{8}{Length} & \bitbox{4}{0x0} & \bitbox{4}{Ver} & \bitbox{1}{\tiny{A}} &  \bitbox{1}{\tiny{B}}  &  \bitbox{1}{\tiny{C}}  &  \bitbox{1}{\tiny{D}}  &  \bitbox{1}{\tiny{E}}  &  \bitbox{1}{\tiny{F}}  &  \bitbox{1}{\tiny{G}}  &  \bitbox{1}{H} \\
   \wordbox{2}{Sender's key (key) (64 bits)} \\
   \wordbox{2}{Receiver's key (key) (64 bits)} \\
   \bitbox{16}{Data-level length} & \bitbox{16}{Checksum (optional)}
   \end{bytefield}
   };   
   




.. tikz:: The MP_TCPRST option 

   \node (A) at (0,0)  {
   \begin{bytefield}{32}
   \bitheader{0-31}\\
   \bitbox{8}{Kind} & \bitbox{8}{Length} & \bitbox{4}{0x8} & \bitbox{1}{\tiny{U}} & \bitbox{1}{\tiny{V}} \bitbox{1}{\tiny{W}} \bitbox{1}{\tiny{T}} & \bitbox{8}{Reason} \\
   \wordbox{2}{Receiver's key \\ 64 bits} \\
   \end{bytefield}
   };   
   
   
.. tikz:: The FAST_CLOSE option 

   \node (A) at (0,0)  {	  
   \begin{bytefield}{32}
   \bitheader{0-31}\\
   \bitbox{8}{Kind} & \bitbox{8}{Length} & \bitbox{4}{0x7} & \bitbox{12}{reserved} \\
   \wordbox{2}{Receiver's key \\ 64 bits} \\
   \end{bytefield}
   }; 

