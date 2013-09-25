PEP: 453
Title: Explicit bootstrapping of pip in Python installations
Version: $Revision$
Last-Modified: $Date$
Author: Donald Stufft <donald@stufft.io>,
        Nick Coghlan <ncoghlan@gmail.com>
BDFL-Delegate: Martin von Löwis
Status: Draft
Type: Process
Content-Type: text/x-rst
Created: 10-Aug-2013
Post-History: 30-Aug-2013, 15-Sep-2013, 18-Sep-2013, 19-Sep-2013, 23-Sep-2013


Abstract
========

This PEP proposes that the `pip`_ package manager be made available by
default when installing CPython and when creating virtual environments
using the standard library's ``venv`` module via the ``pyvenv`` command line
utility).

To clearly demarcate development responsibilities, and to avoid
inadvertently downgrading ``pip`` when updating CPython, the proposed
mechanism to achieve this is to include an explicit `pip`_ bootstrapping
mechanism in the standard library that is invoked automatically by the
CPython installers provided on python.org.

The PEP also strongly recommends that CPython redistributors and other Python
implementations ensure that ``pip`` is available by default, or
at the very least, explicitly document the fact that it is not included.


Proposal
========

This PEP proposes the inclusion of an ``ensurepip`` bootstrapping module in
Python 3.4, as well as in the next maintenance releases of Python 3.3 and
2.7.

This PEP does *not* propose making pip (or any dependencies) directly
available as part of the standard library. Instead, pip will be a
bundled application provided along with CPython for the convenience
of Python users, but subject to its own development life cycle and able
to be upgraded independently of the core interpreter and standard library.


Rationale
=========

Currently, on systems without a platform package manager and repository,
installing a third-party Python package into a freshly installed Python
requires first identifying an appropriate package manager and then
installing it.

Even on systems that *do* have a platform package manager, it is unlikely to
include every package that is available on the Python Package Index, and
even when a desired third-party package is available, the correct name in
the platform package manager may not be clear.

This means that, to work effectively with the Python Package Index
ecosystem, users must know which package manager to install, where to get
it, and how to install it. The effect of this is that third-party Python
projects are currently required to choose from a variety of undesirable
alternatives:

* Assume the user already has a suitable cross-platform package manager
  installed.
* Duplicate the instructions and tell their users how to install the
  package manager.
* Completely forgo the use of dependencies to ease installation concerns
  for their users.

All of these available options have significant drawbacks.

If a project simply assumes a user already has the tooling then beginning
users may get a confusing error message when the installation command
doesn't work. Some operating systems may ease this pain by providing a
global hook that looks for commands that don't exist and suggest an OS
package they can install to make the command work, but that only works
on systems with platform package managers (such as major Linux
distributions). No such assistance is available for Windows and
Mac OS X users. The challenges of dealing with this problem are a
regular feature of feedback the core Python developers receive from
professional educators and others introducing new users to Python.

If a project chooses to duplicate the installation instructions and tell
their users how to install the package manager before telling them how to
install their own project then whenever these instructions need updates
they need updating by every project that has duplicated them. This is
particular problematic when there are multiple competing installation
tools available, and different projects recommend different tools.

This specific problem can be partially alleviated by strongly promoting
``pip`` as the default installer and recommending that other projects
reference `pip's own bootstrapping instructions
<http://www.pip-installer.org/en/latest/installing.html>`__ rather than
duplicating them. However the user experience created by this approach
still isn't good (especially on Windows, where downloading and running
the ``get-pip.py`` bootstrap script with the default OS configuration is
significantly more painful than downloading and running a binary executable
or installer). The situation becomes even more complicated when multiple
Python versions are involved (for example, parallel installations of
Python 2 and Python 3), since that makes it harder to create and maintain
good platform specific ``pip`` installers independently of the CPython
installers.

