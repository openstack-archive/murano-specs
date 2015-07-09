..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=================================================================
Download bundle of packages to local directory using muranoclient
=================================================================

https://blueprints.launchpad.net/murano/+spec/bundle-save

The purpose is to add command to muranoclient which allows to download the
bundle from application catalog to local dir.

Problem description
===================

There are cases when there is no Internet access cloud with murano installed.
Then if user wants to add some bundle of packages into murano, he has to
download all of them one-by-one from application catalog using local computer
with access to the Internet. After that he saves them somewhere on data storage
device and moves all files to cloud.

It is necessary to simplify process of saving packages to avoid manual
downloading.


Proposed change
===============

It's proposed to add new CLI command *bundle-save* to murano-client.

Method *do_bundle_save* will corresponds with new command. It will take three
arguments:

* *filename* is a bundle name, bundle url or path to the bundle file;
* *--path* (optional) is a path to directory in which user wants save packages.
  If it is not specified, current directory will be used;
* *--no-images* (optional) is flag. If it is set, downloading of all required
  images will be skipped.

Method will build whole list of packages and its dependencies. This ability is
already implemented and used in 'bundle-import'. Then method will save bundle
file and each package to specified path. For this, method 'save' will be added
to *FileWrapperMixin* -- the parent class for  *Bundle* and *Package* classes.
This method will take one argument *dst* -- destination for file. It will copy
already downloaded file to the specified path.

Method *do_bundle_save* will also save images which packages require.
*save_image_local* method will be used for that. All images will be downloaded
if *--no-images* is not set.

After bundle saving directory with packages can be moved to lab with murano
where all of them can be imported to murano application catalog in one
command.

CLI command *package-save* also must be implemented. It will give to user the
opportunity to download specific package or several packages he need.
The implementation of command will be based on the methods described above.

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
-------------------------

None

Other end user impact
---------------------

User will have access to a new command.

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

Dmytro Dovbii

Primary assignee:
  ddovbii

Work Items
----------

* Add method *save_image_local*
* Add method *save()* to *FileWrapperMixin* class
* Implement CLI command *bundle-save*
* Implement CLI commamd *package-save*

Dependencies
============

None

Testing
=======

Unit tests for CLI client must be updated

Documentation Impact
====================

CLI reference should be updated manually

References
==========

None
