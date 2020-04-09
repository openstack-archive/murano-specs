..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================================================
Policy Guided Fulfillment - Congress Support in Murano
======================================================

URL of launchpad blueprint:

https://blueprints.launchpad.net/murano/+spec/congress-support-in-murano

Problem description
===================

As a part of policy guided fulfillment we need to call Congress from Murano,
to enforce the policy on Murano environment. For current release
the enforcement will be done by Congress simulation API, where Murano will
query table *predeploy_error* in *murano_system* policy for passed update
sequence created by mapping of environment into congress schema.

Proposed change
===============

We need to provide

* python congress client in Murano

    We will use simulation feature of `congress API
    <https://docs.google.com/document/d/14hM7-GSm3CcyohPT2Q7GalyrQRohVcx77hx
    Ex4AO4Bk/edit#heading=h.ll03wo2z9pcb>`_.
    Using this API Murano will send decomposed Murano environment to Congress
    tables, so simulation can evaluate *predeploy_error* rule without changing
    existing data in Congress.

* mapping of Murano Environment into *update sequence* used in simulation API

Murano To Congress Mapping Details
----------------------------------
This section provides congress schema created for Murano environment mapping.
It will be created by Congress datasource drivers - see `murano driver spec
<https://blueprints.launchpad.net/congress/+spec/murano-driver>`_ .

* **Policies**

    * **murano**

        Dedicated policy for Murano data.

    * **murano_system**
        Dedicated policy for rules

* **Schema**

    * *murano:objects(obj_id, owner_id, type)*

      This table holds every MuranoPL object instance in an environment.

          * *obj_id* - uuid of the object as used in Murano
          * *owner_id* - uuid of the owner object as used in Murano
          * *type* - string with full type indentifier as used in Murano
            (e.g., io.murano.Environment,...)

    * *murano:parent_types(obj_id, parent_type)*

      This table holds parent types of *obj_id* object. Note that Murano
      supports multiple inheritance, so there can be several parent types for
      one object

    * *murano:properties(obj_id, name, value)*

        This table stores object's properties. For multiple cardinality
        properties, there can be number of records here. MuranoPL properties
        referencing class type (i.e., another object) are stored in
        *murano:relatinoship*. Properties with structured content will be
        stored component by component.

    * *murano:relationships(src_id, trg_id, name)*

        This table stores relationship between objects (i.e., MuranoPL property
        to *class*). For multiple cardinality relationships several records
        should be stored.

    * *murano:connected(src_id, trg_id)*

        This table stores tuples of objects connected directly and indirectly
        via relationship. It is necessary since Congress does not support
        recursive rules yet.

    * *murano:states(end_id, state)*

        This table stores *EnvironmentStatus* of Murano environment ( one of
        'ready', 'pending', 'deploying', 'deploy failure', 'deleting',
        'delete failure' ).


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
  filip-blaha

Other contributors:
  ondrej-vojta, radek-pospisil

Work Items
----------

1. Introduce Congress python client into Murano

2. Implement mapping of Murano Environment into Congress simulation API
   update-sequence.

3. Provide tests

Dependencies
============

* Congress python client ( `GIT <https://opendev.org/openstack/python-congressclient>`_ )
  will be added into Murano.
* *Murano datasource driver* in Congress `murano driver spec
  <https://blueprints.launchpad.net/congress/+spec/murano-driver>`_

Testing
=======

Testing will use predefined Congress policy rules in order to test client and
mapping. See https://etherpad.openstack.org/p/policy-congress-murano-spec for
an example mapping and test.

Documentation Impact
====================

Documentation impact is specified in `Policy Enforcement Point <https://bluepri
nts.launchpad.net/murano/+spec/policy-enforcement-point>`_ blueprint.


References
==========

* *Murano datasource driver* in Congress
  https://blueprints.launchpad.net/congress/+spec/murano-driver
* https://blueprints.launchpad.net/murano/+spec/policy-enforcement-point
* https://etherpad.openstack.org/p/policy-congress-murano-spec
