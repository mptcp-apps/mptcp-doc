Analyzing the output of nstat
-----------------------------


.. spelling:word-list::

   nstat

The Linux TCP/IP stack maintains a lot of counters that track various
events inside the kernel. These counters are very useful for system
administrators who need to manage Linux hosts and debug some specific
networking problems.

Linux supports a few hundred counters associated to the protocols in the
network and transport layers. Other operating systems have defined their
own counters to track similar networking events. Fortunately, the IETF
has standard some counters that are common to different operating
systems and TCP/IP implementations. These counters are exported as
variables which can be queried using a management protocol such as SNMP.
This enables a management server to collect statistics for a series of
hosts to process and analyze them.  Several versions of SNMP exist, but
we will not discuss them in details in this document. Instead, we
focus on the Linux TCP/IP implementation and explain the different counters
that the `nstat <https://www.man7.org/linux/man-pages/man8/nstat.8.html>`_
application exposes to the user.

Linux kernel version 5.18 collects 363 different counters that are divided in 7 categories :

 - 67 counters track the IPv4 implementation
 - 80 counters track the ICMPv4 implementation
 - 32 counters track the IPv6 implementation
 - 46 counters track ICMPv6
 - 135 counters track TCP
 - 35 counters track UDP
 - 46 counters track Multipath TCP

Some of these counters are part of the Management Information Bases (MIB) defined
within the IETF, e.g. :rfc:`1213` for IPv4 and ICMPv4, :rfc:`4293` for IPv6 and ICMPv6,  :rfc:`4022` for TCP, :rfc:`4113` for UDP. As of this writing, there is no official IETF MIB for Multipath TCP.


Using nstat
+++++++++++

In this document, we describe the counters that are exposed by
`nstat <https://www.man7.org/linux/man-pages/man8/nstat.8.html>`_
for the different protocols of the TCP/IP stack. Before discussing
these counters, it is useful to understand how
`nstat <https://www.man7.org/linux/man-pages/man8/nstat.8.html>`_ works.

`nstat <https://www.man7.org/linux/man-pages/man8/nstat.8.html>`_ is a
command line tool that supports a small number of arguments
   
.. code:: console

   nstat --help
   Usage: nstat [OPTION] [ PATTERN [ PATTERN ] ]
	  -h, --help		this message
	  -a, --ignore	ignore history
	  -d, --scan=SECS	sample every statistics every SECS
	  -j, --json		format output in JSON
	  -n, --nooutput	do history only
	  -p, --pretty	pretty print
	  -r, --reset		reset history
	  -s, --noupdate	don't update history
	  -t, --interval=SECS	report average over the last SECS
	  -V, --version	output version information
	  -z, --zeros		show entries with zero activity


By default, `nstat <https://www.man7.org/linux/man-pages/man8/nstat.8.html>`_
displays the counters whose value has changed since the latest invocation
of the tool. This is usually a small subset of the counters
that depends on the networking activity of the host. 

`nstat <https://www.man7.org/linux/man-pages/man8/nstat.8.html>`_ can collect
historical information and provides average counters.

.. todo: to be explained, nstat -d lanches history and runs as daemon, you
.. need to stop it when finished

`nstat <https://www.man7.org/linux/man-pages/man8/nstat.8.html>`_ can also list
the current value of the different counters.


