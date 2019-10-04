**************
Prerequisities
**************

``sid`` udevd builtin command
=============================

We need to establish a communication channel between udevd and new Storage
Instantiation Daemon (SID) to make it possible to effectively interchange
information between the two.

Udevd supports implementation of builtin commands which are initialized as
soon as udevd is started and finalized when udevd is stopped. This way,
udevd dos not need to fork and execute a new process for each uevent that
is evaluated like it does when external commands are referenced in udev
rules.

We can call builtin commands in udev rules in very similar way to calling
any external commands – the rules only need to specify that we are
requesting a builtin command instead of an external one with
``RUN{builtin}`` or ``IMPORT{builtin}`` rule.

.. note::
  A udevd builtin command is directly integrated into udevd source code
  using an internal modular interface. The udevd does not support external
  and dynamically loaded modules yet to implement this functionality.

The channel between udevd/sid builtin command and SID itself is implemented
using local stream and connection oriented socket IPC (the
``AF_UNIX``/``AF_LOCAL`` socket domain, ``SOCK_STREAM`` socket type). 

If udevd creates a new worker process, the connection is established on
first ``sid`` builtin command call and it is kept open until udevd worker
exits (or until it is terminated by main udevd process in case of a
timeout). This also counts with any udevd worker reuse for subsequent
uevent processing if there are more uevents queued - the connection is kept
alive.

Since we use stream and connection oriented channel, the receiving side
(SID) can detect peer's (udevd worker) hang up. This way, SID is able to
detect worker timeouts and it can trigger subsequent fallback operation if
the hang up happens at unexpected time.

Subcommands supported by ``sid`` udevd builtin command
------------------------------------------------------

* ``sid active``

  Purpose:
    Check SID availability.

  Input:
    None.

  Output:
    ``SID_BRIDGE_STATUS=<status>`` pair where ``<status>`` is one of:

      * ``active`` if compatible version of SID is running,

      * ``inactive`` if not running,

      * ``incompatible`` if SID is running, but its version is
        incompatible with current ``sid`` udevd builtin command.

  Notes:
    ``sid active`` is primarily used in udev rules to switch functionality
    to fallback mode without SID if needed.

  Example:
    .. code-block:: console

      IMPORT{builtin}="sid active"
      ENV{SID_BRIDGE_STATUS}=="active", ...
      ENV{SID_BRIDGE_STATUS}=="incompatible", ...
      ENV{SID_BRIDGE_STATUS}=="inactive", ...

* ``sid scan``

  Purpose:
    Execute SID identification and scanning routines, update SID database
    accordingly, schedule and execute associated actions.

  Input:
    udev environment in ``KEY=VALUE`` format for currently processed uevent
    that is available at the time of the call.

  Output:
    Environment in ``KEY=VALUE`` format with changed and/or newly added
    items.

  Example:
    .. code-block:: console

      IMPORT{builtin}="sid scan"

* ``sid checkpoint <checkpoint_name> [<key> ...]``

  Purpose:
    Send notification to SID about reaching a checkpoint.

  Input:
    ``<checkpoint_name>`` that is reached while evaluating udev rules.
    Optionally, a key list referencing keys from current udev environment
    can be passed in (or just ``all`` keyword to mean all available keys).
    The command will then, in addition to the checkpoint name itself, send
    over selected ``KEY=VALUE`` pairs to SID from current udev's per-device
    environment. There is one reserved checkpoint name called ``complete``
    to notify SID that udev processing in udev rules is reaching its end.

  Output:
    None.

  Notes:
    Checkpoints can be used for debugging purposes where we install or
    enable additional udev rules with these checkpoints executed for SID to
    be able to track the progression of uevent processing.

  Example:
    .. code-block:: console

      PROGRAM{builtin}="sid checkpoint my_checkpoint A B C"
      RUN{builtin}="sid checkpoint complete"

