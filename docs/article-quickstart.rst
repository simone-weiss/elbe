************************
ELBE Quickstart
************************

This guide explains how to build a root filesystem using ELBE from an XML
file, and how to make basic customizations.

====================
Required steps
====================

1. If not installed already, install **Debian 12 (bookworm)** or later on your
   host. The latest ELBE version should be compatible with this system.

2. Install **ELBE** on the host system.

3. Generate the **initvm** — a dedicated virtual machine that runs the build
   environment.

4. Build the root filesystem inside the initvm.

Steps **1–3 only need to be performed once** unless the environment is recreated.

.. note::
   If Debian is running inside a virtual machine (VMware, etc.), ensure that
   nested KVM is enabled. Without hardware virtualization support,
   builds will be significantly slower.

.. note::
   As of Windows 11 24H2, ELBE can also be used under WSL2 by installing
   Debian Bookworm from the Microsoft Store.
.. _installation:

====================
Installing ELBE
====================

There are two ways to install ELBE:

1. install prebuilt Debian packages from Linutronix, or
2. build ELBE from the upstream git repository.

-------------------------
Binary Debian Packages
-------------------------

Repository
~~~~~~~~~~

The latest ELBE packages are available from:

::

   http://debian.linutronix.de/elbe

This repository can be enabled using the `extrepo` utility:

::

   # apt install extrepo
   # extrepo enable elbe

Repository (manual setup)
~~~~~~~~~~~~~~~~~~~~~~~~~

If `extrepo` is not working for you, configure the repository manually.

Install the repository key to a known place (as root):

::

   # apt install elbe-archive-keyring || \
     wget -O /usr/share/keyrings/elbe-archive-keyring.gpg \
     http://debian.linutronix.de/elbe/elbe-repo.pub.gpg

Add the repository to /etc/apt/sources.list:

::

   # echo "deb [signed-by=/usr/share/keyrings/elbe-archive-keyring.gpg] \
     http://debian.linutronix.de/elbe bookworm main" \
     >> /etc/apt/sources.list

Install ELBE (as root):

::

   # apt update
   # apt install elbe

============================================
Creating the initvm and Submitting XML Files
============================================

The **initvm** is a virtual machine used exclusively by ELBE to build
root filesystems. It should match the host architecture so that hardware
virtualization (KVM) can be used.

To allow your user to manage VMs:

::

   # adduser <youruser> kvm
   # adduser <youruser> libvirt

Then create the initvm:

::

   $ mkdir initvm
   $ cd initvm
   $ elbe initvm create

======================
Submitting an XML file
======================

Submitting an XML file triggrts an image inside the initvmi. Once
the initvm has been created and is running, you can submit XML files
using:

::

   $ elbe initvm submit examples/x86_64-pc-rescue-busybox-dyn-cpio.xml
    Build started, waiting till it finishes
    [INFO] Build started
    [INFO] ELBE Report for Project x86_64-rescue-image
    Report timestamp: 20191001-135512
    [CMD] reprepro --basedir "/var/cache/elbe/63e09968-c9e7-45d8-8dd2-82c1a8f54f8d/repo" export stretch
    [CMD] mkdir -p "/var/cache/elbe/63e09968-c9e7-45d8-8dd2-82c1a8f54f8d/chroot"
    [INFO] Debootstrap log
    [CMD] dpkg --print-architecture
    [CMD] debootstrap  --include="gnupg" --arch=amd64 "stretch" "/var/cache/elbe/63e09968-c9e7-45d8-8dd2-82c1a8f54f8d/chroot" "http://ftp.de.debian.org//debian"
    I: Retrieving InRelease
    I: Retrieving Release
    I: Retrieving Release.gpg
    I: Checking Release signature
    I: Valid Release signature (key id 067E3C456BAE240ACEE88F6FEF0F382A1A7B6500)
    I: Retrieving Packages
    I: Validating Packages
    I: Resolving dependencies of required packages...
    I: Resolving dependencies of base packages...
    I: Checking component main on http://ftp.de.debian.org//debian...
    I: Retrieving libacl1 2.2.52-3+b1
    I: Validating libacl1 2.2.52-3+b1

    ...
     79.05% done, estimate finish Mon Aug  1 09:53:26 2022
     81.77% done, estimate finish Mon Aug  1 09:53:26 2022
     84.49% done, estimate finish Mon Aug  1 09:53:26 2022
     87.22% done, estimate finish Mon Aug  1 09:53:26 2022
     89.95% done, estimate finish Mon Aug  1 09:53:26 2022
     92.67% done, estimate finish Mon Aug  1 09:53:27 2022
     95.39% done, estimate finish Mon Aug  1 09:53:27 2022
     98.12% done, estimate finish Mon Aug  1 09:53:27 2022
    Total translation table size: 0
    Total rockridge attributes bytes: 73534
    Total directory bytes: 355120
    Path table size(bytes): 2354
    Max brk space used bd000
    183454 extents written (358 MB)
    [INFO] Build finished successfully

    Build finished !

    ELBE Package validation
    =======================

    Package List validation
    -----------------------

    No Errors found
    Binary CD
    Source CD

    Getting generated Files

    Saving generated Files to elbe-build-20220801-095330
    source.xml              (Current source.xml of the project)
    rescue.cpio             (Image)
    licence-chroot.txt      (License file)
    licence-chroot.xml      (xml License file)
    licence-target.txt      (License file)
    licence-target.xml      (xml License file)
    validation.txt          (Package list validation result)
    elbe-report.txt         (Report)
    log.txt                 (Log file)
    bin-cdrom.iso           (Repository IsoImage)
    src-cdrom-target.iso    (Repository IsoImage)
    src-cdrom-main.iso      (Repository IsoImage)
    src-cdrom-added.iso     (Repository IsoImage)

The result of the build is stored in elbe-build-<TIMESTAMP> below your
current working directory.

====================================
Ports Opened by the initvm
====================================

The initvm will open:

- **localhost:7587** — used by ELBE on your host to communicate with the initvm.

==========================
Customization of the Build
==========================

The ELBE XML can contain an **archive** element which includes configuration
files, and additional software. This archive is unpacked onto the target image
during the buildprocess. It allows you override any file provided from the default
Debian Install.

This section shows how to:

- extract the archive from the XML file,
- repack the archive back into the XML,
- apply customizations using `<finetuning>` rules.

Finetuning allows modifying the generated root filesystem to among other things:

- add users,
- remove files,
- adjust permissions,
- execute commands during the build.


---------------------
ELBE Archive
---------------------

An ELBE XML can include one or more `archivedir` entries:

.. code:: xml

   <archivedir>foo</archivedir>

Content from archivedirs is copied into the target filesystem during build.
If multiple archivedirs are specified, later ones override conflicting files.

Example:

.. code:: xml

   <archivedir>foo</archivedir>
   <archivedir variant="production">bar</archivedir>

--------------------------------------------
Adding packages to the installation List
--------------------------------------------

Packages are listed inside `<pkg-list>` under the `<target>` XML node.
To install e.g. util-linux to the target-rfs:

.. code:: xml

   <pkg>util-linux</pkg>

--------------------------
Using Finetuning Rules
--------------------------

An ELBE XML file can contain a set of finetuning rules. Finetuning is
used to customize the target-rfs, e.g. remove man-pages. Here is an
example finetuning from
``/usr/share/doc/elbe-doc/examples/elbe-desktop.xml``:

