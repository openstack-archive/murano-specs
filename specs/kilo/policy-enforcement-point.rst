..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode


====================================================
Policy Guided Fulfillment - Policy Enforcement Point
====================================================

URL of launchpad blueprint:

https://blueprints.launchpad.net/murano/+spec/policy-enforcement-point


Problem description
===================

As a part of policy guided fulfillment we need to implement *predeploy* policy enforcement point - i.e., Murano calls Congress to evaluate *predeploy* policy rules on data representing Murano environment being deployed. If evaluation returns *predeploy error* data (i.e., enforcement failed), then deployment of Murano environment fails.

*Predeploy* policy rules are represented by *predeploy_error(env_id, obj_id, message)* congress table.
It means

* Congress administrator is responsible for creating predeploy rules, which has this table on the left side (see examples).
* Murano is not involved in policy rule evaluation, except that Murano provides data about Murano environments. Thus user can use any data (e.g., datasource tables, policy rules) available in Congress to define when environment can be deployed.

Table *predeploy_error(env_id, obj_id, message)* reports list of found errors:

* *env_id* environment id where error was detected

* *obj_id* object id (in environment id) on which error was detected

* *message* text message of error

Murano environment serialization/decomposition is described in `Congress Support in Murano <https://blueprints.launchpad.net/murano/+spec/congress-support-in-murano>`_ and `Murano Congress Driver <https://blueprints.launchpad.net/congress/+spec/murano-driver>`_

Example (generic):
::

    predeploy_error(env_id, obj_id, message) :-
        murano:state(env_id,"PENDING"),
        my-rule-table1(env_id, obj_id),
        concat("", "some error message 1", message)

    predeploy_error(env_id, obj_id, message) :-
        murano:state(env_id,"PENDING"),
        my-rule-table2(env_id, obj_id),
        concat("", "some error message 2", message)

    my-rule-table1(env_id, obj_id) :- ....

    my-rule-table2(env_id, obj_id) :- ....


Example (allow only environments where all VM instances has flavor with max 2048MB RAM):
::

    predeploy_error(eid, oid, msg) :-
        murano:object(oid, eid, type),
        checkIfError(oid),
        concat("", "Instance flavor has RAM size over 2048MB", msg)

    checkIfError(oid) :-
        murano:parent_type(oid, "io.murano.resources.Instance"),
        murano:property(oid, "flavor", fname),
        nova:flavors(i,fname,v,r,d,e,rx),
        gt(r,2048)


Enforcement point will use Congress simulation api to evaluate rules on passed mapped environment into Congress schema.


Proposed change
===============

When user executes *deploy* action on an environment following steps will be done

* environment will be mapped into *update sequence*

* Congress simulation API will be executed:

::

 openstack congress  policy simulate murano_system 'predeploy-error(envId, objId, message)'  'env+(1000,"myEnv") obj+(1,1000,"VM") prop+(100,1,"name", "vm1")  prop+(101,1,"image", "ubuntulinuximg") obj+(2,1000,"Tomcat") prop+(110,2,"name", "tomcat") prop+(111,2,"port", 8080) rel+(200,2,1, "instance") '  action

* if response will contain non empty result, then deployment will fails


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

Environment deployment may fail due to policy validation failure.

Deployer impact
---------------

Policy enforcement will be used only if

* Enforcement is enabled in *murano.conf*
* Congress is available in Keystone catalog (i.e., it is deployed in OpenStack)

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
  ondrej-vojta

Other contributors:
  filip-blaha, radek-pospisil

Work Items
----------

1. Use implementation of *Congress Support in Murano* in order to implement policy enforcement point as advised by Stan (see below). The Congress support must correctly deal with following setups

* Openstack with Congress installed
* Openstack without Congress

::

 Stan: such approach makes PolicyEnforces (and thus dependency on Congress) be   mandatory for Murano. Better approach for now could be just insert
 at https://github.com/stackforge/murano/blob/master/murano/common/engine.py#L112
 something like

 if config.CONF.enable_policy_enforcer:
      policyenforcer.validate(self.model)

2. Provide Developer and User Documentation (see Documentation section).

Dependencies
============
* *Congress Support in Murano* https://blueprints.launchpad.net/congress/+spec/murano-driver
* *Murano datasource driver* in Congress https://blueprints.launchpad.net/congress/+spec/murano-driver
* *Policy enforcement specification* https://etherpad.openstack.org/p/policy-congress-murano-spec



Testing
=======

Unit and integration tests must be done.

Integration tests must cover following setups

* Openstack has Congress installed
    * situations when Congress is running (i.e., responding) and not running (i.e., not responding) must be tested
* Openstack has not Congress installed


Documentation Impact
====================

Policy enforcement must be documented from following perspectives

* Setup configuration (e.g., with and without congress)
* Murano rules (i.e., Murano environment data decompositiion) in Murano policy
* How Murano policy affects environment deployment

Following Murano documentation will be affected

* Murano Installation Guide
    Add section on Congress requirement and section on enabling policy enforcement
* Murano Workflow
    Add section on Murano policy enforcement
* Murano Article (new)
    Article on Murano policy rules (e.g., Murano environment decomposition to Congress)

References
==========

* *Congress Support in Murano* https://blueprints.launchpad.net/congress/+spec/murano-driver
* *Murano datasource driver* in Congress https://blueprints.launchpad.net/congress/+spec/murano-driver
* *Policy enforcement specification* https://etherpad.openstack.org/p/policy-congress-murano-spec
