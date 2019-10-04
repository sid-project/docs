*******
Context
*******

Uevents and udev
================

Linux kernel provides a way to send simple notification messages to
userspace related to changes of device's state and we call these *udev
events* or *uevents* for short.

The uevents are sent from kernel to userspace using *netlink* interface
(*man 7 netlink*). The exact netlink type reserved for this purpose is
``NETLINK_KOBJECT_UEVENT``. One or more userspace listeners can register
to receive the events and if there is more than one listener, these events
are sent in multicast manner.

Currently supported set of *action names* used for uevents are:

  * | ``add``
    | device added,

  * | ``change``
    | device changed,

  * | ``remove``
    | device removed,

  * | ``move`` 
    | device moved to a new parent or device renamed,

  * | ``offline``
    | device is put offline,

  * | ``online``
    | device is put back online after being offline,

  * | ``bind``
    | driver is bound to a device (since kernel version 4.14),

  * | ``unbind`` 
    | driver is unbound from a device (since kernel version 4.14).

The most frequently used ones are ``add``, ``change`` and ``remove``.
Each kernel uevent contains a set of environment variables in ``KEY=VALUE``
format. The minimal and basic set of keys we can find in the kernel uevent,
which is added by kernel's common uevent code, contains at least:

  * | ``ACTION``
    | device's action name,

  * | ``DEVPATH``
    | device's canonical path in *sysfs* (see also *man 5 sysfs*),

  * | ``SUBSYSTEM``
    | subsystem the device belongs to,

  * | ``SEQNUM``
    | this uevent's sequence number.

The Linux kernel's driver core then adds further keys to extend the basic
set, if values for these keys are available:

  * | ``MAJOR``
    | device's major number,

  * | ``MINOR``
    | device's minor number,

  * | ``DEVNAME``
    | device's canonical kernel name,

  * | ``DEVMODE``
    | device's permissions mode,

  * | ``DEVUID``
    | device's user ID (if not global root UID),

  * | ``DEVGID``
    | device's group ID (if not global root GID),

  * | ``DEVTYPE``
    | device's type name,

  * | ``DRIVER``
    | device's driver name.

Various device subsystems and device drivers in kernel can add even more
additional ``KEY=VALUE`` pairs. However, the overall size of the uevent is
limited: maximum number of ``KEY=VALUE`` pairs is 32 (``UEVENT_NUM_ENVP``
constant as found in kernel source code) and the overall size limit for the
whole uevent sent from kernel is 2048 bytes (``UEVENT_BUFFER_SIZE``
constant as found in kernel source code).

For the purpose of this text, we will call the uevents which are sent from
kernel the **genuine kernel uevents** (or *kernel uevents* shortly).

In general, each uevent for a block device has: ``ACTION``, ``DEVPATH``,
``SUBSYSTEM``, ``SEQNUM``, ``MAJOR``, ``MINOR``, ``DEVNAME`` and ``DEVTYPE``
keys set in its uevent environment.

Besides genuine kernel uevents generated based on execution within kernel
and its drivers, there are also **synthetic kernel uevents** (or *synthetic
uevents* shortly). Even though these uevents are generated in kernel, they
are provoked directly in userspace by writing the uevent action name to
``/sys/…/uevent`` file. Such uevents look exactly like *genuine kernel
uevents*, the only difference is that they do not contain any additional
keys in their environment that drivers may add, only the basic key set.
Such uevents are usually used to trigger device state reevaluation back in
userspace once the synthetic uevent is received in userspace.

It is also possible to send uevents directly from userspace back to
userspace, hence providing a way to send messages between two or more
userspace processes. We call these **udev uevents**.

General term that encompasses all the uevents and the logic of dynamic
device management based on these uevents in userspace is **udev**.
On userspace side, the important component is the *udev daemon*.

Udev daemon
===========

One of userspace uevent listeners has a primary role and this is the
**udev daemon**, or shortly **udevd**, which is a part of systemd_
project.