The projects that have decided to forgo dependencies altogether are forced
to either duplicate the efforts of other projects by inventing their own
solutions to problems or are required to simply include the other projects
in their own source trees. Both of these options present their own problems
either in duplicating maintenance work across the ecosystem or potentially
leaving users vulnerable to security issues because the included code or
duplicated efforts are not automatically updated when upstream releases a new
version.

By providing a cross-platform package manager by default it will be easier
for users trying to install these third-party packages as well as easier
for the people distributing them as they should now be able to safely assume
that most users will have the appropriate installation tools available.
This is expected to become more important in the future as the Wheel_
package format (deliberately) does not have a built in "installer" in the
form of ``setup.py`` so users wishing to install from a wheel file will want
an installer even in the simplest cases.

Reducing the burden of actually installing a third-party package should
also decrease the pressure to add every useful module to the standard
library. This will allow additions to the standard library to focus more
on why Python should have a particular tool out of the box, and why it
is reasonable for that package to adopt the standard library's 18-24 month
feature release cycle, instead of using the general difficulty of installing
third-party packages as justification for inclusion.

Providing a standard installation system also helps with bootstrapping
alternate build and installer systems, such as ``setuptools``,
``zc.buildout`` and the ``hashdist``/``conda`` combination that is aimed
specifically at the scientific community. So long as
``pip install <tool>`` works, then a standard Python-specific installer
provides a reasonably secure, cross platform mechanism to get access to
these utilities.


Why pip?
--------

``pip`` has been chosen as the preferred default installer, as it
addresses several design and user experience issues with its predecessor
``easy_install`` (these issues can't readily be fixed in ``easy_install``
itself due to backwards compatibility concerns). ``pip`` is also well suited
to working within the bounds of a single Python runtime installation
(including associated virtual environments), which is a desirable feature
for a tool bundled with CPython.

Other tools like ``zc.buildout`` and ``conda`` are more ambitious in their
aims (and hence substantially better than ``pip`` at handling external
binary dependencies), so it makes sense for the Python ecosystem to treat
them more like platform package managers to inter operate with rather than
as the default cross-platform installation tool. This relationship is
similar to that between ``pip`` and platform package management systems
like ``apt`` and ``yum`` (which are also designed to handle arbitrary
binary dependencies).


Explicit bootstrapping mechanism
================================

An additional module called ``ensurepip`` will be added to the standard
library whose purpose is to install pip and any of its dependencies into the
appropriate location (most commonly site-packages). It will expose a
callable named ``bootstrap()`` as well as offer direct execution via
``python -m ensurepip``.

The bootstrap will *not* contact PyPI, but instead rely on a private copy
of pip stored inside the standard library. Accordingly, only options
related to the installation location will be supported (``--user``,
``--root``, etc).

It is considered desirable that users be strongly encouraged to use the
latest available version of ``pip``, in order to take advantage of the
ongoing efforts to improve the security of the PyPI based ecosystem, as
well as benefiting from the efforts to improve the speed, reliability and
flexibility of that ecosystem.

In order to satisfy this goal of providing the most recent version of
``pip`` by default, the private copy of ``pip`` will be updated in CPython
maintenance releases, which should align well with the 6-month cycle used
for new ``pip`` releases.


Security considerations
-----------------------

The design in this PEP has been deliberately chosen to avoid making any
significant changes to the trust model of the CPython installers for end
users that do not subsequently make use of ``pip``.

The installers will contain all the components of a fully functioning
version of Python, including the ``pip`` installer. The installation
process will *not* require network access, and will *not* rely on
trusting the security of the network connection established between
``pip`` and the Python package index.

Only users that choose to use ``pip`` directly will need to pay
attention to any PyPI related security considerations.


Implementation strategy
-----------------------

To ensure there is no need for network access when installing Python or
creating virtual environments, the ``ensurepip`` module will, as an
implementation detail, include a complete private copy of pip and its
dependencies which will be used to extract pip and install it into the target
environment. It is important to stress that this private copy of pip is
*only* an implementation detail and it should *not* be relied on or
assumed to exist beyond the public capabilities exposed through the
``ensurepip`` module (and indirectly through ``venv``).

