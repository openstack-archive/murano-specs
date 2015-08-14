..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============================
MuranoPL object construction
============================

https://blueprints.launchpad.net/murano/+spec/object-construction

Object construction is ability to create instances of MuranoPL classes
(both YAML and Python-based) and initialize it property. It also directly
affect relations between those 2 types of classes just because they can
inherit from each other and have initializer of their own


Problem description
===================

There are several major problems with current object initialization:

* object created by ``new()`` function will always have an owner set to null.
  Because of this you cannot create objects that expect themselves to be
  nested in something (e.g. say ``find(std:Environment).require()``)
* there are ``initialize()`` method in Python classes and method with the same
  name in YAML part, but they work completely different. YAML version is
  currently doesn't invoked at all for ``new()`` making it useful for Python
  classes only
* Python initializer may me called instead of YAML version in some cases as
  they have the same name
* Python classes inherit from MuranoObject even if they not aware of it.
  Because MuranoObject has ``__init__`` with many parameters and user
  classes have constructor parameters of their own user class cannot have
  ``__init__`` or it will break MuranoObject. It looks very strange for a class
  to inherit from Python's object and not being able to have ``__init__``.
* also because MuranoObject has many methods and properties of its own if user
  class will define method with the same name it will break how MuranoPL work.

Proposed change
===============

* rename YAML initialize and destroy methods to ``.init"`` and ``.destroy``
  as they are special. Make class loader automatically replace old names with
  new ones for backward compatibility. Dot-prefixed names cannot be called from
  MuranoPL directly as this will be a YAQL syntax error so there is no more
  need to have special treatment for those names in MuranoDslExecutor and
  block them from being called explicitly
* ``.init`` will be allowed to have parameters. During object initialization
  object model values will go into .init parameters instead of properties
  if parameter name matches key name in object model. Usual parameter contract
  system applies here
* because ``.init`` need to be able to save values it receives in parameters to
  corresponding objects it should be allowed to modify properties with
  usage=In that are immutable in normal operation mode. The same extends to
  functions that get called during initialization
* separate Python classes from MuranoObject. Classes will no longer inherit
  from MuranoObject and thus will have ``__init__`` instead of initialize.
  MuranoObject will store their native part in a property called ``extension``
* using YAQL injected parameters provide Python class with interface to access
  its MuranoObject counterpart and invoke methods defined in YAML
* initialization should go as following:
    1. YAML-defined properties initialized (in 1 pass for ``new()``, 2 passes
       for object model load) except for those that can be found in ``.init``
       parameters
    2. Python ``__init__`` get called if present. The same property value used
       to initialize object used as ``__init__`` parameters (i.e. subset of
       them since ``__init__`` may have less parameters then number of declared
       properties or not to have parameters at all)
    3. For MuranoObject counterpart all ``.init`` methods in hierarchy get
       invoked starting from the top-level classes and down to the class that
       we are initializing. It should be invoked with special flag passed in
       context that will allow it to write to even read-only properties.
       Property values that were skipped on first stage become ``.init``
       parameters on each hierarchy level. It is ``.init``'s job to set
       corresponding properties (that might have different contract)
* Add ability to explicitly pass created object owner as a ``new()`` parameter
* Update object model serializer so that if object A is specified as an owner
  of object B but doesn't references it and instead referenced by some other
  object C that is nested inside A then reattach B to C. This will not break
  object A since it remains nested in B. If A is referenced by several objects
  conforming this criteria any one of them can be used.


Alternatives
============

None

Data model impact
=================

None

REST API impact
===============

None

Versioning impact
=================

* initialize and destroy will automatically be renamed to .init and .destroy
  by the class loader so previously written apps won't break
* because currently initialize cannot have parameters at all backward
  compatibility retained
* the same is true for ability to modify read-only properties in initializer

Other end user impact
=====================

Users gets ability to implement things like auto-scaling with
ability to create instances from within MuranoPL

Deployer impact
===============

None

Developer impact
================

None

Murano-dashboard / Horizon impact
=================================

None

Implementation
==============

Assignee(s)
```````````

Primary assignee:
  Stan Lagun <slagun>


Work Items
``````````

Bullets from "proposed changes" section may be used as a work items directly


Dependencies
============

https://blueprints.launchpad.net/murano/+spec/migrate-to-yaql-vnext

Testing
=======

Please discuss how the change will be tested. We especially want to know what
tempest tests will be added. It is assumed that unit test coverage will be
added so that doesn't need to be mentioned explicitly, but discussion of why
you think unit tests are sufficient and we don't need to add more tempest
tests would need to be included.

Is this untestable in gate given current limitations (specific hardware /
software configurations available)? Is this untestable in murano-ci? If so,
are there mitigation plans (3rd party testing, gate enhancements, etc).


Documentation Impact
====================

Mentioned changes need to be included into MuranoPL documentation

References
==========

None