.. code:: xml

   <finetuning>
       <rm>var/cache/apt/archives/*.deb</rm>
       <adduser passwd="elbe" shell="/bin/bash">elbe</adduser>
   </finetuning>

rm
~~

The ``<rm>`` node removes files from the target-rfs.

adduser
~~~~~~~

The adduser node allows to create a user. The following example creates
the user ``elbe`` with the password ``foo``.

It is also possible to specify groups the new user should be part of:

.. code:: xml

   <adduser passwd="foo" shell="/bin/bash" groups="audio,video,dialout">elbe</adduser>

Instead of specifying a plain-text password, it is also possible to use
hashed passwords in the XML. Hashed passwords can be either converted by
the Elbe preprocessing (``elbe preprocess <xml>``), with the tool
``mkpasswd`` or with various hashing libraries like crypt (C/C++) or
passlib (Python).

In this example, the command ``mkpasswd`` is used to hash the plain-text
password ``elbe``. If the salt is not specified, ``mkpasswd`` will use a
random salt.

::

   mkpasswd --method=sha512crypt --rounds=656000 --salt=7vWuOPVX0YKaISh5 "elbe"

The generated line contains the hashing parameters and the hashed
password and has to be copied completely to the ``passwd_hashed``
attribute in the XML.

.. code:: xml

   <adduser passwd_hashed="$6$rounds=656000$7vWuOPVX0YKaISh5$cJhevq/z7kJ215n18dnksv/zOeUf6uPoLgICwLeTSu/2xoLHkyYQABaM7a99sQmpilCV.SlK9jfHZz3m7/s2a."
            shell="/bin/bash">elbe</adduser>

------------------------------------------
Changing ownership of directories or files
------------------------------------------

There is currently no special finetuning node for ``chmod`` and
``chown``. These commands needs to be specified via the command tag,
which allows running any command that is available in the target-rfs.

.. code:: xml

   <command>chown elbe:elbe /mnt</command>
   <command>chmod 777 /mnt</command>

Further Example
~~~~~~~~~~~~~~~

A more complete example can be found in the ELBE overview document that
is installed at ``/usr/share/doc/elbe-doc/elbeoverview-en.html``.

===============================
Using the Elbe Pbuilder Feature
===============================

Since Version 1.9.2, elbe is able to create a pbuilder Environment. You
can create a pbuilder for a specific xml File inside the initvm.

The repositories and architecture specified in the xml File will be used
to satisfy build dependencies. It is possible to crosscompile packages
for a foreign architecture. To do so use the *elbe pbuilder create*
command with the --cross option. This will setup the right environment
for crosscompiling. To use this environment you have to use the --cross
option with the build command. (If the environment was created with the
--cross option, the build command must be used with --cross too.
Otherwise it will throw an error.) By creating an environment the
compiler cache ``ccache`` gets installed by default to speed up
recompilations. It is possible to change the size or to deactivate it if
it is not needed. Pbuilder will only build debianised Software.

A pbuilder instance is always associated with a project inside the
initvm. The ``pbuilder create`` command will write the project uuid to a
file, if instructed to do so.

``pbuilder build`` works like ``pdebuild``, in that it uploads the
current working directory into the initvm pbuilder project, and then
builds it using the pbuilder instance created earlier.

Here is an example:

::

   $ elbe pbuilder create --xmlfile examples/x86_64-pc-rescue-busybox-dyn-cpio.xml --writeproject ../pbuilder.prj
   $ git clone https://github.com/Linutronix/libgpio.git
   $ cd libgpio/
   $ elbe pbuilder build --project `cat ~/repos/elbe/pbuilder.prj` --out ../out/

With these steps, elbe builds the libgpio project inside the initvm and
stores the built packages in an internal repository. Every package,
built in this manner, will also be stored in that repository. This
repository can be used for later RFS builds.

List contents of the repository with the following command:

::

   $ elbe prjrepo list_packages `cat ~/repos/elbe/pbuilder.prj`
   libgpio-dev_3.0.0_amd64.deb
   libgpio1_3.0.0_amd64.deb
   libgpio1-dbgsym_3.0.0_amd64.deb

To use this repository for further RFS builds download the repo with:

::

   $ elbe prjrepo download `cat ~/repos/elbe/pbuilder.prj`

The repository is download as elbe-projectrepo-20191002-114244.tar.gz.
This should be unpacked in the DocumentRoot of your webserver and
customized with your key as explained in the next chapter.

Here is an example for crosscompiling a linux kernel with debian
profiles:

::

   $ elbe pbuilder --cross create --xmlfile examples/armhf-ti-beaglebone-black.xml --writeproject pbuilder.prj
   $ apt source linux
   $ cd linux*/
   $ ../elbe pbuilder --cross --origfile ../linux*.orig.tar.xz --profile nodoc,nopython build --project `cat ../pbuilder.prj`

