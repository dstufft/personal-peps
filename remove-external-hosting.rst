PEP: 470
Title: Using Multi Index Support for External to PyPI Package File Hosting
Version: $Revision$
Last-Modified: $Date$
Author: Donald Stufft <donald@stufft.io>,
BDFL-Delegate: Richard Jones <richard@python.org>
Discussions-To: distutils-sig@python.org
Status: Draft
Type: Process
Content-Type: text/x-rst
Created: 12-May-2014
Post-History: 14-May-2014, 05-Jun-2014
Replaces: 438


Abstract
========

This PEP proposes a mechanism for project authors to register with PyPI an
external repository where their project's downloads can be located. This
information can than be includes as part of the simple API so that installers
can use it to tell users where the item they are attempting to install is
located and what they need to do to enable this additional repository. In
addition to adding discovery information to make explicit multiple repositories
easy to use, this PEP also deprecates and removes the implicit multiple
repository support which currently functions through directly or indirectly
linking offsite via the simple API. Finally this PEP also proposes deprecating
and removing the functionality added by PEP 438, particularly the additional
rel information and the meta tag to indicate the API version.

This PEP *does* not propose mandating that all authors upload their projects to
PyPI in order to exist in the index nor does it propose any change to the human
facing elements of PyPI.


Rationale
=========

Historically PyPI did not have any method of hosting files nor any method of
automatically retrieving installables, it was instead focused on providing a
central registry of names, to prevent naming collisions, and as a means of
discovery for finding projects to use. In the course of time setuptools began
to scrape these human facing pages, as well as pages linked from those pages,
looking for things it could automatically download and install. Eventually this
became the "Simple" API which used a similar URL structure however it
eliminated any of the extraneous links and information to make the API more
efficient. Additionally PyPI grew the ability for a project to upload release
files directly to PyPI enabling PyPI to act as a repository in additional to
an index.

This gives PyPI two equally important roles that it plays in the Python
ecosystem, that of index to enable easy discovery of Python projects and
central repository to enable easy hosting, download, and installation of Python
projects. Due to the history behind PyPI and the very organic growth it has
experienced the lines between these two roles are blurry, and this blurriness
has caused confusion for the end users of both of these roles and this has in
turn caused ire between people attempting to use PyPI in different capacities,
most often when end users want to use PyPI as a repository but the author wants
to use PyPI soley as an index.

By moving to using explict multiple repositories we can make the lines between
these two roles much more explicit and remove the "hidden" surprises caused
by the current implementation of handling people who do not want to use PyPI
as a repository. However simply moving to explicit multiple repositories is
a regression in discoverablity, and for that reason this PEP adds an extension
to the current simple API which will enable easy discovery of the specific
repository that a project can be found in.

PEP 438 attempted to solve this issue by allowing projects to explicitly
declare if they were using the repository features or not, and if they were
not, it had the installers classify the links it found as either "internal",
"verifiable external" or "unverifiable external". PEP 438 was accepted and
implemented in pip 1.4 (released on Jul 23, 2013) with the final transition
implemented in pip 1.5 (released on Jan 2, 2014).

PEP 438 was successful in bringing about more people to utilize PyPI's
repository features, an altogether good thing given the global CDN powering
PyPI providing speed ups for a lot of people, however it did so by introducing
a new point of confusion and pain for both the end users and the authors.


Why Additional Repositories?
----------------------------

The two common installer tools, pip and easy_install/setuptools, both support
the concept of additional locations to search for files to satisify the
installation requirements and have done so for many years. This means that
there is no need to "phase" in a new flag or concept and the solution to
installing a project from a repository other than PyPI will function regardless
of how old (within reason) the end user's installer is. Not only has this
concept existed in the Python tooling for some time, but it is a concept that
exists across languages and even extending to the OS level with OS package
tools almost universally using multiple repository support making it extremely
likely that someone is already familar with the concept.

Additionally, the multiple repository approach is a concept that is useful
outside of the narrow scope of allowing projects which wish to be included on
the index portion of PyPI but do not wish to utilize the repository portion
of PyPI. This includes places where a company may wish to host a repository
that contains their internal packages or where a project may wish to have
multiple "channels" of releases, such as alpha, beta, release candidate, and
final release.

