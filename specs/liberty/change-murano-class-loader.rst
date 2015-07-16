..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=================================
Rework package class loader logic
=================================

https://blueprints.launchpad.net/murano/+spec/change-murano-class-loader

This spec describes rework of murano package class loader as part of the
another blueprint `simulated-execution-mode-murano-engine
<https://blueprints.launchpad.net/murano/+spec/simulated-execution-mode-murano-engine>`_.

The detailed logic regarding class modification would be described in this
specification.

Problem description
===================

Class loader should be able to load packages not only from the external
repository, but from a local directory. It's needed not only for testing new
packages, but for accepting packages on-the-fly.
During the development phase the package gets frequently updated, and the need
to upload it to the repository on each update complicates the development.
Ability to load it from the local filesystem will streamline the process

Another case, when there is no connection to the external repository.

Proposed change
===============

Need to create one more class, responsible for loading packages.
It will look up at the provided directory for a package, that name was requested.
It worth noting, that packages should not be zipped.

Class loader from repository (RCL) and new class loader from local dir (LCL)
will provide the following logic:

* If local dir is not provided, LCL is not operating.
  Logic stays same as it's now. RCL do all the work.

* If directory path or several paths to load local packages from are provided,
  LCL will check all packages in dir and compare with requested name.
  If the desired class is found, package gets loaded. If not - next dir in the
  provided list will be scaned.

  If is not found in all the provided directories - RCL sends API call to find
  it in the repository as usual.

We do have both class loader implemented, we need an ability to combine two
(or more) package loaders in one, with the ability to prioritize the queries.
For example, a ``CombinedPackageLoader`` may be created with an instances of
``DirectoryPackageLoader`` and ``ApiPackageLoader`` underneath. When a
request for a package or class comes in, it is first executed against loader
with the highest priority (say, ``DirectoryPackageLoader``), and if it is not
found there, then goes to the next one.

Alternatives
------------

Use separate class loader in test framework.
But support two different class loaders is not good.

Also as an alternative we can also mention
`this <https://docs.djangoproject.com/en/1.7/ref/settings/#std:setting-TEMPLATE_LOADERS>`_
approach.
That is: introduce package_loaders config parameter. It can be a list of
python-dot-notation class names, that murano would import and try to import pkg
from each loader. This could further be modified to a list of lists, to allow params
like this:
`package_loaders: [pkg_loader1, [pkg_loader2, param1, param2]]`
This would allow to easily add custom loaders without changing murano code itself.

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

Enabling loading packages from local directory will be set up in config file.

New key in config file will be added under *[engine]* section and look
like that:

# Directory used as a way to load packages from. (string value)
# local_packages_dir = <None>

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
  <efedorova>

Other contributors:
  <ativelkov>, <slagun>

Work Items
----------

* Implement ``CombinedPackageLoader``:
* Perform testing

Dependencies
============

* Murano Simulation Mode:
  https://blueprints.launchpad.net/murano/+spec/simulated-execution-mode-murano-engine

Testing
=======

Unit tests should be added/updated.

Documentation Impact
====================

Opportunity to load apps from local directory should be described in the
documentation.


References
==========

None