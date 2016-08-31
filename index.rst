:tocdepth: 1

.. sectnum::

.. _intro:

Introduction
============

.. _change-record:

There are two primary methods of data access. The first, which appears
primarily as a library to the user, is the Butler. The second is a set of HTTP
interfaces collectively known as the DAX APIs or loosely referred to as
webserv. There may be some overlap between the two of these; in some cases,
the Butler may call out to the DAX APIs or the DAX APIs may use the Butler
underneath, and we plan to have a DAX API which operates as a Butler bridge
service. This last service will serve primarily as a proxy for remote
applications which may be coded in any language.


Butler
======

Overview - What is the Data Butler?
-----------------------------------

The Butler is a framework for generic I/O and data management. It isolates
application code from the underlying data-access implementation in terms of
storage formats, physical locations, data staging, database mapping, etc. 

Butler stores data in and retrieves data from Repositories. Each repository has
a mapper that is used to find data in the repository. The mapper is configured
by a Policy that provides configuration details.

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

When a Repository class is instantiated, it uses input configuration arguments
to locate or create a repository in storage. It then instantiates the Mapper and
Access classes as well as its parent & peer Repositories.

If the configuration arguments point to a repository that already exists then
the arguments must be consistent with the stored repository configuration. This
includes which mapper to use and what the parents of that repository are.

Root
^^^^

Root is the 'top-level' location of a Repository and provides access to
within-repository items such as RepositoryCfg_, a Policy, and the Registry.
These provide further information about where actual Datasets can be found
(currently with in-filesystem repositories, the files are always stored under
the folder indicated by Root). 

Root is passed to the Butler by URI.

CfgRoot
^^^^^^^
If a RepositoryCfg_ is not located within the repository it can exist at a 
"configuration root". It's ``root`` parameter must indicate where the repository
is.

Parents
^^^^^^^

Repositories may have zero or more parents that are other repositories. These
repositories are used by the butler as inputs. Repositories' parent
configuration are referenced by URI in the repository configuration.

When a repository is used as a Butler output, all of the butler's input and
writable output repositories are listed in that output repository's list of
parent repositories. (But not parents-of-parents etc; these are found again the
next time the parent is loaded).

Repository Version
^^^^^^^^^^^^^^^^^^
This feature is still being designed and developed under
`DM-4168 - Data repository selection based on version
<https://jira.lsstcorp.org/browse/DM-4168>`_.

The stated requirement is: Must support a system so that data may be referred to
by version at repository load time at the latest. (Won't be selectable by dataId
when calling ``Butler.get(...)``) .

Butler
------
The ``Butler`` class is the  overall interface and manager for repositories. 
The Butler init function takes a list of input and output repositories (see
below for a description of inputs and outputs) that are used as locations for
i/o.

The new butler initializer API is ``Butler(inputs=None, outputs=None)``. Values
for both ``inputs`` and ``outputs`` can be an instance of the ``RepositoryArgs``
class or can be a string. Additionally, the value can be either a single item or
many items in a sequence container. If the value is only a string, it is treated
as a URI. In inputs it must refer to a persisted ``RepositoryCfg`` and in
outputs it can refer to an existing ``RepositoryCfg`` or can be a location to
create a new repository.

Inputs and Outputs
^^^^^^^^^^^^^^^^^^

Butler will only perform read actions on input repositories and will perform
read and write actions on output repositories. Repositories may also have a
mode that can be one of:

* read
* write
* read-write 

When the mode of an output repository is read-write it will also be used as an
input. Attempting to pass a read-only repo as a butler output or a write-only
repo as a butler input will raise an exception.