.. code:: console

   #nstat -az
	  #kernel
	  IpInReceives                    1073367            0.0
	  IpInHdrErrors                   0                  0.0
	  IpInAddrErrors                  0                  0.0
	  IpForwDatagrams                 0                  0.0
	  IpInUnknownProtos               0                  0.0
	  IpInDiscards                    0                  0.0
	  IpInDelivers                    1072518            0.0
	  IpOutRequests                   484889             0.0
	  IpOutDiscards                   0                  0.0
	  IpOutNoRoutes                   0                  0.0
	  IpReasmTimeout                  0                  0.0
	  IpReasmReqds                    0                  0.0
	  IpReasmOKs                      0                  0.0
	  IpReasmFails                    0                  0.0
	  IpFragOKs                       0                  0.0
	  IpFragFails                     0                  0.0
	  IpFragCreates                   0                  0.0
	  IcmpInMsgs                      561                0.0
	  IcmpInErrors                    125                0.0
	  IcmpInCsumErrors                0                  0.0
	  IcmpInDestUnreachs              6                  0.0
	  IcmpInTimeExcds                 125                0.0
	  IcmpInParmProbs                 0                  0.0
	  IcmpInSrcQuenchs                0                  0.0
	  IcmpInRedirects                 0                  0.0
	  IcmpInEchos                     298                0.0
	  IcmpInEchoReps                  0                  0.0
	  IcmpInTimestamps                33                 0.0
	  IcmpInTimestampReps             0                  0.0
	  IcmpInAddrMasks                 99                 0.0
	  IcmpInAddrMaskReps              0                  0.0
	  IcmpOutMsgs                     331                0.0
	  IcmpOutErrors                   0                  0.0
	  IcmpOutDestUnreachs             0                  0.0
	  IcmpOutTimeExcds                0                  0.0
	  IcmpOutParmProbs                0                  0.0
	  IcmpOutSrcQuenchs               0                  0.0
	  IcmpOutRedirects                0                  0.0
	  IcmpOutEchos                    0                  0.0
	  IcmpOutEchoReps                 298                0.0
	  IcmpOutTimestamps               0                  0.0
	  IcmpOutTimestampReps            33                 0.0
	  IcmpOutAddrMasks                0                  0.0
	  IcmpOutAddrMaskReps             0                  0.0
	  IcmpMsgInType3                  6                  0.0
	  IcmpMsgInType8                  298                0.0
	  IcmpMsgInType11                 125                0.0
	  IcmpMsgInType13                 33                 0.0
	  IcmpMsgInType17                 99                 0.0
	  IcmpMsgOutType0                 298                0.0
	  IcmpMsgOutType14                33                 0.0
	  TcpActiveOpens                  3330               0.0
	  TcpPassiveOpens                 252                0.0
	  TcpAttemptFails                 0                  0.0
	  TcpEstabResets                  78                 0.0
	  TcpInSegs                       3202615            0.0
	  TcpOutSegs                      6431616            0.0
	  TcpRetransSegs                  7584               0.0
	  TcpInErrs                       0                  0.0
	  TcpOutRsts                      102                0.0
	  TcpInCsumErrors                 0                  0.0
	  UdpInDatagrams                  18972              0.0
	  UdpNoPorts                      0                  0.0
	  UdpInErrors                     0                  0.0
	  UdpOutDatagrams                 19257              0.0
	  UdpRcvbufErrors                 0                  0.0
	  UdpSndbufErrors                 0                  0.0
	  UdpInCsumErrors                 0                  0.0
	  UdpIgnoredMulti                 19989              0.0
	  UdpMemErrors                    0                  0.0
	  UdpLiteInDatagrams              0                  0.0
	  UdpLiteNoPorts                  0                  0.0
	  UdpLiteInErrors                 0                  0.0
	  UdpLiteOutDatagrams             0                  0.0
	  UdpLiteRcvbufErrors             0                  0.0
	  UdpLiteSndbufErrors             0                  0.0
	  UdpLiteInCsumErrors             0                  0.0
	  UdpLiteIgnoredMulti             0                  0.0
	  UdpLiteMemErrors                0                  0.0
	  Ip6InReceives                   2198489            0.0
	  Ip6InHdrErrors                  0                  0.0
	  Ip6InTooBigErrors               0                  0.0
	  Ip6InNoRoutes                   200                0.0
	  Ip6InAddrErrors                 0                  0.0
	  Ip6InUnknownProtos              0                  0.0
	  Ip6InTruncatedPkts              0                  0.0
	  Ip6InDiscards                   0                  0.0
	  Ip6InDelivers                   2177604            0.0
	  Ip6OutForwDatagrams             0                  0.0
	  Ip6OutRequests                  1567967            0.0
	  Ip6OutDiscards                  0                  0.0
	  Ip6OutNoRoutes                  6                  0.0
	  Ip6ReasmTimeout                 0                  0.0
	  Ip6ReasmReqds                   0                  0.0
	  Ip6ReasmOKs                     0                  0.0
	  Ip6ReasmFails                   0                  0.0
	  Ip6FragOKs                      0                  0.0
	  Ip6FragFails                    0                  0.0
	  Ip6FragCreates                  0                  0.0
	  Ip6InMcastPkts                  20785              0.0
	  Ip6OutMcastPkts                 13                 0.0
	  Ip6InOctets                     2578707266         0.0
	  Ip6OutOctets                    3533261025         0.0
	  Ip6InMcastOctets                1442288            0.0
	  Ip6OutMcastOctets               1252               0.0
	  Ip6InBcastOctets                0                  0.0
	  Ip6OutBcastOctets               0                  0.0
	  Ip6InNoECTPkts                  2060704            0.0
	  Ip6InECT1Pkts                   0                  0.0
	  Ip6InECT0Pkts                   137799             0.0
	  Ip6InCEPkts                     0                  0.0
	  Icmp6InMsgs                     7525               0.0
	  Icmp6InErrors                   0                  0.0
	  Icmp6OutMsgs                    7511               0.0
	  Icmp6OutErrors                  0                  0.0
	  Icmp6InCsumErrors               0                  0.0
	  Icmp6InDestUnreachs             10                 0.0
	  Icmp6InPktTooBigs               0                  0.0
	  Icmp6InTimeExcds                0                  0.0
	  Icmp6InParmProblems             0                  0.0
	  Icmp6InEchos                    2                  0.0
	  Icmp6InEchoReplies              6                  0.0
	  Icmp6InGroupMembQueries         0                  0.0
	  Icmp6InGroupMembResponses       0                  0.0
	  Icmp6InGroupMembReductions      0                  0.0
	  Icmp6InRouterSolicits           0                  0.0
	  Icmp6InRouterAdvertisements     0                  0.0
	  Icmp6InNeighborSolicits         4316               0.0
	  Icmp6InNeighborAdvertisements   3189               0.0
	  Icmp6InRedirects                0                  0.0
	  Icmp6InMLDv2Reports             2                  0.0
	  Icmp6OutDestUnreachs            0                  0.0
	  Icmp6OutPktTooBigs              0                  0.0
	  Icmp6OutTimeExcds               0                  0.0
	  Icmp6OutParmProblems            0                  0.0
	  Icmp6OutEchos                   6                  0.0
	  Icmp6OutEchoReplies             2                  0.0
	  Icmp6OutGroupMembQueries        0                  0.0
	  Icmp6OutGroupMembResponses      0                  0.0
	  Icmp6OutGroupMembReductions     0                  0.0
	  Icmp6OutRouterSolicits          0                  0.0
	  Icmp6OutRouterAdvertisements    0                  0.0
	  Icmp6OutNeighborSolicits        3179               0.0
	  Icmp6OutNeighborAdvertisements  4316               0.0
	  Icmp6OutRedirects               0                  0.0
	  Icmp6OutMLDv2Reports            8                  0.0
	  Icmp6InType1                    10                 0.0
	  Icmp6InType128                  2                  0.0
	  Icmp6InType129                  6                  0.0
	  Icmp6InType135                  4316               0.0
	  Icmp6InType136                  3189               0.0
	  Icmp6InType143                  2                  0.0
	  Icmp6OutType128                 6                  0.0
	  Icmp6OutType129                 2                  0.0
	  Icmp6OutType135                 3179               0.0
	  Icmp6OutType136                 4316               0.0
	  Icmp6OutType143                 8                  0.0
	  Udp6InDatagrams                 460                0.0
	  Udp6NoPorts                     0                  0.0
	  Udp6InErrors                    0                  0.0
	  Udp6OutDatagrams                95                 0.0
	  Udp6RcvbufErrors                0                  0.0
	  Udp6SndbufErrors                0                  0.0
	  Udp6InCsumErrors                0                  0.0
	  Udp6IgnoredMulti                0                  0.0
	  Udp6MemErrors                   0                  0.0
	  UdpLite6InDatagrams             0                  0.0
	  UdpLite6NoPorts                 0                  0.0
	  UdpLite6InErrors                0                  0.0
	  UdpLite6OutDatagrams            0                  0.0
	  UdpLite6RcvbufErrors            0                  0.0
	  UdpLite6SndbufErrors            0                  0.0
	  UdpLite6InCsumErrors            0                  0.0
	  UdpLite6MemErrors               0                  0.0
	  TcpExtSyncookiesSent            0                  0.0
	  TcpExtSyncookiesRecv            0                  0.0
	  TcpExtSyncookiesFailed          0                  0.0
	  TcpExtEmbryonicRsts             0                  0.0
	  TcpExtPruneCalled               3791               0.0
	  TcpExtRcvPruned                 0                  0.0
	  TcpExtOfoPruned                 0                  0.0
	  TcpExtOutOfWindowIcmps          0                  0.0
	  TcpExtLockDroppedIcmps          0                  0.0
	  TcpExtArpFilter                 0                  0.0
	  TcpExtTW                        2283               0.0
	  TcpExtTWRecycled                0                  0.0
	  TcpExtTWKilled                  0                  0.0
	  TcpExtPAWSActive                0                  0.0
	  TcpExtPAWSEstab                 11                 0.0
	  TcpExtDelayedACKs               31995              0.0
	  TcpExtDelayedACKLocked          47                 0.0
	  TcpExtDelayedACKLost            282                0.0
	  TcpExtListenOverflows           0                  0.0
	  TcpExtListenDrops               0                  0.0
	  TcpExtTCPHPHits                 699069             0.0
	  TcpExtTCPPureAcks               997468             0.0
	  TcpExtTCPHPAcks                 1235546            0.0
	  TcpExtTCPRenoRecovery           0                  0.0
	  TcpExtTCPSackRecovery           2526               0.0
	  TcpExtTCPSACKReneging           0                  0.0
	  TcpExtTCPSACKReorder            36858              0.0
	  TcpExtTCPRenoReorder            0                  0.0
	  TcpExtTCPTSReorder              85                 0.0
	  TcpExtTCPFullUndo               1                  0.0
	  TcpExtTCPPartialUndo            67                 0.0
	  TcpExtTCPDSACKUndo              11                 0.0
	  TcpExtTCPLossUndo               0                  0.0
	  TcpExtTCPLostRetransmit         184                0.0
	  TcpExtTCPRenoFailures           0                  0.0
	  TcpExtTCPSackFailures           0                  0.0
	  TcpExtTCPLossFailures           0                  0.0
	  TcpExtTCPFastRetrans            7084               0.0
	  TcpExtTCPSlowStartRetrans       0                  0.0
	  TcpExtTCPTimeouts               168                0.0
	  TcpExtTCPLossProbes             345                0.0
	  TcpExtTCPLossProbeRecovery      82                 0.0
	  TcpExtTCPRenoRecoveryFail       0                  0.0
	  TcpExtTCPSackRecoveryFail       0                  0.0
	  TcpExtTCPRcvCollapsed           0                  0.0
	  TcpExtTCPBacklogCoalesce        10938              0.0
	  TcpExtTCPDSACKOldSent           300                0.0
	  TcpExtTCPDSACKOfoSent           49                 0.0
	  TcpExtTCPDSACKRecv              317                0.0
	  TcpExtTCPDSACKOfoRecv           2                  0.0
	  TcpExtTCPAbortOnData            25                 0.0
	  TcpExtTCPAbortOnClose           54                 0.0
	  TcpExtTCPAbortOnMemory          0                  0.0
	  TcpExtTCPAbortOnTimeout         4                  0.0
	  TcpExtTCPAbortOnLinger          0                  0.0
	  TcpExtTCPAbortFailed            0                  0.0
	  TcpExtTCPMemoryPressures        0                  0.0
	  TcpExtTCPMemoryPressuresChrono  0                  0.0
	  TcpExtTCPSACKDiscard            0                  0.0
	  TcpExtTCPDSACKIgnoredOld        2                  0.0
	  TcpExtTCPDSACKIgnoredNoUndo     272                0.0
	  TcpExtTCPSpuriousRTOs           0                  0.0
	  TcpExtTCPMD5NotFound            0                  0.0
	  TcpExtTCPMD5Unexpected          0                  0.0
	  TcpExtTCPMD5Failure             0                  0.0
	  TcpExtTCPSackShifted            34290              0.0
	  TcpExtTCPSackMerged             11301              0.0
	  TcpExtTCPSackShiftFallback      40480              0.0
	  TcpExtTCPBacklogDrop            0                  0.0
	  TcpExtPFMemallocDrop            0                  0.0
	  TcpExtTCPMinTTLDrop             0                  0.0
	  TcpExtTCPDeferAcceptDrop        0                  0.0
	  TcpExtIPReversePathFilter       0                  0.0
	  TcpExtTCPTimeWaitOverflow       0                  0.0
	  TcpExtTCPReqQFullDoCookies      0                  0.0
	  TcpExtTCPReqQFullDrop           0                  0.0
	  TcpExtTCPRetransFail            0                  0.0
	  TcpExtTCPRcvCoalesce            100585             0.0
	  TcpExtTCPOFOQueue               15954              0.0
	  TcpExtTCPOFODrop                0                  0.0
	  TcpExtTCPOFOMerge               38                 0.0
	  TcpExtTCPChallengeACK           0                  0.0
	  TcpExtTCPSYNChallenge           0                  0.0
	  TcpExtTCPFastOpenActive         0                  0.0
	  TcpExtTCPFastOpenActiveFail     0                  0.0
	  TcpExtTCPFastOpenPassive        0                  0.0
	  TcpExtTCPFastOpenPassiveFail    0                  0.0
	  TcpExtTCPFastOpenListenOverflow 0                  0.0
	  TcpExtTCPFastOpenCookieReqd     0                  0.0
	  TcpExtTCPFastOpenBlackhole      0                  0.0
	  TcpExtTCPSpuriousRtxHostQueues  0                  0.0
	  TcpExtBusyPollRxPackets         0                  0.0
	  TcpExtTCPAutoCorking            73847              0.0
	  TcpExtTCPFromZeroWindowAdv      40                 0.0
	  TcpExtTCPToZeroWindowAdv        40                 0.0
	  TcpExtTCPWantZeroWindowAdv      2870               0.0
	  TcpExtTCPSynRetrans             91                 0.0
	  TcpExtTCPOrigDataSent           5948573            0.0
	  TcpExtTCPHystartTrainDetect     34                 0.0
	  TcpExtTCPHystartTrainCwnd       1880               0.0
	  TcpExtTCPHystartDelayDetect     3                  0.0
	  TcpExtTCPHystartDelayCwnd       261                0.0
	  TcpExtTCPACKSkippedSynRecv      0                  0.0
	  TcpExtTCPACKSkippedPAWS         9                  0.0
	  TcpExtTCPACKSkippedSeq          11                 0.0
	  TcpExtTCPACKSkippedFinWait2     0                  0.0
	  TcpExtTCPACKSkippedTimeWait     0                  0.0
	  TcpExtTCPACKSkippedChallenge    0                  0.0
	  TcpExtTCPWinProbe               0                  0.0
	  TcpExtTCPKeepAlive              67                 0.0
	  TcpExtTCPMTUPFail               0                  0.0
	  TcpExtTCPMTUPSuccess            0                  0.0
	  TcpExtTCPDelivered              5951000            0.0
	  TcpExtTCPDeliveredCE            0                  0.0
	  TcpExtTCPAckCompressed          3021               0.0
	  TcpExtTCPZeroWindowDrop         0                  0.0
	  TcpExtTCPRcvQDrop               0                  0.0
	  TcpExtTCPWqueueTooBig           0                  0.0
	  TcpExtTCPFastOpenPassiveAltKey  0                  0.0
	  TcpExtTcpTimeoutRehash          72                 0.0
	  TcpExtTcpDuplicateDataRehash    0                  0.0
	  TcpExtTCPDSACKRecvSegs          371                0.0
	  TcpExtTCPDSACKIgnoredDubious    0                  0.0
	  TcpExtTCPMigrateReqSuccess      0                  0.0
	  TcpExtTCPMigrateReqFailure      0                  0.0
	  IpExtInNoRoutes                 0                  0.0
	  IpExtInTruncatedPkts            0                  0.0
	  IpExtInMcastPkts                62                 0.0
	  IpExtOutMcastPkts               24                 0.0
	  IpExtInBcastPkts                19989              0.0
	  IpExtOutBcastPkts               0                  0.0
	  IpExtInOctets                   533061309          0.0
	  IpExtOutOctets                  5153892360         0.0
	  IpExtInMcastOctets              7448               0.0
	  IpExtOutMcastOctets             3592               0.0
	  IpExtInBcastOctets              2082276            0.0
	  IpExtOutBcastOctets             0                  0.0
	  IpExtInCsumErrors               0                  0.0
	  IpExtInNoECTPkts                1073527            0.0
	  IpExtInECT1Pkts                 0                  0.0
	  IpExtInECT0Pkts                 0                  0.0
	  IpExtInCEPkts                   0                  0.0
	  IpExtReasmOverlaps              0                  0.0
	  MPTcpExtMPCapableSYNRX          0                  0.0
	  MPTcpExtMPCapableSYNTX          2203               0.0
	  MPTcpExtMPCapableSYNACKRX       2172               0.0
	  MPTcpExtMPCapableACKRX          0                  0.0
	  MPTcpExtMPCapableFallbackACK    0                  0.0
	  MPTcpExtMPCapableFallbackSYNACK 22                 0.0
	  MPTcpExtMPFallbackTokenInit     0                  0.0
	  MPTcpExtMPTCPRetrans            0                  0.0
	  MPTcpExtMPJoinNoTokenFound      0                  0.0
	  MPTcpExtMPJoinSynRx             0                  0.0
	  MPTcpExtMPJoinSynAckRx          0                  0.0
	  MPTcpExtMPJoinSynAckHMacFailure 0                  0.0
	  MPTcpExtMPJoinAckRx             0                  0.0
	  MPTcpExtMPJoinAckHMacFailure    0                  0.0
	  MPTcpExtDSSNotMatching          0                  0.0
	  MPTcpExtInfiniteMapRx           0                  0.0
	  MPTcpExtDSSNoMatchTCP           0                  0.0
	  MPTcpExtDataCsumErr             0                  0.0
	  MPTcpExtOFOQueueTail            0                  0.0
	  MPTcpExtOFOQueue                0                  0.0
	  MPTcpExtOFOMerge                0                  0.0
	  MPTcpExtNoDSSInWindow           0                  0.0
	  MPTcpExtDuplicateData           0                  0.0
	  MPTcpExtAddAddr                 0                  0.0
	  MPTcpExtEchoAdd                 0                  0.0
	  MPTcpExtPortAdd                 0                  0.0
	  MPTcpExtAddAddrDrop             0                  0.0
	  MPTcpExtMPJoinPortSynRx         0                  0.0
	  MPTcpExtMPJoinPortSynAckRx      0                  0.0
	  MPTcpExtMPJoinPortAckRx         0                  0.0
	  MPTcpExtMismatchPortSynRx       0                  0.0
	  MPTcpExtMismatchPortAckRx       0                  0.0
	  MPTcpExtRmAddr                  0                  0.0
	  MPTcpExtRmAddrDrop              0                  0.0
	  MPTcpExtRmSubflow               0                  0.0
	  MPTcpExtMPPrioTx                0                  0.0
	  MPTcpExtMPPrioRx                0                  0.0
	  MPTcpExtMPFailTx                0                  0.0
	  MPTcpExtMPFailRx                0                  0.0
	  MPTcpExtMPFastcloseTx           0                  0.0
	  MPTcpExtMPFastcloseRx           0                  0.0
	  MPTcpExtMPRstTx                 17                 0.0
	  MPTcpExtMPRstRx                 0                  0.0
	  MPTcpExtRcvPruned               0                  0.0
	  MPTcpExtSubflowStale            0                  0.0
	  MPTcpExtSubflowRecover          0                  0.0
	  
