PEP: XXXX
Title: Deprecation and Removal of Externally Linked Files
Version: $Revision$
Last-Modified: $Date$
Author: Donald Stufft <donald@stufft.io>
BDFL-Delegate: Nick Coghlan <ncoghlan@gmail.com>
Discussions-To: distutils-sig@python.org
Status: Draft
Type: Process
Content-Type: text/x-rst
Created: 12-May-2014
Post-History: 30-Aug-2002
Replaces: 438


Abstract
========

This PEP proposes a change in policy on PyPI to deprecate and ultimately remove
the concept of externally hosted files. It further proposes a mechanism to
replace this in a way that is simpler to implement and easier to reason about
from an end user perspective.

It is important to remember that this is **not** about forcing anyone to host
their files on PyPI. If someone does not wish to do so they will never be under
any obligation too. They can still list their project in PyPI as an index, and
the tooling will still allow them to host it elsewhere.


Rationale
=========

There is a long history on PyPI that explains why externally hosted files
exist today in the state that they do documented in PEP438.

PEP438 proposed a system of classifying file links as either internal,
external, or unsafe. It recommended that by default only internal links would
be installed by an installer however users could opt into external links on
either a global or a per package basis. Additionally they could also opt into
unsafe links on a per package basis.

This system has turned out to be *extremely* unfriendly towards the end users
and it is the position of this PEP that the situation has become untenable. The
situation as provided by PEP438 requires an end user to be aware not only of
the difference between internal, external, and unsafe, but also to be aware of
what hosting mode the package they are trying to install is in, what links are
available on that project's /simple/ page, whether or not those links have
a properly formatted hash fragment, and what links are available from pages
linked to from that project's /simple/ page.

There are a number of common confusion/pain points with this system that I
have witnessed:

* Users unaware what the simple installer api is at all or how an installer
  locates installable files.
* Users unaware that even if the simple api links to a file, if it does
  not include a ``#md5=...`` fragment that it will be counted as unsafe.
* Users unaware that an installer can look at pages linked from the
  simple api to determine additional links, or that any links found in this
  fashion are considered unsafe.
* Users are unaware and often surprised that PyPI supports hosting your files
  someplace other than PyPI at all.

In addition to that, the information that an installer is able to provide
when an installation files is pretty minimal. We are able to detect if there
are externally hosted files directly linked from the ``/simple/`` however we
cannot detect if there are files hosted on a linked page without fetching that
page which would cause a massive performance hit just to see if there might be
a file there so we can provide a better error message.

Finally very few projects have properly linked to their external files so that
they can be safely downloaded and verified. At the time of this writing there
are a total of 65 projects which rely on externally and safely hosted files at
all.

The end result of all of this, is that with PEP438, when a user attempts to
install a file that is not hosted on PyPI typically the steps they follow are:

1. First, they attempt to install it normally, using ``pip install foobar``.
   This fails because the file is not hosted on PyPI and PEP438 has us default
   to only hosted on PyPI, and if pip detected any externally hosted files
   or other pages that we *could* have attempted to find other files at it
   will give an error message suggesting that they try
   ``--allow-external foobar``.
2. They then attempt to install their package using
   ``pip install --allow-external foobar foobar``. If they are lucky foobar is
   one of the packages which is hosted externally and safely and this will
   succeed. If they are unlucky they will get a different error message
   suggesting that they *also* try ``--allow-unverified foobar``.
3. They then attempt to install their package using
   ``pip install --allow-external foobar --allow-unverified foobar foobar``
   and this finally works.

This is the same basic steps that practically everyone goes through every time
they try to install something that is hosted on PyPI. If they are lucky it'll
only take them two steps, but typically it requires three steps. Worse there is
no real indication to these people why one package might install after two
but most require three. Even worse than that most of them will never get an
externally hosted package that does not take three steps, so they will be
increasingly annoyed and frustrated at the intermediate step and will likely
eventually just start skipping it.


Utilize Additional Indexes
==========================

It is the opinion of this PEP that package authors *must* be allowed to host
their files off of PyPI if they desire, and that installing projects hosted
in this fashion must be made reasonably easy to do.

Both pip and easy_install support the concept of additional URLs to use to
locate files during the dependency resolution phase. This provides an easy
way for end users to opt into installing projects from locations external to
PyPI that also ensures they are aware that they are doing so.

To support projects that wish to externally host their files, regardless of
reason, PyPI will gain the ability for project's to register external index
URLs and additionally a comment associated with each index URL. These URLs
will be made available on the simple page however they will not be linked or
provided in a form that older installers will automatically search them.

When an installer fetches the simple page for a project, if it finds this
additional meta-data and it cannot find any files for that project in it's
configured URLs then it should use this data to tell the user how to add one
or more of the additional URLs to search in. This message should include any
comments that the project has included to enable them to communicate to the
user and provide hints as to which URL they might want if some are only
useful or compatible with certain platforms or situations.

This feature *must* be added to PyPI prior to starting the deprecation and
removal process.


