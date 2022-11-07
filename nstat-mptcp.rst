The Multipath TCP counters
--------------------------

Linux version 5.18 maintains 46 counters for Multipath TCP. These counters
correspond to different parts of the protocol and can be organized in four
groups. The first group gathers the counters that are incremented when TCP
packets containing the ``MP_CAPABLE`` option are processed. The second group 
gathers the counters that are incremented when processing packets with
the ``MP_JOIN`` option. The third group gathers the counters that are
modified when packets with the ``ADD_ADDR``, ``RM_ADDR`` or ``MP_PRIO`` option
are processed. The fourth group gathers the remaining counters of the
Multipath TCP stack.

Two versions of Multipath TCP have been specified within the IETF. Version
0 was initially defined in :rfc:`6824`. The off-tree but well
maintained set of patches distributed by
`https://www.multipath-tcp.org <https://www.multipath-tcp.org>`_ implemented
this version of Multipath TCP. Based on the experience gathered with this
implementation and also Apple's implementation, Multipath TCP evolved and
the IETF published version 1 in :rfc:`8684`. The Multipath TCP counters
correspond to this version of Multipath TCP.



The MPCapable counters
++++++++++++++++++++++

This group gathers the following counters: ``MPTcpExtMPCapableSYNRX``, 
``MPTcpExtMPCapableSYNTX``, ``MPTcpExtMPCapableSYNACKRX``,
``MPTcpExtMPCapableACKRX``, ``MPTcpExtMPCapableFallbackACK``,
``MPTcpExtMPCapableFallbackSYNACK`` and ``MPTcpExtMPFallbackTokenInit``. They
relate to the establishment of the initial Multipath TCP subflow
which is described in the :ref:`mptcp-initial-mptcp-handshake` section. 


The ``MPTcpExtMPCapableSYNTX`` counter is similar to the ``TcpActiveOpens``
counter maintained by TCP. It counts the number of Multipath TCP connections
that this host has tried to establish. Its value will usually be much smaller than ``TcpActiveOpens``. When a Multipath connection is initiated using the
``connect`` system call, both ``MPTcpExtMPCapableSYNTX`` and
``TcpActiveOpens`` are incremented. Although the name of the counter is
``MPTcpExtMPCapableSYNTX``, it is only incremented once per Multipath
TCP connection if the ``SYN`` packet needs to be retransmitted.

The ``MPTcpExtMPCapableSYNACKRX`` counter is incremented every time
a Multipath TCP connection is confirmed by the reception of a
``SYN+ACK`` with the ``MP_CAPABLE`` option to a ``SYN`` packet
that it sent earlier. The value of this counter should be lower than
``MPTcpExtMPCapableSYNTX`` since only a subset of the connections initiated
by a host will typically reach a Multipath TCP compliant server.
If a client receives a ``SYN+ACK`` without the ``MP_CAPABLE`` option
in response to a ``SYN`` sent with the ``MP_CAPABLE`` option, then
the ``MPTcpExtMPCapableFallbackSYNACK`` counter is incremented. This
counter tracks the Multipath TCP connections that were forced to fall back
to regular TCP during the three-way handshake of the initial subflow.



On the other hand, the ``MPTcpExtMPCapableSYNRX`` counter tracks the
number of Multipath TCP connections that were accepted by the host.
Its value will usually be much smaller than ``TcpPassiveOpens`` which
tracks all accepted TCP connections. When a Multipath connection is accepted,
both ``MPTcpExtMPCapableSYNRX`` and ``TcpPassiveOpens`` are incremented.
As for ``MPTcpExtMPCapableSYNTX``, the ``MPTcpExtMPCapableSYNRX`` counter is
only incremented once per connection and not each time a packet is received.
Upon reception of a ``SYN`` with the ``MP_CAPABLE`` option, a
Multipath TCP server returns a ``SYN+ACK`` with the ``MP_CAPABLE``
option. The ``MPTcpExtMPCapableACKRX`` counter is incremented upon reception
of the third ``ACK`` containing the ``MP_CAPABLE`` option. If this option
is not present in this ``ACK``, then the ``MPTcpExtMPCapableFallbackACK``
gets incremented.
If this counter increases, it probably indicates some interference with a
middlebox that injects acknowledgments during the three-way handshake.