Setting up an external repository is very simple, it can be achieved with
nothing more than a filesystem, some files to host, and any web server capable
of serving files and generating an automated index of directories (commonly
called "autoindex"). This can be as simple as:

::

    $ mkdir -p /var/www/index.example.com/
    $ mkdir -p /var/www/index.example.com/myproject/
    $ mv ~/myproject-1.0.tar.gz /var/www/index.example.com/myproject/
    $ twistd -n web --path /var/www/index.example.com/


Using this additional location within pip is also simple and can be included
on a per invocation, per shell, or per user basis. The next version of pip will
also include the ability to configure this on a per virtual environment or per
machine basis as well. This can be as simple as:

::

    $ pip install --extra-index-url https://index.example.com/ myproject
    $ PIP_EXTRA_INDEX_URL=https://pypi.example.com/ pip install myproject
    $ echo "[global]\nextra-index-url = https://pypi.example.com/" > ~/.pip/pip.conf
    $ pip install myproject


Why Not PEP 438 or Similar?
---------------------------

While the additional search location support has existed in pip and setuptools
for quite some time support for PEP 438 has only existed in pip since the 1.4
version, and still has yet to be implemented in setuptools. The design of
PEP 438 did mean that users still benefited for projects which did not require
external files even with older installers, however for projects which *did*
require those users are still silently being given either potentionally
unreliable or, even worse, unsafe files to download. This system is also unique
to Python as it arises out of the history of PyPI, this means that it is almost
certain that this concept will be foreign to most, if not all users, until they
encounter it while attempting to use the Python toolchain.

Additionally, the classification system proposed by PEP 438 has, in practice,
turned out to be extremely confusing to end users, so much so that it is a
position of this PEP that the situation as it stands is completely untenable.
The common pattern for a user with this system is to attempt to install a
project possibly get an error message (or maybe not if the project ever
uploaded something to PyPI but later switched without removing old files), see
that the error message suggests ``--allow-external``, they reissue the command
adding that flag most likely getting another error message, see that this time
the error message suggests also adding ``--allow-unverified``, and again issue
the command a third time, this time finally getting the thing they wish to
install.

This UX failure exists for several reasons.

1. If pip can locate any files at all for a project on the Simple API it will
   simply use that instead of attempting to locate anymore. This is generally
   the right thing to do any attempting to locate any more would erase a large
   part of the benefit of PEP 438. This means that if a project *ever* uploaded
   a file that matches what the user has requested for install that will be
   used regardless of how old it is.

2. PEP 438 makes an implicit assumption that most projects would either upload
   themselves to PyPI or would update themselves to directly linking to release
   files. While a large number of projects *did* ultimately decide to upload
   to PyPI, some of them did so only because the UX around what PEP 438 was so
   bad that they felt forced to do so. More concerning however, is the fact
   that very few projects have opted to directly and safely link to files and
   instead they still simply link to pages which must be scraped in order to
   find the actual files, thus rendering the safe variant
   (``--allow-external``) largely useless.

3. Even if an author wishes to directly linked to their files, doing so safely
   is non-obvious. It requires the inclusion of a MD5 hash (for historical
   reasons) in the hash of the URL. If they do not include this then their
   files will be considered "unverified".

4. PEP 438 takes a security centric view and disallows any form of a global
   opt in for unverified projects. While this is generally a good thing, it
   creates extremely verbose and repetive command invocations such as:

   ::

      $ pip install --allow-external myproject --allow-unverified myproject myproject
      $ pip install --allow-all-external --allow-unverified myproject myproject


Multiple Repository/Index Support
=================================

Installers SHOULD implement or continue to offer, the ability to point the
installer at multiple URL locations. The exact mechanisms for a user to
indicate they wish to use an additional location is left up to each indidivdual
implementation.

Additionally the mechanism discovering an installation candidate when multiple
repositories are being used is also up to each individual implementation,
however once configured an implementation should not discourage, warn, or
otherwise cast a negative light upon the use of a repository simply because it
is not the primary repository.

Currently both pip and setuptools implement multiple repository support by
using the best installation candidate it can find from either repository,
essentially treating it as if it were one large repository.


External Index Discovery
========================

One of the problems with using an additional index is one of discovery. Users
will not generally be aware that an additional index is required at all much
less where that index can be found. Projects can attempt to convey this
information using their description on the PyPI page however that excludes
people who discover their project organically through ``pip search``.

