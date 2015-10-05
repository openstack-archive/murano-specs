..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===========================
Artifact Repository Support
===========================


https://blueprints.launchpad.net/murano/+spec/artifact-repository-support

Glance has recently got a new feature - ability to store not only the VM images
but also data assets and their metadata for other OpenStack projects.

This specification defines the usage of this feature in Murano, so Murano may
store its packages in Glance and benefit from all its features.

Problem description
===================

The current implementation of Murano Package Repository as part of murano-api
has the following major drawbacks:


* Code duplication. Lot's of the logic to manage the packages is very similar
  for other OpenStack projects. Sharing that code with others will help to
  improve the overall quality of OpenStack ecosystem.

* Database storage is a bad idea. Currently we store all the murano-packages as
  blob fields in the database tables. This dramatically decreases the
  performance and limits the maximum size of the packages, thus preventing us
  from putting actual application binaries inside the packages. Using glance
  for this purpose will allow us to utilize all the backing stores supported by
  glance, including Swift, Ceph, S3 and others.

* No cross-package dependencies. Current implementation of package repository
  does not have any concept of cross-package dependency, so MuranoPL package
  loaders have to detect package dependencies only in runtime. This leads to
  issues when some of the package's dependencies are not present in the repo,
  but this cannot be detected until the actual deployment starts. Using glance
  which has a notion of cross-artifact dependencies solves this issue natively
  at the repository level.

* Image dependencies are manual. Current implementation has some manual way of
  managing dependencies of murano packages to glance images. When glance is
  user dependencies on Images will have the same nature as cross-package
  dependencies.

* No versioning of packages. Current implementation does not allow to attach
  semver-based version strings to the packages, and so no filtering is possible
  by this info. Glance Artifact Repository provides a rich versioning tools and
  allows to filter by specific version spec and range.

* No ability to query packages based on dependency hierarchy. We need a way to
  query the repository for the classes which inherit some specific base class.
  Glance Artifact Repository provides such queries out of the box.



Proposed change
===============

It is proposed to create a Glance Plugin which will define the artifact type
"Murano Package" with the following type-specific metadata attributes:

* **type** - a string defining the type of the package, may have values of
  either `Application` or `Library`. Immutable. Required.

* **author** - a string defining the author of the package. Immutable.
  Required.

* **display_name** - a display name for the package to be shown in the catalog.
  Mutable. Required.

* **enabled** - a boolean indicating if that package is available for usage.
  Proposed temporary, until Glance v3 has a support of "disable" operation for
  all the types of artifacts. Mutable, True by default.

* **categories** - a set of strings containing the categories attached to the
  package. Mutable.

* **class_definitions** - a set of strings containing the fully-qualified names
  of muranoPL classes contained within the package. Immutable.

* **inherits** - a dictionary in which the keys are the interfaces inherited by
  the classes of the packages and the values are the list of the names of these
  classes. Immutable.

* **keywords** - a set of keywords to simplify the search of the package in the
  catalog. Mutable.


..and the following binary objects:

* **logo** - a blob containing the logo of a package

* **archive** - a blob containing the zip archive with the package

* **ui_definition** - a blob containing the yaml-based definition of app's UI
  if the one is present.


If worth noting that currently v3 of Glance API is tagged as EXPERIMENTAL, and
both its implementation and interfaces may (and will) change in Mitaka release
cycle. So, some of the plugin implementation details may have to be changed as
well in the next cycle.

Because of this, it is suggested to add this functionality as EXPERIMENTAL as
well, maintaining the old behavior as a preferred alternative, selected by
default.


Alternatives
------------

We may continue using murano-api to manage our packages, at least till the
Glance v3 is STABLE, however this will require us to introduce lots of
temporary code to support things like versioning and dependencies. So, it is
better to add thr Glance support as an experimental feature, facade it under
the python-muranoclient bindings, and migrate to it completely in the next
release cycle.


Data model impact
-----------------

No impact. When the migration is complete and Glance is the only storage of
packages, we'll need to drop the old data tables which are currently storing
the old package repositories.


REST API impact
---------------

No impact.


Versioning impact
-----------------

No impact.


Other end user impact
---------------------

No impact.

Deployer impact
---------------

The deployer will have to add the glance plugin package into their environment.
It should be deployed into the same python env with the regular Glance. They'll
have to enable the V3 API in the glance-api config as well.


Developer impact
----------------

None

Murano-dashboard / Horizon impact
---------------------------------

Only the configuration changes should be made in murano-dashboard.


Implementation
==============

Assignee(s)
-----------


Primary assignee: ativelkov

Work Items
----------

* Implement a glance plugin

* Copy the artifact-specific access logic from python-glanceclient into the
  python-muranoclient, as glance is not going to release v3-aware clients until
  the API is stable.

* Implement artifacts adapter in python-muranoclient


Dependencies
============

* Most of the artifacts functionality has been merged to Glance by L-1
  milestone. However, several important bugfixes were merged only into the RC1.
  So, the latest Glance master is required.


Testing
=======

As the new backend substitutes the old one, the regular testing of
package-related actions should be sufficient.


Documentation Impact
====================

New configuration settings have to be documented.

References
==========

None
