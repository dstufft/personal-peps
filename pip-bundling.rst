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

This PEP proposes the inclusion of an explicit bootstrapping method to fetch
the common packaging tools as in use by the greater Python ecosystem. It also
proposes for these tools to be pre-installed in the distributions of software
provides on Python.org as well as recommendations to third party distributors
of Python to include, in a way reasonable for their distributions, these
common tools.

This PEP does not propose the inclusion of any tooling to the python standard
library nor does it propose any new tooling or standards.


Rationale
=========

Currently installing third party packages into a freshly installed Python
is not particularly easy. It requires users to know ahead of time what the
common tools are, where they can be found, and how to install them. The effect
of this is that third party packages tend to either document with the
assumption that the user has already gone through this process, add repetitive
instructions telling people how to install the installer, or to completely
forgo the use of dependencies to ease installation concerns for their users.

If a project has chosen to simply assume the user already has the tools
available it gives users attempting to follow those instructions an unhelpful
error message. This leaves them further confused about how to install that
package.

The duplicated nature of the instructions for the projects that choose to
document how to install the installer has caused there to be a large corpus
of these instructions, often times slightly different or out of date. This
leaves new users confused as to which set of instructions they should listen.

For the projects that simply decided to forgo dependencies in order to not
need to deal with the complexities of teaching their users how to use a
package manager this has left them to one of a few bad choices. Often they
end up "vendoring", or directly including the files from another package
into their own package. The other thing that they might do is forgo all
dependencies all together, choosing to utilize only what is in the standard
library.

The instructions themselves are difficult to be written in a cross platform
way. They typically involve invoking curl or wget and then invoking the
downloaded Python file to install the packages. Additionally these scripts
often times don't include many of the security checks available in the tools
themselves because of the need to be a single file.

Further more the new package standards such as `Wheel`_ do not have a built
in method for installing if downloaded manually such as
``python setup.py install``. The lack of a built in method for installation
makes having a simple ability to install a downloaded file more important
than previous to make it as simple as ``pip install downloaded-file.whl``.

Finally making it easier to install third party packages should ideally lessen
the desire to add anything that people might find useful to the standard
library and instead allow the maintainers of Python to more easily say that
a particular library belongs outside of the standard library as a third party
package.


Explicit Bootstrapping
======================

This proposes an additional module added to the standard library called
``getpip``. This module has a single function for it's public api called
``bootstrap()``. This function will bootstrap pip, and any dependencies of
pip, into the Python installation. It will include options for installing it
to the user packages as well as other install time options.

The ``getpip`` module is also directly executable using the
``python -m getpip`` syntax. This command line form will expose all of the
same arguments as the ``bootstrap()`` function and is expected to be the
primary method of invoking the bootstraping.

It is important to balance users who will be installing this file who have
network access and want the absolute latest version of pip with users who
are attempting to install offline and want to be able to install some files
that they have already downloaded previously. It is also important that we
do not attempt to reinvent the wheel, installing a package is a domain specific
action and is tricky to get right.

In order to solve the above issues I propose that we include inside the
``getpip`` module a copy of a package for pip and any dependency it might have.
The ``getpip`` module should then attempt to use a bundled copy of pip in
order to download and install pip and it's dependencies. If for any reason
pip is unable to use the network to download the packages it will fall back
to installing the bundled package files. Their should also be the ability
to force to either using network installation or networkless installation.


Which Software
--------------

The proposal is to bundle anything that is required to make the ``pip`` command
work and be able to install packages. Currently this is `pip`_ itself as well
as `setuptools`_ however there is a desire to make the setuptools dependency
of pip optional and have pip install setuptools itself as required.

Pip has been chosen as it is the third party installer with the largest number
of users. Alternative installers may still be used and the bundling of pip
should make it simpler for those alternative installers to bootstrap themselves
as well.


Updating the Bundled pip
------------------------

It is important that the version of pip used to fetch the pip to be installed,
as well as the bundled fall back package. The ``getpip`` module should be
ensured to be updated to the latest version of all packages it installs before
any official release. Ideally it should also be updated periodically with
inside of the source tree. As part of this PEP a script will be provided to
handle updating the bundled copies of software.


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