To support projects that wish to externally host their files and to enable
users to easily discover what additional indexes are required, PyPI will gain
the ability for projects to register external index URLs along with an
associated comment for each. These URLs will be made available on the simple
page however they will not be linked or provided in a form that older
installers will automatically search them.

This ability will take the form of a ``<meta>`` tag. The name of this tag must
be set to ``external-repository`` and the content will be a link to the location
of the external repository. An optional data-description attribute will convey
any comments or description that the author has provided.

An example would look something like:

::

    <meta name="external-repository" content="https://index.example.com/" data-description="Primary Repository">
    <meta name="external-repository" content="https://index.example.com/Ubuntu-14.04/" data-description="Wheels built for Ubuntu 14.04">


An external repository cannot be registered with PyPI while the project has any
files uploaded nor will any new files be permitted to be uploaded while the
external repository registration exists.

When an installer fetches the simple page for a project, if it finds this
additional meta-data and it cannot find any files for that project in it's
configured URLs then it should use this data to tell the user how to add one
or more of the additional URLs to search in. This message should include any
comments that the project has included to enable them to communicate to the
user and provide hints as to which URL they might want (e.g. if some are only
useful or compatible with certain platforms or situations). When the installer
has implemented the auto discovery mechanisms they should also deprecate any
of the mechanisms added for PEP 438 (such as ``--allow-external``) for removal
at the end of the deprecation period proposed by the PEP.

This feature *must* be added to PyPI prior to starting the deprecation and
removal process for the implicit offsite hosting functionality.


Deprecation and Removal of Link Spidering
=========================================

A new hosting mode will be added to PyPI. This hosting mode will be called
``pypi-only`` and will be in addition to the three that PEP 438 has already
given us which are ``pypi-explicit``, ``pypi-scrape``, ``pypi-scrape-crawl``.
This new hosting mode will modify a project's simple api page so that it only
lists the files which are directly hosted on PyPI and will not link to anything
else.

Upon acceptance of this PEP and the addition of the ``pypi-only`` mode, all new
projects will be defaulted to the PyPI only mode and they will be locked to
this mode and unable to change this particular setting. ``pypi-only`` projects
will still be able to register external index URLs as described above - the
"pypi-only" refers only to the download links that are published directly on
PyPI.

An email will then be sent out to all of the projects which are hosted only on
PyPI informing them that in one month their project will be automatically
converted to the ``pypi-only`` mode. A month after these emails have been sent
any of those projects which were emailed, which still are hosted only on PyPI
will have their mode set to ``pypi-only``.

After that switch, an email will be sent to projects which rely on hosting
external to PyPI. This email will warn these projects that externally hosted
files have been deprecated on PyPI and that in 6 months from the time of that
email that all external links will be removed from the installer APIs. This
email *must* include instructions for converting their projects to be hosted
on PyPI and *must* include links to a script or package that will enable them
to enter their PyPI credentials and package name and have it automatically
download and re-host all of their files on PyPI. This email *must also*
include instructions for setting up their own index page and registering that
with PyPI, including the fact that they can use pythonhosted.org as a host
for an index page without requiring them to host any additional infrastructure
or purchase a TLS certificate. This email must also contain a link to the Terms
of Service for PyPI as many users may have signed up a long time ago and may
not recall what those terms are.

Five months after the initial email, another email must be sent to any projects
still relying on external hosting. This email will include all of the same
information that the first email contained, except that the removal date will
be one month away instead of six.

Finally a month later all projects will be switched to the ``pypi-only`` mode
and PyPI will be modified to remove the externally linked files functionality.
At this point in time any installers should finally remove any of the
deprecated PEP 438 functionality such as ``--allow-external`` and
``--allow-unverified`` in pip.


Impact
======

The largest impact of this is going to be projects where the maintainers are
no longer maintaining the project, for one reason or another. For these
projects it's unlikely that a maintainer will arrive to set the external index
metadata which would allow the auto discovery mechanism to find it.

Looking at the numbers factoring out PIL (which has been special cased below)
the actual impact should be quite low, with it affecting just 3.8% of projects
which host any files only externally or 2.2% which have their latest version
hosted only externally.