The way udevd processes uevents in userspace is driven by **udev rules**
which are usually placed in ``/lib/udev/rules.d`` and ``/etc/udev/rules.d``
directory. There is a common set of rules provided directly by upstream
udevd, other rules are installed by foreign tools and system components.

Whenever there is a new kernel uevent (genuine or synthetic one) coming
from the kernel and when received by udevd, the udevd creates a new process
(also called *udevd worker*) to handle the event or reuses existing one if
it is available. The udevd keeps worker processes if previous event has
just been processed and the queue is not empty yet so it reuses the worker
immediately to execute the rules for next uevent, hence optimizing and
saving machine time and resources.

Only one event can be handled at a time for a single device. That means all
processing of uevents that are issued for a single device is serialized,
queued and processed one by one while uevents for different devices can
still be processed in parallel. Devices are distinguished based on their
canonical device path in sysfs.

There is a limit to the number of worker processes that are created to
handle the uevents in parallel and this is controlled by udevd's
``--children-max`` command line option or provided on kernel command
line as ``udev.children_max`` argument. This way, it is possible to control
the degree of parallelism the udevd uses. With current implementation,
the default value for this option is computed using a simple formula that
is based on number of CPU cores available:

.. math::
  2 * AvailableCpuCoreCount + 8

..
  The calculation for the limit on number of worker processes `has changed
  in systemd v243 <https://github.com/systemd/systemd/commit/88bd5a32e89125d79d68bf846d1af419ebb8cb4f>`_.

Udevd's primary role is to collect any additional information that is
needed to create various symlinks under /dev directory and to set
permissions driven by instructions written in udev rules. Udevd has
no control over device node names (with the exception of network
devices). With devtmpfs filesystem in use, the device nodes are created
directly by kernel and udevd only adjusts their permissions. Udev rules
can also access information present in sysfs for the device that is being
processed. To collect any other information, udev rules need to instruct
udevd to execute external commands or to gather this information in a
special way. This is accomplished by executing one of these rules:

  * | ``IMPORT``
    | Executes a command that exports the information in ``KEY=VALUE``
      pairs that is then imported into udev context which further udev
      rules can access; the actual call is made right at the time when the
      rule is hit.

  * | ``RUN``
    | Adds a command to the list of commands to be executed after all the
      rules are processed – so delaying the execution up to the end of udev
      rule processing.

  * | ``PROGRAM``
    | Executes a command where the string output from the last executed
      command can be matched with accompanying ``RESULT`` rule.

The ``IMPORT`` and ``RUN`` rule can either execute external command or
it can execute udevd's own *builtin command* by specifying
``IMPORT{builtin}`` or ``RUN{builtin}``. The builtin commands have
advantage over external commands in fact that they do not require a new
process to get created (forked) and these commands are initialized as
soon as udevd is started. However, builtin commands need to be integrated
directly into udevd's code base - they are not designed as external modules
loaded on udevd startup.

Udevd poses a restriction on time to execute all the udev rules for
particular uevent. Currently, the default value is 180 seconds. It is
possible to override the default value by specifying udevd's
``event-timeout`` option or by specifying the timeout value on kernel
command line with ``udev.event-timeout`` argument. The timer starts
counting as soon as the worker process is forked or reused and it is
stopped when main udevd process receives a message from worker processes
that it has finished the processing. Simplified list of steps taken to
execute udev worker on incoming kernel uevent is following:

  1. kernel uevent is received by main udevd process

  2. udevd create or reuses udevd worker process to handle the uevent

  3. udevd starts timer for the udevd worker

  4. udevd worker executes and applies udev rules

  5. udevd worker updates *udev database*

  6. udevd worker executes run queue
     (all the calls as instructed by ``RUN`` rule)

  7. udevd worker sends udev uevent

  8. udevd worker sends  *worker finished* message to main udevd process

  9. udevd receives the *worker finished* message from worker

  10. udevd stops timer for the udevd worker

