.. _chapter-tcp:

The Transmission Control Protocol (TCP)
***************************************


TCP is a connection-oriented transport protocol. This means that a TCP connection must be established before communicating hosts can exchange data. A connection is a logical relation between the two communication hosts. Each hosts maintains some state about the connection and uses it to manage the connection.


TCP uses the three-way handshake as shown in :numref:`fig-tcp-handshake`. To initiate a connection, the client sends a TCP segment with the ``SYN`` flag set. Such a segment is usually called a ``SYN`` segment. It contains a random sequence number (`x` in :numref:`fig-tcp-handshake`). If the server accepts the connection, it replies with a ``SYN+ACK`` segment whose ``SYN`` and ``ACK`` flags are set. The acknowledgment number of this segment is set to ``x+1`` to confirm the reception of the ``SYN`` segment sent by the client. The server selects a random sequence number (`y` in :numref:`fig-tcp-handshake`). Finally, the client replies with an `ACK` segment that acknowledges the reception of the ``SYN+ACK`` segment. 

.. _fig-tcp-handshake:
.. tikz:: Establishing a TCP connection using the three-way handshake
   :libs: positioning, matrix, arrows, math

   \tikzmath{\c1=1;\c2=1.5; \s1=8; \s2=8.5; \max=5; }
   
   \tikzstyle{arrow} = [thick,->,>=stealth]
   \tikzset{state/.style={rectangle, dashed, draw, fill=white} }
   \node [black, fill=white] at (\c1,\max) {Client};
   \node [black, fill=white] at (\s1,\max) {Server};
   
   \draw[blue,very thick,->] (\c1,\max-0.5) -- (\c1,0.5);
   \draw[blue,very thick,->] (\s1,\max-0.5) -- (\s1,0.5);
   
   \tikzmath{\y=\max-1;}
   
   \draw[blue,thick, ->] (\c1,\y) -- (\s1,\y-1) node [midway, fill=white] {SYN\small{[seq=x]}};
   \draw[blue,thick, ->] (\s1,\y-1) -- (\c1,\y-2) node [midway, fill=white] {SYN+ACK\small{[seq=y,ack=x+1]}};
   \draw[blue,thick, ->] (\c1,\y-2) -- (\s1,\y-3) node [midway, fill=white] {ACK\small{[seq=x+1,ack=y+1]}};


TCP was designed to be extensible. The TCP header contains a TCP Header Length (THL) field that indicates the total length of the TCP header in four-bytes words. For the normal header, this field is set to 5, which corresponds to the 20 bytes long TCP header. Larger values of the THL field indicate that the segment contains one or more TCP options. TCP options are encoded as a Type-Length-Value field. The first byte specifies the Type, the second byte indicates the length of the entire TCP option in bytes. The utilization of TCP options is usually negotiated during the three-way-exchange. The client adds a TCP option in the ``SYN`` segment. If the server does not recognize the option, it simply ignores it. If the server wants to utilize the extension for the connection, it simply adds the corresponding option in the ``SYN+ACK`` segment. This is illustrated in :numref:`fig-tcp-handshake-sack` with the Selective Acknowledgments extension :cite:p:`rfc2018` as an example.

.. _fig-tcp-handshake-sack:
.. tikz:: Negotiating the utilization of Selective Acknowledgments during the three-way handshake
   :libs: positioning, matrix, arrows, math

   \tikzmath{\c1=1;\c2=1.5; \s1=8; \s2=8.5; \max=4.5; }
   
   \tikzstyle{arrow} = [thick,->,>=stealth]
   \tikzset{state/.style={rectangle, dashed, draw, fill=white} }
   \node [black, fill=white] at (\c1,\max) {Client};
   \node [black, fill=white] at (\s1,\max) {Server};
   
   \draw[blue,very thick,->] (\c1,\max-0.5) -- (\c1,0.5);
   \draw[blue,very thick,->] (\s1,\max-0.5) -- (\s1,0.5);
   
   \tikzmath{\y=\max-1;}
   
   \draw[blue,thick, ->] (\c1,\y) -- (\s1,\y-1) node [midway, align=left, fill=white] {SYN\small{[seq=x]}\\\small{SACK-Permitted}};
   \draw[blue,thick, ->] (\s1,\y-1) -- (\c1,\y-2) node [midway, align=left, fill=white] {SYN+ACK\small{[seq=y,ack=x+1]}\\\small{SACK-Permitted}};
   \draw[blue,thick, ->] (\c1,\y-2.1) -- (\s1,\y-3) node [midway, align=left, fill=white] {ACK\small{[seq=x+1,ack=y+1]}};

A TCP connection is identified by using four fields that are included inside each TCP packet:

 - the client IP address
 - the server IP address
 - the client-selected port
 - the server port

