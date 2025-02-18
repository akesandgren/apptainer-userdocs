.. _security:

###########################
 Security in {Project}
###########################

*****************
 Security Policy
*****************

If you suspect you have found a vulnerability in {Project} we want
to work with you so that it can be investigated, fixed, and disclosed in
a responsible manner. Please follow the steps in our published `Security
Policy <https://apptainer.org/security-policy/>`__, which begins with
contacting us privately via security@apptainer.org

We disclose vulnerabilities found in {Project} through public
CVE reports, and notifications on our community channels. We encourage
all users to monitor new releases of {Project} for security
information. Security patches are applied to the latest open-source
release.

************
 Background
************

{Project} grew out of the need to implement a container platform
that was suitable for use on shared systems, such as HPC clusters. In
these environments multiple people access a shared resource. User
accounts, groups, and standard file permissions limit their access to
data, devices, and prevent them from disrupting or accessing others'
work.

To provide security in these environments a container needs to run as
the user who starts it on the system. Before the widespread adoption of
Linux user namespaces, only a privileged user could perform the
operations which are needed to run a container. A default Docker
installation uses a root-owned daemon to start containers. Users can
request that the daemon starts a container on their behalf. However,
coordinating a daemon with other schedulers is difficult and, since the
daemon is privileged, users can ask it to carry out actions that they
wouldn't normally have permission to do.

When a user runs a container with {Project}, it is started as a
normal process running under the user's account. Standard file
permissions and other security controls based on user accounts, groups,
and processes apply. 

**************************
 Setuid & User Namespaces
**************************

Using a setuid binary to run container setup operations used to be
essential to support containers on the older Linux distributions that were
previously common in HPC and enterprise.

Most distributions now have
support for unprivileged user namespaces. This means a normal, unprivileged
user can create a user namespace, in which most operations needed
to run a container can be run.

{Project} still supports running containers with a setuid starter, but
by default it runs containers without setuid, using user namespaces.
If user namespaces are available when compiling, the ``--without-suid``
option is implied.
If user namespaces are not available when compiling, the installer
must choose between ``--with-suid`` and ``--without-suid``.
Packages are compiled with ``--with-suid`` but then the setuid
component is not installed by default and the installer must separately
install the ``{command}-suid`` package if setuid mode is desired.

In non-suid mode *all* operations run as the user who starts the
``{command}`` program. 
This has some advantages over suid mode:

-  Setuid-root programs are notoriously difficult to make fully secure.
   {Project} keeps the setuid portions to a minimum and has passed a
   careful review, but still it is a risk.

-  Linux kernel developers believe that it is inherently unsafe to 
   allow unprivileged users to modify an underlying filesystem at will
   while kernel code is actively accessing the filesystem
   (see `this article <https://lwn.net/Articles/652468/>`__). 
   Kernel filesystem drivers do not and cannot protect against all kinds
   of modifications to that data which it has not itself written, and
   that fact could potentially be used to attack the kernel.
   By the way it does mounts (details below), {Project} prevents the
   most obvious modifications which would enable elevated privileges,
   and there are not currently any publicly known kernel attacks for
   this, but this is a significant risk.

-  Non-suid {command} can run nested inside another {command} command
   or in other container runtimes that restrict setuid-root.

However, there are also some disadvantages of the non-suid mode:

-  Mounting from unprivileged user namespaces makes use of FUSE
   filesystems, which run extra processes in user space.
   This has slightly lower performance than kernel filesystems,
   but it is has been shown to not be a very significant overhead for
   typical HPC workflows.
   Metadata operations are still moved to the node running the
   container, which is a big advantage over having many files directly
   on networked filesystems.

-  Encryption is not yet supported. In suid mode, {Project} uses kernel LUKS2
   mounts to run encrypted containers without decrypting their content
   to disk.
   An unprivileged FUSE filesystem will hopefully be able to perform this
   operation in a future release.

-  Some little used :ref:`security options <security-options>` and
   :ref:`network options <networking>` of {Project} that give users elevated
   privileges through configuration are only available in suid mode.