* ``sid version``

  Purpose:
    Return version information for ``sid`` udevd builtin command and SID
    daemon (if available).

  Input:
    None.

  Output:
    A list of ``KEY=VALUE`` pairs:

        * ``SID_PROTOCOL=<protocol_version>``
        * ``SID_MAJOR=<major_version>``
        * ``SID_MINOR=<minor_version>``
        * ``SID_RELEASE=<release_version>``
        * ``SID_BUILTIN_PROTOCOL=<protocol_version>``
        * ``SID_BUILTIN_MAJOR=<major_version>``
        * ``SID_BUILTIN_MINOR=<minor_version>``
        * ``SID_BUILTIN_RELEASE=<release_version>``

  Notes:
    Even though it is possible to call ``sid version`` in udev rules, it
    makes more sense to call this as part of ``udevadm test-builtin``
    command for the result to be included in various tools which collect
    debugging and version information about current system environment.
    Note that ``udevadm test-builtin`` command requires a ``sysfs`` path
    for an existing device given as argument. However, the ``sid version``
    builtin command is not bound to any device as it only collects version
    information, therefore, we can use arbitrary existing ``sysfs`` path
    here just to fulfill the requirement - it is ignored by the
    ``sid version`` command.

  Example:
    .. code-block:: console

      # udevadm test-builtin "sid version" /sys/kernel
      SID_PROTOCOL=1
      SID_MAJOR=0
      SID_MINOR=0
      SID_RELEASE=1
      SID_BUILTIN_PROTOCOL=1
      SID_BUILTIN_MAJOR=0
      SID_BUILTIN_MINOR=0
      SID_BUILTIN_RELEASE=1

Enhanced synthetic uevents
==========================

When processing uevents, we need to take into consideration the fact that
not all uevents have their origins in direct processing within
kernel itself. There are also synthetic uevents which are generated as a
result of userspace actions, that is, writing uevent's action name to
``/sys/.../uevent`` file. For us to be able to make a better distinction
between synthetic and genuine uevents, we need to enhance the synthetic
uevent interface.

Kernel changes to support enhanced synthetic uevents
----------------------------------------------------

We can enhance the original synthetic uevent interface to make it possible
to pass additional arguments with the action name written to
``/sys/…/uevent`` file.

The proposal is to pass a unique identifier with the event name which marks
the generated uevent as being part of a transaction. The transaction may
span one or more uevents. The identifier then appears as an environment
variable in the generatad uevent. If we use a single identifier with more
uevents, we logically group them together - in this case any userspace
process reading such uevents can watch and/or collect all members of the
group. Also, this way, it is possible to synchronize with the synthetic
uevent processing and we can wait for all the changes in the group to
settle down before continuing with subsequent processing.

Before, we could not wait for a single (or selected group) of uevent
processing to finish. We could do this only at global level using
``udevadm settle`` command which in turn waits until whole udev processing
queue is empty - in which case, we also wait for any unrelated uevents
which may happen for any other devices in parallel and no matter if they
are synthetic or genuine ones or if they are of any significance to us. 

For backwards compatibility, the identifier is not required for synthetic
uevent to be generated. In this case, the kernel will automatically add an
identifier with zero value.

.. note::
  We have chosen UUID to represent the identifier – there are already
  existing tools to generate UUIDs easily on command line (for example
  ``uuidgen``  that is part of essential ``util-linux`` package) or by
  using various libraries (for example ``libuuid`` that is also part of
  ``util-linux`` or ``sd-id128`` from ``libsystemd`` that is part of
  ``systemd`` pack).

Userspace processes should be also able to pass more variables for the
generated synthetic uevents. The list of extra environment variables is
then passed as ``KEY=VALUE`` pairs following the identifier. The ``KEY``
and ``VALUE`` allowed character set will be limited to alphanumeric
characters. Each ``KEY=VALUE`` pair is delimited by a space character.
To avoid name clashes for existing udev environment variable keys, the
kernel will prepend a prefix automatically for each ``KEY`` that is passed
through the ``/sys/.../uevent`` file - that prefix is ``SYNTH_``.

