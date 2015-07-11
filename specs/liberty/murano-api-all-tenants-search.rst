..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===============================
Murano API - All Tenants Search
===============================

https://blueprints.launchpad.net/murano/+spec/murano-api-all-tenants-search

Congress Murano datasource driver pulls environments from one tenant only. The goal
is to pull all environments from all tenants (as nova driver does for servers).

Problem description
===================

Murano - Congress integration is part of a part of Policy Guided Fulfillment.
It uses Congress policy framework to define and evaluate restrictions on Murano
environments. So Murano environments are pulled by Congress Murano datasource driver,
so Congress policy rules can be evaluated.

The problem is that Murano REST API returns environments of one tenant of authenticated user's token only.
Thus Congress policy rules evaluation is run on data from one tenant only.

Other Congress datasource drivers are dealing with similar requirements also - for example
Nova datasource driver pulls data about all servers across all tenants
in its *nova* policy. It is possible because Nova REST API supports search option *all_tenants*.

Note that *Congress policy* is a place of both rules and data related to one *service*.
If *policy* is defined by datasource driver, then its configuration have *user*, *password* and *tenant*,
which are used to get token to access the service.


Proposed change
===============

Search option *all_tenants* will be added to operation *List Environments* of
Murano REST API. When set, the returned list will contain all environments accessible
by the user (specified by token) regardless of tenant.
Listing environments from all tenants can only admin user.


Alternatives
------------

The requested behavior can be also achieved by iterating operation
*List Environment* over all tenants available to the configuration user.
This solution has following performance issues:

* each pull cycle executes the REST operation for every tenant where user is member,
  instead of one execution in case of *all_tenants*

* user's tenant assignment has to be periodically updated, so it leads to another
  requests to keystone each such period


Data model impact
-----------------

None

REST API impact
---------------

* List Environments

  * *all_tenants* parameter is added. When set to *true*, then search over all tenants
    is executed, otherwise search on token's tenant is done

Example (without *all_tenants*):
::

    GET http://<server-name>:8082/v1/environments

    {
      "environments": [
        {
          "status": "deploying",
          "updated": "2015-05-06T08:14:06",
          "networking": {},
          "name": "test",
          "created": "2015-05-06T08:08:40",
          "tenant_id": "cd9e218f9b894ebdb421e9906fbec15e",
          "version": 1,
          "id": "8cc3187c763f4ca9bc58cdaf89f926d3"
        }
      ]
    }

Example (with *all_tenants* - note different *tenant_id*):
::

    GET http://<server-name>:8082/v1/environments?all_tenants=true

    {
      "environments": [
        {
          "status": "deploying",
          "updated": "2015-05-06T08:14:06",
          "networking": {},
          "name": "test",
          "created": "2015-05-06T08:08:40",
          "tenant_id": "cd9e218f9b894ebdb421e9906fbec15e",
          "version": 1,
          "id": "8cc3187c763f4ca9bc58cdaf89f926d3"
        },
        {
          "status": "deploying",
          "updated": "2015-05-08T09:34:16",
          "networking": {},
          "name": "test 2",
          "created": "2015-05-08T08:18:20",
          "tenant_id": "8908989abbeec239023489023ccc1234f",
          "version": 1,
          "id": "abecbf88328932bbecbefe82348238b"
        }
      ]
    }



Versioning impact
-------------------------

None

Other end user impact
---------------------

*python-muranoclient* will be changed as follows:

* *--all-tenants* on CLI

Example:
::

    $ murano environment-list --all-tenants

* *search options* will be supported on API level

Example:
::

    class EnvironmentManager(base.ManagerWithFind):
       def list(self):
       ...

       def list(self, search_opts):
       ...


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

* Introduce *all_tenants* search option in

  * file *murano/api/v1/environments.py*

* Modify *policy.json* file with rules

  * file *etc/murano/policy.json*

* Add support for search options in *python-muranoclient*

  * file *muranoclient/v1/environments.py*

* Add support for *--all-tenants* in *python-muranoclient* CLI

  * file *muranoclient/shell.py*


Dependencies
============

None

Testing
=======

Unit tests should cover server API side also client and shell should be covered.


Documentation Impact
====================

REST API documentation will be modified to mention *all_tenants* search option.

References
==========

* https://wiki.openstack.org/wiki/PolicyGuidedFulfillmentLibertyPlanning

* https://wiki.openstack.org/wiki/PolicyGuidedFulfillmentLibertyPlanning_MuranoAPI
