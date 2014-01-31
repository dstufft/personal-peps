PEP: XXXX
Title: Wheel 2.0 - Moar Better Wheels
Version: $Revision$
Last-Modified: $Date$
Author: Donald Stufft <donald@stufft.io>
BDFL-Delegate: Nick Coghlan <ncoghlan@gmail.com>
Discussions-To: distutils-sig@python.org
Status: Draft
Type: Standards Track
Content-Type: text/x-rst
Created: 30-Jan-2014
Post-History:
Replaces: 427


Abstract
========

This PEP refines the specification of Wheel from PEP 427 and defines version
2.0 of the Wheel format.

      The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL
      NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED",  "MAY", and
      "OPTIONAL" in this document are to be interpreted as described in
      RFC 2119.


Rationale
=========

PEP 427 introduced the Wheel format. In the year since it's acceptance Wheel
has been implemented in pip and is now on by default and well on it's way to
being adopted. In that time a number of problems have been found with the
current specification including undefined behavior, missing features, and
included mis-features.

Wheel 2.0 is explicitly defined to be an installer oriented distribution file
aimed at making the job of distributing and installing a binary package easier.
The ramifications of this is that Wheel 2.0 outright rejects any feature or
change that makes it more difficult or more ambiguous for an *installer* to
work with Wheel 2.0 but instead makes it easier for a *human* to manually
interact with the contents of a Wheel 2.0 file.


Specification
=============


Name and Version Normalization
------------------------------

There are several locations in this specification where the name or version
of a project is used to construct file or directory names. In these cases
these constructed names are designed to be machine parseable and use the ``-``
character as a delimiter. For this reason a ``-`` character in a project name
or version **MUST** be escaped to be a ``_`` character and implementations
**SHOULD** consider ``-`` and ``_`` to be equivalent within the data extracted
from a file or directory name.


Container and Compression
-------------------------

A Wheel file **MUST** be a Zip-format archive which **SHOULD** be compatible
with the Zip64 extension for archive sizes larger than 4GB. For compatibility
with Python installations that do not have the optional zlib module
implementations **MUST** support uncompressed Wheels through the ``ZIP_STORED``
constant and **SHOULD** support Wheels compressed with the deflate algorithm
through the ``ZIP_DEFLATE`` constant.


Archive File Layout
-------------------

.. code::

   .
   ├── {name}-{version}.dist-info/
   │   ├── METADATA
   │   ├── WHEEL
   │   └── wheel.json
   ├── {name}-{version}.data/
   │   ├── data/
   │   ├── include/
   │   ├── platinclude/
   │   ├── platlib/
   │   ├── platstdlib/
   │   ├── purelib/
   │   ├── scripts/
   │   └── stdlib/
   └── __main__.py


A Wheel **MUST** have a dist-info directory where the directory name **MUST**
follow the format ``{name}-{version}.dist-info``.

A Wheel **MUST** have a ``METADATA`` file located in the dist-info directory.

A Wheel **MUST** have a ``WHEEL`` file located in the dist-info directory.

A Wheel **MUST** have a ``wheel.json`` file located in the dist-info directory.

A Wheel **SHOULD** have a data directory where the directory name **MUST**
follow the format ``{name}-{version}.data``.

A Wheel **MAY** have a single sub directory per sysconfig path name within the
``{name}-{version}.data`` directory.

A Wheel **MAY** be made into an executable Wheel by including a ``__main__.py``
in the root of the Wheel file.


Backwards Incompatibilities
===========================


Specialized Installer vs "Simple" Installer
-------------------------------------------

One goal of the Wheel 1.0 specification was that that simple Wheel files could
be installed with a "simple" installer, namely the ``unzip`` tool. With Wheel
2.0 this is no longer a goal of the Wheel format. While it is true that the
simplest of Wheels could be "manually" installed in this fashion, it is not
the case that all Wheels could be installed in this fashion. Information about
whether or not a Wheel could be reliably installed this way is not available
and would require end users attempting to do this to inspect the Wheel file
manually *and* be aware of the inner workings of the Wheel file format to know
if it is using any features that require additional installer support.

As more features are added the gap between what types of Wheels can be
installed by simply unzipping them into site-packages will only widen. This
includes features such as generated script wrappers and install hooks.

Additionally attempting to make it easy to install a Wheel by hand dilutes
the specification, causing divergent concerns to create hackish work-arounds
such as the ``Root-Is-Purelib`` metadata.


Wheel and zipimport
-------------------

Another goal of the Wheel 1.0 specification was that Wheels could be added to
the ``sys.path`` or ``PYTHONPATH`` and directly imported without unzipping them
through the user of the standard ``zipimport`` loader. Wheel 2.0 rejects the
compatibility with the standard ``zipimport`` loader as a goal.

The ``zipimport`` module was designed to operate on "dumb" zip files that
contain no information about what they support or if they are expected to work
at all. This causes end users to have to try and determine if any particular
Wheel file is compatible with ``zipimport`` where they may or may not have
any idea what makes a Wheel compatible or not. Further more the standard
``zipimport`` module loader imposes restrictions on the file layout of the
Wheel that makes it less convenient for an installer to actually install them.


Wheel Signing
-------------

Wheel 1.0 includes somewhat adhoc support for signing the files inside of the
Wheel. It does this by modifying the ``RECORD`` from PEP 376 and allowing
either JWS or S/MIME detached signatures. Wheel 2.0 rejects Wheel specific
signature schemes in favor of format agnostic signature schemes.

Signing a file without a mechanism for establishing trust is largely useless.
Additionally signing and trust is a much larger problem that is outside the
scope of any specific package format. Finally a signature solution needs to
support more than just Wheels, therefore the existence of Wheel signing
exists primarily to confuse the situation when such a signing scheme is finally
available.


TODO:
=====

* Move filename format into it's own PEP. Package discovery isn't part of
  the file format and deserves it's own PEP.
* WheelImporter. Add .whl to PYTHONPATH, get compatibility checks and other
  fun things.
* Decide about wheel.json, Key: Value is lame but so is WHEEL + wheel.json,
  however the WHEEL file is required for compatibility


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
