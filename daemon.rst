****************************
Storage Instantiation Daemon
****************************

Stages and phases
=================

Storage Instantiation Daemon (SID) is layered on top of udev. Its
functionality is driven by interception of two kinds of events based on
which execution of a SID stage is initiated:

  * | **Stage A**
    | Also called *udev stage*. It is started by reception of
      ``sid scan`` request out of a udev rule while processing device's
      uevent within udevd. The stage completes by reception of
      ``sid checkpoint complete`` call out of another udev rule which is
      positioned within udev rules so that it is executed as the last one
      for current device. Therefore, we are making use of
      ``RUN+="sid checkpoint complete"`` rule to queue the call up until
      the point when all the other udev rules are processed, but before the
      udev uevent is triggered. At this stage, processing any subsequent
      uevents for this device is still blocked by udevd.

  * | **Stage B**
    | Also called *trigger-action stage*. It is started by reception of
      udev uevent for the device, that is, after all udev rule processing
      for the uevent is completely finished, including any ``RUN`` rules.
      At this stage, processing any subsequent uevents for this device is
      no longer blocked by udevd, that means, while we are processing stage
      B in SID, udevd can already be processing another uevent for this
      device.

Besides SID **core** that handles both stage A and B, SID provides a way
to load **modules** to extend the stage A and stage B processing. Each
stage is divided further into discrete **phases** where each module can
register callback functions to handle specific processing.

**Stage A** consists of these consecutive top-level phases (SID components
which handle each phase are given in square brackets):

  * | ``IDLE`` [core]
    | Awaiting request from a udev rule running ``sid scan`` command.

  * | ``INIT`` [core]
    | The ``sid scan`` request received. Initializing processing sequence.

  * | ``IDENT`` [core, module]
    | Executing basic device type identification and decision-making
      routines. The core looks up device type name in ``/proc/devices``
      based on its major number. The device type name is then used to
      determine a module to load (if not already loaded) for further
      processing and subsequent phases.

  * | ``SCAN`` [core, module]
    | This is the main phase executing device scanning, decision-making
      routines and storing collected information. At this phase, we are
      able to schedule possible actions to execute during stage B
      processing.

  * | ``WAIT`` [core]
    | Awaiting a confirmation from a udev rule running ``sid checkpoint
      complete`` command.

There is one specialized error handling phase:

  * | ``ERROR`` [core, module]
    | Handling a failure hit in any of previous phases.

**Stage B** is responsible for executing actions and it consists of one
phase:

  * | ``TRIGGER-ACTION`` [core, module]
    | Checking trigger conditions and executing associated actions.

Phase modules
-------------

The modules can handle ``IDENT``, ``SCAN`` and ``ERROR`` phase - we call
these *phase modules*. SID differentiates between two types of phase
modules:

  * | **generic phase module**
    | This is a module that is always executed for a phase by SID,
      irrespective of  what the detected device type within ``IDENT`` and
      ``SCAN`` phase is.

  * | **dedicated phase module**
    | This is a module that is executed only if it matches detected device
      type within ``IDENT`` and ``SCAN`` phase.

Database
========

SID uses a ``KEY=VALUE`` in-memory database as its information storage
backend. The database consists of two layers:

  * | **Database abstraction layer**
    | This is a layer that abstracts away actual access to a database. This
      way, it is possible to support and choose from various low-level
      database backends without a need to modify the layers above.

  * | **SID database layer**
    | This layer provides access to SID's own database for modules to
      access through SID module API.

Database abstraction layer
--------------------------

The database abstraction layer defines common API to access a ``KEY=VALUE``
database where the key at this layer is always defined as a null-terminated
string. All values have defined size and the value itself is either:

  * | **Single value**
    | Here, the size is the actual size of the value. The value is a direct
      pointer to the object in memory.
      

  * | **Value vector**
    | Here, the size is the number of elements in the vector. The value is
      a pointer to the vector. Each element of the vector is then a touple
      ``[pointer, size]``, where the pointer is a direct pointer to the
      object in memory. The elements of the vector are not ordered.