Following is an example of enhanced synthetic uevent interface usage and
functionality:

.. code-block:: console

  # uuid=$(uuidgen)

  # echo $uuid
  4f60b88c-3052-4daa-8904-2e4efe8563ef

  # echo "change $uuid A=1 B=abc" > /sys/block/sda/uevent

Monitoring generated synthetic uevent for this uevent trigger shows that a
uevent with the additional ``KEY=VALUE`` pairs we passed in are now a part
of the uevent as ``SYNTH_KEY=VALUE`` pairs:

.. code-block:: console

  # udevadm monitor --kernel --property

  monitor will print the received events for:

  KERNEL - the kernel uevent

  KERNEL[12557.917744] change   /devices/pci0000:00/0000:00:08.0/virtio4/host2/target2:0:0/2:0:0:0/block/sda (block)
  ACTION=change
  DEVNAME=/dev/sda
  DEVPATH=/devices/pci0000:00/0000:00:08.0/virtio4/host2/target2:0:0/2:0:0:0/block/sda
  DEVTYPE=disk
  MAJOR=8
  MINOR=0
  SEQNUM=2027
  SUBSYSTEM=block
  SYNTH_ARG_A=1
  SYNTH_ARG_B=abc
  SYNTH_UUID=4f60b88c-3052-4daa-8904-2e4efe8563ef

The ``/sys/.../uevent`` interface still remains backwards compatible. That
means, if we use only the action name for synthetic uevent trigger and no
further additional variables, this still works as expected:

.. code-block:: console

  # echo "change" > /sys/block/sda/uevent

.. code-block:: console

  # udevadm monitor --kernel --property

  monitor will print the received events for:

  KERNEL - the kernel uevent

  KERNEL[12646.761609] change   /devices/pci0000:00/0000:00:08.0/virtio4/host2/target2:0:0/2:0:0:0/block/sda (block)
  ACTION=change
  DEVNAME=/dev/sda
  DEVPATH=/devices/pci0000:00/0000:00:08.0/virtio4/host2/target2:0:0/2:0:0:0/block/sda
  DEVTYPE=disk
  MAJOR=8
  MINOR=0
  SEQNUM=2028
  SUBSYSTEM=block
  SYNTH_UUID=0

In this case, there is ``SYNTH_UUID=0`` added automatically by kernel so
even if the extra arguments are not used when writing to
``/sys/.../uevent`` to generate the synthetic uevent, we can still make a
difference between synthetic and genuine uevents when processing them - the
synthetic uevents will always contain ``SYNTH_UUID`` key  while genuine
uevents will not have this variable set.

.. note::
  The `enhanced synthetic uevent interface
  <https://www.kernel.org/doc/Documentation/ABI/testing/sysfs-uevent>`_
  is supported since Linux kernel version 4.13.

Userspace changes to support enhanced synthetic uevents
-------------------------------------------------------

Udevd itself uses synthetic uevent interface to generate synthetic uevents
to implement udevd's ``OPTIONS+="watch"`` rule support and
``udevadm trigger`` functionality. The ``libudev`` library currently does
not cover any synthetic uevent functionality yet.

When using patched kernel with the enhanced synthetic uevent interface and
without any further changes in ``OPTIONS="watch"`` and ``udevadm trigger``
implementation, we end up with synthetic uevents generated with
``SYNTH_UUID=0`` pair in the uevent and no further keys defined with
``SYNTH_`` prefix. This improves the situation for SID and udev rules to
recognize such uevents as synthetic ones, but we would like to specify
further what kind of synthetic uevent this is and make a difference among
other synthetic uevents that may be generated. Therefore, we are proposing
an approach where whenever udevd enerates any synthetic uevents, they will
be marked appropriately.

Enhanced synthetic uevents for ``OPTIONS+="watch"`` rule
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

All synthetic uevents generated as a result of applying the
``OPTIONS+="watch"`` rule will contain these ``KEY=VALUE`` pairs::

 SYNTH_UUID="00000000-0000-0000-0000-00000000000"
 SYNTH_ARG_UDEV_WATCH="1"