6674 unique IP addresses have accessed the Simple API for these 3.8% of
projects in a single day (2014-09-30). Of those, 99.5% of them installed
something which could not be verified, and thus they were open to a Remote Code
Execution via a Man-In-The-Middle attack, while 7.9% installed something which
could be verified and only 0.4% only installed things which could be verified.


Projects Which Rely on Externally Hosted files
----------------------------------------------

This is determined by crawling the simple index and looking for installable
files using a similar detection method as pip and setuptools use. The "latest"
version is determined using ``pkg_resources.parse_version`` sort order and it
is used to show whether or not the latest version is hosted externally or only
old versions are.

============ ======= ================ =================== =======
\             PyPI    External (old)   External (latest)   Total
============ ======= ================ =================== =======
 **Safe**     43313   16               39                  43368
 **Unsafe**   0       756              1092                1848
 **Total**    43313   772              1131                45216
============ ======= ================ =================== =======


Top Externally Hosted Projects by Requests
------------------------------------------

This is determined by looking at the number of requests the
``/simple/<project>/`` page had gotten in a single day. The total number of
requests during that day was 10,623,831.

============================== ========
Project                        Requests
============================== ========
PIL                            63869
Pygame                         2681
mysql-connector-python         1562
pyodbc                         724
elementtree                    635
salesforce-python-toolkit      316
wxPython                       295
PyXML                          251
RBTools                        235
python-graph-core              123
cElementTree                   121
============================== ========


Top Externally Hosted Projects by Unique IPs
--------------------------------------------

This is determined by looking at the IP addresses of requests the
``/simple/<project>/`` page had gotten in a single day. The total number of
unique IP addresses during that day was 124,604.

============================== ==========
Project                        Unique IPs
============================== ==========
PIL                            4553
mysql-connector-python         462
Pygame                         202
pyodbc                         181
elementtree                    166
wxPython                       126
RBTools                        114
PyXML                          87
salesforce-python-toolkit      76
pyDes                          76
============================== ==========


PIL
---

It's obvious from the numbers below that the vast bulk of the impact come from
the PIL project. On 2014-05-17 an email was sent to the contact for PIL
inquiring whether or not they would be willing to upload to PyPI. A response
has not been received as of yet (2014-10-01) nor has any change in the hosting
happened. Due to the popularity of PIL this PEP also proposes that during the
deprecation period that PyPI Administrators will set the PIL download URL as
the external index for that project. Allowing the users of PIL to take
advantage of the auto discovery mechanisms although the project has seemingly
become unmaintained.


Rejected Proposals
==================

Keep the current classification system but adjust the options
-------------------------------------------------------------

This PEP rejects several related proposals which attempt to fix some of the
usability problems with the current system but while still keeping the
general gist of PEP 438.

This includes:

* Default to allowing safely externally hosted files, but disallow unsafely
  hosted.
* Default to disallowing safely externally hosted files with only a global
  flag to enable them, but disallow unsafely hosted.
* Continue on the suggested path of PEP 438 and remove the option to unsafely
  host externally but continue to allow the option to safely host externally.


These proposals are rejected because:

* The classification system introduced in PEP 438 in an entirely unique concept
  to PyPI which is not generically applicable even in the context of Python
  packaging. Adding additional concepts comes at a cost.

* The classification system itself is non-obvious to explain and to
  pre-determine what classification of link a project will require entails
  inspecting the project's ``/simple/<project>/`` page, and possibly any
  URLs linked from that page.

* The ability to host externally while still being linked for automatic
  discovery is mostly a historic relic which causes a fair amount of pain and
  complexity for little reward.

* The installer's ability to optimize or clean up the user interface is limited
  due to the nature of the implicit link scraping which would need to be done.
  This extends to the ``--allow-*`` options as well as the inability to
  determine if a link is expected to fail or not.

* The mechanism paints a very broad brush when enabling an option, while PEP
  438 attempts to limit this with per package options. However a project that
  has existed for an extended period of time may often times have several
  different URLs listed in their simple index. It is not unsusual for at least
  one of these to no longer be under control of the project. While an
  unregistered domain will sit there relatively harmless most of the time, pip
  will continue to attempt to install from it on every discovery phase. This
  means that an attacker simply needs to look at projects which rely on unsafe
  external URLs and register expired domains to attack users.

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