.. todo:: find example ?


.. todo:: MPTcpExtMPFallbackTokenInit  seems related to the failure of getting a token during the handshake (e.g. because there are too many tokens already or not enough randomness), but there are cases in subflow joins where this counter is also incremented
   
.. code-block:: console
   :caption: The MPCapable counters (active opens)

                            ||
			    ||
    MPTcpExtMPCapableSYNTX+++++++++++++++
                            ||
			    ||
			    |+-----------> MPTcpExtMPCapableSYNACKRX 
			    ||
			    ||
			    |+-----------> MPTcpExtMPCapableFallbackSYNACK 
                            ||
                            ||


.. code-block:: console
   :caption: The MPCapable counters (passive opens)

                            ||
			    ||
    MPTcpExtMPCapableSYNRX+++++++++++++++
                            ||
			    ||
			    |+-----------> MPTcpExtMPCapableACKRX 
			    ||
			    ||
			    |+-----------> MPTcpExtMPCapableFallbackACK 
                            ||
                            ||

			    
			 
The Join counters
+++++++++++++++++

There are thirteen counters in this group. They are incremented when a host
processes ``SYN`` packets corresponding to additional subflows.


The first counter, ``MPTcpExtMPJoinSynRx`` is incremented every time a
``SYN`` packet with the ``MP_JOIN`` option is received. Upon reception
of a such packet, the host first verifies that it knows the token of
the Multipath TCP connection. If so, the processing continues and
the host returns a ``SYN+ACK`` packet with the ``MP_JOIN`` option, its
random number and a HMAC. Otherwise, the ``MPTcpExtMPJoinNoTokenFound``
counter is incremented. The host then waits for the third ``ACK``
which contains the ``MP_JOIN`` option and the HMAC computed by
the remote host. It then checks the validity of the received HMAC. If
the HMAC is invalid, then the ``MPTcpExtMPJoinAckHMacFailure`` counter
is incremented.

The ``MPTcpExtMPJoinSynRx`` counter will increase on Multipath TCP hosts
that accept subflows, typically servers. The value of the
``MPTcpExtMPJoinACKRX`` counter should be close to the previous one.
If the two other counters, ``MPTcpExtMPJoinNoTokenFound`` or
``MPTcpExtMPJoinAckHMacFailure`` increase, then the system administrator
should probably investigate as these are indication of possible attacks.


.. code-block:: console
   :caption: The Join counters when accepting subflows
	     
                            ||
			    ||
    MPTcpExtMPJoinSynRX +++++++++++++++
                            ||
			    ||
			    |+-----------> MPTcpExtMPJoinNoTokenFound 
			    ||
			    ||
    MPTcpExtMPJoinACKRX +++++++++++++++
    			    ||
			    |+-----------> MPTcpExtMPJoinAckHMacFailure 
                            ||
                            ||


Unfortunately, there is no counter that tracks the creation of new subflows
by a host. The TCP stack counts these new subflows as active opens, but
there is no specific Multipath TCP counter. However, the
``MPTcpExtMPJoinSynAckRX`` counter tracks the reception of ``SYN+ACK``
packets containing the ``MP_JOIN`` option. This is thus an indirect
way to track the creation of new subflows. Upon reception of such a
packet, in response to a previously sent ``SYN`` packet with the ``MP_JOIN``
option, a host checks the validity of the received HMAC. If the HMAC is
invalid, the ``MPTcpExtMPJoinSynAckHMacFailure`` is incremented. This counter
should rarely increase. If it increases, then the problem should be
investigated by collecting packet traces. 

			    
.. code-block:: console
   :caption: The Join counters when initiating subflows
	     
                            ||
			    ||
    MPTcpExtMPJoinSynAckRX +++++++++++++++
                            ||
			    ||
			    |+-----------> MPTcpExtMPJoinSynAckHMacFailure
			    ||
			    ||


			    

.. MPTcpExtMPJoinNoTokenFound      0                  0.0
.. MPTcpExtMPJoinSynRx             0                  0.0
.. MPTcpExtMPJoinSynAckRx          0                  0.0
.. MPTcpExtMPJoinSynAckHMacFailure 0                  0.0
.. MPTcpExtMPJoinAckRx             0                  0.0
.. MPTcpExtMPJoinAckHMacFailure    0


