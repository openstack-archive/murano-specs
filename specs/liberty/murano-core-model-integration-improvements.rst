..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=========================================================
Murano API - Core Model Component Integration Improvement
=========================================================

https://blueprints.launchpad.net/murano/+spec/murano-core-model-integration-improvements

Core Model can be seen as API, because user is using it when writing Datalog queries
in Congress, or integrating with Mistral workflows. Current Murano Core model does not
provide means to easy link them with realized OpenStack entities (for example Murano
Instance does not provide UUID of provisioned Nova Server).


Problem description
===================

Congress datalog queries are one of core features used by Policy Guided Fulfillment.
These queries are used to express validity of Murano environment either
in pre-deployment and/or runtime. In order to evaluate environment validity it is
necessary to work with realized entities by core Murano components - for example

* *I want to check if Murano Instance's Nova server exists and is running*

  * In kilo I have to use IP address of the Murano Instance and do multiple joins of
    neutron tables to identify Nova server.

* *I want to check if owner of network used by Murano Environment is from given group of
  users*

  * In kilo it is impossible because Murano network object contains only name
    of the network (*a-network*), while realized network (via Heat) contains name
    with Murano object id (*a-network-bed7a70ed791434c8acdd53a52a8d4ca*)

Proposed change
===============

Changes of core Murano model:

* **io.murano.resources.Instance**

  * add property *openstackId* and fill the property with value once Heat provisioned
    Instance

::

 openstackId:
   Contract: $.string()
   Usage: Out

* **io.murano.resources.Network**

  * add property *openstackId* and fill the property with value once Heat provisioned
    network (as part of Instance provisioning)

::

 openstackId:
   Contract: $.string()
   Usage: Out



Alternatives
------------

Instead of adding the same property to each class aware of openstack ID we can create
mixin class ( e.g. **OpenstackIdMixin**) with this property and all classes aware of
openstack ID will extend that mixin.

::

 Name: OpenstackIdMixin
 Properties:
   openstackId:
     Contract: $.string()
     Usage: Out


Data model impact
-----------------

None

REST API impact
---------------

None

Versioning impact
-------------------------

Murano package versioning is currently analyzed and it is planned for Liberty.

It makes sense to introduce it (modifications of the core Murano packages) as new
versions of core Murano packages.

On the other hand, proposed changes are backward compatible, so they can be
done prior versioning.


Other end user impact
---------------------

None

Deployer impact
---------------

If proposed changes will be done prior Murano package versioning, then after upgrade
the Murano objects won't have initialized introduced properties (*openstackId*).


Developer impact
----------------

None

Murano-dashboard / Horizon impact
---------------------------------

The change will simplify implementation of Horizon UI (instance detail)

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  filip-blaha

Work Items
----------

* introduce *openstackId* properties to

  * *meta/io.murano/Classes/resources/Instance.yaml*
  * *meta/io.murano/Classes/resources/Network.yaml*

* implemented instance and network *openstackId* property population

  * *meta/io.murano/Classes/resources/Instance.yaml* , *deploy* method
  * *meta/io.murano/Classes/resources/NeutronNetwork.yaml* , *deploy* method
  * *meta/io.murano/Classes/resources/NovaNetwork.yaml* won't be modified, as nova
    networking is deprecated

Dependencies
============

https://blueprints.launchpad.net/murano/+spec/murano-versioning

Testing
=======

Both unit and tempest tests of policy guided fulfillment will be enhanced to test properties *openstackId*.

Documentation Impact
====================

None

References
==========

* https://wiki.openstack.org/wiki/PolicyGuidedFulfillmentLibertyPlanning
* https://wiki.openstack.org/wiki/PolicyGuidedFulfillmentLibertyPlanning_MuranoAPI