This means, each time udevd writes to ``/sys/.../uevent`` file to generate
synthetic uevents in this scenario, it will use this string to trigger the
uevent::

  change 00000000-0000-0000-0000-00000000000 UDEV_WATCH=1

The reason we use null UUIDs in synthetic uevents generated based on
``OPTIONS="watch"`` rule is that we do not have any use for the UUID in
this scenario yet. This use is left for possible future enhancements.

Enhanced synthetic uevents for ``udevadm trigger`` call
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

All synthetic uevents generated as a result of executing the
``udevadm trigger`` call will contain these ``KEY=VALUE`` pairs::

 SYNTH_UUID="<uuid>"
 SYNTH_ARG_UDEV_TRIGGER="1"

This means, each time udevd writes to ``/sys/.../uevent`` file to generate
synthetic uevents in this scenario, it will use this string to trigger the
uevent::

  <action_name> <uuid> UDEV_TRIGGER=1

Here, the ``<action_name>`` is ``ACTION`` as provided with
``-c, --action=ACTION`` argument which is already supported by
``udevadm trigger``. The ``<uuid>`` is passed in by using a new
``--uuid=UUID`` argument. The UUID is then used to trigger all the
uevents - all the uevents which are generated as a result of this trigger
are then grouped together into a single transaction by this UUID. If the
UUID is not specified, the ``udevadm trigger`` itself generates a new
random one.

In addition, we propose a new ``--wait-uevent`` argument for the
``udevadm trigger`` which will cause it wait for all related uevent
processing in udevd (including all the udev rule processing) to finish
before it exits. Simply, this will collect and wait for all udev uevents
having the exact and same UUID before it exits.

For example, to trigger change uevent for all devices belonging to block
subsystem and waiting for all associated uevent processing in udevd to
finish before exiting the ``udevadm trigger``, we will call:

.. code-block:: console

  # udevadm trigger --subsystem-match block --action change --wait-uevent

To trigger change uevent for all devices belonging to block subsystem
so that all the generated uevents have 6cab53e2-b9c9-4c43-9d1d-0d8673fb62b0
UUID set in the uevent environment, we will call:

.. code-block:: console

  # udevadm trigger --subsystem-match block --action change --uuid 6cab53e2-b9c9-4c43-9d1d-0d8673fb62b0

In this case, it is up to the uevent listener how it proceeds with all the
uevents having this single UUID set.

Enhanced synthetic uevents support in ``libudev``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The ``libudev`` library will contain a utility function to generate
synthetic uevent on demand. The proposed function prototype for
``libudev.h`` interface is:

.. code-block:: c

  int udev_device_synth_uevent(struct udev_device *udev_device, const char *action, bool wait, unsigned long long timeout);

The ``device_synth_uevent`` function generates synthetic uevent of type
``action`` for device ``udev_device``. If ``wait`` is ``true``, the
function sets up uevent monitor internally to wait for corresponding udev
uevent and exits if the uevent is received or timeout happened with return
codes set up appropriately. The function matches synthesized uevent based
on the UUID which is set automatically by ``libudev`` for this uevent.

All synthetic uevents generated with ``device_synth_uevent`` function will
contain these ``KEY=VALUE`` pairs::

 SYNTH_UUID="<uuid>"
 SYNTH_ARG_LIBUDEV_TRIGGER="1"

This means, each time libudev writes to ``/sys/.../uevent`` file to
generate synthetic uevents in this scenario, it will use this string to
trigger the uevent::

  <action_name> <uuid> LIBUDEV_TRIGGER=1

Further functions may be provided in the future to extend this
functionality more to include sets of devices and hence providing the
functionality similar to ``udevadm trigger`` through ``libudev`` library.

Eventually, this interface should be preferred in the future by various
tools and userspace components instead of relying on ``OPTIONS+="watch"``
rule.
