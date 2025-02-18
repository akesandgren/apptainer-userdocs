#####################
 Persistent Overlays
#####################

Persistent overlay directories allow you to overlay a writable file
system on an immutable read-only container for the illusion of
read-write access. You can run a container and make changes, and these
changes are kept separately from the base container image.

**********
 Overview
**********

A persistent overlay is a directory or file system image that "sits on
top" of your immutable SIF container. When you install new software or
create and modify files the overlay will store the changes.

If you want to use a SIF container as though it were writable, you can
create a directory, an ext3 file system image, or embed an ext3 file
system image in SIF to use as a persistent overlay. Then you can specify
that you want to use the directory or image as an overlay at runtime
with the ``--overlay`` option, or ``--writable`` if you want to modify
the overlay embedded in SIF.

If you want to make changes to the image, but do not want them to
persist, use the ``--writable-tmpfs`` option. This stores all changes in
an in-memory temporary filesystem which is discarded as soon as the
container finishes executing.

You can use persistent overlays with the following commands: ``run``,
``exec``, ``shell``, and ``instance start``.

*******
 Usage
*******

File system image overlay
=========================

{Project} provides a command ``{command} overlay
create`` to create persistent overlay images. You can create a single
EXT3 overlay image or adding a EXT3 writable overlay partition to an
existing SIF image.

.. note::

   ``dd`` and ``mkfs.ext3`` must be installed on your system.
   Additionally ``mkfs.ext3`` must support ``-d`` option in order to
   create an overlay directory tree usable by a regular user.

For example, to create a 1 GiB overlay image:

.. code::

   $ {command} overlay create --size 1024 /tmp/ext3_overlay.img

To add a 1 GiB writable overlay partition to an existing SIF image:

.. code::

   $ {command} build ubuntu.sif library://ubuntu
   ...
   $ {command} overlay create --size 1024 ubuntu.sif

.. warning::

   It is not possible to add a writable overlay partition to a
   **signed**, **encrypted** SIF image or if the SIF image already
   contain a writable overlay partition.

``{command} overlay create`` also provides an option ``--create-dir``
to create additional directories owned by the calling user, it can be
specified multiple times to create many directories. This is
particularly useful when you need to make a directory writable by your
user.

So for example:

.. code::

   $ {command} build /tmp/nginx.sif docker://nginx
   ...
   $ {command} overlay create --size 1024 --create-dir /var/cache/nginx /tmp/nginx.sif
   $ echo "test" | {command} exec /tmp/nginx.sif sh -c "cat > /var/cache/nginx/test"

Fakeroot with overlay
=====================

If you want to be able to modify the container with an overlay
(including with ``--writable-tmpfs``) you will generally want to run it
either as root or with ``--fakeroot`` because usually containers are
modifiable only by root. 

If that is the way you plan to use the image, then when creating the
filesystem image with ``overlay create`` also give it a ``--fakeroot``
option.

For example:

.. code::

   $ {command} build ubuntu.sif docker://ubuntu
   ...
   $ {command} overlay create --fakeroot --size 1024 overlay.img
   $ {command} shell --fakeroot --overlay overlay.img ubuntu.sif
   {Project}> which vim
   {Project}> apt-get update && apt-get install -y vim
   ...
   {Project}> which vim
   /usr/bin/vim

An exception is if you are using the 4th :ref:`fakeroot mode <fakeroot>`
with a setuid installation and no unprivileged user namespaces available. 
In that case the ``--fakeroot`` option to ``overlay create`` makes 
the overlay image unwritable, so leave it out.
This case also has other restrictions in that it only works when the
underlying image is a sandbox directory, and yet the overlay itself must
not be a directory.

Directory overlay
=================

A directory overlay is simpler to use than a filesystem image overlay.
On the other hand, a directory of modifications to a base container image
cannot be transported or shared as easily as a single overlay file,
and it generally does not work well on network file servers
(see the `NFS <{admindocs}/installation.html#nfs>`_ and
`Lustre / GPFS <{admindocs}/installation.html#lustre-gpfs>`_
sections of the admin guide).
It is supported, however, and this section describes how to use it.

