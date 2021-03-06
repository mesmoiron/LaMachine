# Adding new software to LaMachine

## Requirements and guidelines

* Only add relevant NLP software
* LaMachine is only intended for POSIX-compliant systems (i.e. Linux, BSD, Mac OS X).
  Other systems such as Windows are only supported as a host system for a VM.
* All software must be in public version control (we recommend *github*), be *public*, and be fully *open source* with
  an explicitly stated licence.
* LaMachine distinguishes between latest development versions and stable
  releases. If your software is mature enough to be released, please do so
  using the releases mechanism on github. Version numbers should use semantic versioning and start with a **v** (like
  v0.1.2; the ``major.minor.revision`` format is recommended)
* The latest state of the ``master`` branch of your repository will be considered the development version.
* If there is a suitable software repository for your software (such as the [Python Package
  Index](https://pypi.python.org) for Python, [CPAN](https://wwww.cpan.org) for
  Perl, [CRAN](https://www.cran.org) for R, [Maven](https://search.maven.org) for Java); use it to publish stable releases.
  LaMachine can in turn obtain it from these
  repositories.
* Your software should have some form of documentation, at least a decent README
* Any Python software should support Python 3 (and not just 2.7)
* The software should be maintained and should work on modern linux distributions.
  LaMachine is not intended for legacy or archiving purposes.

## Why use LaMachine?

Why should you want to participate in LaMachine to distribute your software?

* LaMachine comes in many flavours and is not just a containerisation solution (e.g. Docker). We use Ansible to offer
  a good and clear level of abstraction, allowing it to work with multiple technologies (Vagrant, Docker, local installation, global installation).
  It also removes the need to mess with Dockerfile or Vagrantfile yourself, as this is something LaMachine handles for
  you.
* LaMachine already offers a lot of NLP software; this may include software your tool depends on or software people like
  to use independently in combination with your tool.
* LaMachine supports multiple modern Linux distributions, offering flexibility rather than forcing a single one.
* LaMachine aims to provide a Virtual Research Environment
* From a user perspective, LaMachine is easy to install (one command bootstrap).
* LaMachine comes with an extensive test framework.
* LaMachine is extensively documented.
* LaMachine does not replace or reinvent existing technologies, but builds on them: Linux Distributions, Ansible, Vagrant, Docker, pip, virtualenv
    * This means LaMachine remains an optional solution to make things easier, but build on established installation methods that remain usable outside LaMachine.
* LaMachine handles software metadata

## How to contribute?

Contributors are expected to be familiar with git and github:

* Fork the LaMachine github repository
* Add a role for your software in ``roles/``  (read the rest of this documentation to learn how)
* When all done, create a pull request on Github

## LaMachine Architecture

LaMachine consists primarily of installation and configuration recipes for [Ansible](https://ansible.org). These recipes
are called *playbooks* by Ansible, or more specifically they are called *roles* when used in a modular fashion as we do.
Roles are defined in a simple [YAML syntax](http://docs.ansible.com/ansible/latest/YAMLSyntax.html) in combination with
a powerful [templating language](http://docs.ansible.com/ansible/latest/playbooks_templating.html).

A role or playbook defines *tasks*, each task is an installation or configuration
step. A task has a name and references a certain Ansible module based on the
type of work to do, there is for example a ``copy`` module to copy files onto
the destination system, a ``file`` module to create files/directories and set
proper owners ship, an ``include_role`` module that calls another role with
specific parameters (this is heavily used in LaMachine), and hundreds more.
A task may also define [a condition](http://docs.ansible.com/ansible/latest/playbooks_conditionals.html)
for its execution through a ``when`` statement.

LaMachine provides a full framework with various predefined *roles* and preset *variables*.  The framework allows for
installation in various forms (docker, VM, local, global). A single ``bootstrap.sh`` script is used build any desired
form; it does the necessary preprocessing and finally always invokes Ansible.  We will look into that first.

### The bootstrap process

The LaMachine bootstrap consists of a single bash script that is capable of downloading almost all necessary dependencies and
kicking of the installation process in whatever form the user choses. The script can be downloaded
and piped directly to the shell, allowing a *single command* to be sufficient to get everything downloaded and started.

By default, the script interactively queries the user for his
installation preferences and creates at least the following files:

 * **Installation Manifest** - (``install.yml``) - This is the main ansible playbook that contains the packages to install
   or roles to perform.
 * **Host Configuration File** - (``host_vars/$HOSTNAME.yml``) - This contains the
   configuration variables for your LaMachine installation.
 * If a Virtual Machine is chosen, a ``Vagrantfile`` will be generated.
 * If a Docker container is to be constructed, then the provided ``Dockerfile`` will be used.
 * A ``hosts.ini`` file may be generated, this is the Ansible inventory.

The bootstrap initially creates a directory in  ``lamachine-controller/`` (which in turn is created in the directory
where you run the bootstrap). This controller directory will hold a git clone of the LaMachine
repository with all the aforemention generated files. The installation manifest and host configuration file will initially be staged
in the current working directory and presented in a text editor a text editor so end-users can adapt these files to
their liking (both are heavily commented with instructions). Afterwards they are copied into the LaMachine controller.
In most cases, the LaMachine controller performs a temporary role during the bootstrap process, the controller will be
copied into the LaMachine environment, allowing the environment to be updatable; we call this the *internal* controller.
In some advanced situations however, you may prefer an *external* controller. This means the ``lamachine-controller``
directory is kept and used to updated a LaMachine installation from the outside. This is used for managing *remote*
systems directly using Ansible, but may also be useful in development settings.

Note that the bootstrap procedure can also be run in a non-interactive fashion and parameters can be passed on the command line,
see ``bootstrap.sh --help`` for details.

First we take a look at the variables available to LaMachine and defined in the host configuration file, afterwards we
look at the installation manifest and learn how to add software to LaMachine.

### Variables

The variables are generally set by the end-user in the LaMachine host configuration file
when building or updating LaMachine. These determine the type of environment to
build, as LaMachine offers quite some flexibility through its different
flavours, versions, and is meant to work on multiple Linux distributions.

We distinguish the following variables, all of which you can read and use in your Ansible roles for LaMachine:

* *Generic:*
  * ``version`` -- The version of LaMachine, is either ``stable``, ``development``, or ``custom``.
  * ``locality`` - Set to either ``local`` or ``global`` and determines whether LaMachine is installed in a local user environment (see ``localenv_type``) or globally on the system. For Virtual Machines and Docker Containers, this value will be set to global.
    * ``localenv_type`` - The type of local environment (only makes sense if ``locality == "local"``), can only be set to ``virtualenv`` for now, meaning we use a Python VirtualEnv.
* *Permissions:*
  * ``root``	 - A boolean indicating whether LaMachine has root permission on the system (necessarily true if ``locality == "global"``)
  * ``unix_user`` - The unix user that owns and runs LaMachine.
  * ``unix_group`` - The unix group that owns and runs LaMachine and has write access.
  * ``controller`` - Set to either ``internal`` or ``external``. Indicates whether the LaMachine installation manages itself (i.e. upgrades are done from within the LaMachine environment), or whether it is externally managed. Externally managed installation are useful in development environments or for provisioning of remote machines.
* *Paths:*
  * ``lm_prefix`` - The path where LaMachine installs its software. This equals to either:
    * ``local_prefix`` - The local installation path (i.e., the path of the virtual enviroment)
    * ``global_prefix`` - The global installation path (by default ``/usr/local``)
  * ``homedir`` - The path to the home directory for the user that owns and runs LaMachine
  * ``source_path`` - The path to where sources for packages will be downloaded
  * ``lm_path`` - The path to the LaMachine source repository (where are the ansible roles, templates and configurations are). This equals to either:
      * ``lamachine_path`` - The path upon first installation or remote location where the installation is managed (if controller == `external`)
      * ``source_path``/LaMachine - The path inside LaMachine (if controller == 'internal')
  * ``data_path`` - The path where the end-user can store data (this is typically shared with the host system, if applicable)
* *Network:*
  * ``hostname`` - The hostname of the system
  * ``webserver`` - (boolean) Include a webserver or not
    * ``webservertype`` - The type of webserver, defaults to ``nginx``. LaMAchine does not install any other webserver, so if you change this nginx won't be installed but you have to set up any alternative webserver (e.g. Apache) yourself.
  * ``http_port`` - port the webserver will listen on
  * ``web_user`` - The unix user that runs the webserver and webservices
  * ``web_group`` - The unix group that runs the webserver and webservices; ``unix_user`` should always be a member of this group
  * ``services`` - This is a list of services to provide, by default it is set to ``[ all ]``, meaning all services provided by the software categories you install will be enabled, you can remove ``all`` and provide only specific services.
* *Other:*
  * ``private`` - (boolean) Send basic analytics back to us
  * ``minimal`` - (boolean) A minimal installation is requested (might break some stuff)

### Directory layout

It is important to understand the directory layout used by LaMachine and to adhere to it when adding software yourself.
The variable ``lm_prefix`` is most important here, as it holds the base directory under which all software is installed.
For a global installation (in the Docker and Virtual Machine flavours for instance), this by default corresponds to
``/usr/local`` (don't let the word *local* confuse you here, we are still talking about a global installation). In a
local installation ``lm_prefix`` corresponds to the directory containing the virtual environment, which can be anywhere the user desires.

We try to follow the [Filesystem Hierarchy Standard](https://wiki.linuxfoundation.org/lsb/fhs) as much as possible:

 * ``{{lm_prefix}}/bin`` - Holds executable binaries other executable scripts; this will be added to your
   environment's ``$PATH`` upon activation
   * ``{{lm_prefix}}/bin/activate.d`` - Extra shell scripts for activating the environment for specific software, will be sourced automatically by the main LaMachine activation script
 * ``{{lm_prefix}}/lib`` - Holds shared libraries; this will be added to your environments library path.
   * ``{{lm_prefix}}/lib/python3.*/site-packages`` - Contains installed Python modules.
 * ``{{lm_prefix}}/include`` - Holds development header files
 * ``{{lm_prefix}}/src`` - Holds sources of the software, symlinks to ``{{source_path}}``
 * ``{{lm_prefix}}/opt`` - Holds optional application software, for which each application is stored in a single directory under this path. This is common for software that is not typically distributed in a more unix-like fashion.
 (e.g. Alpino, PICCL, Nextflow), but we also use it to symlink to certain software.
 * ``{{lm_prefix}}/share`` - Shared data files
 * ``{{lm_prefix}}/var`` - Holds variable files.
   * ``{{lm_prefix}}/var/log/nginx`` - Holds log files by the webserver.
   * ``{{lm_prefix}}/var/log/uwsgi`` - Holds log files for the individual uwsgi-powered (e.g. CLAM, Django) webservices/webapplications.
 * ``{{lm_prefix}}/etc`` - Holds configuration files
   * ``{{lm_prefix}}/etc/nginx/nginx.conf`` - Generic webserver configuration (in global installations this will symlink to the system-wide ``/etc/nginx``)
   * ``{{lm_prefix}}/etc/nginx/conf.d/*.conf`` - Configurations per webservice/webapplication.
   * ``{{lm_prefix}}/etc/uwsgi-emperor/vassals/`` - Holds individual uwsgi configuration files for uwsgi-powered webservices/webapplications
   * ``{{lm_prefix}}/etc/*.clam.yml`` - External configuration files for specific CLAM webservices

### Reusable Roles

We supply  *roles* that are built for reuse (they all start with
``lamachine-*``) and are meant to to install software from one or more external
repositories:

* ``lamachine-python-install`` - Install Python software
	* Expects a ``package`` variable that is a dictionary/map with the following fields:
      * ``github_user`` - The user/group that holds the github repository (used by the development version of LaMachine)
      * ``github_repo`` - The name of the github repository (used by the development version of LaMachine)
      * ``pip`` - The name of the package in the Python Package Index (used by the stable version of LaMachine)
* ``lamachine-package-install`` - Install Distribution packages
	* Expects a ``package`` variable that is a dictionary/map with one or more of the following fields:
      * ``debian`` - The package name for APT on debian/ubuntu/mint systems.
      * ``redhat`` - The package name for YUM on fedora/redhat/rhel/centos systems.
      * ``arch`` - The package name for Arch Linux
      * ``homebrew`` - The package name for Homebrew on Mac OS X
* ``lamachine-git-autoconf`` -  Install C++ software hosted in git and which makes use of the GNU autotools (autoconf/automake), i.e. software that follows the classic  ``./configure && make && make install`` paradigm.
	* Expects a ``package`` variable that is a dictionary/map with the following fields:
      * ``user`` - The user/group that holds the github repository (used by the development version of LaMachine)
      * ``repo`` - The name of the github repository (used by the development version of LaMachine)
      * You can also pass any of the variables used by ``lamachine-register``, as this will be called automatically.

Some lower-level roles:
* ``lamachine-git`` - Clone a particular git repository
* ``lamachine-run`` - Run a particular command in the LaMachine environment.
* ``lamachine-register`` -  Registers software metadata manually
    * Holds a ``metadata`` dictionary/map with the relevant metadata according to the  Codemeta model (Read the *metadata* section below).
    * Set ``codemeta`` to point to a specific ``codemeta.json`` file to load, any metadata supplied in ``metadata`` will be appended.
    * When using ``lamachine-python-install``, metadata registration is entirely automatic, as PyPI contains all relevant information already. So you never need ``lamachine-register``, you can again set a ``metadata`` map and/or ``codemeta`` field.
    * When using ``lamachine-git-autoconf``,  ``lamachine-register`` is automatically called and a few variables (name,identifier,version) can be pre-filled. Others you will need to provide explicitly if wanted (preferably in the upstream project in a ``codemeta.json`` file).
    * When using ``lamachine-git``,  ``lamachine-register`` is only called if you set ``do_registration: true``. Certain variables (name, version) can be pre-filled. Others you will need to provide explicitly if wanted.

The use of specific LaMachine roles is always preferred over the use of comparable generic ansible modules as the
LaMachine roles take care of a lot of specific things for you so it works in all environments. So use
``lamachine-python-install`` rather than Ansible's ``pip`` module, and ``lamachine-git`` rather than ansible's ``git``
or ``github`` module.

To add your own software, you add a *role* yourself which includes one (or more) of the above, with specific parameters, to do the
actual work. Your role, in turn, is referenced by the end-user who has final control over the installation playbook.
Rather than just installing a single piece of software, a role in LaMachine usually installs multiple inter-connected
software components and all their dependencies.

This may sound a bit cryptic still, so let's go through some examples:

## Examples

Learning by example is usually the most efficient. We provide somes examples below, but also want to encourage you to
just browse around in the ``roles/`` directory and see how some of the existing software packages are installed:

### Example: Python software

* For this example, we use a Python software package named *babelente*
* The source code is on github as https://github.com/proycon/babelente
* The software is released on the Python Package Index as *babelente*, meaning a simple ``pip install babelente`` is enough to
  install it and all dependencies.
* Fork the LaMachine github repository
* Git clone your fork
* Create a file  *roles/babelente/tasks/main.yml* (create the necessary directories) with the following contents:

```
 - name: Install Foobar
   include_role:
      name: lamachine-python-install
   vars:
      package:
         github_user: proycon
         github_repo: babelente
         pip: babelente
```

That's it, the ``lamachine-python-install`` role works in such a way that the stable version of LaMachine will use
``pip`` with PyPI, whilst the development version of LaMachine will draw from github directly and run ``python3 setup.py
install``. Note that this automatically covers any Python dependencies the package has declared.

### Example: Distribution packages

If a software package already commonly included in our supported Linux distributions, then we can pull straight from the
distribution's repository. To this end, we use the ``lamachine-package-install`` role. The following example
demonstrates an installation of the Tesseract OCR system, supported on different distributions:

```
 - name: Install Tesseract
   include_role:
      name: lamachine-package-install
   with_items:
      - { debian: tesseract-ocr, redhat: tesseract, arch: tesseract, homebrew: tesseract }
      - { debian: tesseract-ocr-eng, redhat: tesseract-langpack-eng, arch: tesseract-data-eng }
   loop_control:
       loop_var: package
```

Note that ``with_items`` and ``loop_control`` is a standard [Ansible looping construct](http://docs.ansible.com/ansible/latest/playbooks_loops.html). The role is called two times, and the
variable ``package`` is assigned the value provided in ``with_items``.

It is not always feasible to include all distributions and this it not obligatory, if a platform isn't mentioned then
nothing will be installed. This however may break the process, so you if you decide not to support a certain platform,
we encourage you to set up a task that produces an error if the platform is unsupported. This can be done as follows:

```
 - name: Check for unsupported OS
   fail:
     msg: "This software is not supported on Mac OS X or CentOS"
   when: ansible_distribution|lower == "macosx" or ansible_distribution|lower == "centos"
```

## Software Metadata

LaMachine attempts to register software metadata of all software it explicitly installs, the ``lamachine-register`` role
that is invoked by various installation procedures takes care of this. All software metadata is collected in a
single ``lamachine-registry.json`` file.
LaMachine uses the [Codemeta](https://codemeta.github.io) standard for software metadata,
ensuring a unified metadata format. Metadata from certain sources such as the Python Package Index or CRAN can be
automatically converted quite well.  Our philosophy is that software metadata should be in a simple format and either live right alongside the source code in a
version controlled repository (e.g. on github, bitbucket, etc), or be obtained from a software repository such as as the
Python Package Index, CRAN, CPAN, Maven Central and automatically converted to a unified format. This automatic
conversion is performed by [CodeMetaPy](https://github.com/proycon/codemetapy) or [CodeMetar](https://ropensci.github.io/codemetar/)), included in LaMachine.

You can, and are in fact encouraged to, place a ``codemeta.json`` file in the root of your source code repository, following the proper Codemeta specifications. If you do so, LaMachine
will automatically use this. Alternatively, you can augment metadata from within the ansible roles themselves if updating the upstream project
is not an option.

Within LaMachine, the ``lamachine-list`` allows users to access the available metadata from the command line (it will be serialised as YAML).

If you enabled and started the webserver in LaMachine, then you have access to a rich portal page giving an overview of
all installed software and providing access to any software with a web-based interface. This portal is powered by
[Labirinto](https://github.com/proycon/labirinto).

## Testing

If you are doing development on LaMachine, you will be working from a cloned LaMachine repository on a particular git
branch. We recommend to fork it on github and then clone your fork. Make sure you navigate to the root of the git
repository, and then simply invoke ``./bootstrap.sh`` to build a LaMachine build. LaMachine will simply reuse your git
repository for the controller environment rather than download and create a new one, allowing you to test your
additions.

Once you are ready, issue a pull request on https://github.com/proycon/LaMachine/ to merge your changed into the
``develop`` branch.

Various predefined tests are also available for multiple linux distributions, through vagrant and docker. The test
script is ``builds/test.py`` and the build are defined in ``builds/build.py``. Note that the former script references a particular
LaMachine git repository and branch, so may need to be adapted to fit your situation.
