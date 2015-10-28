..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=======================
Pluggable Package Types
=======================

URL of launchpad blueprint:

https://blueprints.launchpad.net/murano/+spec/pluggable-package-types

This spec is about making possible to support different package formats
(besides `MuranoPL/1.x` and `Heat.HOT/1.0`) via package format plugins.


Problem description
===================

There are many different formats that can be used to describe applications
besides MuranoPL. Currently Murano supports 2 package types `MuranoPL/1.x` and
`Heat.HOT/1.0`. However because they both are parts of Murano source codes
it is impossible to add additional types without merging them into main source
tree thus making Murano team responsible for all of them.


Proposed change
===============

It is proposed to have support for non-MuranoPL packages the same way HOT
support is implemented: by dynamic generation of MuranoPL code at run time.
However the package types themselves need to be pluggable so that anyone
could extend Murano with additional package type by installing corresponding
plugin.

It is proposed to use stevedore library and similar approach to how MuranoPL
python plugins are currently handled.

Package types plugins will be identified by dedicated namespace
`io.murano.plugins.packages`. To specify package type one should append to
plugin's `setup.cfg` file

.. code-block:: ini

   io.murano.plugins.packages =
       FORMAT_STRING = CLASS_IDENTIFIER

For example:

.. code-block:: ini

   io.murano.plugins.packages =
       Cloudify.TOSCA/1.0 = murano_cloudify_plugin.cloudify_tosca_package:CloudifyToscaPackage

If target package type requires some utility to construct it (e.g. put right
files in right folders, zip them, generate manifest and so on) than it
should also be included in the plugin as additional shell endpoint.


Alternatives
------------

None


Data model impact
-----------------

None


REST API impact
---------------

None


Versioning impact
-----------------

None. There cannot be 2 versions of the same plugin simultaneously.
However single plugin may support several different package formats including
severals different versions of the same format using single of several Python
classes. Version number remains part of format string.


Other end user impact
---------------------

None


Deployer impact
---------------

Plugins need to be deployed to each Murano node (or to all machines running
either Murano API or Murano Engine) in order to support particular package
type. Installation is done as usually in python (`pip install PATH` or
`python setup.py install` etc.)


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
  Stan Lagun <slagun@mirantis.com>


Work Items
----------

1. Refactor current package class hierarchy, change order of `__init__`
   parameters so that format name and version will become the first two.
   This is required because they will be supplied automatically by plugin
   loader.
#. Implement plugin loader that would find all installed stevedore plugins
   in `io.murano.plugins.packages` and build internal mapping
   format name/version -> Python implementation class
#. Add function to register in that mapping implementations that remain inside
   Murano sources like MuranoPL package type
#. Add method to retrieve `Package` instance of supplied format name/version.
   Plugin-loader should return class factory that will instantiate appropriate
   Python class for particular format by substituting first 2 arguments
   (format name and version) while the rest arguments will be supplied by the
   caller.
#. Refactor `load_utils` to use plugin-loader rather than its own hardcoded
   mapping.


Dependencies
============

stevedore library


Testing
=======

Testing can be performed by attempt to deploy application that is not in
native Murano format like TOSCA is.


Documentation Impact
====================

Guide that tells how to develop custom package type plugins need to be
published.


References
==========

None
