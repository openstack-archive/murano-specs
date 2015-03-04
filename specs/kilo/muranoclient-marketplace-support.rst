..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=========================
Murano Repository Support
=========================

https://blueprints.launchpad.net/murano/+spec/muranoclient-marketplace-support


Murano applications provide a powerful and flexible way to move workloads
from other cloud environments to OpenStack. In order to accelerate
application migrating we need a way to deliver Murano
packages and Glance images to customers incrementally, independent from major
releases of OpenStack.


Problem description
===================

Typical use cases:

* After the end users installed and configured Murano they would need to
  install murano-enabled applications. To do so they would use `murano` CLI
  client. Invoking a command like
  `murano install-package io.apps.application --version=2.0` would
  install the application in question alongside with all requirements
  (applications and glance images)
* A developer would want to provide murano applications to end users, by
  hosting them on a web server and providing http-access to appliaction
  packages. In that case end users would be able to install the application by
  invoking a CLI command `murano install-package http://path/to/app`
* End user would want to install bundles of applications that are often
  required to work together, but not necessarily depend on each other.
  In that case end user would invoke a command
  `murano install-bundle bundle_name` and murano-client would download all
  the applications mentioned in the bundle alongside with their requirements.
* End users would want to perform same operations through
  murano-dashboard instead of CLI tools.


Proposed change
===============

The proposition is to implement a set of features in python-muranoclient and
murano-dashboard, that would allow end users to install applications from
http application repository.

* Enable `package-import` command to support url as a parameter for import by
  modifying the way package creation of murano-client works.
* Enable `package-import` command to support package name as a parameter for
  import, introduce `--murano-repo-url` as a base path to the repository.
* Introduce `bundle-import` command and allow it to support local-files, urls
  and bundle names as parameters for import. The command should parse the
  bundle file and import all the packages, mentioned in the file.
  Bundle should be a simple json and/or yaml-structured file.
* Introduce `Require:` section to `manifest.yaml`, that would specify which
  packages are required by the appication.
* Enable package creation suite to automatically import all the packages
  mentioned in `Require:` section.
* Introduce `images.lst` file in the package structure, that would contain a
  list of images, required for the application to work.
* Enable package creation suite to automatically import required images into
  glance image service.
* Allow `bundle-import` command to import bundles from local-files. In that
  case first search for package files and image files in the local filesystem,
  relative to the location of the bundle file. If file is not found locally
  attempt to connect to the repository.
* Enable murano-dashboard to support changes made to client and introduce a way
  to upload packages via url/name to dashboard.
* Implement importing of bundles in murano-dashboard by url or by name.
  Since bundles are currently a simple json/yaml optionally we could optionally
  support direct input.
* Implement error handling both for CLI-tools and dashboard, that would inform
  end users about any errors that might have happened alont the way of
  importing.
* Optionally implement a progress marker and a ETA-marker for CLI `import`
  commands.


Alternatives
------------

Implementing a server, that would hold versions and paths to package files
and implementing a client to that server might be a good alternative to a
simple http-server, although it seems a bit excessive at the moment.

Another idea would be to implement a service, that would download packages
asynchronously. This would allow client to download large package files and
image files with possibly better error handling mechanisms. This also seems a
bit excessive at the moment.

Data model impact
-----------------

It might be a good idea to store information about required apps and required
images, although it is not strictly required for the task.

REST API impact
---------------

It might be a good idea to warn the end user if the client installed a
package, that depends on other packages, not present in app catalog,
although it is not strictly required for the task.

Versioning impact
-----------------

This proposition adds functionality both to python-muranoclient and to
murano-dashboard. It should be fully backward compatible.

Minor version of python-muranoclient should be increased, because we add
functionality.

Other end user impact
---------------------

See *Proposed Change* section, as it describes all the ways users would
interact with the feature


Deployer impact
---------------

None

Developer impact
----------------

None

Murano-dashboard / Horizon impact
---------------------------------

Package import dialog should be changed to reflect different ways of
importing an application.
Additional bundle-import dialog should be implemented.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  <kzaitsev@mirantis.com>


Work Items
----------

* Add support for importing packages by url in client
* Add support for importing packages by name in client
* Add support for importing bundles by name/url in client
* Add support for recursive parsing of Require section of `manifest.yaml`
  file in client
* Add support for image download/upload from `images.lst` file in client
* Handle `package exists` error in CLI client
* Add support for different ways to import a package in murano-dashboard
* Add support for bundle-import in murano-dashboard
* Add support for image/package requirements while importing packages and
  bundles in murano-dashboard

Dependencies
============

None

Testing
=======

Unit testing should be sufficient to cover most of the use cases.
However an integration test, that would setup a simple repository is very
desirable, since we add changes to both `python-muranoclient` and
`muranodashboard`.

Documentation Impact
====================

Additional capabilities of the CLI client should be documented, otherwise
there should be no impact, since we do not change the API.

References
==========

None
