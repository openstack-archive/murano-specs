..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================
murano-mistral-integration
==========================

https://blueprints.launchpad.net/murano/+spec/murano-mistral-integration-1

The purpose is to add integration between Murano and Mistral.
We would like to allow invocation of a Mistral workflow from Murano.
No prerequisite for uploaded Mistral workflow.

Problem description
===================

A new capability in Murano modeling process should be added in order to answer
several use cases, including:

#. The application modeler wishes to leverage existing workflow that deploys
   a specific component as part of a new application model creation process.

#. The application modeler wishes to add post-deployment logic to an
   application model.

   For example:

   a. Check that the application has been successfully deployed.
   b. Inject initial data to the deployed application.

Proposed change
===============

Adding a new system class for Mistral Client that allows to call Mistral
APIs from the Murano application model.

The system class will allow you to:

#. Upload a Mistral workflow to Mistral.
#. Trigger the already-deployed Mistral workflow, wait for completion and
   return the execution output.


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
Natasha Beck

Work Items
----------

#. Add mistral client system class that can trigger pre-deployed Mistral
   workflow.
#. Add ability to the client to upload the Mistral workflow.
#. Add test that deploys application which uploads the Mistral workflow from
   Resources, triggers it and gets output as a result.


Dependencies
============

Openstack Mistral component.


Testing
=======

Functional test will be added to Murano.
The test deploys application which uploads the Mistral workflow from
Resources, triggers it and gets output as a result.


Documentation Impact
====================
Murano-Mistral integration must be documented from the following perspectives:

* Setup configuration (for example with and without mistral)
* Include usage of Mistral workflow as a part of an application model

The following Murano documentation will be affected:

* Murano Installation Guide
    Add section on Mistral requirement
* Murano Workflow
    Add section on Murano-Mistral integration
* Murano Article (new)
    Article on Murano-Mistral integration


References
==========

None