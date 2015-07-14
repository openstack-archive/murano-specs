..

 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

========================================
Policy Based Modification of Environment
========================================

https://blueprints.launchpad.net/murano/+spec/policy-based-env-modification

Goal is to be able to define modification of an environment by Congress policies prior
deployment. This allows to add components (for example monitoring), change/set properties
(for example to enforce given zone, flavors, ...) and relationships into environment,
so modified environment is after that deployed.

Problem description
===================

Currently it is possible to reject deployment of an environment if it does not follows
set of so called pre-deployment policies set by admin. Administrator wants to also modify
environment prior it is deployed:

* add/set/remove component properties
* add/remove relationships
* add/remove objects

**Example Use Case: Policy Based Monitoring**

Admin wants to monitor an environment, so he wants to

* install monitoring agent on each Instance

 * it is done by adding component with the agent and creating relationship between
   agent and Instance. It is done at pre-deploy time

* register monitoring agent on Monitoring server

 * it is done by calling monitoring server API during deployment of monitoring agent.


Proposed change
===============

**Changes**

* Introduce new Congress policy rule *predeploy_modify(eid,oid,modify-action-id,priority,
  [key-val]\*)*

  *predeploy_modify* policy rule is queried on all actions.
  Simulation Congress API is used like in case of *predeploy_errors* policy rule.

  If it returns non empty list of *modifications* for given environment, then

 * *deploy* action is temporarily paused, until all modifications are processed

  * if any of modification fails, then environment *deploy* fails

* Pluggable modification actions
  Modification actions can be plug using setup *entry_points*.

  Out of box, there will be following modification actions

 * add_property( name=name, value=value)
 * remove_property( name=name)
 * set_property( name=name, value=value)
 * add_relationship( name=name, source=source-uuid, target=target-uuid)
 * remove_relationship( name=name, object=object-uuid)
 * add_object( type=type, owner=owner-uuid, owner-rel-name=name, [name=val]*)
 * remove_object( object=object-uuid)

Alternatives
------------

Alternative can be usage of *executes[]* of Congress policy, which executes modify
actions. In this approach

* modify action has to be implemented as Congress datasource action
* triggering of executes[] has to be solved
* it is not possible to order modify action ordering
* Murano session-id of REST API must be passed to Congress
* actions can be executed only as asynchronous, so it is not possible to postpone
  *deploy* environment action until all modify actions are finished

Thus it is not alternative.

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

User (admin) can control modification by creating *predeploy_modify* Congress policy
rules.

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
  filip-blaha

Work Items
----------

* design API of modify actions
* framework for pluggable modify actions - registering and managing available actions
* implement out-of-box actions
* add point to engine where congress called and returned action list is processed on given environment

Dependencies
============

None

Testing
=======

We need to cover by unit tests:
* framework for registering/managing modify actions
* applying modify actions on environment
* processing action list returned by congress

We need to create functional tests covering end-to-end scenario.

Documentation Impact
====================

It is documented as part of policy guided fulfillment.

References
==========

