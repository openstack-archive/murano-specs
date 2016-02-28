..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=================================================
MuranoPL metadata to properties, classes, methods
=================================================

https://blueprints.launchpad.net/murano/+spec/metadata-in-muranopl

MuranoPL metadata is a way to attach additional information to various MuranoPL
entities such as classes, packages, methods etc. That information might be used
by both applications (to implement dynamic programming techniques) or by the
external callers (API consumers like UI or even by the Murano Engine itself
to impose some runtime behavior based on well known meta values).

Problem description
===================

With time and development of the language new challenges arrived and MuranoPL
need to be extended with new keywords for new features. Metadata will be used
to store new information about various MuranoPL entities which can be needed
in the future. With this new feature MuranoPL will become more flexible and
go away from the "new keyword for new feature" rule.

Proposed change
===============

The proposed solution is to introduce another type of classes - meta-classes.
Meta classes are similar to regular classes but has additional attributes
that control how and where instances of that meta-class can be attached.

To distinguish meta-classes from regular classes new class-level attribute
will be introduced called `Usage`. When `Usage` is `Class` (which is a default)
the rest of markup is interpreted as a class. Usage `Meta` is used ot define
meta-class.

In addition to Usage the following attributes are available for meta-classes:

#. `Cardinality` - either `One` or `Many` - controls if there can be more than
   one instance of the meta-class attached to a single language entity.
   Default is `One`.

#. `Applies` - one of `Package`, `Type`, `Method`, `Property`, `Argument` or
   `All` - controls to which of the language entities instances of the meta-
   class can be attached. It is possible to specify several values using YAML
   list notation. Default is `All`.

#. `Inherited` - `true` or `false` - specifies if the metadata retained for
   child classes, overridden methods and properties. Default is `false`.

Now, let's take a look at the examples of the meta-class in MuranoPL:

    .. code-block:: yaml

       Name: FooMetaOne
       Usage: Meta
       Applies: Property
       Cardinality: One
       Properties:
         description:
          Contract: $.string()
          Default: null
         count:
          Contract: $.int()
          Default: null


    .. code-block:: yaml

       Name: FooMetaMany
       Usage: Meta
       Applies: [Property, Method]
       Cardinality: Many

The instances of meta-classes will never have an owner and thus cannot use
`find()` function.

Instances of `FooMetaOne` class can be attached to properties only and each
property may have at most on attached `FooMetaOne` instance.
To attach this class to a property is used `Meta` keyword in a property
description.

    .. code-block:: yaml

       Namespaces:
         =: io.murano.apps.apache
         std: io.murano
         res: io.murano.resources
         sys: io.murano.system
         meta: io.murano.meta
       Name: ApacheHttpServer
       Extends: std:Application
       Properties:
         enablePHP:
           Contract: $.bool()
           Default: false
         instance:
           Contract: $.class(res:Instance).notNull()
           Meta:
              meta:FooMetaOne:
                description: "Stub metaclass"
                count: 2

In example above Meta keyword has a scalar value because it is only one
instance get attached. However it can also be an array:

    .. code-block:: yaml

       Meta:
        - meta:FooMetaOne:
            description: "Stub metaclass"
            count: 2
        - meta:FooMetaMany:
        - meta:FooMetaMany:

Metadata can be accessed from MuranoPL using reflection capabilities and from
Python code using existing yaql mechanism (additional yaql smart type/helper
interface may be needed to simplify the task).

Alternatives
------------

Add new keyword for new feature when we need it.

Data model impact
-----------------

None

REST API impact
---------------

None

Versioning impact
-----------------

None

Other end user impact
---------------------

None

Deployer impact
---------------

None

Developer impact
----------------

None

Murano-dashboard / Horizon impact
---------------------------------

None


Implementation
==============

Assignee(s)
-----------

Primary assignee:
 starodubcevna

Work Items
----------

#. Create a new basic class instead of `MuranoType`.

#. Create 2 new classes which will inherit from new basic class - one for regular
   data structures and the other one will be `MetaAttribute`.

#. Provide possibility to create instances of meta-classes.

#. Provide an access to meta-classes.

#. Create a mechanism to attach instances of meta-classes to related objects and
   store it.

Dependencies
============

None


Testing
=======

New unit tests should be added to packages.

Documentation Impact
====================

New documents about metadata usage should be added to the documents pool.

References
==========

None
