PEP: XXX
Title: Explicit Bootstrapping of pip and Bundling of pip with Python
Version: $Revision$
Last-Modified: $Date$
Author: Donald Stufft <donald@stufft.io>
BDFL-Delegate: Nick Coghlan <ncoghlan@gmail.com>
Discussions-To: distutils-sig@python.org
Status: Draft
Type: Process
Content-Type: text/x-rst
Created: 10-Aug-2013
Post-History: 04-Aug-2013


Abstract
========

This PEP proposed the inclusion of a method for explicitly bootstrapping pip
as the default package manager for Python. It also proposes that the
distributions of Python available via Python.org will automatically run this
explicit bootstrapping method and recommendations to third part distributors of
Python to include, in a way reasonable for their distributions, pip by default
as well.

This PEP does *not* proposes the inclusion of pip to the standard library
itself.


Rationale
=========

Installing a third party package into a freshly installed Python requires first
installing the package manager. This requires users ahead of time to know what
the package manager is, where to get them from, and how to install them. The
effect of this is that these external projects are required to either blindly
assume the user already has the package manager installed, needs to duplicate
the instructions and tell their users how to install the package manager, or
completely forgo the use of dependencies to ease installation concerns for
their users.

All of the available options have their own drawbacks.

If a project simply assumes a user already has the tooling then they get a
confusing error message when the installation command doesn't work. Some
operating may ease this pain by providing a global hook that looks for commands
that don't exist and suggest an OS package they can install to make the command
work.

If a project chooses to duplicate the installation instructions and tell their
users how to install the package manager before telling them how to install
their own project then whenever these instructions need updates they need
updated by every project that has duplicated them. This will inevitably not
happen in every case leaving many different instructions on how to install it
many of them broken or less than optimal. These additional instructions might
also confuse users who try to install the package manager a second time
thinking that it's part of the instructions of installing the project.

The projects that have decided to forgo dependencies all together are forced
to either duplicate the efforts of other projects by inventing their own
solutions to problems or are required to simply include the other projects
in their own source trees. Both of these options present their own problems
either in duplicating maintenance work across the ecosystem or potentially
leaving users vulnerable to security issues because the included code or
duplicated efforts are not automatically updated when upstream releases a new
version.

By providing the package manager by default it will be easier for users trying
to install these third party packages as well as easier for the people
distributing them as they no longer need to pick the lesser evil. This will
become more important in the future as the Wheel_ package format does not have
a built in "installer" in the form of ``setup.py`` so users wishing to install
a Wheel package will need an installer even in the simple case.

Reducing the burden of actually installing a third party package should also
decrease the pressure to add every useful module to the standard library. This
will allow additions to the standard library to focus more on why Python should
have a particular tool out of the box instead of needing to use the difficulty
in installing a package as justification for inclusion.


Explicit Bootstrapping
======================

An additional module called ``getpip`` will be added to the standard library
whose purpose is to install pip and any of its dependencies into the
appropriate location (most commonly site-packages). It will expose a single
callable named ``bootstrap()`` as well as offer direct execution via
``python -m getpip``. Options for installing it such as index server,
installation location (``--user``, ``--root``, etc) will also be available
to enable different installation schemes.

It is believed that users will want the most recent versions available to be
installed so that they can take advantage of the new advances in packaging.
Since any particular version of Python has a much longer staying power than
a version of pip in order to satisfy a user's desire to have the most recent
version the bootstrap will contact PyPI, find the latest version, download it,
and then install it. This process is security sensitive, difficult to get
right, and evolves along with the rest of packaging.

Instead of attempting to maintain a "mini pip" for the sole purpose of
installing pip the ``getpip`` module will, as an implementation detail, include
a private copy of pip which will be used to discover and install pip from PyPI.
It is important to stress that this private copy of pip is *only* an
implementation detail and it should *not* be relied on or assumed to exist.

Not all users will have network access to PyPI whenever they run the bootstrap.
In order to ensure that these users will still be able to bootstrap pip the
bootstrap will fallback to simply installing the included copy of pip.

This presents a balance between giving users the latest version of pip, saving
them from needing to immediately upgrade pip after bootstrapping it, and
allowing the bootstrap to work offline in situations where users might already
have packages downloaded that they wish to install.


Updating the Bundled pip
------------------------

In order to keep up with evolutions in packaging as well as providing users
who are using the offline installation method with as recent version as
possible the ``getpip`` module should be updates to the latest versions of
everything it bootstraps. During the preparation for any release of Python a
script, provided as part of this PEP, should be run to update the bundled
packages to the latest versions.


Pre-installation
================

During the installation of Python from Python.org ``python -m getpip`` should
be executed. Leaving people using the Windows or OSX installers with a working
copy of pip once the installation has completed. The exact method of this is
left up to the maintainers of the installers however if the bootstrapping is
optional it should be opt out rather than opt in and it should default to
allowing either networking or networkless installs.

While the installers distributed by Python itself will automatically attempt
to run ``python -m getpip``, the ``make install`` and ``make altinstall``
commands of the source distribution will not. This is because this PEP is
aimed at primarily helping new users as well as enabling existing users to
reduce some repetitive steps however the primary consumers of the source
distributions will be other distributors (such as Linux Distros) who will
desire to not have pip installed by default using ``make``. For anyone who
*is* actually using the source distribution from Python personally they
are unlikely to be a new user and ``python -mgetpip`` is still not a
particularly onerous requirement for someone advanced enough to install from
source.


Python Virtual Environments
===========================

Since Python 3.3 there has been a standard library approach to virtual
environments for Python inside of the ``venv`` module. However experience
has shown that very few users have begun to use this feature due to the lack
of an installer present inside of the virtual environment by default. Instead
users have continued to use the existing ``virtualenv`` package to create
virtual environments which *do* include pip pre-installed inside of them.