===================
Custom Repository
===================

You might have your own packages which should be installed into your
image. This can be done with a custom repository. You can use
`reprepro <https://mirrorer.alioth.debian.org/>`__ to create your own
repository or the above mentioned pbuilder feature.

--------------
Repository Key
--------------

Because the repository needs to be signed using ``gpg``, a key needs to
be generated.

::

   -> gpg --default-new-key-algo rsa4096 --gen-key
   gpg (GnuPG) 2.1.18; Copyright (C) 2017 Free Software Foundation, Inc.
   This is free software: you are free to change and redistribute it.
   There is NO WARRANTY, to the extent permitted by law.

   Note: Use "gpg --full-generate-key" for a full featured key generation dialog.

   GnuPG needs to construct a user ID to identify your key.

   Real name: Torben Hohn
   Email address: torben.hohn@linutronix.de
   You selected this USER-ID:
       "Torben Hohn <torben.hohn@linutronix.de>"

   Change (N)ame, (E)mail, or (O)kay/(Q)uit? O
   We need to generate a lot of random bytes. It is a good idea to perform
   some other action (type on the keyboard, move the mouse, utilize the
   disks) during the prime generation; this gives the random number
   generator a better chance to gain enough entropy.
   gpg: key 68E68615BB6CB47C marked as ultimately trusted
   gpg: directory '/home/torbenh/.gnupg/openpgp-revocs.d' created
   gpg: revocation certificate stored as '/home/torbenh/.gnupg/openpgp-revocs.d/CF837F1AAAC35E084062AE4468E68615BB6CB47C.rev'
   public and secret key created and signed.

   Note that this key cannot be used for encryption.  You may want to use
   the command "--edit-key" to generate a subkey for this purpose.
   pub   rsa4096 2018-10-08 [SC] [expires: 2020-10-07]
         CF837F1AAAC35E084062AE4468E68615BB6CB47C
         CF837F1AAAC35E084062AE4468E68615BB6CB47C
   uid                      Torben Hohn <torben.hohn@linutronix.de>

Please note the keyname (here
``CF837F1AAAC35E084062AE4468E68615BB6CB47C``). This keyname can then be
used to export the public key into a repo.pub file.

::

   gpg --export --armor CF837F1AAAC35E084062AE4468E68615BB6CB47C > repo.pub

----------------------
reprepro configuration
----------------------

To create your own repository with reprepro or the elbe pbuilder feature
you need only the ``distributions`` configuration file. For an ``amd64``
and ``source`` repository for Debian ``stretch`` it might look as
follows:

::

   Origin: mylocal
   Label: mylocal
   Suite: stable
   Codename: stretch
   Architectures: amd64 source
   Components: main
   Description: my local repo
   SignWith: CF837F1AAAC35E084062AE4468E68615BB6CB47C

.. note::
   the ``SignWith:`` field needs to be the key of the previously
   generated key.

Now place the ``distributions`` file in a ``conf`` named directory. also
put ``repo.pub`` into your ``repo`` directory.

::

   repo/
   ├── conf
   │   └── distributions
   └── repo.pub

---------------------
insert pkgs into repo
---------------------

To include packages in your repository you might use the following
command from inside the ``repo`` directory:

::

   $ reprepro include stretch ../path/to/your/*.changes

To use this repository from ELBE you need a webserver. Simply place the
repository inside the document root of your webserver.

If the webserver is running on the same machine as the initvm you can
use the following to access the repository:

.. code:: xml

   <url-list>
       <url>
           <binary>http://LOCALMACHINE/repo/ bookworm main</binary>
           <source>http://LOCALMACHINE/repo/ bookworm main</source>
           <key>http://LOCALMACHINE/repo/repo.pub</key>
       </url>
   </url-list>

ELBE replaces the string ``LOCALMACHINE`` with the ip address of your
machine. If you use an external machine as webserver you need to replace
``LOCALMACHINE`` with the name or the ip of it.

Now you can install packages from your custom repository the same way
you can install from any other repository.
