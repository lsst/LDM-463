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

Parents
^^^^^^^

Repositories may have none or more parents that are other repositories. These
repositories are used by the butler as inputs. Repositories' parent
configuration are persisted along with the repository's configuration.

When a repository is used as a Butler output, all of the current input
repositories are added to its list of parent repositories.

Repository Version
^^^^^^^^^^^^^^^^^^
This feature is still being designed and developed under
`DM-4168 - Data repository selection based on version
<https://jira.lsstcorp.org/browse/DM-4168>`_.

The stated requirement is: Must support a system so that data may be referred to
by version at repository load time at the latest. (Won't be selectable by dataId
when calling ``Butler.get(...)``) .

Configuration
^^^^^^^^^^^^^
A repository is created via configuration specification. Details about how
configuration works can be found under `Repository Configuration`_

Butler
------
The ``Butler`` class is the  overall interface and manager for repositories. 
The Butler init function takes a list of input and output repositories (see
below for a description of inputs and outputs) that are used as locations for
i/o.

Inputs and Outputs
^^^^^^^^^^^^^^^^^^

Butler will only perform read actions on input repositories and will perform
read and write actions on output repositories. Repositories may also have an
internal mode that can be one of:

* read
* write
* read-write 

Internally Butler keeps a list of input repositories and a list of output
repositories. When an output repository's mode is read-write it will also be
added to the list of inputs as well as the list of outputs. Attempting to pass a
read-only repo as a butler output or a write-only repo as a butler input will
raise an exception.

Input repository configurations must specify certain parameters. In some cases
output configurations may be more sparsely populated and derive parameter values
from input configs, but the inputs must be uniform. See the next section.

Output configuration derived from inputs
""""""""""""""""""""""""""""""""""""""""

Some settings for output configurations can be derived from input
configurations. For example, if an output configuration does not specify a
mapper, the input mapper may possibly be assumed (this will work as long as all
the input repositories use the same type of mapper; if the inputs use different
types of mapper then a single type mapper can not be inferred to use for the
output repositories). When possible the butler will use settings from input
configurations to complete output configurations. 

Search Order
""""""""""""
The order of repositories passed to inputs and outputs is meaningful; search is
depth-first and in order (left to right). Parents of inputs will be searched
before the next passed-in input. See the attached diagram.

Tagging 
^^^^^^^

Input repositories can be “tagged” with a temporary id that gets used when
reading from a repository. ``RepositoryCfg`` will have a method to ‘tag’ a
repository with a value or object. A repository can be tagged with more than one
tag by passing in a container of tags. The tag is not persisted with the
repository.

When performing read operations on the butler, if the DataId contains one or
more tags, the repository will only be used for lookups if it is also tagged
with one of the tags in the DataId. If the DataId has no tags, then all input
repositories will be used. More information about DataId and its tag are
available in the DataId section.

Repository Configuration
^^^^^^^^^^^^^^^^^^^^^^^^

``RepositoryCfg`` instances are used to instantiate repositories in Butler. It is
best to create the ``RepositoryCfg`` by using the member function
``Repository.cfg(...)``. Its parameters include:

* ``mode`` 
    * Required 
    * string - This can be one of 'r', 'w', or 'rw' (read, write, read-write). 
    * It is used to indicate the read/write state of the repositories. Input
      repositories are always read-only and an exception will be raised if
      the mode of an input repository is 'w', but it may be 'rw'. Output
      repositories must be 'w' or 'rw'. If it is 'rw' the repository will
      also be used as an input repository.
* ``mapper`` 
    * Optional if the repository already exists and specifies its mapper or if
      the mapper can be inferred from the input repositories (if more than 1
      input repository, for the mapper to be inferred they must all use the
      same mapper). Otherwise required.
    * Can be an importable & instantiatable string (e.g.
      ``lsst.daf.persistence.CameraMapper``), an class object, or a class
      instance.
    * This specifies the mapper to be used by the repository.
* ``mapperArgs``
    * Optional
    * dict
    * These arguments are passed to the mapper when it is being instantiated (if
      it needs to be instantiated). If the mapper requires root it does not need
      to be included in mapperArgs. When creating the mapper if root is needed the
      butler will get root from storage and use that. 
* ``storageCfg``
    * Required
    * Instance of a StorageCfg
    * This is used to instantiate a storage class for the repository. The
      easiest way to create this is to use the ``cfg`` method of the ``Storage``
      subclass. For example, for Posix storage, use ``PosixStorage.cfg(root=...)``
* ``parentCfgs``
    * Optional
    * Instance of a ``RepositoryCfg``
    * This is used to indicate the parents of a repository. In normal use the
      Butler will be setting the parents on output repositories. However
      there may be times when it will be useful for users to indicate a
      parent relationship using this mechanism.
* ``tags``
    * Optional
    * Any tag type
    * Indicates the tags that a repository should be labeled with in the
      butler. (There is also a member function of ``RepositoryCfg`` to set
      tags on an instantiated cfg.

What parameters of a ``RepositoryCfg`` must be populated depends on how it is
going to be used:

* Ready to use as Butler input:
    * Points to a location of an existing repository but no other info is known.
        * Mapper not defined.
        * Storage & Root must be defined.
        * Parents not defined.
        * Mode not defined.
    * Info is known (already deserialized cfg or otherwise not deserializing)
        * Mapper might be defined (in Butler it could be inferred from input repository/repositories).
        * Storage & Root must be defined.
        * Parents must be defined (if any).
        * Mode must be defined.
* Ready to use as Butler output
    * Mapper may be defined. (If it is not and Butler’s inputs all have the
      same mapper then that mapper will be added to the cfg. If Butler’s
      inputs have different mapper types then Butler will throw instead of
      assigning the mapper).
    * Storage & Root must be defined.
    * Parents must not be defined. (The Butler’s inputs will be added as parents to the cfg).
    * Mode must be defined (one of ‘w’ or ‘rw’. ‘r’ (read-only) will throw).

Mapper
------

A Mapper is used by a Repository to find datasets (when reading) or
locations for datasets (when writing). the ``Mapper`` class must be subclassed
to implement meaningful behavior. The most commonly used Mapper subclass in LSST
is ``CameraMapper``.

Typically a Mapper instance is configured by the Policy.

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

 * Concurrency control that cooperates with their actual storage. Handle-to-
 * stored-Parent for persisted data so that the parent may be found at load
   time.

It is worth noting that the Storage classes are interfaces and may contain
datasets (e.g. in-memory storage), but they do not necessarily contain datasets,
and in some cases absolutely do not contain them.


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

DataId
------
A class that extends dict. As a dict it contains scientifically meaningful
key-value pairs the mapper to find a lcoation of one or more datasets that
should be read or written.

It also contains a member variable called ``tag``:

* ``tag`` may be a string or other type, including container types. When
  searching repositories, if the tag argument is not None, then repositories will
  only be searched if their tag equals the value of tag (or if a match is found in
  either container of tags).
* When searching, if an input repository is tagged, all of its parents will be
  searched (even if they do not have a tag).
* The Butler API allows a dict to be passed instead of a DataId; as needed it
  will constructed dict into a DataId object (with no tags) internally.

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