All TCP packets that belong to a connection contain these four fields in the IP and TCP header. When a host receives a packet, it uses them to match the connection to which it belongs. A TCP implementation maintains some state for each established TCP connection. This state is a data structure that contains fields which can vary from one implementation to another. The TCP specification defines some state variables that any implementation should remember. On the sender side, these include:
 - ``snd.una``, the oldest unacknowledged sequence number
 - ``snd.nxt``, the next sequence number of be sent
 - ``rcv.win``, the latest window advertised by the remote host

A TCP sender also stores the data that has been sent but has not yet been acknowledged. It also measures the round-trip-time and its variability to set the retransmission timer and maintains several variables that are related to the congestion control scheme.

A TCP receiver also maintains state variables. These include ``rcv.next``, the next expected sequence number. Data received in sequence can be delivered to the application while out-of-sequence data must be queued.

Finally, TCP implementations store the state of the connection according to the TCP state machine :cite:p:`rfc793`.

TCP implementations include lots of optimizations that are outside the scope of this brief introduction. Let us know briefly describe how TCP sends data reliably. Consider a TCP connection established between a client and a server. :numref:`fig-tcp-simple-data` shows a simple data transfer between these two hosts. The sequence number of the first segment starts at ``1234``, the current value of ``snd.nxt``. For TCP, each transmitted byte consumes one sequence number. Thus, after having sent the first segment, the client's ``snd.nxt`` is set to ``1238``.  The server receives the data in sequence and immediately acknowledges it. A TCP receiver always sets the acknowledgment number of the segments that it sends with the next expected sequence number, i.e. ``rcv.nxt``. 


.. _fig-tcp-simple-data:
.. tikz:: TCP Reliable data transfer
   :libs: positioning, matrix, arrows, math

   \tikzmath{\c1=1;\c2=1.5; \s1=8; \s2=8.5; \max=4; }
   
   \tikzstyle{arrow} = [thick,->,>=stealth]
   \tikzset{state/.style={rectangle, dashed, draw, fill=white} }
   %\node [black, fill=white] at (\c1,\max) {Client};
   %\node [black, fill=white] at (\s1,\max) {Server};
   
   \draw[blue,very thick,->] (\c1,\max-0.5) -- (\c1,0.5);
   \draw[blue,very thick,->] (\s1,\max-0.5) -- (\s1,0.5);
   
   \tikzmath{\y=\max-1;}
   
   \draw[blue,thick, ->] (\c1,\y) -- (\s1,\y-1) node [midway, align=left, fill=white] {\small{[seq=1234,data="abcd"]}};
   \draw[blue,thick, ->] (\s1,\y-1) -- (\c1,\y-2) node [midway, align=left, fill=white] {ACK\small{[ack=1237]}};
   \draw[blue,thick, ->] (\c1,\y-1) -- (\s1,\y-2) node [midway, align=left, fill=white] {\small{[seq=1238,data="efgh"]}};
   \draw[blue,thick, ->] (\s1,\y-2) -- (\c1,\y-3) node [midway, align=left, fill=white] {ACK\small{[ack=1224]}};


In practice, TCP implementations use the Nagle algorithm :cite:p:`rfc896` and thus usually try to send full segments. They use the Maximum Segment Size (MSS) option during the handshake and PathMTU discovery the determine the largest segment which can be safely sent over a connection. Furthermore, TCP implementations usually delay acknowledgments and only acknowledge every second segment when these are received in sequence. This is illustrated in :numref:`fig-tcp-data-delack`.


.. _fig-tcp-data-delack:
.. tikz:: TCP Reliable data transfer with delayed acknowledgments.
   :libs: positioning, matrix, arrows, math

   \tikzmath{\c1=1;\c2=1.5; \s1=8; \s2=8.5; \max=5.0; }
   
   \tikzstyle{arrow} = [thick,->,>=stealth]
   \tikzset{state/.style={rectangle, dashed, draw, fill=white} }
   %\node [black, fill=white] at (\c1,\max) {Client};
   %\node [black, fill=white] at (\s1,\max) {Server};
   
   \draw[blue,very thick,->] (\c1,\max-0.5) -- (\c1,0.5);
   \draw[blue,very thick,->] (\s1,\max-0.5) -- (\s1,0.5);
   
   \tikzmath{\y=\max-1;}
   
   \draw[blue,thick, ->] (\c1,\y) -- (\s1,\y-1) node [midway, align=left, fill=white] {\small{[seq=1000,len=1460,data="x...x"]}};
   \draw[blue,thick, ->] (\c1,\y-0.5) -- (\s1,\y-1.5) node [midway, align=left, fill=white] {\small{[seq=2460,len=1460,data="x...x"]}};
   \draw[blue,thick, ->] (\s1,\y-1.6) -- (\c1,\y-2.6) node [midway, align=left, fill=white] {ACK\small{[ack=3920]}};


