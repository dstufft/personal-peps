PEP: XXX
Title: Removal of the PyPI Mirror Authenticity API
Version: $Revision$
Last-Modified: $Date$
Author: Donald Stufft <donald@stufft.io>
BDFL-Delegate: Richard Jones <richard@python.org>
Discussions-To: distutils-sig@python.org
Status: Draft
Type: Process
Content-Type: text/x-rst
Created: 02-Mar-2014
Post-History:
Replaces: 381


Abstract
========

This PEP proposes the deprecation and removal of the PyPI Mirror Authenticity
API, this includes the /serverkey URL and all of the URLs under /serversig.


Rationale
=========

The PyPI mirroring infrastructure (defined in PEP 381) provides a means to
mirror the content of PyPI used by the automatic installers, and as a component
of that, it provides a method for verifying the authenticity of the mirrored
content.

This PEP proposal the removal of this API due to:

* No known implementations that utilize this API are known.
* Because this API uses DSA it is vulnerable to leaking the private key if
  there is *any* bias in the random nonce.
* This API solves one small corner of the trust problem, however the problem
  itself is much larger and it would be better to have a fully fledged system,
  such as The Update Framework [1]_, instead.

Due to the issues it has and the lack of use it is the opinion of this PEP
that it does not provide any benefit and instead is legacy cruft.


Plan for Deprecation & Removal
==============================

Immediately upon the acceptance of this PEP the Mirror Authenticity API will be
considered deprecated and mirroring agents and installation tools should stop
accessing it.

Instead of actually removing it from the current code base (PyPI 1.0) the
current work to replace PyPI 1.0 with a new code base (PyPI 2.0) will simply
not implement this API. This would cause the API to be "removed" when the
switch from 1.0 to 2.0 occurs.





References
==========

.. [1] The Update Framework
   (https://python.org/dev/peps/pep-0458/)


Copyright
=========

This document has been placed in the public domain.



..
   Local Variables:
   mode: indented-text
   indent-tabs-mode: nil
   sentence-end-double-space: t
   fill-column: 70
   coding: utf-8
   End:
