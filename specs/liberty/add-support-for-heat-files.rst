..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===========================================
Add support for heat environments and files
===========================================

https://blueprints.launchpad.net/murano/+spec/add-support-for-heat-environments-and-files

Add support for using Heat additional files, when saving and deploying Heat
templates.


Problem description
===================

Today there is no option to create stacks neither with an environment nor with
nested stacks via Murano. Only basic stacks are supported.

Use cases for deploying a stack with additional files:

* Supporting Heat nested stacks
* Supporting scripts

For more information see references section of this spec


Proposed change
===============

The change described here will focus on supporting additional files. Another
spec will be submitted for Heat environments.

Allowing Heat additional files to be saved and deployed as part of a Murano
Heat applications. Adding additional files to a package should be optional.
Such files should be placed in the package under '/Resources/HeatFiles'. When a
Heat generated package is being deployed, if the package contains Heat files,
they should be sent to Heat together with the template and params. If there are
any Heat additional files located in the package under '/Resources/HeatFiles',
they should all be sent with the template during stack creation in Heat.

There can be more nested directories under '/Resources/HeatFiles', and if they
have files in them, those should be send to heat as well together with their
relative path.

This part does not require any UI nor API changes.

Alternatives
------------

* The package structure can be different.

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

User will have the option to add Heat files to the package, and have them
deployed with the package as part of the Murano environment.

At this point there will be no change in the python-muranoclient. If the user
will wish to add files to a heat generated package, he can edit the package and
continue normally from there.

Deployer impact
---------------

None

Developer impact
----------------

None

Murano-dashboard / Horizon impact
---------------------------------

None.


Implementation
==============

new data objects:
* files

This new data object will be generated when parsing the package, so if there
is nothing in the package, the new data objects will be empty.

A new object called files of type dictionary will be added in HeatStack class
in the init method. It can be referenced from inside HeatStack by using
self._files when sending the create request to Heat. This object will be
initialized to be empty.

A new method called setFiles will be added in HeatStack class. It will allow to
enter a value into _files just like setParameters allows to enter values into
_parameters.

A new method called _translate_files will be add to HotPackage class. it will
get a path to the files directory and will return a dictionary of files
locations and files values. It will be in the next format:
fileRelativePathStartingHeatFilesDirectory -> stringFileContent
For example if there is a file with a full path of
/Resources/HeatFiles/my_heat_file.yaml and content of "echo hello world" and it
is the only file in the folder, than this will be the dictionary returned:
{"my_heat_file.yaml": "echo hello world"}

A new parameter of called files will be added to _generate_workflow method that
can be found inside HotPackage class. It will be included in the deploy
variable in the same way the template_parameters is included.

Assignee(s)
-----------

Primary assignee:
  michal-gershenzon

Other contributors:
  noa-koffman

Work Items
----------

* Add support for Heat additional-files. If such files exist in the package,
  they should be parsed and send with the template, when deploying a Murano
  environment.


Dependencies
============

None


Testing
=======

Unit tests should cover API calls changes:

* Test that the request for Heat is build correctly with Heat files


Documentation Impact
====================

None


References
==========

* http://docs.openstack.org/developer/heat/template_guide/hot_spec.html#get-file