TCP uses a single segment type and each segment contains both a sequence number and an acknowledgment number. The sequence number is mainly useful when a segment contains data. A receiver only processes the acknowledgment number if the ``ACK`` flag is set. In practice, TCP uses cumulative acknowledgments and all the segments sent on a TCP connection have their ``ACK`` flag set. The only exception is the ``SYN`` segment sent by the client to initiate a connection.


.. _fig-tcp-piggyback:
.. tikz:: TCP piggybacking.
   :libs: positioning, matrix, arrows, math

   \tikzmath{\c1=1;\c2=1.5; \s1=8; \s2=8.5; \max=5.0; }
   
   \tikzstyle{arrow} = [thick,->,>=stealth]
   \tikzset{state/.style={rectangle, dashed, draw, fill=white} }

   
   \draw[blue,very thick,->] (\c1,\max-0.5) -- (\c1,0.5);
   \draw[blue,very thick,->] (\s1,\max-0.5) -- (\s1,0.5);
   
   \tikzmath{\y=\max-1;}
   
   \draw[blue,thick, ->] (\c1,\y) -- (\s1,\y-1) node [midway, fill=white] {ACK\small{[seq=1234,ack=5678,len=4,data="abcd"]}};
   \draw[blue,thick, ->] (\s1,\y-1) -- (\c1,\y-2) node [midway, fill=white] {ACK\small{[seq=5678,ack=1238,len=2,data="ef"]}};
   \draw[blue,thick, ->] (\c1,\y-2) -- (\s1,\y-3) node [midway, fill=white] {ACK\small{[seq=1238,ack=5680,len=4,data="ghij"]}};
   
   
TCP uses different techniques to retransmit corrupted or lost data. The TCP header contains a 16 bits checksum that is computed over the entire TCP segment and a part of the IP header. The value of this checksum is computed by the sender and checked by the receiver to detect transmission errors. TCP copes with these errors by retransmitting data. The simplest technique is to rely on a retransmission timer. TCP continuously measure the round-trip-time, i.e. the delay between the transmission of a segment and the reception of the corresponding acknowledgment. It then sets a per-connection retransmission timer based on its estimations of the mean rtt and its variance :cite:p:`rfc6298`. This is illustrated in :numref:`fig-tcp-retrans` where the arrow terminated with red cross corresponds to a lost segment. Upon expiration of the retransmission timer, the client retransmits the unacknowledged segment. 

.. _fig-tcp-retrans:
.. tikz:: TCP protects data by a retransmission timer
   :libs: positioning, matrix, arrows, math, arrows.meta

   \tikzmath{\c1=1;\c2=1.5; \s1=8; \s2=8.5; \max=7; }
   
   \tikzstyle{arrow} = [thick,->,>=stealth]
   \tikzset{state/.style={rectangle, dashed, draw, fill=white} }
   %\node [black, fill=white] at (\c1,\max) {Client};
   %\node [black, fill=white] at (\s1,\max) {Server};
   
   \draw[blue,very thick,->] (\c1,\max-0.5) -- (\c1,0.5);
   \draw[blue,very thick,->] (\s1,\max-0.5) -- (\s1,0.5);
   
   \tikzmath{\y=\max-1;}
   
   \draw[blue,thick,-{Rays[color=red]}] (\c1,\y) -- (\s1,\y-1) node [midway, fill=white] {ACK\small{[seq=1234,ack=5678,len=4,data="abcd"]}};
   \draw[black,thick,<->]  (\c1-0.5,\y) -- (\c1-0.5,\y-3) node [midway, fill=white] {retransmission timer};
   \draw[blue,thick, ->] (\c1,\y-3) -- (\s1,\y-4) node [midway, fill=white]  {ACK\small{[seq=1234,ack=5678,len=4,data="abcd"]}};
   \draw[blue,thick, ->] (\s1,\y-4.1) -- (\c1,\y-5) node [midway, fill=white] {ACK\small{[seq=5678,ack=1238]}};

For performance reasons, TCP implementations try to avoid relying on the retransmission timer to retransmit the lost segments. Modern TCP implementations use selective acknowledgments which can be negotiated during the handshake. This is illustrated in :numref:`fig-tcp-retrans-sack`. A selective acknowledgment reports blocks of sequence number that have been received correctly by the receiver. Upon reception of the ``SACK`` option, the sender knows that sequence numbers ``1234-1237`` have not been received while sequence numbers ``1238-1250`` have been correctly received.

