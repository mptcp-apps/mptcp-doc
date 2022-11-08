.. Multipath TCP documentation

   
Multipath TCP
=============

.. only:: builder_html

	  Multipath TCP, defined in :rfc:`8684`, is a major extension ot the TCP protocol which is used by the majority of Internet applications. Despite the benefits of Multipath TCP, this protocol `is not yet widely deployed <https://mptcp.io>`_. To encourage the adoption and the deployment of Multipath TCP, this document first presents the main use cases for Multipath TCP. Then, after a `brief recap of TCP <chapter-tcp>`_, it provides a `detailed description of the Multipath TCP protocol <chapter-mptcp>_` and its main design choices. Then, the document focuses on the main Multipath TCP implementations on `recent Linux kernels <chapter-linux>`_ and `Apple devices <chapter-apple>`_.
	  
	  Contributions and comments are more than welcome via `https://github.com/mptcp-apps/mptcp-doc <https://github.com/mptcp-apps/mptcp-doc>`_. A pdf version of this document is also available from :download:`MPTCP-doc.pdf <docs/latex/MPTCP-doc.pdf>`


.. toctree::
   :maxdepth: 2
   :caption: Contents:

	     
   usecases
   tcp
   mptcp
   mptcp-linux
   mptcp-apple
   biblio

   
Indices and tables
==================

* :ref:`genindex`
* :ref:`modindex`
* :ref:`search`