.. note::

   For security reasons, with setuid mode only root may use a bare
   directory as an overlay.
   If unprivileged user namespaces are also available, however, the
   ``--userns`` or ``--fakeroot`` options should make it work.

Create a directory as usual:

.. code::

   $ mkdir my_overlay

The example below shows the directory overlay in action.

.. code::

   $ {command} shell --fakeroot --overlay my_overlay ubuntu.sif
   {Project}> mkdir /data
   {Project}> apt-get update && apt-get install -y vim
   ...
   {Project}> which vim
   /usr/bin/vim

You will find that your changes persist across sessions as though you
were using a writable container.

.. code::

   $ {command} shell --userns --overlay my_overlay ubuntu.sif
   {Project}> ls -ld /data
   drwxr-xr-x 2 user group 4096 Apr  9 10:21 /data
   {Project}> which vim
   /usr/bin/vim

If you mount your container without the ``--overlay`` directory, your
changes will be gone.

.. code::

   $ {command} shell ubuntu.sif
   {Project}> ls /data
   ls: cannot access 'data': No such file or directory
   {Project}> which vim

Readonly overlay
================

After all modifications to an overlay (either ext3 image or directory) 
have been completed,
it can be mounted read-only by appending a ``:ro`` to the overlay path
and no longer needs to use ``--fakeroot``.

Continuing the above example:

.. code::

   $ {command} shell --userns --overlay my_overlay:ro ubuntu.sif
   {Project}> which vim
   /usr/bin/vim
   {Project}> touch /usr/bin/myfile
   touch: cannot touch '/usr/bin/more': Read-only file system

.. _overlay-sif:

Overlay embedded in SIF
=======================

It is possible to embed an overlay image in the SIF file that holds a
container. This allows the read-only container image and your
modifications to it to be managed as a single file.
An example of doing that directly with the ``{command} overlay create``
command was shown above,
but an external image file can also be added to a SIF file with the
``{command} sif add`` command like this:

.. code::

   $ {command} sif add --datatype 4 --partfs 2 --parttype 4 --partarch 2 --groupid 1 ubuntu.sif overlay.img
   $ {command} sif list ubuntu.sif | grep -i ext3
   5    |1       |NONE    |29810688-1103552512       |FS (Ext3/Overlay/amd64)

Below is the explanation what each parameter means, and how it can
possibly affect the operation:

-  ``datatype`` determines what kind of an object we attach, e.g. a
   definition file, environment variable, signature.
-  ``partfs`` should be set according to the partition type, e.g.
   SquashFS, ext3, raw.
-  ``parttype`` determines the type of partition. In our case it is
   being set to overlay.
-  ``partarch`` must be set to the architecture against you're building.
   In this case it's ``amd64``.
-  ``groupid`` is the ID of the container image group. In most cases
   there's no more than one group, therefore we can assume it is 1.

All of these options are documented within the CLI help. Access it by
running ``{command} sif add --help``.

Unlike the ``--overlay`` option, an overlay image inside a SIF is by
default mounted readonly.
To modify the overlay image, use the ``--writable`` option (and likely
also the ``--fakeroot`` option):

.. code::

   $ {command} shell --writable --fakeroot ubuntu.sif
   {Project}> apt-get update && apt-get install -y vim
   ...
   {Project}> exit
   $ {command} exec ubuntu.sif which vim
   /usr/bin/vim

Final note
==========

To resize an overlay, standard Linux tools which manipulate ext3 images
can be used. For instance, to resize the 500MB file created above to
700MB one could use the ``e2fsck`` and ``resize2fs`` utilities like so:

.. code::

   $ e2fsck -f overlay.img && \
       resize2fs overlay.img 700M

Hints for creating and manipulating ext3 images on your distribution are
readily available online and are not treated further in this manual.
