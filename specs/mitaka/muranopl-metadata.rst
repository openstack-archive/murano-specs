..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=================================================
MuranoPL metadata to properties, classes, methods
=================================================

https://blueprints.launchpad.net/murano/+spec/metadata-in-muranopl

Now in MuranoPL is impossible to add metadata for properties, classes or methods,
which can be used in various situations, such as UI form definitions or action
calls. To solve this problem MuranoPL metadata is introduced.

Problem description
===================

With time and development of the language new challenges arrived and MuranoPL
need to be extended with new keywords for new features. Metadata will be used to store
new information about various MuranoPL entities which can be needed in the future.
With this new feature MuranoPL will become more flexible and go away from the
"new keyword for new feature" rule.

Proposed change
===============

To resolve a problem that was addressed above it was decided to introduce a new
way to describe data in MuranoPL - metadata. Python class called `MetaAttribute`
will be a class describing meta-classes. In MuranoPL app developers will create
their own meta-classes. Instances of those classes will be attached to classes,
properties, methods and packages.

To support metadata functionality it's planned to introduce several new class-level
keywords:

#. `Usage` which will help to separate metadata classes from normal classes. At first
   `Usage` will support only two values `Meta` and `Class`.

#. `Cardinality` (only for `Meta`) will be used to set how much values
   will be returned from one key in metadata object. Values can be `One` or `Many`.

#. `Applies` (only for `Meta`) which will define to what type of MuranoPL
   objects the metadata class can be attached. Values can be `All`, `Class`,
   `Method`, `Property`, `Package` or combinations of them, e.g. `[Method, Class]`.

Now, let's take a look at the examples of the meta-class in MuranoPL:

    .. code-block:: yaml

       Name: FooMetaOne
       Usage: MetaAttribute
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
       Usage: MetaAttribute
       Applies: [Property, Method]
       Cardinality: Many


The instances of meta-classes are going to be immutable since they are part of the
class/property/etc definitions which are also immutable. As a result there cannot
be Out/InOut properties and instances of those classes cannot have an owner and
thus cannot use `find()` function.

Instances of `FooMetaOne` class can be attached to propeties and will return only one
value to each key-value pair. To attach this class to a property is used `Meta`
keyword in a property description.

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
            - meta:FooMetaOne:
                description: "Stub metaclass"
                count: 2

As you can see from example above we organize `Meta` attribute as an array. If we
need to use two examples of the same meta-class it will look like:

    .. code-block:: yaml

       Meta:
        - meta:FooMetaOne:
            description: "Stub metaclass"
            count: 2
        - meta:FooMetaMany:
        - meta:FooMetaMany:

Metadata attributes are never inherited. It's decided to be so, because MuranoPL
supports multiple inheritance and `metadata inheritance` can produce conflicts which
will be hard to solve.

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