The udev uevent, in contrast to kernel uevent, is the uevent sent by udevd
directly to all its listeners after all the rules have been processed and
hence such uevent contains all the environment variables in ``KEY=VALUE``
format that have been added by execution and application of the udev rules.
The udev uevent is sent by udevd using the same netlink interface as udevd
used to receive the kernel uevent, the netlink interface makes this
possible. Usually, udev uevent as well as kernel uevent listeners subscribe
for these uevents using libudev_ library (*man 3 libudev*) which wraps up
these uevents in a structure for easier manipulation and for further
processing using various libudev functions and also it abstracts out the
actual netlink usage for the library user.

The udev database is a simple filesystem-based database (usually stored
in ``/run/udev`` directory). It contains current environment for each
device – the ``KEY=VALUE`` pairs and other information used and recorded
by udevd: list of symlinks, symlink priority, tags and monitoring of
device content changes requested by ``OPTIONS+="watch"`` udev rule.

.. note::
  The ``OPTIONS+="watch"`` udev rule is internally implemented using
  ``inotify`` monitoring mechanism (*man 7 inotify*). Whenever a monitored
  device is closed after being open for writing before, udev daemon
  receives the inotify event. Then, udev daemon generates synthetic uevent
  for the device based on the inotify event. The ``OPTIONS+="watch"``
  udev rule is usually used when we expect that a write operation to
  the device can change its content in a way that this also changes the
  way udev rules are evaluated and that in turn can change the udev
  database content.

Block device uevent processing
==============================

Block device uevent processing is driven by udev rules provided by both
upstream udev itself as well as block device subsystems.

Rules provided by udev itself
-----------------------------

  * | ``60-block.rules``
    | Eenables media presence polling, forwards scsi events to
       corresponding block device and sets ``OPTIONS+="watch"`` for
       selected block devices.

  * | ``60-persistent-storage.rules``
    | Imports parent information from udev database for partitions, calls
      ``ata-id``, ``scsi_id``, ``usb_id``, ``path_id``, ``blkid``, sets
      device symlinks.

  * | ``60-persistent-storage-tape.rules``
    | Calls ``blkid``, sets device symlinks.

  * | ``60-cdrom_id.rules``
    | Calls ``cdrom_id``, sets device symlinks.

  * | ``64-btrfs.rules``
    | Calls ``btrfs_ready`` builtin command, marks device as not ready if
      needed and sets ``SYSTEMD_READY`` variable appropriately.

  * | ``99-systemd.rules``
    | Sets ``SYSTEMD_READY`` variable based on various other variables
      and/or ``sysfs`` content. It also includes handling of loop devices.

Rules provided by device-mapper (DM) subsystem
----------------------------------------------

  * | ``10-dm.rules``
    | Calls ``dmsetup udevflags`` to decode flags out of ``DM_COOKIE``
      variable, calls ``dmsetup info`` if needed, sets device symlinks,
      imports variables from previous udev database state if needed.

  * | ``13-dm-disk.rules``
    | Calls ``blkid``, sets device symlinks.

  * | ``95-dm-notify.rules``
    | Calls ``dmsetup udevcomplete`` to notify waiting process about udev
      rule processing completion.

Rules provided by DM-LVM subsystem
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

  * | ``11-dm-lvm.rules``
    | Calls ``dmsetup splitname`` to split DM name into VG/LV/layer parts,
      imports variables from previous udev state if needed, sets device
      symlinks.

  * | ``12-dm-lvm-permissions.rules``
    | This is a template to add rules to set device permissions.

  * | ``69-dm-lvm-metad.rules``
    | Detects when the device is ready for use and schedules
      ``lvm2-pvscan@<major>:<minor>.service`` systemd unit containing
      ``pvscan --cache -a ay call`` to update ``lvmetad`` and to activate
      a VG once it is complete.

