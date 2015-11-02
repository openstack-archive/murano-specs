..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===================================================
Support TOSCA-Cloudify definitions for applications
===================================================

URL of launchpad blueprint:

https://blueprints.launchpad.net/murano/+spec/support-tosca-cloudify-format

TOSCA is a standard developed under OASIS foundation. It is aimed to cover
definition of complex enterprise applications which might consist of different
loosely coupled components. Components can be tied by using requirements and
capabilities which are part of TOSCA standard. As TOSCA covers most of the
Applications aspects (excluding application compilation and build) it should
be straightforward to support TOSCA format in Murano. TOSCA is adopted by
enterprises so it will allow the OpenStack ecosystem to integrate with
enterprise IT applications.

Cloudify is an open source TOSCA-based cloud orchestration software platform
written in Python, and YAML created by GigaSpaces Technologies.
It is licensed under the Apache License Version 2.0

With Murano-Cloudify TOSCA-based integration it will be possible to package
Cloudify-compatible TOSCA applications as Murano packages and deploy them via
Cloudify not necessary on OpenStack but on all cloud platforms currently
supported by Cloudify.

Problem description
===================

Murano currently does not support TOSCA and TOSCA-based applications. With
the growing popularity of TOSCA for describing application orchestrations
it would be nice if murano supports TOSCA-based orchestration templates.

Murano has accepted HOT-translator based TOSCA support but that is not the
only possible TOSCA implementation possible. Cloudify brings alternate
TOSCA solution that is not limited to Heat and OpenStack in general.

One standard for defining TOSCA-based application packages is CSAR
(http://tinyurl.com/tosca-yaml-csar). It is a compressed file that includes the
main application definition template called "blueprint" in Cloudify along with
the supporting templates and files that are referenced in the main template.

This specification addresses adding support for Cloudify blueprints
application packages in Murano application catalog with ability to deploy them
via Cloudify.


Proposed change
===============

Make it possible for Murano packages to ship Cloudify blueprints.
Cloudify blueprints will be stored in the Resource folder of the package.
The manifest will be in the package root folder.
The Cloudify code will be converted to Murano PL code on the fly.

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

There should be no user impact. Users who are creating HOT-based packages would
do it the same way as before. Users who are creating TOSCA-based packages would
do it in a similar way. Application packages can be added to environments for
deployment the same way as before too.

There should be also a utility to help users package CSARs into Murano package
but it is out of the scope of this spec.


Deployer impact
---------------

None


Developer impact
----------------

None


Murano-dashboard / Horizon impact
---------------------------------

Murano would need to represent TOSCA-based components using a specific
logo, to distinguish them from HOT-based components. This applies to other
specification formats that may come on board in the future.


Implementation
==============

Assignee(s)
-----------

  Trammell <trammell@gigaspaces.comx>

Work Items
----------

#. Define new package format Cloudify.TOSCA/1.0 and create a package-type
   class that will process packages with such format string.
#. Cloudify Blueprint directory that contains a blueprint in the format
   filename.yaml. The name of the file needs to be declared in the
   `manifest.yaml` file.
#. From main blueprint yaml file extract information about application
   inputs and outputs.
#. Generate UI form definition by generating 1 input field per each TOSCA
   input and also generate Application section of UI form that will map
   those fields to MuranoPL application class properties.
#. From TOSCA inputs/outputs and manifest data generate MuranoPL class
   that will represent TOSCA application. There should be a MuranoPL
   property for each TOSCA input and output and usual application methods
   as in other Murano applications.
#. In order to avoid generation of complex MuranoPL code create base class
   for Cloudify applications that will implement deploy, ``destroy`` and other
   common methods. Generated application will inherit from this base class
   and implement only application-specific method that will be used by the
   base class to get required information (like input values, entry point
   name, etc.). Bacause base class is going to be regular MuranoPL class
   it will form a separate MuranoPL package that need to be present in
   catalog as well. Cloudify package processing code will need to tell
   Murano Engine that there is a dependency to that package without
   explicitly specifying that in manifest file as normaly done in MuranoPL
   packages.
#. For Cloudify application to be able to talk to Cloudify manager and
   do the actual provisioning/deployment MuranoPL bindings to Cloudify
   rest-client need to be developed. It should provide all required methods
   to upload the blueprint, create deployment and execute TOSCA workflows.
#. In base CloudifyApplication class implement common application workflows:
   for installation:

    #. Provide name of the blueprint file in the Cloudify Blueprint directory.
       Cloudify Rest Client will tar that directory and upload the payload
       to the Cloudify Manager.
    #. Create blueprint deployment (Cloudify term similar to Stack in Heat or
       Environment in Murano). Generated application object ID can be used
       as a deployment ID
    #. Invoke "install" workflow on created deployment. Watch its progress.

   for uninstallation (.destroy method):
    #. Invoke "uninstall" workflow on previously created deployment.
    #. Delete deployment.

Dependencies
============

Applications should be compatible with Cloudify 3.1 (latest GA version - 1)

Testing
=======

  #. Follow these instructions to create a Cloudify Manager in a vagrant
     box on the same machine as your development environment. From there
  #. Add the manager IP to your etc/murano.conf file and install the
     plugin.
  #. Upload the Cloudify Application Package
     (contrib/plugins/cloudify_application).
  #. Upload the Nodecellar Example Deployment.
     (contrib/plugins/cloudify-nodecellar-example-murano).

Documentation Impact
====================

How TOSCA packages can be added to catalog and deployed should be
documented, perhaps with some examples.


References
==========

* `Cloudify <http://getcloudify.org>`_
* `Tosca-Parser on GitHub <https://github.com/openstack/tosca-parser>`_