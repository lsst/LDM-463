:tocdepth: 1

.. sectnum::

.. _intro:

Introduction
============

.. _change-record:

Butler
======



Overview - What is the Data Butler?
-----------------------------------

The Butler is a framework for generic I/O and data management. It isolates
application code from the underlying data-access implementation in terms of
storage formats, physical locations, data staging, database mapping, etc. Butler
is configured by a Policy that provides configuration details, as well as by
parameters provided at initialization time.

This general concept makes the Data Butler potentially able to serve as a data
*router* (or hub or switch). Data can be published and sent to multiple
locations based on the Butler configuration. Those locations may include
"persistent storages" that are actually dynamic (like displays or subscription
streams) rather than truly persistent storage.

Dataset
-------

A dataset is the persisted form of an in-memory object. It could be a single
item, a composite, or a collection. Examples: `int`/`long`, `PropertySet`,
`ExposureF`, WCS, PSF, `set`/`list`/`dict`.

Repository
----------

.. warning::

    This section describes New Butler Repositories, much of which should be
    considered internal-only, non-public API for the time being. See
    documentation in daf_persistence and daf_butlerUtils for current butler use
    and API.

A Repository refers to data storage. In persisted form (i.e. in on disk, in a
database, or in other storage) a repository contains any/all of:

- a collection of datasets
- the name of the mapper to use
- the policy that is used to configure the mapper
- a registry to expedite data lookup
- references to parent repositories.

When a Repository class is instantiated, it uses input configuration parameters
to locate or create a repository in storage. It then instantiates the Mapper and
Access classes as well as it's parent & peer Repositories.

Root
^^^^

Root is the 'top-level' location of a Repository and provides access to
within-repository items such as a Mapper, a Policy, and the Registry. These
provide further information about where actual Datasets can be found (currently
with in-filesystem repositories, the files are always stored under the folder
indicated by Root).

TBD if Root will be a proper class, a named tuple, some other data type, or just
parameters passed to an initializer in Butler, Repository, or other. It should
contain basic bootstrap info for a Repository. It seems reasonable that the
exact contents of Root should depend on the type of Storage being initialized.

The only current example of Root is in posix storage: Root is the path to the
top folder of the Repository.

Parent and Self
^^^^^^^^^^^^^^^

Repositories can point to other repositories as inputs. This relationship is
defined in the repository configuration.

Repositories read from their parent repositories. Repositories write to
themselves and do not read from themselves. Repositories can have more than one
parent, and parents can have parents. This provides what is in effect a search
path for datasets, allowing repositories to share access to datasets without
copying data and modification of data without overwriting previously written
data.

It is possible to have a Repository be both 'self' and 'parent' to allow both
input and output behavior in a single butler instance.

The search order is set by the repository configuration; parents that are first
in the list will be searched first, and search is depth-first. Repositories can
be configured to return one result or to return all results.

Peer and Self
^^^^^^^^^^^^^

Repositories can have 'peer' repositories as outputs. This allows a single call
to ``Butler.put`` to write data to multiple outputs, by writing to self and to
all the peers. Peer repositories are defined in the repository configuration.

Recursive Calls to Self and Peers
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The public Butler API methods are mostly used to make calls on mappers in self
and related repositories. The public methods implement recursive behaviors, via
``doParents`` and ``doSelfAndPeers``, which forward to 'do' methods in self and
related repositories. For example calling

 ``repository.map(datasetType, dataId, write=False)``

calls ``doMap`` via

 ``return self.doSelfAndPeers(Repository.doMap, *args, **kwargs)``.

doSelfAndPeers will call doMap on self and all the peer repositories, returning
the value or list of values.

To change or extend the recursive behavior of ``Repository``, subclass it and
override ``doParents``, ``doSelfAndPeers``, and/or public API methods as needed.

Version
^^^^^^^
This feature is still being designed and developed under
`DM-4168 - Data repository selection based on version
<https://jira.lsstcorp.org/browse/DM-4168>`_.

The stated requirement is: Must support a system so that data may be referred to
by version at repository load time at the latest. (Won't be selectable by dataId
when calling ``Butler.get(...)``) .

Configuration
^^^^^^^^^^^^^
A repository is created with a configuration specification. Details about how
configuration works can be found under `Butler Configuration`_

Mapper
------

A Mapper is used by a Repository to find datasets (when reading) or
locations for datasets (when writing). the ``Mapper`` class must be subclassed
to implement meaningful behavior. The most commonly used Mapper subclass in LSST
is ``CameraMapper``.

Typically a Mapper instance is configured by data in the policy.

Access, Storage, and Transport
------------------------------

.. warning::

    This section describes New Butler classes, and should be considered
    internal-only, non-public API for the time being.

The ``Access`` class is intended to be an interface for the ``Storage`` class.

``Access`` may also become an interface that contains connections and i/o for
remote Repositories. TBD.

Storage
^^^^^^^

.. warning::

    This section describes New Butler classes, and should be considered
    internal-only, non-public API for the time being.

