..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=================
Murano versioning
=================

https://blueprints.launchpad.net/murano/+spec/murano-versioning

This specification proposes set of arrangements that are going to allow Murano
keep backward compatibility with applications written for older version while
still making possible to introduce incompatible changes to any part of the
project.

Problem description
===================

Murano is a complex system that comprises from a number of services and
technologies including several domain specific/markup languages. All of them
evolve and change from release to release. New features are introduced, code is
being refactored and not always remains compatible with previous release.
But applications written for previous Murano versions must still work on newer
releases otherwise all applications would require at least retesting and often
code changes on each release.

Proposed change
===============

Because there are many different parts in Murano that can be broken it is
necessary to have versioning support to manage backward compatibility.


Murano packages versioning
''''''''''''''''''''''''''

Murano packages will have new optional attribute in their manifest called
``Version``. It will be standard SemVer format version string consisting
of 3 parts: ``Major.Minor.Patch`` and optional SemVer suffixes
``[-dev-build.label[+metadata.label]]``.

All MuranoPL classes are said to have version of the package they contained in.
If no version specified package version will be equal to "0.0.0"

Version number will become very important not just to distinguish modifications
of the same package but to determine compatibility between them. Two classes
(packages) said to be compatible for particular operation if class with newer
version number can be used instead of older class version without breaking
application/component written for that older version.

Compatibility rules differ for various class usages but they all assume
semantic versioning rules be strictly followed. Changes which break
backwards-compatibility should increment the major segment, non-breaking new
features increment the minor segment and all non-breaking bug fixes increment
the patch segment.

Changes that require major version segment to be incremented:
 * class was removed from package
 * public method or property was removed from any class in the package
 * public method changed argument signature to incompatible with previous one.
   For example argument removed or added without default value
 * input property (or initializer argument) was added without ``Default`` value
   and restrictive contract (e.g. one must provide its value)
 * any other change (including in class behavior) that can break class
   consumers (callers)

Minor version segment should be incremented upon:
 * new class was introduced in the package
 * new method or property was introduced in any package's class
 * list of parents (``Extends`` section) changed for any class
 * any other interface (including behavior) changes that don't break external
   class users but can possibly break class inheritors

For all other backward compatible changes patch segment must be incremented.
One should not make changes to package contents without also updating version
number. However there is no requirement to do it on each change. Only released
packages need to differ in version. As a result:

#. several breaking changes can be accumulated into one version number change
#. major segment change may also imply changes that would normally affect minor
   and/or patch segments and the same goes for minor version segment change
#. during one development iteration packages may be incompatible with each
   other even if they have compatible version number

SemVer suffixes have no special meaning for Murano and affect how two version
strings compare to each other.

Murano core library is also a package which has its own version. The first
version to have a number will be 0.1.0 and be released in Liberty.