Rules provided by DM-multipath subsystem
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

  * | ``11-dm-mpath.rules``
    | Imports variables from previous udev database state if needed, marks
      multipath device either as ready or not or whether scanning can be
      done on this device.

  * | ``62-multipath.rules``
    | Calls ``multipath -c`` and ``multipath -T`` to check for multipath
      components, imports variables from previous udev database state if
      needed, calls ``partx`` to remove partitions on multipath components
      and it calls ``kpartx`` to create partition mappings on top of a
      multipath device.

Rules provided by multiple device (MD) subsystem
------------------------------------------------
    
  * | ``63-md-raid-arrays.rules``
    | Handles arrays with external metadata: DDF and Intel Matrix RAID,
      calls ``mdadm –-detail``, calls ``blkid``, creates device symlinks,
      schedules MD array monitoring.

  * | ``65-md-incremental.rules``
    | Calls ``mdadm -I`` for incremental addition or removal of a device
      to/from an MD array if the device is ready/removed, requests
      ``mdadm-last-resort@<md_device>.timer`` systemd unit to get started
      to implement a timeout on MD devicefor it to be started in degraded
      mode.


Rules provided by Ceph subsystem
--------------------------------

  * | ``50-rbd.rules``
    | Calls ``ceph-rbdnamer`` and creates device symlinks based on results.

  * | ``60-ceph-by-parttypeuuid.rules``
    | Forwards SCSI events to corresponding block device, imports parent
      information from udev database for partitions, calls ``blkid``,
      creates device symlinks for partitions.

  * | ``95-ceph-osd.rules``
    | Sets permissions, calls ``ceph-disk``)

Rules provided by btrfs subsystem
---------------------------------

  * | ``64-btrfs-dm.rules``
    | Calls ``btrfs ready`` to let btrfs subsystem know underlying DM
      device is ready.

  * | ``64-btrfs.rules``
    | Calls ``btrfs ready`` to let btrfs subsystem know the underlying
      device is ready.

Problematic areas
=================

The udevd was primarily designed to collect additional information that is
needed for a specific device and then let udevd create additional symlinks
in ``/dev`` and set proper permissions for the device node based on rules.

Although the majority of the rules to handle block devices do contain rules
that set device node symlinks, the fact is that over the years the number
of various other calls within these rules has risen too. Currently, it is
not only that additional information collection that the rules do, but it
is also other functionality, like further activation and various helper
calls to support various specific aspects of block device subsystems. As a
consequence, there are various problems and shortcomings related with this
approach which became significant.

This section lists and briefly describes various problems and shortcomings
in general which we have identified while trying to deploy storage-related
solutions over time and then trying to integrate them with udev.

These problems are not completely discrete. Instead, they are very closely
related to each other and a solution to one of these problems usually
reduces degree of impact of other problematic parts.

Multistep activation
--------------------

Some block devices have more complex nature when it comes to activation and
detecting current device state.

This is mainly the case for subsystems like DM (including device-mapper
multipath and LVM subsystem) and MD devices where they are are created
first (that generates ``add`` uevent), but the device may not be usable
right away. Usually, there is another step or more to make these devices
ready for use (that generates further ``change`` uevents).

Notion of device groups and stack awareness
-------------------------------------------

One of the most important features we also need to take into account is the
fact that some block devices can be stacked on top of each other and they
can form an abstraction over a set of devices which logically groups them
together.

Udev has no direct notion of grouping or stack awareness within the device
groups.

Intermediate steps during device management
-------------------------------------------

Some subsystems also support conversions from one type to another which may
require several deactivation and activation steps and transforming the
device with intermediate steps in between.

Unless we mark the intermediate states with additional ``KEY=VALUE`` pairs
within the uevents the kernel driver generates or unless we use an external
information or tool to decide on what the current state is, we cannot make
a difference within udev rules and we act as if this was usual device
activation or deactivation or a generic change. The usual set of rules are
executed even though the commands executed within those rules may interfere
with the process of device transformation or conversion.

Also, such processing may not be efficient if the result is outdated right
in the next step that follows and we are only interested in the overall
result when the device is fully set up again and ready for use.

