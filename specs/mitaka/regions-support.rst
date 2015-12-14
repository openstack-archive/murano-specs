..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================
Support for OpenStack regions
=============================

Related blueprints:

https://blueprints.launchpad.net/murano/+spec/default-region-for-services
https://blueprints.launchpad.net/murano/+spec/assign-environment-to-region

Many if not most production grade OpenStack installations are comprised of
several so called OpenStack Regions - autonomous OpenStack clouds joined
together through shared Keystone identity server and managed through common
interface. However Murano is not region-aware and not always works correctly in
such setups.

Problem description
===================

There are several possible topologies for multi-region deployment:

#. There is a copy of all murano services at each region. Environments
   consume resources in (or in other words get deployed to) the same regions
   that murano-api/murano-engine run in. Because each murano-api instance is
   going to use independent database each regions will have its own list of
   environments, packages etc.

#. Single murano-engine is capable of deploying environments to different
   regions. In this case environment gets deployed in one specific region but
   not necessary the one engine is located in. The target region is chosen
   according to attribute of the environment rather than location of the the
   engine. Because in this case region of murano services is independent from
   target environment region it is still possible that murano services are
   present in more than one region up to the case where there is a copy in each
   region and each one of them is capable of deploying applications to any of
   the regions (but still each region has its own database and everything it
   implies).

#. Single environment may contain applications and/or components that deploy
   into different regions. Because environments cross region boundaries it is
   no longer single heat stack per environment and applications, components and
   resources need to be associated with regions and with stacks.

In the list above each subsequent item includes and extends the previous one
and thus can be implemented in turn.

This specification covers first two items because they can be implemented
without introducing major changes to applications design or existing Heat
API bindings.


Proposed change
===============

Only a few minor changes are needed in order to enable basic region support:

* Murano-engine should probe for correct region when querying endpoints from
  keystone's service catalog.

* Add attribute (property) to the Environment class saying what is the
  target region for the environment.

* Improve murano-dashboard with ability to select target environment.


Alternatives
------------

None

Data model impact
-----------------

Optionally target region name for the environment can be extracted from
object model and put into a separate column in environment/session tables.
This will allow to filter (group) environments that target different regions
in dashboard.


REST API impact
---------------

None

Versioning impact
-----------------

Because the changes are backward compatible a minor version component of
the core library need to be incremented (possibly by the end of Mitaka cycle).

Other end user impact
---------------------

For each OpenStack setup that has more than one region Horizon will
automatically presents user a drop-down that allow to switch between API
instances in different regions and thus see independent lists of environments,
packages etc. Nothing need to be done from murano side here.

In addition there should be a way to select target region for particular
environment independently of currently selected API region. However this
setting should remain optional and not affect existing user experience.

Murano CLI interface should not require any changes because region selection
for API is already supported using --os-region-name option or OS_REGION_NAME
environment variable. Target region name of environment is an ordinary
environment property as is set using all the same interface.


Deployer impact
---------------

Each region that has a copy of murano needs to have murano-api and
murano-engine as well as its own RabbitMQ for api to engine and engine to
agents communications (this can be either one shared or two independent
instances of RabbitMQ). Thus murano in different regions will have different
configuration files. In the same time the version of murano itself must be the
same in all regions. In addition service endpoint of each murano-api needs to
be registered in keystone.

In scenario where single murano instance can deploy to different regions it
might be that it is required to use different RabbitMQ instances in different
regions in order for engine to communicate with murano agents. Because
RabbitMQ instances are currently not registered in keystone service catalog
address and credentials for all RabbitMQs need to be put into murano config
file.


Developer impact
----------------

None

Murano-dashboard / Horizon impact
---------------------------------

* Additional control (drop box with region names) is added to environment
  configuration form.

* Dashboard need to correctly handle situation when murano is present only
  in some of available regions.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Stan Lagun <slagun>


Work Items
----------

Add setting that specifies default region name
``````````````````````````````````````````````

murano-engine needs to be aware what OpenStack region it belongs to.
When deploying environment that doesn't explicitly target some particular
environment (as it is now) this setting will control in what region should
a Heat stack be created. We assume that there is a Heat instance in each
region so we just need to use correct endpoint of the heat API.

There is already a setting `region_name_for_services` but it is located in
the wrong section ([murano] rather than [DEFAULT]).


Make ClientManager region aware
```````````````````````````````

There is a class in murano-engine that responsible for creation of specific
service clients. Typical workflow is to ask keystone for an endpoint of
desired type (using url_for() function) and then pass the endpoint to client's
`__init__` method. However in multi-region setups there can be several
endpoints for each service that differ by region name.

ClientManager's methods need to be extended with optional `region_name`
parameter to be used for url_for(). If no region name specified by the caller
then default region name must be used.

Two work items above alone is enough to enable first use case for regions.

Add region property to the Environment
``````````````````````````````````````

Add optional string property to the Environment class that will hold target
region name for the environment.

OpenStack API binding classes (like `HeatStack` and `NetExplorer`) that
internally request a client instance from ClientManager need to modified to
pass a region name of environment they are belong to (can be None)

Note, that murano-client must always use the default region endpoint because
it is used to update environment status and download packages and this is
always done in current region rather then in one we're deploying to.

Also optionally value from this property can be replicated to a dedicated
column in database so that it will be possible to filter on.


Extend [rabbitmq] configuration setting group
`````````````````````````````````````````````

Murano uses RabbitMQ to interact with guest-VM agents. But for multi-region
setup it is desirable to have dedicated RabbitMQ instance in each region.
Thus a single section with RabbitMQ parameters is not enough.

Instead it is proposed to have variable number of sections with a common
format [rabbitmq_RegionName] (i.e. [rabbitmq_RegionOne]).

When somebody needs RabbitMQ parameters it need to look in region specific
section and only if it is not present (or some particular setting is absent)
look in [rabbitmq] that now can be used to hold default values.
This will requre modifications to both Agent/AgentListener classes and
LinuxMuranoInstance/WindowsInstance core library classes.


Add UI control to environment configuration dialog
``````````````````````````````````````````````````

In order for introduced Environment property to get non-default value
a control need to be added to environment configuration form in dashboard.
It should be a drop-down with region names with possibility to leave it empty
to user server default.

A change of the region must also reset a network selection control because
the selected region is likely to have another set of networks.


Check that dashboard works when murano is missing in some regions
`````````````````````````````````````````````````````````````````

Murano services may be present just in one or in some of available regions.
Selection of region that doesn't run murano must not lead to errors.
Instead user must be presented with a message saying that it is impossible
to create environment in this region (or upload a package etc). However
this region can still be a target for environments in other regions.


Dependencies
============

None

Testing
=======

We should test application deployment with all possible combinations of
API region and environment region (which can be absent). In addition to manual
testing it would be good to improve automated CI tests to deploy applications
to different regions. However it will requires multi-region OpenStack setup for
tests.


Documentation Impact
====================

* Config settings that affect region selection need to be documented in
  murano configuration guide.

* Changes made to UI need to be documented in user's guide.

References
==========

None