Murano Extension Plugins (i.e python packages which declare
setuptools-entrypoints in 'io.murano.extensions' namespace) also will have
similar versioning semantics: they will have a fully qualified name (defined
by the setuptools' package name) and a version (also defined by setuptools).
From the class-loader perspective the MuranoPL classes defined in the plugins
are no difference from the classes defined in a regular package. However
for initial implementation we may just assume that all exposed plugin
classes reside in a single MuranoPL package having a name of the Python package
and version `0.0.0` to keep backward compatibility until we find a better
way to represent MuranoPL packages using Python counterpart.


Package requirements
''''''''''''''''''''

Packages may require other packages for their work. Those requirements are
expressed via ``Require`` key in  package manifest. It has a form of

.. code-block:: yaml

  Require:
      package1_FQN: version_spec_1
      ...
      packageN_FQN: version_spec_N

``version_spec`` here denotes allowed version range.
It can be either in `semantic_version.Spec` pip-like format  (for example
`>=1.2,<2.0,!=1.5` or `*`) or as partial version string. In the later case:
* x <==> x.0.0 <= version < (x+1).0.0
* x.y <==> x.y.0 <= version < x.(y+1).0
* z.y.z <==> version == x.y.z
* empty (null) <==> 0.0.0  <= version < 1.0.0

Upon package upload/validation it is checked that at least one package in
specified version range exists.

All packages considered to be dependent on ``io.murano`` package (core
library). If such requirement is absent from the list (or the list is empty or
even there is no ``Require`` key in package manifest) then dependency
``io.murano: 0`` will be automatically added (for package types other than
MuranoPL default version may differ).

Packages must explicitly request all other packages which types are mentioned.
This includes class() contracts, inheritance (``Extends``) and dynamic object
construction (``new()``). For any type name use type should not be resolved
by the engine if its not contained either in the class package itself nor in
any of its requirements. Transitional dependencies doesn't have to be
explicitly required.

Murano Extension Plugins treated as regular MuranoPL packages and thus need
to have a way to specify requirement via Python means.

Each package will also have implicit (unless specified explicitly) requirement
on itself of a form ``PackageFQN: X`` where ``X`` is the package major version
segment meaning package can tall to other versions of itself having the same
major version. This requirement is automatically satisfied by the package
itself.

Object version
''''''''''''''
Object in Object Model are also versioned by the version of object's class.
The version will be stored in ``?/classVersion`` attribute of each object.

When model loader tries to create object from its JSON representation it is
going to ask class loader for class of specified version. Versions of object's
parts that belong to parent classes are determined by normal package dependency
resolution rules (parent classes must be in package requirements for the class
version being loaded)
If no version specified then latest available version should be used.

If version is specified but is not present in catalog then:

#. latest class with the same major and minor segments used
#. if there is no match use latest class having the same major version segment
   only
#. if there is no matching class then fail

class() contracts must validate object version against package requirements.
If class A has a property with contract $.class(B) then object passed in this
property when upcasted to B must have a version compatible with requirement
specification in A's package (requesting B's package)


Side by side versioning of packages
'''''''''''''''''''''''''''''''''''

There exist cases when several version of the same package may live in the
same environment:

 * there are different versions of the same MuranoPL class inside single object
   model (environment)
 * several class versions encounter within class parents. For example class A
   extends B and C and class C inherits B2 where B and B2 are two different
   versions of the same class

This implies that class loader needs to:

#. be able to load/keep in memory several versions of the same class
   independently
#. when asked for a class object (i.e. MuranoClass object) from MuranoPL code
   (via ``class()`` contract or ``new()`` function analyzes caller's package
   requirements and returns latest class version matches those requirements
   (or fail otherwise)
#. when accessed from object model loader be able to return class version
   specified in object header (and find compatible replacement when the request
   cannot be satisfied)

The first case when 2 different versions of the same class need to talk to each
other is handled by the fact that in order to do that there must be a
``class()`` contract for that value and it will be validated by the rules from
previous section.

For the second case where single class will attempt to inherit from two
different versions of a same class engine (dsl) should attempt to to find a
version of this class which satisfies all parties and use it instead.
However if it is impossible all remained different versions of the same class
will be threatened as if they be unrelated classes.

For example: classA inherits classB from packageX and classC from packageY.
Both classB and classC inherit from classD from packageZ, however packageX
depends on the version 1.2.0 of packageZ, while packageY depends on the
version 1.3.0. This leads to a situation when classA transitively inherits
classD of both versions 1.2 and 1.3. So, an exception will be thrown.
However, if packageY's dependency would be just "1" (which means any of the
1.x.x family) the conflict would be resolved and a 1.2 would be used as it
satisfies both inheritance chains.


Engine versioning
'''''''''''''''''

Each package has a manifest attribute named ``Format`` of a form
[PackageType/]Version where PackageType is a language used for the package
with “MuranoPL” as a default and Version is a 3-component version string
with shortening rules ``null = 1 = 1.0 = 1.0.0``. Version indicates minimum
engine/language version to run.

Language, engine and manifest format versions are tied together into a
single version.

We assume that all of above are backward compatible or internally can handle
differences between versions. Engine internally knows what changed in each
version and can use that knowledge to emulate behavior of older versions.

Engine must ensure that for each incompatibility in execution behavior or in
YAQL functions particular class method gets properly initialized context with
functions and flags from requested engine version.


UI forms versioning
'''''''''''''''''''