Recognizing uevents, device's state and overloaded uevents
----------------------------------------------------------

All block device subsystems use udevd to drive userspace actions based on
uevents coming from kernel - either originating in the kernel driver itself
or synthesized in userspace by writing the ``/sys/…/uevent`` file.
Inherently, some of the special uevents that these block subsystems would
need to have processed are mapped onto a single ``change`` uevent instead
of distinct uevents directly describing the nature of the event.

This fact makes  the udev rules complex because they need to deal with
these device state transitions and they need to recognize uevents properly
to know what the transition is exactly, possibly comparing udev's
environment (the ``KEY=VALUE`` pairs) with previous environment stored in
udev database.

Udevd was not designed for this task. Even though there are rules to import
previous udev database values (the ``IMPORT{db}`` udev rule), we cannot do
direct comparisons of previous and current values for certain keys which
are in udev's environment in an efficient way. We can only do simple string
matching so only rules in the form of
``ENV{KEY}=="direct_string_to_match"`` are possible, but not
``ENV{KEY1}==ENV{KEY2}``. Also, udevd does not support number comparisons
directly within udev rules, because the only operator supported is a match
against an explicit string value.

Udev rule language and related restrictions
-------------------------------------------

It is up to the driver or udev rules to properly recognize current state.
The kernel driver can add a set of various additional ``KEY=VALUE`` pairs
that it passes with the kernel uevent it generates. Alternatively, if we
try to handle this in udev rules directly, we need to get previous udev
database state, do comparisons of the states and/or call an external tool
to evaluate the environment and return the results back to udev context
to evaluate the rules further.

Considering the fact that the language used to articulate udev rules is
very simple and restricted, we may end up with complex rules even for
relatively simple device state detection or detection of device state
advancement within a state machine we need to track, including the burden
of calling an external tool to make further decisions.

This makes it hard to implement state machines within udev rules to track
devices properly.

Debugging and logging
---------------------

As the rules get more and more complex, whenever a problem appears, it is
complicated to perform effective debugging - udevd does not report current
environment it is working with nor does it have support for adding
additional logging hooks into the rules directly. With this, it is hard to
track what the actual path was taken when the udev rules were processed
and what the actual states were.

This status quo is also a consequence of the fact that some device
subsystems try to implement more complex logic with the udev rules tha
what they were originally designed for.

Marking devices as ready
------------------------

The udev rules are responsible for triggering device activation based on
current state at proper time. This becomes even more prominent if we are
considering device stacks where one block device subsystem is layered on
top of another one and so on.

We need to have a proper and standard way of marking devices in the layer
below as ready for any layer above. This standard is currently missing.
Each subsystem has its own way of marking the device as ready – there are
various ``KEY=VALUE`` pairs to check in udev's environment (e.g.
``DM_ACTIVATION``, ``MD_STARTED``, ``SYSTEMD_READY``, ...).

The same problem arises when considering event subscribers using udev
monitoring which have no standardized way to know whether a device is ready
for use or not.	

Amount of work in udevd context
-------------------------------

Another problem that arises is related with the amount of work that needs
to be done to process the uevent while processing udev rules.

As per udevd design, this extra work and processing needs to be minimized
as much as possible and it should be restricted to acquiring the
information that is needed to have all the needed symlinks in ``/dev``
created. That means, all the rules and processing that is not related to
collecting basic device identification and information collection should be
moved out of udevd context and executed later or, if possible, in parallel
to udevd.

Timeouts
--------

Udevd sets up timeout for each uevent's processing. On heavy-loaded system,
this can pose a problem as default timeout may not be enough. The timeouts
cannot be set in runtime - support for ``OPTIONS="event_timeout"`` rule
has been removed from udevd.

If the timeout occurs, the udevd worker with any of its children processes
is killed by udevd using ``SIGTERM`` signal. For this reason, commands
which may take longer to execute must be executed in background. On systems
with systemd, the command needs to be instantiated as a service even,
completely out of udevd's context and its control group. There is no
special handling for these timeouts – if a timeout occurs and the udevd
worker is killed, any udev uevent listener will receive the uevent without
any additional variables set – udevd just relays the kernel uevent it
receives as udev uevent to all its listeners.

