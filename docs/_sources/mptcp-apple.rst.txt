Multipath TCP on Apple devices
==============================

`Apple <https://www.apple.com>`_ is among the early adopters of Multipath TCP.
The first implementation appeared in iOS7 in September 2013. The implementation has been refined later and incorporated in MacOS as well. As of 2022, it is likely that iOS, iPadOS and MacOS use the same Multipath TCP implementations.

Apple does not publish lots of details about the features of its Multipath TCP implementations, but interested readers can explore the source code of recent OS releases. 



Multipath TCP on MacOS Big Sur
------------------------------

On MacOS, part of the Multipath TCP parameters can be specified using ``sysctl`` variables. The following variables are supported on Big Sur :

.. code-block:: console

   sysctl -a | grep mptcp
   net.inet.mptcp.dss_csum: 0
   net.inet.mptcp.enable: 1
   net.inet.mptcp.fail: 1
   net.inet.mptcp.keepalive: 840
   net.inet.mptcp.mptcp_cap_retr: 4
   net.inet.mptcp.probecnt: 5
   net.inet.mptcp.probeto: 1000
   net.inet.mptcp.rto_thresh: 1500
   net.inet.mptcp.rtthist_thresh: 600
   net.inet.mptcp.alternate_port: 0
   net.inet.mptcp.dbg_area: 31
   net.inet.mptcp.dbg_level: 1
   net.inet.mptcp.pcbcount: 0
   net.inet.mptcp.allow_aggregate: 0
   net.inet.mptcp.expected_progress_headstart: 5000
   net.inet.mptcp.no_first_party: 0
   net.inet.mptcp.nrto: 3
   net.inet.mptcp.rto: 3
   net.inet.mptcp.tw: 60


There is no detailed description of the definition of these ``sysctl``
variables on MacOS, but some can be inferred based on their names and the
sources of the  `xnu sources <https://github.com/apple-oss-distributions/xnu/tree/main/bsd/netinet>`_.

The first important ``sysctl`` variable is ``net.inet.mptcp.enable``. It controls whether Multipath TCP is enabled (variable set to ``1``, the default) or not.

The second ``sysctl`` variable is ``net.inet.mptcp.dss_csum``. It controls the utilization of the ``DSS`` checksum. By default, this checksum is disabled, but it can be enabled if required. 

The third ``sysctl`` variable, ``net.inet.mptcp.mptcp_cap_retr``  probably controls the number of ``SYN`` with the ``MP_CAPABLE`` option that are sent before removing this option to prevent broken middleboxes from blocking the establishment of Multipath TCP connections. The default value for this parameter is 4. 

The fourth ``sysctl`` variable, ``net.inet.mptcp.allow_aggregate`` probably controls whether Multipath TCP can actively use two different subflows to send data. The initial use case for Multipath TCP on Apple devices was mainly failover and the ability to aggregate bandwidth was added later. The default value of this option is ``0`` which indicates that it is disabled.


Multipath TCP on MacOS Ventura
------------------------------

.. todo: Get recent Mac


Adding Multipath TCP support to existing applications
-----------------------------------------------------

Apple exposes some of the Multipath TCP features to applications written in Objective C using the `Network Framework <https://developer.apple.com/documentation/network/2976692-nw_connection_start?language=objc>`_. In particular, the `example <https://developer.apple.com/documentation/network/implementing_netcat_with_network_framework?language=objc>` showing how to implement a ``netcat``-like application would useful for application developers.





Multipath TCP enabled applications
----------------------------------

Apple uses Multipath TCP for some of its own applications, notably:

 - Siri
 - Apple Maps
 - Apple Music


Third party applications can also use Multipath TCP. These include :

 - the excellent `Secure ShellFish <https://secureshellfish.app>`_ ``ssh`` client


   