.. todo:: This part is unclear and needs to be checked based on the tests

A Multipath TCP host will usually accept additional subflows on the address
and ports where the initial subflow was accepted. The following counters
track the arrival of packets destined to different port numbers:

 - ``MPTcpExtMPJoinPortSynRx``
 - ``MPTcpExtMPJoinPortSynAckRx``
   ``MPTcpExtMPJoinPortAckRx``



The last two counters, ``MPTcpExtMismatchPortSynRx`` and
``MPTcpExtMismatchPortAckRx`` are a bit different. They are incremented when
a ``SYN`` or ``ACK`` sent to a different port number are received.


The ``MP_JOIN`` option contains a ``B`` that indicates whether the new
subflow should be considered as a backup subflow or a regular one. This
information is used by the path manager, but no counter tracks the value of
the backup bit in the ``MP_JOIN`` option. Once a subflow has been established,
its backup status can be changed using the ``MP_PRIO`` option. The
``MPTcpExtMPPrioTx`` counter is incremented every time such an option is sent.
The ``MPTcpExtMPPrioRx`` counter is incremented by each received ``MP_PRIO``
option. 


   
.. MPTcpExtMPPrioTx                0                  0.0
.. MPTcpExtMPPrioRx       


The address advertisement counters
++++++++++++++++++++++++++++++++++

There are six counters in this group. The advertisement of addresses by
Multipath TCP is described in ref:`Address management <mmtpbook:mptcp-addr-management>`.

When a host receives a packet with a valid ``ADD_ADDR`` option with its
``Echo`` bit set to zero, the ``MPTcpExtAddAddr`` counter is incremented.
If this option includes an optional port number, the ``MPTcpExtPortAdd``
counter is also incremented. In addition to these two counters, the
``MPTcpExtAddAddrDrop`` tracks the address advertisements that were received
by the host, but not processed by the path manager, e.g. because no user
space path manager was active.

Multipath TCP does not track the advertisements of addresses by sending
the ``ADD_ADDR`` option. However, it tracks the reception of packets
containing the ``ADD_ADDR`` option with the ``Echo`` bit set to one with
the ``MPTcpExtEchoAdd`` counter. These packets are echoed by the remote host. 

Similarly, the ``MPTcpExtRmAddr`` counter tracks the number of received
``RM_ADDR`` options. These options typically indicate a change in the
addresses owned by a remote peer. Mobile hosts are likely to send these
options when they move from one type of network to another. The
``MPTcpExtRmAddrDrop`` is incremented when the path manager cannot process an
incoming ``RM_ADDR`` option.

.. todo:: is this a rare event ?


.. MPTcpExtAddAddr                 0                  0.0
.. MPTcpExtEchoAdd                 0                  0.0
.. MPTcpExtPortAdd                 0                  0.0
.. MPTcpExtAddAddrDrop    
.. MPTcpExtRmAddr                  0                  0.0
.. MPTcpExtRmAddrDrop              0                  0.0

When a host receives a ``RM_ADDR`` option from a remote peer, its path
manager should remove the subflows associated with this address. The
``MPTcpExtRmSubflow`` counter tracks the number of subflows that have
been destroyed by a path manager.
   

.. MPTcpExtRmSubflow               0                  0.0



The connection termination counters
+++++++++++++++++++++++++++++++++++

There are seven counters in this group. They track the abnormal termination of
a Multipath TCP connection. A normal Multipath TCP connection should end
with the exchange of ``DATA_FIN`` in both directions. However, are scenarios
are possible. First, one of the hosts may wish to quickly terminate the
Multipath TCP connection without having to maintain state. Multipath TCP
uses the ``FAST_CLOSE`` option
in this case. The ``MPTcpExtMPFastcloseTx`` and ``MPTcpExtMPFastcloseRx``
counters track the transmission and the reception of such options.

.. MPTcpExtMPFastcloseTx           0                  0.0
.. MPTcpExtMPFastcloseRx           0                  0.0