.. _fig-tcp-retrans-sack:
.. tikz:: TCP leverages selective acknowledgments to retransmit lost data
   :libs: positioning, matrix, arrows, math, arrows.meta

   \tikzmath{\c1=1;\c2=1.5; \s1=8; \s2=8.5; \max=8; }
   
   \tikzstyle{arrow} = [thick,->,>=stealth]
   \tikzset{state/.style={rectangle, dashed, draw, fill=white} }

   
   \draw[blue,very thick,->] (\c1,\max-0.5) -- (\c1,0.5);
   \draw[blue,very thick,->] (\s1,\max-0.5) -- (\s1,0.5);
   
   \tikzmath{\y=\max-1;}
   
   \draw[blue,thick,-{Rays[color=red]}] (\c1,\y) -- (\s1,\y-1) node [midway, fill=white] {\small{[seq=1234,ack=5678,data="abcd"]}};
   \draw[blue,thick, ->] (\c1,\y-1) -- (\s1,\y-2) node [midway, fill=white]  {\small{[seq=1234,data="efgh"]}};
   \draw[blue,thick, ->] (\c1,\y-2) -- (\s1,\y-3) node [midway, fill=white]  {\small{[seq=1238,data="ijkl"]}};
    \draw[blue,thick, ->] (\c1,\y-2) -- (\s1,\y-3) node [midway, fill=white]  {\small{[seq=1242,data="mnop"]}};  
   \draw[blue,thick, ->] (\c1,\y-3) -- (\s1,\y-4) node [midway, fill=white]  {\small{[seq=1246,data="qrst"]}};
   \draw[blue,thick, ->] (\s1,\y-4.1) -- (\c1,\y-5) node [midway, fill=white] {ACK\small{[ack=1234]}SACK[1238:1250]};
   \draw[blue,thick, ->] (\c1,\y-5.1) -- (\s1,\y-6) node [midway, fill=white]  {\small{[seq=1234,ack=5678,data="abcd"]}};

When the client and the sender have exchanged all the required data, they can terminate the connection. TCP supports two different methods to terminate a connection. The reliable manner is that each host closes its direction of data transfer by sending a segment with the ``FIN`` flag set. The sequence number of this segment marks the end of the data transfer and the recipient of the segment acknowledges it once it has delivered all the data up to the sequence number of the ``FIN`` segment to its application. The release of a TCP connection is illustrated in :numref:`fig-tcp-fin`. To reduce the size of the figure, we have set the ``FIN`` flag in segments that contains data. The server considers the connection to be closed upon reception of the ``FIN+ACK`` segment. It discards the state that it maintained for this now closed TCP connection. The client also considers the connection to be closed when it sends the ``FIN+ACK`` segment since all data has been acknowledged. However, it does not immediately discard the state for this connection because it needs to be able to retransmit the ``FIN+ACK`` segment in case it did not reach the server.

.. _fig-tcp-fin:
.. tikz:: Closing a TCP connection using the ``FIN`` flag
   :libs: positioning, matrix, arrows, math, arrows.meta

   \tikzmath{\c1=1;\c2=1.5; \s1=8; \s2=8.5; \max=6; }
   
   \tikzstyle{arrow} = [thick,->,>=stealth]
   \tikzset{state/.style={rectangle, dashed, draw, fill=white} }

   
   \draw[blue,very thick,->] (\c1,\max-0.5) -- (\c1,0.5);
   \draw[blue,very thick,->] (\s1,\max-0.5) -- (\s1,0.5);
   
   \tikzmath{\y=\max-1;}
   
   \draw[blue,thick,->] (\c1,\y) -- (\s1,\y-1) node [midway, fill=white] {FIN\small{[seq=1234,data="abcd"]}};
   \draw[blue,thick, ->] (\s1,\y-1) -- (\c1,\y-2) node [midway, fill=white]  {ACK \small{[ack=1239]}};

   \draw[blue,thick, ->] (\s1,\y-3) -- (\c1,\y-4) node [midway, fill=white]  {FIN\small{[seq=5678,date="xyz"]}};
   \draw[blue,thick,->] (\c1,\y-4) -- (\s1,\y-5) node [midway, fill=white] {FIN+ACK\small{[seq=1239,ack=5681]}};


   
.. _fig-tcp-rst:
.. tikz:: Closing a TCP connection using a ``RST`` segment
   :libs: positioning, matrix, arrows, math, arrows.meta

   \tikzmath{\c1=1;\c2=1.5; \s1=8; \s2=8.5; \max=4; }
   
   \tikzstyle{arrow} = [thick,->,>=stealth]
   \tikzset{state/.style={rectangle, dashed, draw, fill=white} }

   
   \draw[blue,very thick,->] (\c1,\max-0.5) -- (\c1,0.5);
   \draw[blue,very thick,->] (\s1,\max-0.5) -- (\s1,0.5);
   
   \tikzmath{\y=\max-1;}
   
   \draw[blue,thick,->] (\c1,\y) -- (\s1,\y-1) node [midway, fill=white] {\small{[seq=1234,data="abcd"]}};
   \draw[blue,thick, ->] (\s1,\y-1) -- (\c1,\y-2) node [midway, fill=white]  {RST\small{[ack=1239]}};