Storage is intended to be a protocol (or abstract base class TBD) that defines
the api for concrete Storage classes that implement read and write access.
Storage classes can be added by client code and are to be pluggable; i.e.
provided by client code.

Concrete classes include support for one of:

* file system (FilesystemStorage or PosixStorage)
* database (DatabaseStorage)
* in-memory (InMemoryStorage)
* stream (StreamStorage)
* others, can be implemented by 3rd party users

Concrete Storage classes are responsible for implementing:

 * Concurrency control that cooperates with their actual storage.
 * Handle-to-stored-Parent for persisted data so that the parent may be found at load time.

It is worth noting that the Storage classes are interfaces and may contain the
data (e.g. in-memory storage), but they do not necessarily contain it, and in
some cases absolutely do not contain it.

Butler
------
The ``Butler`` class is the  overall interface and manager for repositories. The
Butler has a single Repository that may have zero or more input repositories and
one or more write-only output Repositories.

Butler Configuration
--------------------

.. warning::

    The Butler configuration mechanism is still being developed and details will
    be provided here once it solidifies a little more. More information about
    current use is available under `Butler with Legacy Repositories`_.

Mapper Configuration
--------------------

Policy
^^^^^^

The policy provides configuration details for the butler framework that will
access a dataset. The policy may be defined in any/all of:

1. repository
2. butler subclass
3. butler framework

If policy keys conflict, settings will override in that order, where the
in-repository settings will have highest priority.

Dataset Type
^^^^^^^^^^^^

A label given to a one or more datasets reflecting their meaning or usage
(not their persisted representation). Each dataset type corresponds to
exactly one Python type. Dataset types are used by convention by Tasks for
their inputs and outputs. Examples: `calexp`, `src`, `icSrc`.

Dataset Prototype
^^^^^^^^^^^^^^^^^

.. warning::

    Dataset Prototype is currently concept-ware and does not exist at all in
    code. See details below.

This concept is work-in-progress, and is related to making it possible to define
dataset types at runtime.
`DM-4180 - Butler: provide API so that a task can define the output dataset type
<https://jira.lsstcorp.org/browse/DM-4180>`_.

A labeled set of basic access characteristics serving as the basis for a
group of dataset types, used to define new dataset types. The characteristics
may include code, template strings, and other configuration data. Dataset
genres are often (but not necessarily) common to all dataset types with the
same Python type, making it easy for an application to select which genre is
applicable to a new dataset type that it is creating.

dataId
------
Scientifically meaningful key-value pairs used by ``butler.get`` and
``butler.put`` to select one or more datasets that should be read or written.

Butler with Legacy Repositories
-------------------------------

_parent
^^^^^^^

Until March 2016 Butler did not have a class abstraction for repositories, and
a Butler was instantiated with a single repository. That single repository could
have "parent" repositories. This allowed the repository to access datasets from
other repositories. This was implemented putting a symlink at the top level of
the repository on disk (at  the location specified by "root") named ``_parent``
whose target was the root of the parent repository.

There is still support for ``_parent`` symlinks in the locations it was used as
of March 2016 (there is minimal support in the Butler framework classes and it
is mostly used by ``CameraMapper``). To the extent possible this will be
maintained but new code and features may not make any attempt to support it.

When searching multiple repositories (current implementation; parents and peers
set by the cfg) an 'old style' repo with _parent symlinks will be treated as a
single repository. IE the _parent symlinks get followed before the next repo in
``repository._parents`` is searched.

Subset
------

ButlerSubset is a container for ButlerDataRefs.  It represents a collection of
data ids that can be used to obtain datasets of the type used when creating the
collection or a compatible dataset type.  It can be thought of as the result of
a query for datasets matching a partial data id.

The ButlerDataRefs are generated at a specified level of the data id hierarchy.
If that is not the level at which datasets are specified, the
ButlerDataRef.subItems() method may be used to dive further into the
ButlerDataRefs.

DataRef
^^^^^^^
A ButlerDataRef is a reference to a potential dataset or group of datasets that
is portable between compatible dataset types.  As such, it can be used to create
or retrieve datasets.

ButlerDataRefs are (conceptually) created as elements of a ButlerSubset by
Butler.subset().  They are initially specific to the dataset type passed to that
call, but they may be used with any other compatible dataset type. Dataset type
compatibility must be determined externally (or by trial and error).

ButlerDataRefs may be created at any level of a data identifier hierarchy. If
the level is not one at which datasets exist, a ButlerSubset with lower-level
ButlerDataRefs can be created using ButlerDataRef.subItems().

DataRefSet
^^^^^^^^^^

Logically, a set of 'DataRef's. This may be implemented as an iterator/generator
in some contexts where materializing the set would be expensive. The
'DataRefSet' is usually generated by listing existing datasets of a particular
dataset type, but its component 'DataRef's can be used with other dataset types.

Change Record
=============

+-------------+------------+----------------------------------+-----------------+
| **Version** | **Date**   | **Description**                  | **Owner**       |
+=============+============+==================================+=================+
| 0.1         | 2/15/2016  | Initial version.                 | Jacek Becla     |
+-------------+------------+----------------------------------+-----------------+