-  {Project} configuration options that restrict the use of containers
   are not enforceable, because if unprivileged user namespaces are
   available then people could always compile their own copy from source
   and set their own configuration options.

-  Since the Linux kernel is subject to a much greater amount of
   scrutiny than the {Project} setuid software, there have been a greater
   number of announced vulnerabilities that are exploitable through
   kernel namespace code than have been announced for {Project} and
   its predecessor. 
   Security experts generally argue that it is better to have the
   scrutiny than to have "security by obscurity",
   but urgently responding to those vulnerabilities causes additional
   administrator effort and can cause disruption to operations.
   See the `User Namespaces section
   <{admindocs}/user_namespace.html#disabling-network-namespaces>`_
   of the admin guide for details about mitigating the impact of user
   namespace vulnerabilities through disabling network namespaces.

********************************
 Runtime & User Privilege Model
********************************

While other runtimes have aimed to safely sandbox containers executing
as the ``root`` user, so that they cannot affect the host system,
{Project} has adopted an alternative security model that protects
against attacks even with the setuid-root mode:

-  Containers should be run as an unprivileged user.

-  The user should never be able to elevate their privileges inside the
   container to gain control over the host.

-  All permission restrictions on the user outside of a container should
   apply inside the container.

-  Favor integration over isolation. Allow a user to use host resources
   such as GPUs, network filesystems, high speed interconnects easily.
   The process ID space, network etc. are not isolated in separate
   namespaces by default.

To accomplish this, {Project} uses a number of Linux kernel
features. The container file system is mounted using the ``nosuid``
option, and processes are started with the ``PR_NO_NEW_PRIVS`` flag set.
This means that even if you run ``sudo`` inside your container, you
won't be able to change to another user, or gain root privileges by
other means.

If you do require the additional isolation of the network, devices, PIDs
etc. provided by other runtimes, {Project} can make use of
additional namespaces and functionality such as seccomp and cgroups.

********************************
 Singularity Image Format (SIF)
********************************

{Project} uses SIF as its default container format. A SIF container
is a single file, which makes it easy to manage and distribute. Inside
the SIF file, the container filesystem is held in a SquashFS object. When
in suid mode, {Project} mounts the container filesystem directly using SquashFS,
otherwise it mounts the filesystem with squashfuse. In either case, on a
network filesystem this means that reads from the container are
data-only. Metadata operations happen locally, speeding up workloads
with many small files.

Holding the container image in a single file also enables unique security
features. The container filesystem is immutable, and can be signed. The
signature travels in the SIF image itself so that it is always possible
to verify that the image has not been tampered with or corrupted.

{Project} uses private PGP keys to create a container signature, and the corresponding
public key in order to verify the container signature. Verification of signed
containers can be done at any time by a user and happens automatically in
``{command} pull`` commands against Library API registries. The prevalence
of PGP key servers, (like https://keys.openpgp.org/), make sharing and obtaining
public keys for container verification relatively simple.

A container may be signed once, by a trusted individual who approves its
use. It could also be signed with multiple keys to signify it has passed
each step in a CI/CD QA & Security process. {Project} can be
configured with an execution control list (ECL), which requires the
presence of one or more valid signatures, to limit execution to approved
containers.

In addition, the root filesystem of a container (stored in the squashFS
partition of SIF) can be encrypted. As a result, everything inside the container
becomes inaccessible without the correct key or passphrase. The content of the
container is private, even if the SIF file is shared in public.

When in suid mode,
encryption and decryption are performed using the Linux kernel's LUKS2
feature. This is the same technology routinely used for full disk
encryption. The encrypted container is mounted directly through the
kernel. Unlike other container formats, an encrypted container is not
decrypted to disk in order to run it.
Encryption and decryption is not currently supported in non-suid mode.

*********************************
 Configuration & Runtime Options
*********************************

System administrators who manage {Project} can use configuration
files to set security restrictions, grant or revoke a user's
capabilities, manage resources and authorize containers etc.

For details see the `Security section <{admindocs}/security.html>`_
of the admin guide.