In order to make the ``venv`` module supremely more useful this PEP also
proposes that the creation of a virtual environment using the ``venv`` module
will cause the bootstrap script to be run as part of the creation process. This
will allow people the same convenience inside of a virtual environment as
outside of it and make the ``venv`` module a much better replacement for
``virtualenv``.


Reasons for a Private Bundled pip
=================================

This proposal includes a private copy of pip which is then used to either fetch
a copy of pip from `PyPI`_ or is used to install itself. An obvious question
would be why not simply include a public copy of pip and remove the need for
the ``python -m getpip`` all together?

* A typical version of Python far outlives a version of pip. For instance
  Python 2.6 is still in popular use today and it was released in August of
  2010 (or March of 2012 for source only releases). The version of pip
  available at that time was 0.8 (or 1.1 for the source only release) while the
  current version is 1.4.1. There have been numerous improvements to pip
  including several major security fixes. Installing the latest version allows
  people who are still attempting to install a particular version of Python
  far into the future to still get a newer version of pip automatically (if
  they have networking available) allowing us to further satisfy the goal of
  making it easy for packaging to evolve separately from the Python release
  schedule.

  The logical place to "hook" into installing pip is during the installation of
  Python itself as this is the time when everything else available in a default
  installation of Python is being installed.
* Given that we want to be able to install the latest available version of pip
  we need code to handle finding the latest version, downloading the latest
  version, and then installing the latest version. By including pip itself in
  order to handle these activities the Python standard library does not need to
  reinvent the Wheel, instead deferring the domain experts who are working on
  pip and centralizing the efforts there.
* Given that we also want to be able to install offline if a network connection
  is not available, we need to have an installable copy of pip available to a
  Python distribution to fall back to. The simplest method of doing this with
  the least amount of new code is to steal a page from `virtualenv`_'s book
  and simply include a pip package.



Recommendations for Other Distributors
======================================

A significant number of Python installations come from other sources such as
Linux Distributions [#ubuntu]_ [#debian]_ [#fedora]_, OSX Package Managers
[#homebrew]_, or even other python specific tools [#conda]_. In order to
provide a consistent experience for all Python users as well as to maintain
compatibility with upstream Python it is recommended that:

* Using whatever means makes sense for your users ensure that installing
  Python installs pip as well. For Linux distributions this could use the
  "Depends" or "Recommends" meta-data on Debian like systems.
* Do not remove the bundled copy of pip.
  * This is required for offline installation of pip into a virtual environment
  * A similar mechanism can be found inside the "virtualenv" package.
* Migrating build systems to utilize `pip`_ and `Wheel`_ where appropriate
  could be a very good idea.

Specifically this pep supports:

* Online installation of the latest version of pip into a global Python using
  ``python -m getpip``.
* Offline installation of the bundled version pip into a global Python using
  ``python -m getpip``.
* Automatic online installation of the latest version of pip into a virtual
  environment.
* Automatic offline installation of the bundled version of into a virtual
  environment.
* ``pip install --upgrade pip`` in a global installation should not affect any
  already created virtual environments.
* ``pip install --upgrade pip`` in a virtual environment should not affect the
  global installation.

Any changes made to Python by a distributor *SHOULD* support all of these
options.


Policies & Governance
=====================

The maintainers of the bundled software and the CPython core team will work
together in order to have a harmonious relationship. However the bundled
software remains external to CPython and does not fall under the governance
of CPython. The community has placed it's trust in the developers of this
software and the decision to bundle them is a pragmatic decision to make the
lives of developers simpler not one to have one project subsume another.


Backwards Compatibility
-----------------------

The ``getpip`` module itself will fall under the typical backwards
compatibility of Python. However the details of it's implementation and how
packages are discovered are not (due to the nature of evolving tools). The
externally bundled software such as pip do not fall under the banner of CPython
and thus does not fall under the backwards compatibility banner of Python.


Security Releases
-----------------

Any security update that affects the ``getpip`` module will be shared prior to
release with the PSRT. The PSRT will then decide if the issue inside warrants
a security release of Python.


Counter Points
==============


Implicit Bootstrap
------------------

`PEP439`_ proposes a solution to the same problem this PEP does. However
it's solution is that of an implicit bootstrap that would run the first time
a user attempted to invoke the ``pip`` command. This is a bad idea because
users cannot be sure when the installation of pip is occurring. This makes it
difficult to predict if they need network access or not nor does it provide any
no provisions for non network installs. A number of people have also raised
concerns about the "magic"-ness of the implicit bootstrap.


Including pip In the Standard Library
-------------------------------------

A simpler proposal would be to simply include pip as part of the standard
library and remove the need to bootstrap or bundle external software at all.
However this has a very serious side effect of removing the ability for pip
to easily evolve. Additionally by tying it into the standard library it is tied
to the release schedule of Python which would mean any improvements to
packaging could not be used for several years by the wider community.

Enabling the packaging tools to progress externally to Python enables
improvements in these areas that can be used by *all* of the Python community
members.


.. _Wheel: http://www.python.org/dev/peps/pep-0427/
.. _pip: http://www.pip-installer.org
.. _setuptools: https://pypi.python.org/pypi/setuptools
.. _PEP439: http://www.python.org/dev/peps/pep-0439/


References
==========

.. [#ubuntu] `Ubuntu <http://www.ubuntu.com/>`
.. [#debian] `Debian <http://www.debian.org>`
.. [#fedora] `Fedora <https://fedoraproject.org/>`
.. [#homebrew] `Homebrew  <http://brew.sh/>`
.. [#conda] `Conda <http://www.continuum.io/blog/conda>`


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