The database abstraction layer API supports:

  * | **Storing and retrieving values**
    | The way the value is stored is controlled by providing *flags*:

      * | no flags
        | Defaults are used: a value is a single value and the object in
          memory is copied before storing it in the database.

      * | ``VECTOR``
        | The value is a value vector.

      * | ``REF``
        | The value is stored directly as a pointer to the object in
          memory, that is, the value is not copied before storing it in the
          database.

      * | ``AUTOFREE``
        | The value is automatically freed as soon as it is no longer
          stored in the database.

  * | **Iterating through existing keys and values**

  * | **Calling callback functions on key conflicts**
    | Whenever a key already exists when trying to store a value with
      provided key, a *key conflict resolver* (if provided) is called. The
      conflict is then resolved by one of:

        * | **Using the newly provided value**
          | The newly provided value is used by default if no key conflict
            resolver is defined or if the resolver confirms it.

        * | **Keeping the existing value**
          | The existing value is used if the resolver confirms the
            existing value.

        * | **Creating a new value on the fly**
          | The resolver can create a completely new value on the fly based
            on the existing and newly provided value. Then the created
            value is stored instead of the newly provided value.


SID database layer
------------------

This layer handles the actual SID database content which is used by SID's
core as well as its modules.

The main daemon process contains master copy of the database which is then
shared with all SID worker processes. The database is available as long as
SID main process is running.

Each time stage A is started, a snapshot of the master database is created
which is used throughout the whole stage A processing. At the end of stage
A, all the changes in the snapshot copy are synchronized with the master
copy.

When stage B processing starts, again, a new snapshot of the master copy
of the database is created and then this one is used throughout the whole
stage B processing. At the end of stage B, the changes in the snapshot are
synchronized with the master copy.

The snapshotting supports complete access to database records without a
need to take locks when performing database reads or writes as well as
consistent views of the whole database while accessing it even several
times during either A or B stage processing.

Internally, the key is compounded of these **key parts** each separated by
``:`` character:

  * | ``prefix``
    | The *prefix* is reserved for operation specification. Currently, this
      is used for snapshot copy synchronization:

        * | blank
          | No operation.

        * | ``+``
          | Add a value to master record.

        * | ``-``
          | Remove a value from master record.

  * | ``ns``
    | This stands for *namespace* and it is used to separate records into
      top-level categories:

        * | ``U``
          | *Udev* namespace. This is a virtual namespace which is not
            recorded in master database. Instead, it is available only in
            the snapshots during stage A and stage B processing. This
            namespace contains all variables which were available and then
            imported at the time of the ``sid scan`` request.

        * | ``D``
          | *Device* namespace containing records which are unique per
            device.

        * | ``M``
          | *Module* namespace containing records which are unique per
            module.

        * | ``G``
          | *Global* namespace containing global which are globally unique.

  * | ``ns_part``
    | This stands for *namespace partition*. It is bound to the ``ns``
      field and it specifies it further:

        * | ``major_minor``
          | Device number for ``U`` udev namespace.

        * | ``major_minor``
          | Device number for ``D`` device namespace.

        * | ``module_name``
          | Module's name for ``M`` module namespace.

        * | blank
          | Used for ``G`` global namespace.

  * | ``dom``
    | This stands for *domain*. It is a domain of the record within the
      ``ns:ns_part`` pair. Currently, these domains are used:

        * | ``USR``
          | User domain (user-specified records).

        * | ``LYR``
          | Device layer domain. Records describing device layering and
            associated dependencies.

  * | ``id``
    | This stands for *identifier*. That is the main identifier for the
      record.

  * | ``id_part``
    | This stands for *identifier partition*. It is bound to the ``id``
      field and it specifies it further.

The complete **compound key** is then::

  prefix:ns:ns_part:dom:id:id_part

The **value** is either a *single value* or a *set of values*. Each value
always has specified size. The values are not limited to strings only and
it is possible to store raw binary values.

.. note::

  Future revisions of SID should provide a database that is persistent over
  SID's restarts and system reboots. Such database would provide useful
  hints  when devices, layers and whole stacks are discovered and
  instantiated.