Deprecation and Removal
=======================

A new hosting mode will be added to PyPI. This hosting mode will be called
``pypi-only`` in addition to the three that PEP438 has already given us
(``pypi-explicit``, ``pypi-scrape``, ``pypi-scrape-crawl``). This hosting mode
will modify a project's ``/simple/`` page so that it only lists the files which
are directly hosted on PyPI and will not link to anything else.

Upon acceptance of this PEP and the addition of the ``pypi-only`` mode, all new
projects will by defaulted to the PyPI only mode and they will be locked to
this mode and unable to change this particular setting.

An email will then be sent out to all of the projects which are hosted only on
PyPI informing them that in one month their project will be automatically
converted to the ``pypi-only`` mode. A month after these emails have been sent
any of those projects which were emailed, which still are hosted only on PyPI
will have their mode set to ``pypi-only``.

At the same time an email will be sent to projects which rely on hosting
external to PyPI. This email will warn these projects that externally hosted
files have been deprecated on PyPI and that in 6 months from the time of that
email that all external links will be removed from the installer APIs. This
email *must* include instructions for converting their projects to be hosted
on PyPI and *must* include links to a script or package that will enable them
to enter their PyPI credentials and package name and have it automatically
download and re-host all of their files on PyPI.

Five months after the initial email, another email must be sent to any projects
still relying on external hosting. This email will include all of the same
information that the first email contained, except that the removal date will
be one month away instead of six.

Finally a month later all projects will be switched to the ``pypa-only`` mode
and PyPI will be modified to remove the externally linked files functionality.


Statistics
==========

=================  ===========
     Hosting         Projects
=================  ===========
Hosted on PyPI      37779
Hosted Externally   65
Hosted Unsafely     2974
=================  ===========


Rejected Proposals
==================

Keep the current classification system but adjust the options
-------------------------------------------------------------

This PEP rejects several related proposals which attempt to fix some of the
usability problems with the current system but while still keeping the
general gist of PEP438.

This includes:

* Default to allowing safely externally hosted files, but disallow unsafely
  hosted.
* Default to disallowing safely externally hosted files with only a global
  flag to enable them, but disallow unsafely hosted.

These proposals are rejected because:

* The classification "system" is complex, hard to explain, and requires an
  intimate knowledge of how the simple API works in order to be able to reason
  about which classification is required. This is reflected in the fact that
  the code to implement it is complicated and hard to understand as well.

* People are generally surprised that PyPI allows externally linking to files
  and doesn't require people to host on PyPI. In contrast most of them are
  familiar with the concept of multiple software repositories such as is in
  use by many OSs.

* PyPI is fronted by a globally distributed CDN which has improved the
  reliability and speed for end users. It is unlikely that any particular
  external host has something comparable. This can lead to extremely bad
  performance for end users when the external host is located in different
  parts of the world or does not generally have good connectivity.

  As a data point, many users reported sub DSL speeds and latency when
  accessing PyPI from parts of Europe and Asia prior to the use of the CDN.

* PyPI has monitoring and an on-call rotation of sysadmins whom can respond to
  downtime quickly, thus enabling a quicker response to downtime. Again it is
  unlikely that any particular external host will have this. This can lead
  to single packages in a dependency chain being un-installable. This will
  often confuse users, who often times have no idea that this package relies
  on an external host, and they cannot figure out why PyPI appears to be up
  but the installer cannot find a package.

* PyPI supports mirroring, both for private organizations and public mirrors.
  The legal terms of uploading to PyPI ensures that mirror operators, both
  public and private have the right to distribute the software found on PyPI.
  However software that is hosted externally does not have this, causing
  private organizations to need to investigate each package individually and
  manually to determine if the license allows them to mirror it.

  For public mirrors this essentially means that these externally hosted
  packages *cannot* be reasonably mirrored. This is particularly troublesome
  in countries such as China where the bandwidth to outside of China is
  highly congested making a mirror within China often times a massively better
  experience.

* Installers have no method to determine if they should expect any particular
  URL to be available or not. It is not unusual for the simple API to reference
  old packages and URLs which have long since stopped working. This causes
  installers to have to assume that it is OK for any particular URL to not be
  accessible. This causes problems where an URL is temporarily down or
  otherwise unavailable (a common cause of this is using a copy of Python
  linked against a really ancient copy of OpenSSL which is unable to verify
  the SSL certificate on PyPI) but it *should* be expected to be up. In this
  case installers will typically silently ignore this URL and later the user
  will get a confusing error stating that the installer couldn't find any
  versions instead of getting the real error message indicating that the URL
  was unavailable.

* In the long run, global opt in flags like ``--allow-all-external`` will
  become little annoyances that developers cargo cult around in order to make
  their installer work. When they run into a project that requires it they
  will most likely simply add it to their configuration file for that installer
  and continue on with whatever they were actually trying to do. This will
  continue until they try to install their requirements on another computer
  or attempt to deploy to a server where their install will fail again until
  they add the "make it work" flag in their configuration file.


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