If the timeout happens, we would need to let the listeners know or provide
a possibility to define fallback actions to keep the system running and
letting the user fix the configuration or increase timeouts if needed.

Synthetic uevents
-----------------

Another problematic area is with the source of uevents. Besides genuine
udev events coming from kernel directly, there are also synthetic events,
as we already mentioned before. There are three usual ways how the
synthetic uevent is triggered from user's perspective:


 - by directly writing the event name to ``/sys/.../uevent`` file,

 - by calling udevadm trigger command (which in turn writes to the
   ``/sys/.../uevent`` file),

 - by using ``OPTIONS="watch"`` udev rule for a device (and then
   whenever the device is opened for writing and then closed,
   the inotify watch triggers that udevd receives that in turn writes
   to the ``/sys/../uevent`` file).


If kernel driver does not provide any additional variables for the uevent
it generates, the genuine uevent is indistinguishable from the synthetic
one – this may make it harder to recognize which event is the one that
makes the device ready for use. For a long time, udev's position was that
these two uevents should remain indistinguishable and uevent listeners
and authors of udev rules should account for this fact.

However, our argument is that there is indeed a difference in these two
types of uevents. The genuine kernel uevents notify userspace about a state
change of the device itself (e.g. device addition or change in device's
configuration that the kernel itself is aware of, device's removal).
The synthetic uevents, which originate in userspace actions, are either
used to refresh udev's state in userspace (e.g. to repopulate udev database
if it was cleared before or started afresh or to notify any uevent
subscribers to simply reread information based on the uevent if it is
needed).

Alternatively, synthetic uevents may be used to notify about changes in
device's content in general – the device's content is something that is
usually not tracked by kernel device drivers (e.g. subsystem or filesystem
signatures are added to device or they are cleared).

At the moment, the two types of uevents are considered equal. This is a
source of confusion when handling the uevents and it may also cause useless
resource consumption due to excessive processing. Trying to solve this
issue, at least partially, within udev context, requires writing even more
complex udev rules to try to make a difference between these two uevent
types.

Also, the synthetic event is completely asynchronous and we cannot
synchronize with that at all at the moment. This creates considerable
burden for any tools trying to access the device exclusively or even
remove the device because synthetic uevents can happen in parallel.

Marking devices as private or public
------------------------------------

Another problematic area is within identification of devices which are
private for the subsystem. Such devices only act as building blocks to
create a higher level device that is supposed to be the one used.

Also, we may need to initialize the device first before marking it as
ready for use. For example, we need to erase any old signatures which may
be left on the device from previous use. Again, there is no standard
defined on how such devices are marked (e.g. DM devices use flags in
``DM_COOKIE`` uevent variable to handle this while MD uses a temporary file
``/run/mdadm/creating-<md_device_name>`` in filesystem to mark device as
not fully initialized yet).

Device initialization
---------------------

We should be able to activate a device in private mode first (without doing
scans), providing time for usersapce tools to do any initialization steps
and cleaning that is necessary to properly make the device ready for use.

After these initialization steps, userspace tool should be able to switch
the device into ready state by issuing synthetic uevent that is properly
recognized for this type of switch from private to public mode.

Eventually, the solution for this initialization and wiping part during
device activation may be centralized and handled by a single external
entity without a need for each subsystem to provide its own code to
implement this. Such solution would be preferred, but it requires the
central entity to have enough knowledge so that the initialization and
wiping operation is safe to do at a specific time. The external entity
needs to recognize this initialization state properly and that is already
the problem we have identified before – with current scheme, we are not
completely sure about states.

.. _systemd: https://www.freedesktop.org/wiki/Software/systemd/
.. _libudev: https://www.freedesktop.org/software/systemd/man/libudev.html