UI forms are versioned using Format attribute inside YAML definition.
The only way to stay compatible with older format versions is either to
support several different formats internally in dashboard or have
auto-conversion utility that will upgrade forms on package upload.


API versioning
''''''''''''''
If single API service can handle several API versions simultaneously it
should be done via endpoint prefixes (/v1/, /v2/ etc.). Otherwise there
should be several separate services listening on different ports and
registered with different names in Keystone.

We assume that dashboard-API-engine are always consistent and upgraded as
the whole. We also assume to be acceptable for different API versions to
have separate databases or tables so that environments created with one
version would not be visible to APIs of other versions.


Alternatives
------------

Keep strict backward compatibility without ability to introduce breaking
changes.

Data model impact
-----------------

For current package API:

* need to store version number for each versioned entity in the database
* in many cases uniqueness must be constrained by name, version number
  and tenant ID rather than by name and tenant ID alone
* requirements for each package need to be stored in database as well

However there is an ongoing process to move packages from Murano API to
Glance v3 Artifact Repository (GLARE).

In GLARE package manifest attributes like FQN and Version are going to be
naturally mapped to corresponding artifact attributes. The package dependencies
will be stored in Glance as cross-artifact dynamic dependencies (i.e.
dependencies not on a particular artifact but on the last artifact matching
the given name and the version range query) as soon as that feature is
implemented in Glance (currently only static dependencies are implemented
there). Until that dependencies are going to be stored as a regular list of
strings, and the Murano engine will process it and query Glance to fetch the
packages.


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

All versions of released packages should be available simultaneously.
That implies that for core library and other essential packages instead
of importing only those packages at system installation all versions of
those libraries need to be imported.

If there would be several independent API services it would require additional
deployment efforts.

Developer impact
----------------

Development process need to be changed for correct version tracking.
Here is an example how it can be organized:

* for each released version of the package there is a separate folder called
  after version number plus one additional folder called “dev”
* during development all changes go to “dev” folder. All packages and
  requirements have a version “0.0.0” effectively turning versioning off.
* after code freeze release manager assign version number to the version about
  to be released by examining commits happened between releases. It is advised
  to mark  such commits somehow (for example [changes major version],
  [changes minor version], [changes revision] or something similar).
* requirements may be defined more accurately from the same source
* new folder for released version is created and final version of the “dev”
  package is put there.
* bug/fix and backport commits can be made in between modifying either already
  released package or making a copy of it with different revision version
  number according to versioning requirements.
* we should write a script to package/upload packages from such versioned
  folder (e.g. producing a package for each released version)

Murano-dashboard / Horizon impact
---------------------------------

In L cycle we are not going to show multiple versions of the same app in
Murano dashboard: only the last one will be shown if the multiple versions
are present. This is to minimize the changes at Dashboard side: in future
releases we'll add the ability to select the proper version.


Implementation
--------------

Assignee(s)
```````````

Primary assignee:
  Stan Lagun <istalker2>

GLARE integration (API to access packages with versions and GLARE plugin):
  Alexander Tivelkov <ativelkov>


Work Items
``````````

* Make Murano Engine access package manifest information
* Implement more flexible processing of package format string (currently it
  requires strict version number)
* Make engine be able to setup independent YAQL context chain depending on
  package's engine version requirements
* Update class loader to work with versions
* Add support for version numbers and package requirements to
  DirectoryPackageLoader (including version-aware directory structure as in
  ``murano-apps`` repo)
* Update APIPackageLoader to work with GLARE Murano plugin
* Implement version compatibility rules for contracts and inheritance


Dependencies
============

None

Testing
=======

In general, after each release we should try to test old applications on new
release to make sure that nothing broke (which may happen if versioning rules
were not followed correctly). But it is clear that in practice this cannot be
done for all applications in catalog. So we may peek selected set of
applications and to try to deploy all versions of those applications on current
release. This set should be representative to test the most possible set of
futures.

Subset of those applications can be used for per-commit tests.


Documentation Impact
====================

Changes introduced by each version change of each component should be
documented. There should be possible to see documentation for specific previous
Murano version as it remains usable.

References
==========

None