Multipath TCP was designed to prevent as much as possible interference
from middleboxes, but there are some types of interferences that force
Multipath TCP to fallback to regular TCP. In this case, the host that first
noticed the interference (e.g. problem during the handshake, DSS checksum
problem, ...) sends a packet with the ``MP_FAIL`` option. This forces the
Multipath TCP connection to fall back to a regular TCP connection.
The ``MPTcpExtMPFailTx`` and ``MPTcpExtMPFailRx`` counters track the
transmission and the reception of the ``MP_FAIL`` option. During some
types of fall backs, a host may also send an infinite DSS mapping. The
``MPTcpExtInfiniteMapRx`` counter tracks the reception of such infinite
DSS mappings. 

.. MPTcpExtInfiniteMapRx           0                  0.0

An increase of
these counters would indicate some type of middlebox interference which
should be investigated since it could prevent a complete utilization of
Multipath TCP.

Like TCP, Multipath TCP uses TCP ``RST`` to terminate subflows. Multipath
TCP also defines the ``MP_TCPRST`` option which can contain an option reason
code and flags indicating some information about the reason for the
transmission of the ``RST``. The ``MPTcpExtMPRstTx`` and ``MPTcpExtMPRstRx``
counters track the transmission and the reception of such ``RST`` packets.

.. MPTcpExtMPFailTx                0                  0.0
.. MPTcpExtMPFailRx                0                  0.0

.. MPTcpExtMPRstTx                 17                 0.0
.. MPTcpExtMPRstRx      




The other counters
++++++++++++++++++
.. mainly data transmission

The remaining eleven counters are mainly related to processing of data.

If the DSS checksum is enabled, the ``MPTcpExtDataCsumErr`` is incremented
every time a check of the DSS checksum fails. This should be a rare event that
likely indicates the presence of middleboxes. It should be correlated with
the ``MPTcpExtMPFailTx`` and ``MPTcpExtMPFailRx`` counters discussed in the
previous section.

.. MPTcpExtDataCsumErr             0                  0.0

Three counters track the DSS option of the incoming packets : 
``MPTcpExtDSSNotMatching``, ``MPTcpExtDSSNoMatchTCP`` and
``MPTcpExtNoDSSInWindow``. The first counter is
incremented when a mapping is received for data that has already been mapped
and the new mapping is not the same as the existing one. The second counter
is incremented when the TCP sequence numbers found in the mapping do not
match with the current TCP sequence numbers. The third counter is incremented
upon reception of a packet that indicates a DSS option that is outside the
current window. These three counters should rarely increase.


.. MPTcpExtDSSNotMatching          0                  0.0
.. MPTcpExtDSSNoMatchTCP           0                  0.0
.. MPTcpExtNoDSSInWindow           0                  0.0
.. MPTcpExtDuplicateData  

The last counter that tracks data at the Multipath TCP connection
level is ``MPTcpExtDuplicateData``. It counts the number of received
packets whose data has been ignored because it had already been received
earlier. Such duplicated data can occur with Multipath TCP when data
sent over a subflow is retransmitted over another subflow. It would
be interesting to follow the evolution of this counter on a server that
interacts with mobile devices.


.. todo:: check with Matthieu

Multipath TCP tracks losses on the subflows that compose a Multipath
TCP connection. If one subflow accumulates losses, it may be marked
as stale and the packet scheduler will stop using it to transmit data
until the losses have been recovered. The ``MPTcpExtSubflowStale`` counter is
incremented every time a subflow is marked as being stale. The 
``MPTcpExtSubflowRecover`` counter tracks the transitions from stale to
active.


.. MPTcpExtSubflowStale            0                  0.0
.. MPTcpExtSubflowRecover

Multipath TCP uses an out-of-order queue to reorder the data received over
the different subflows. The ``MPTcpExtOFOQueueTail`` and ``MPTcpExtOFOQueue``   counters track the insertion of data at the tail and in the out-of-order
queue. The ``MPTcpExtOFOMerge`` is incremented when data present in the
out-or-order queue can be merged.
   

.. MPTcpExtOFOQueueTail            0                  0.0
.. MPTcpExtOFOQueue                0                  0.0
.. MPTcpExtOFOMerge                0                  0.0



Finally, the ``MPTcpExtRcvPruned`` tracks the number of packets that
were dropped because the memory available for Multipath TCP was full.
If this counter increases, you should probably check the memory configuration
of your host.
   
.. MPTcpExtRcvPruned               0                  0.0