There is not yet a reference ``ensurepip`` implementation. The existing
``get-pip.py`` bootstrap script demonstrates an earlier variation of the
general concept, but the standard library version would take advantage of
the improved distribution capabilities offered by the CPython installers
to include private copies of ``pip`` and ``setuptools`` as wheel files
(rather than as embedded base64 encoded data), and would not try to
contact PyPI (instead installing directly from the private wheel files.

Rather than including separate code to handle the bootstrapping, the
``ensurepip`` module will manipulate sys.path appropriately to allow
the wheel files to be used to install themselves, either into the current
Python installation or into a virtual environment (as determined by the
options passed to the bootstrap command).

It is proposed that the implementation be carried out in five separate
steps (all steps after the first are independent of each other and can be
carried out in any order):

* the first step would add the ``ensurepip`` module and the private copies
  of the most recently released versions of pip and setuptools, and update
  the "Installing Python Modules" documentation. This change
  would be applied to Python 2.7, 3.3 and 3.4.
* the Windows installer would be updated to offer the new ``pip``
  installation option for Python 2.7.6, 3.3.3 and 3.4.0.
* the Mac OS X installer would be updated to offer the new ``pip``
  installation option for Python 2.7.6, 3.3.3 and 3.4.0.
* the ``venv`` module and ``pyvenv`` command would be updated to make use
  of ``ensurepip`` in Python 3.4+
* the PATH handling and ``sysconfig`` directory layout on Windows would be
  updated for Python 3.4+


Proposed CLI
------------

The proposed CLI is based on a subset of the existing ``pip install``
options::

    Usage:
      python -m ensurepip [options]

    General Options:
      -h, --help          Show help.
      -v, --verbose       Give more output. Option is additive, and can be used up to 3 times.
      -V, --version       Show the pip version that would be extracted and exit.
      -q, --quiet         Give less output.

    Installation Options:
      -U, --upgrade       Upgrade pip and dependencies, even if already installed
      --user              Install using the user scheme.
      --root <dir>        Install everything relative to this alternate root directory.

In most cases, end users won't need to use this CLI directly, as ``pip``
should have been installed automatically when installing Python or when
creating a virtual environment.

Users that want to retrieve the latest version from PyPI, or otherwise
need more flexibility, should invoke the extracted ``pip`` appropriately.


Proposed module API
-------------------

The proposed ``ensurepip`` module API consists of the following two
functions::

    def version():
        """
        Returns a string specifying the bundled version of pip.
        """

    def bootstrap(root=None, upgrade=False, user=False, verbosity=0):
        """
        Bootstrap pip into the current Python installation (or the given root
        directory).
        """


Invocation from the CPython installers
--------------------------------------

The CPython Windows and Mac OS X installers will each gain a new option:

* Install pip (the default Python package management utility)?

This option will be checked by default.

If the option is checked, then the installer will invoke the following
command with the just installed Python::

    python -m ensurepip --upgrade

This ensures that, by default, installing or updating CPython will ensure
that the installed version of pip is at least as recent as the one included
with that version of CPython. If a newer version of pip has already been
installed then ``python -m ensurepip --upgrade`` will simply return without
doing anything.


Installing from source
----------------------

While the prebuilt binary installers will be updated to run
``python -m ensurepip`` by default, no such change will be made to the
``make install`` and ``make altinstall`` commands of the source
distribution.

``ensurepip`` itself (including the private copy of ``pip`` and its
dependencies) will still be installed normally (as it is a regular
part of the standard library), only the implicit installation of pip and
its dependencies will be skipped.

Keeping the pip bootstrapping as a separate step for ``make``-based
installations should minimize the changes CPython redistributors need to
make to their build processes. Avoiding the layer of indirection through
``make`` for the ``ensurepip`` invocation avoids any challenges
associated with determining where to install the extracted ``pip``.


Changes to virtual environments
-------------------------------

Python 3.3 included a standard library approach to virtual Python environments
through the ``venv`` module. Since its release it has become clear that very
few users have been willing to use this feature directly, in part due to the
lack of an installer present by default inside of the virtual environment.
They have instead opted to continue using the ``virtualenv`` package which
*does* include pip installed by default.

To make the ``venv`` more useful to users it will be modified to issue the
pip bootstrap by default inside of the new environment while creating it. This
will allow people the same convenience inside of the virtual environment as
this PEP provides outside of it as well as bringing the ``venv`` module closer
to feature parity with the external ``virtualenv`` package, making it a more
suitable replacement.

To handle cases where a user does not wish to have pip bootstrapped into
their virtual environment a ``--without-pip`` option will be
added.

The ``venv.EnvBuilder`` and ``venv.create`` APIs will be updated to accept
one new parameter: ``with_pip`` (defaulting to ``False``).

The new default for the module API is chosen for backwards compatibility
with the current behaviour (as it is assumed that most invocation of the
``venv`` module happens through third part tools that likely will not
want ``pip`` installed without explicitly requesting it), while the
default for the command line interface is chosen to try to ensure ``pip``
is available in most virtual environments without additional action on the
part of the end user.

This particular change will be made only for Python 3.4 and later versions.
The third-party ``virtualenv`` project will still be needed to obtain a
consistent cross-version experience in Python 3.3 and 2.7.


Documentation
-------------

The "Installing Python Modules" section of the standard library
documentation will be updated to recommend the use of the bootstrapped
`pip` installer. It will give a brief description of the most common
commands and options, but delegate to the externally maintained ``pip``
documentation for the full details.

The existing content of the module installation guide will be retained,
but under a new "Invoking distutils directly" subsection.


Bundling CA certificates with CPython
-------------------------------------

The ``ensurepip`` implementation will include the ``pip`` CA bundle along
with the rest of ``pip``. This means CPython effectively includes
a CA bundle that is used solely by ``pip`` after it has been extracted.

This is considered preferable to relying solely on the system
certificate stores, as it ensures that ``pip`` will behave the same
across all supported versions of Python, even those prior to Python 3.4
that cannot access the system certificate store on Windows.


Automatic installation of setuptools
------------------------------------

``pip`` currently depends on ``setuptools`` to handle metadata generation
during the build process, along with some other features. While work is
ongoing to reduce or eliminate this dependency, it is not clear if that
work will be complete for pip 1.5 (which is the version likely to be current
when Python 3.4.0 is released).

This PEP proposes that, if pip still requires it as a dependency,
``ensurepip`` will include a private copy of ``setuptools`` (in addition
to the private copy of ``ensurepip``). ``python -m ensurepip`` will then
install the private copy in addition to installing ``pip`` itself.

However, this behavior is officially considered an implementation
detail. Other projects which explicitly require ``setuptools`` must still
provide an appropriate dependency declaration, rather than assuming
``setuptools`` will always be installed alongside ``pip``.

Once pip is able to run ``pip install --upgrade pip`` without needing
``setuptools`` installed first, then the private copy of ``setuptools``
will be removed from ``ensurepip`` in subsequent CPython releases.


Updating the private copy of pip
--------------------------------

In order to keep up with evolutions in packaging as well as providing users
with as recent version a possible the ``ensurepip`` module will be
regularly updated to the latest versions of everything it bootstraps.

After each new ``pip`` release, and again during the preparation for any
release of Python (including feature releases), a script, provided as part
of this PEP, will be run to ensure the private copies stored in the CPython
source repository have been updated to the latest versions.


Updating the ensurepip module API and CLI
-----------------------------------------

Like ``venv`` and ``pyvenv``, the ``ensurepip`` module API and CLI
will be governed by the normal rules for the standard library: no
new features are permitted in maintenance releases.

However, the embedded components may be updated as noted above, so
the extracted ``pip`` may offer additional functionality in maintenance
releases.


Feature addition in maintenance releases
========================================

Adding a new module to the standard library in Python 2.7 and 3.3
maintenance releases breaks the usual policy of "no new features in
maintenance releases".

It is being proposed in this case as the current bootstrapping issues for
the third-party Python package ecosystem greatly affects the experience of
new users, especially on Python 2 where many Python 3 standard library
improvements are available as backports on PyPI, but are not included in
the Python 2 standard library.

By updating Python 2.7, 3.3 and 3.4 to easily bootstrap the PyPI ecosystem,
this change should aid the vast majority of current Python users, rather
than only those with the freedom to adopt Python 3.4 as soon as it is
released.


Uninstallation
==============

No changes are proposed to the uninstallation process by this PEP. The
bootstrapped pip will be installed the same way as any other pip
installed packages, and will be handled in the same way as any other
post-install additions to the Python environment.

At least on Windows, that means the bootstrapped files will be
left behind after uninstallation, since those files won't be associated
with the Python MSI installer.

While the case can be made for the CPython installers clearing out these
directories automatically, changing that behaviour is considered outside
the scope of this PEP.


Script Execution on Windows
===========================

While the Windows installer was updated in Python 3.3 to optionally
make ``python`` available on the PATH, no such change was made to
include the Scripts directory. Independently of this PEP, a proposal has
also been made to rename the ``Tools\Scripts`` subdirectory to ``bin`` in
order to improve consistency with the typical script installation directory
names on \*nix systems.

Accordingly, in addition to adding the option to extract and install ``pip``
during installation, this PEP proposes that the Windows installer (and
``sysconfig``) in Python 3.4 and later be updated to:

- install scripts to PythonXY\bin rather than PythonXY\Tools\Scripts
- add PythonXY\bin to the Windows PATH (in addition to PythonXY) when the
  PATH modification option is enabled during installation

For Python 2.7 and 3.3, it is proposed that the only change be the one
to bootstrap ``pip`` by default.

This means that, for Python 3.3, the most reliable way to invoke pip on
Windows (without tinkering manually with PATH) will actually be
``py -m pip`` (or ``py -3 -m pip`` to select the Python 3 version if both
Python 2 and 3 are installed) rather than simply calling ``pip``.

For Python 2.7 and 3.2, the most reliable mechanism will be to install the
standalone Python launcher for Windows and then use ``py -m pip`` as noted
above.

Adding the scripts directory to the system PATH would mean that ``pip``
works reliably in the "only one Python installation on the system PATH"
case, with ``py -m pip``, ``pipX``, or ``pipX.Y`` needed only to select a
non-default version in the parallel installation case (and outside a virtual
environment). This change should also make the ``pyvenv`` command substantially
easier to invoke on Windows, along with all scripts installed by ``pip``,
``easy_install`` and similar tools.

While the script invocations on recent versions of Python will run through
the Python launcher for Windows, this shouldn't cause any issues, as long
as the Python files in the Scripts directory correctly specify a Python version
in their shebang line or have an adjacent Windows executable (as
``easy_install`` and ``pip`` do).


Recommendations for Downstream Distributors
===========================================

A common source of Python installations are through downstream distributors
such as the various Linux Distributions [#ubuntu]_ [#debian]_ [#fedora]_, OSX
package managers [#homebrew]_ [#macports]_ [#fink]_, or Python-specific tools
[#conda]_. In order to provide a consistent, user-friendly experience to all
users of Python regardless of how they attained Python this PEP recommends and
asks that downstream distributors:

* Ensure that whenever Python is installed pip is also installed.

  * This may take the form of separate packages with dependencies on each
    other so that installing the Python package installs the pip package
    and installing the pip package installs the Python package.
  * Another reasonable way to implement this is to package pip separately but
    ensure that there is some sort of global hook that will recommend
    installing the separate pip package when a user executes ``pip`` without
    it being installed. Systems that choose this option should ensure that
    the ``pyvenv`` command still installs pip into the virtual environment
    by default, but may modify the ``ensurepip`` module in the system Python
    installation to redirect to the platform provided mechanism when
    installing ``pip`` globally.

* Do not remove the bundled copy of pip.

  * This is required for installation of pip into a virtual environment by the
    ``venv`` module.
  * This is similar to the existing ``virtualenv`` package for which many
    downstream distributors have already made exception to the common
    "debundling" policy.
  * This does mean that if ``pip`` needs to be updated due to a security
    issue, so does the private copy in the ``ensurepip`` bootstrap module
  * However, altering the private copy of pip to remove the embedded
    CA certificate bundle and rely on the system CA bundle instead is a
    reasonable change.

* Migrate build systems to utilize `pip`_ and `Wheel`_ instead of directly
  using ``setup.py``.

  * This will ensure that downstream packages can more easily utilize the
    new metadata formats which may not have a ``setup.py``.

* Ensure that all features of this PEP continue to work with any modifications
  made to the redistributed version of Python.

  * Checking the version of pip that will be bootstrapped using
    ``python -m ensurepip --version`` or ``ensurepip.version()``.
  * Installation of pip into a global or virtual python environment using
    ``python -m ensurepip`` or ``ensurepip.bootstrap()``.
  * ``pip install --upgrade pip`` in a global installation should not affect
    any already created virtual environments (but is permitted to affect
    future virtual environments, even though it will not do so when using
    the upstream version of ``ensurepip``).
  * ``pip install --upgrade pip`` in a virtual environment should not affect
    the global installation.

In the event that a Python redistributor chooses *not* to follow these
recommendations, we request that they explicitly document this fact and
provide their users with suitable guidance on translating upstream ``pip``
based installation instructions into something appropriate for the platform.

Other Python implementations are also encouraged to follow these guidelines
where applicable.


Policies & Governance
=====================

The maintainers of the bootstrapped software and the CPython core team will
work together in order to address the needs of both. The bootstrapped
software will still remain external to CPython and this PEP does not
include CPython subsuming the development responsibilities or design
decisions of the bootstrapped software. This PEP aims to decrease the
burden on end users wanting to use third-party packages and the
decisions inside it are pragmatic ones that represent the trust that the
Python community has already placed in the Python Packaging Authority as
the authors and maintainers of ``pip``, ``setuptools``, PyPI, ``virtualenv``
and other related projects.


Backwards Compatibility
-----------------------

The public API and CLI of the ``ensurepip`` module itself will fall under
the typical backwards compatibility policy of Python for its standard
library. The externally developed software that this PEP bundles does not.

Most importantly, this means that the bootstrapped version of pip may gain
new features in CPython maintenance releases, and pip continues to operate on
its own 6 month release cycle rather than CPython's 18-24 month cycle.


Security Releases
-----------------

Any security update that affects the ``ensurepip`` module will be shared
prior to release with the Python Security Response Team
(security@python.org). The PSRT will then decide if the reported issue
warrants a security release of CPython with an updated private copy of
``pip``.


Appendix: Rejected Proposals
============================


Automatically contacting PyPI when bootstrapping pip
----------------------------------------------------

Earlier versions of this PEP called the bootstrapping module ``getpip`` and
defaulted to downloading and installing ``pip`` from PyPI, with the private
copy used only as a fallback option or when explicitly requested.

This resulted in several complex edge cases, along with difficulties in
defining a clean API and CLI for the bootstrap module. It also significantly
altered the default trust model for the binary installers published on
python.org, as end users would need to explicitly *opt-out* of trusting
the security of the PyPI ecosystem (rather than opting in to it by
explicitly invoking ``pip`` following installation).

As a result, the PEP was simplified to the current design, where the
bootstrapping *always* uses the private copy of ``pip``. Contacting PyPI
is now always an explicit separate step, with direct access to the full
pip interface.


Implicit bootstrap
------------------

`PEP439`_, the predecessor for this PEP, proposes its own solution. Its
solution involves shipping a fake ``pip`` command that when executed would
implicitly bootstrap and install pip if it does not already exist. This has
been rejected because it is too "magical". It hides from the end user when
exactly the pip command will be installed or that it is being installed at
all. It also does not provide any recommendations or considerations towards
downstream packagers who wish to manage the globally installed pip through
the mechanisms typical for their system.

The implicit bootstrap mechanism also ran into possible permissions issues,
if a user inadvertently attempted to bootstrap pip without write access to
the appropriate installation directories.


Including pip directly in the standard library
----------------------------------------------

Similar to this PEP is the proposal of just including pip in the standard
library. This would ensure that Python always includes pip and fixes all of the
end user facing problems with not having pip present by default. This has been
rejected because we've learned, through the inclusion and history of
``distutils`` in the standard library, that losing the ability to update the
packaging tools independently can leave the tooling in a state of constant
limbo. Making it unable to ever reasonably evolve in a time frame that actually
affects users as any new features will not be available to the general
population for *years*.

Allowing the packaging tools to progress separately from the Python release
and adoption schedules allows the improvements to be used by *all* members
of the Python community and not just those able to live on the bleeding edge
of Python releases.

There have also been issues in the past with the "dual maintenance" problem
if a project continues to be maintained externally while *also* having a
fork maintained in the standard library. Since external maintenance of
``pip`` will always be needed to support earlier Python versions, the
proposed bootstrapping mechanism will becoming the explicit responsibility
of the CPython core developers (assisted by the pip developers), while
pip issues reported to the CPython tracker will be migrated to the pip
issue tracker. There will no doubt still be some user confusion over which
tracker to use, but hopefully less than has been seen historically when
including complete public copies of third-party projects in the standard
library.

The approach described in this PEP also avoids some technical issues
related to handling CPython maintenance updates when pip has been
independently updated to a more recent version. The proposed pip-based
bootstrapping mechanism handles that automatically, since pip and the
system installer never get into a fight about who owns the pip
installation (it is always managed through pip, either directly, or
indirectly via the ``ensurepip`` bootstrap module).

Finally, the separate bootstrapping step means it also easy to avoid
installing ``pip`` at all if end users so desire. This is often the case
if integrators are using system packages to handle installation of
components written in multiple languages using a common set of tools.


Defaulting to --user installation
---------------------------------

Some consideration was given to bootstrapping pip into the per-user
site-packages directory by default. However, this behavior would be
surprising (as it differs from the default behavior of pip itself)
and is also not currently considered reliable (there are some edge cases
which are not handled correctly when pip is installed into the user
site-packages directory rather than the system site-packages).


.. _Wheel: http://www.python.org/dev/peps/pep-0427/
.. _pip: http://www.pip-installer.org
.. _setuptools: https://pypi.python.org/pypi/setuptools
.. _PEP439: http://www.python.org/dev/peps/pep-0439/


References
==========

.. [1] Discussion thread 1 (distutils-sig)
   (https://mail.python.org/pipermail/distutils-sig/2013-August/022529.html)

.. [2] Discussion thread 2 (distutils-sig)
   (https://mail.python.org/pipermail/distutils-sig/2013-September/022702.html)

.. [3] Discussion thread 3 (python-dev)
   (https://mail.python.org/pipermail/python-dev/2013-September/128723.html)

.. [4] Discussion thread 4 (python-dev)
   (https://mail.python.org/pipermail/python-dev/2013-September/128780.html)

.. [#ubuntu] `Ubuntu <http://www.ubuntu.com/>`
.. [#debian] `Debian <http://www.debian.org>`
.. [#fedora] `Fedora <https://fedoraproject.org/>`
.. [#homebrew] `Homebrew <http://brew.sh/>`
.. [#macports] `MacPorts <http://macports.org>`
.. [#fink] `Fink <http://finkproject.org>`
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