Already existing Repositories as Outputs
""""""""""""""""""""""""""""""""""""""""

When initializing a ``Butler`` with one or more output repositories that already
exist it is important to not add any input repositories that are not already
parents of the output repository. This is because all butler inputs (including
readable outputs) become parents of output repositories, and the repository
configuration is not allowed to change after it has been written.

Output arguments derived from inputs
""""""""""""""""""""""""""""""""""""

Some settings for output repositories can be derived from input repository
configurations. For example, if an output configuration does not specify a
mapper, the input mapper may possibly be assumed (this will work as long as all
the input repositories use the same type of mapper; if the inputs use different
types of mapper then a single type mapper can not be inferred to use for the
output repositories). When possible the butler will use settings from input
configurations to complete RepositoryArgs parameters for output repositories. 

Search Order
""""""""""""
The order of repositories passed to inputs and outputs is meaningful; search is
depth-first and in order (left to right). Readable outputs will be searched
before inputs. Parents of readable outputs/inputs will be searched before the
next passed-in output/input.

Tagging 
^^^^^^^

Readable repositories can be “tagged” with an identifier that gets used when
reading from a repository. A tag can be specified in ``RepositoryArgs`` when
initializing a Butler. A repository can be tagged with more than one tag by
passing in a container of tags. The tag is not persisted with the repository.

When performing read operations on the butler, if the DataId contains one or
more tags, the repository will only be used for lookups if it is also tagged
with one of the tags in the DataId. If the DataId has no tags, then all input
repositories will be used. More information about DataId and its tag are
available in the DataId section.

RepositoryArgs
^^^^^^^^^^^^^^

``RepositoryArgs`` instances are used to instantiate repositories in Butler. Its
parameters are:

* ``mode`` 
    * Optional.
    * string - This can be one of 'r', 'w', or 'rw' (read, write, read-write). 
    * It is used to indicate the read/write state of the repositories. Input
      repositories are always read-only and an exception will be raised if the
      mode of an input repository is 'w'. It may be 'rw' but for inputs the
      behavior will be the same as 'r'. Output repositories must be 'w' or 'rw'.
      If it is 'rw' the repository will also be used as an input repository. If
      mode is not specified, outputs will default to 'w' and inputs will default
      to 'w'.
* ``mapper`` 
    * Optional if the repository already exists - for inputs it's better to
      leave this parameter empty. For outputs it's optional if the mapper can be
      inferred from the input repositories and is otherwise required.
    * Can be an importable & instantiatable string (e.g.
      ``lsst.daf.persistence.CameraMapper``), an class object, or a class
      instance.
    * This specifies the mapper to be used by the repository.
* ``mapperArgs``
    * Optional
    * dict
    * These arguments are passed to the mapper when it is being instantiated (if
      it needs to be instantiated). If the mapper requires Root_ it does not
      need to be included in mapperArgs. When creating the mapper if Root_ is
      needed the butler will get Root_ from storage and use that. 
* ``root`` and ``cfgRoot``
    * at least one is required.
    * string URI
    * ``root`` is a URI to where the repository or repositoryCfg is (if it
      exists already) or to where the repository should be (if it does not exist
      yet). If the RepositoryCfg is or should be stored separately from the
      repository then ``cfgRoot`` should be a URI to the persisted RepositoryCfg.
* ``tags``
    * Optional
    * Any tag type
    * Indicates the tags that a repository should be labeled with in the
      butler. (There is also a member function of ``RepositoryCfg`` to set
      tags on an instantiated cfg.)

If the repository already exists it is best to only to populate:

 * ``root`` (required, to find the repository cfg)
 * ``tags`` - if any are to be used.
 * ``mode`` - for output repositories that should be readable.
 
If ``mapper`` and/or ``mapperArgs`` are populated and the value in args does not 
match the value of the persisted RepositoryCfg an exception will be raised.

Details about the repository configuration are persisted in the
``RepositoryCfg`` object when it is serialized. ``RepositoryArgs`` parameters
that do not appear in the ``RepositoryCfg`` are not persisted (``mode``,
``tags``).

RepositoryCfg
^^^^^^^^^^^^^

When a ``Repository`` is initialized by Butler its ``RepositoryCfg`` is persisted.
The ``RepositoryCfg`` is written only once and can not change. The ``RepositoryCfg``
parameters are:

* ``root``
    * Required (but may not appear in persisted RepositoryCfg). This field is
      populated in the persisted cfg in the case where the cfg is not stored in
      the repository. If the persisted cfg is at the root of the repository then
      the field is left blank.
    * string URI
    * This is a URI to the root location of the repository.
* ``mapper``
    * Required 
    * Can be an importable & instantiatable string (e.g.
      ``lsst.daf.persistence.CameraMapper``), an class object, or a class
      instance.
    * This indicates the mapper to use with this repository
* ``mapperArgs``
    * Required
    * dict or None
    * These arguments are passed to the mapper when it is being instantiated (if
      it needs to be instantiated and the mapper parameter does not have the
      args packed into that value). If the mapper requires root it does not need
      to be included in mapperArgs. When creating the mapper if Root_ is needed
      the butler will get root from storage and use that. 
* ``parents``
    * required
    * list or None
    * This is a list of URI to the ``RepositoryCfg`` of each parent.
* ``_isLegacyRepository``
    * not persisted, required in instantiated RepositoryCfg (but is instantiated
      via the cfg reader).
    * bool
    * this is used to mark when a RepositoryCfg was synthesized by reading an
      old-style repository filesystem layout, including reading the _mapper
      file and populating root from the repository's root. In the case where
      _isLegacyRepository is True, the RepositoryCfg is never persisted; the
      next time the repository is used the cfg will be synthesized again.    
    
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
key-value pairs the mapper to find a location of one or more datasets that
should be read or written.

It also contains a member variable called ``tag``:

* ``tag`` may be a string or other type, including container types. When
  searching repositories, if the tag argument is not None, then repositories will
  only be searched if their tag equals the value of tag (or if a match is found in
  either container of tags).
* When searching, if an input repository is tagged, all of its parents will be
  searched (even if they do not have a tag).
* The Butler API allows a dict to be passed instead of a DataId; as needed it
  will convert the dict into a DataId object (with no tags) internally.

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


DAX
===

Overview - What is DAX?
-----------------------

DAX is a collection of Data Access Interfaces implemented as REST APIs.
Currently, there are three core APIs: dbserv, metaserv, and imgserv. They are
all implemented in python using the `Flask framework
<http://flask.pocoo.org/>`_. The collection of these APIs referred to
as webserv.


dbserv
------
dbserv is the REST API for databases. The primary target for dbserv will be as
a frontend for QServ, but dbserv is generic enough to be used in front of any
database, providing the user a uniform interface and JSON data structure back.
dbserv's interface borrows heavily from the `IVOA Table Access Protocol
<http://www.ivoa.net/documents/TAP/20100327/REC-TAP-1.0.html>`_. While we aim
to provide a complete, TAP-compliant implementation, dbserv is currently a
small subset TAP. We implement only the `/sync` endpoint, and we also return
a JSON format

.. Link to dberv API here once we get sphinx autodoc works for dax_dbserv

Design Requirements
^^^^^^^^^^^^^^^^^^^

dbserv has two primary requirements. The first requirement is to behave as an
abstraction for database access, providing a portable way of performing both
synchronous and asynchronous queries, providing raw data access to any LSST
database through an HTTP API.  The second requirement of is, effectively, to
implement the features in TAP. This can be broken down into four parts:

The second requirement of is, effectively, to implement the features in TAP.
This can be broken down into four parts:

1. Serve and package data to the proper file format
   * (JSON, FITS, VOTable, HDF5, SQLite)
2. Implement ADQL (in some cases, a subset of ADQL)
3. Semantically-useful metadata about results (e.g. UCDs)
4. Handle table uploads from users

The current implementation of dbserv meets a small subset of these requirements.


Future work
^^^^^^^^^^^

In order to best meet these and future requirements moving forward, dbserv
will likely split into two components.

For the first requirement, we will have a component will behave as a lower
level API to the databases, optimized for the datacenter and simplicity of use.
While TAP could conceivably meet most of the needs, TAP's abstractions aren't
efficient for server to server communications, namely streaming, which is an
issue for results larger than a gigabyte.

The second component will act as an intermediary between the user and the lower
level component. In the common case of an ADQL query, this component will parse
the query, validate tables and columns, retrieve UCDs from metaserv (where
appropriate), and possibly rewrite the query to plain SQL for the lower level
API. If the downstream database does not directly implement asynchronous
queries, like the L2 database for example, this component will directly
implement them, otherwise it will defer to the implementation. Finally, this
component will be in charge of stitching together the raw data into an
appropriate file format and attaching semantic metadata about the results to
the file, like UCDs, whenever possible

In order to serve up UCDs and other semantic metadata about a query and/or it's
results, this second component will likely provide an API which a user might
also use in conjunction with the lower level API to mimic the functionality of
the full TAP implementation with the performance benefits of the lower level
API. This will likely be most useful for efficient server-to-server
communications, the likely customer being SUIT. It's also possible this API may
actually be implemented in metaserv.

imgserv
-------

imgserv is a lightweight API to serve up images. The current implementation of
imgserv uses the Butler to acquire relevant images, and imgserv transforms
those images in-process, providing cutouts of images. imgserv may also grow
into a collection of services including
`IVOA's Simple Image Access <http://www.ivoa.net/documents/SIA>`_  and
`SODA <http://www.ivoa.net/documents/SODA>`_.

.. Link to imgserv API here once we get sphinx autodoc works for dax_imgserv

metaserv
--------

metaserv is both a metadata database for dataset repositories and the API to
query that database. Currently, the only repositories that are supported are
databases repositories, and metaserv stores information about tables and their
columns, like UCDs. metaserv will likely grow to include metadata about images
and more generally, Butler repositories. It is not necessarily a goal of
metaserv to facilitate data discovery.

.. Link to metaserv API here once we get sphinx autodoc works for dax_metaserv


Future DAX Services
-------------------

We recognize the need for a Butler service which can act serve as a proxy for
remote Butler instantiations and also serve as a generic gateway for languages
where no Butler implementation exists, like Java. This use case is especially
desired by the SUIT team.


Change Record
=============

+-------------+------------+----------------------------------+-----------------+
| **Version** | **Date**   | **Description**                  | **Owner**       |
+=============+============+==================================+=================+
| 0.1         | 2/15/2016  | Initial version.                 | Jacek Becla     |
+-------------+------------+----------------------------------+-----------------+