.. todo: explain patterns


.. todo explain how to use -n and -s

   
Among all these variables, the ones named ``\*Ext\*`` are Linux specific
variables that are not defined in IETF MIBs. The others are usually defined
in an IETF RFC. The counters maintained by the Linux kernel are defined in
`include/uapi/linux/snmp.h <https://github.com/torvalds/linux/blob/master/include/uapi/linux/snmp.h>`_ and
`net/mptcp/mib.h <https://github.com/torvalds/linux/blob/master/net/mptcp/mib.h>`_ for the Multipath TCP counters. 
Each of the counters exposed by nstat_ correspond to one specific identifier
in the Linux kernel. For example, the beginning of the IP part of the
counters is defined as follows:

.. code:: c

   enum
   {
	IPSTATS_MIB_NUM = 0,
	/* frequently written fields in fast path, kept in same cache line */
	IPSTATS_MIB_INPKTS,			/* InReceives */
	IPSTATS_MIB_INOCTETS,			/* InOctets */
	IPSTATS_MIB_INDELIVERS,			/* InDelivers */
	IPSTATS_MIB_OUTFORWDATAGRAMS,		/* OutForwDatagrams */
	IPSTATS_MIB_OUTPKTS,			/* OutRequests */
	IPSTATS_MIB_OUTOCTETS,			/* OutOctets */
	/* other fields */


Before looking at the precise meaning of each of the counters managed by
`nstat <https://www.man7.org/linux/man-pages/man8/nstat.8.html>`_,
it is interesting to recall the definition of the Case diagrams. This graphical
representation of SNMP variables can be really useful to understand the
meaning of the Linux networking counters.

The Case diagrams
+++++++++++++++++

The :index:`Case diagrams` were introduced by Jeffrey Case and Craig
Partridge in 1989 in the paper `Case diagrams: a first step to diagrammed
management information bases <https://doi.org/10.1145/66093.66094>`_.
This article describes a simple but powerful graphical representation
of the interactions among the different SNMP variables that a networking
stack maintains.

A `Case diagram` represents the flow of packets through a stack and the
different variables that are updated as the packet progress through the
stack. The incoming packets are represented as progressing from
the bottom layer of the stack to the upper layer, while the outgoing
packets are represented in the other direction. The progression of these
packets is represented using a large arrow. An horizontal line
that crosses this arrow indicates the point in the stack where the associated
SNMP counter is updated. A small that leaves the main packet processing
flow indicates a specific treatment for a packet and a counter
that is updated. In some cases, an arrow enters the main workflow and
updates the associated counter.

The original paper used the IP counters of the MIB-2 to illustrate the
`Case diagrams`. This figure is reproduced below in ASCII format to simplify
the updates to the document.

.. code:: console

                      Transport Layer
   -----------------------------------------------------
	                    /\
                            ||
			    ||
    IpInDelivers +++++++++++++++
                            ||
			    ||
			    |+-----------> IpInUnknownProtos
			    ||
			    ||
    IpInDiscards <----------+|
                            ||
			    |<----------------  IpReasmOKs
			    ||                     /\     
			    ||   IpReasmFails  <---+|
			    ||                     ||
			    |+--------------> IpReasmReqds
			    ||
			    ||
    IpForwDatagrams <-------+|
                            ||
			    |+--------------> IpInAddrErrors
			    ||
			    ||
    IpInHdrErrors <---------+|
                            ||
			    ||
			    ||
			 +++++++++++++++++++ IpInReceives   
                            ||
   -----------------------------------------------------	  
                      Interface Layer
			 

The `Case diagram` above shows how the packets are processed by the
IP stack. First, the Interface layer extracts the payload of the
received frame and passes it to the IP layer. At this point, the
``IpInReceives`` counter is incremented. The processing of the
IPv4 packet starts. First, the stack checks for errors inside the IPv4
header. If an error is detected in the IPv4 header, the packet is dropped
and ``IpInHdrErrors`` is incremented. Then, the destination address is checked.
If the address is incorrect, the packet processing stops and ``IpInAddrErrors``
is incremented.

If IP forwarding is enabled and the packet is not destined to this host,
then the packet is forwarded using the FIB. The ``IpForwDatagrams`` counter
is incremented.

The next step is to check whether the received packet is a fragment of
a larger packet that needs to be reassembled. If the received packet is a
fragment, then the ``IpReasmReqds`` counter is incremented and the
packet passed through the reassembly process. This reassembly can take time
since more fragments can be required to recover a complete packet. If
the packet reassembly succeeds, then ``IpReasmOKs`` is incremented and
the processing of the full packet continues. If the reassembly fails, e.g.
because a fragment is missing before the timeout expires, then
``IpReamsFails`` gets incremented.

A this point, the packets have almost finished to be processed by the
IP stack. Most packets will be delivered to the transport layer
and increment the ``IpInDelivers`` counter except if the IP queue becomes
full. In this case, the ``IpInDiscards`` counter is incremented.
The incoming packet could also be discarded if its `Protocol` field
does not match one of the transport layers supported by the stack
(i.e. UDP, TCP, DCCP, ...). In this case, the ``IpInUnknownProtos``
counter is incremented.
