..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==============================
Environment abandoning support
==============================

https://blueprints.launchpad.net/murano/+spec/environment-abandon

The purpose is to add an ability to abandon the environment, which hangs during
deployment or deleting in order to hide it from the list of other environments.


Problem description
===================

There are cases when the process of deleting or deploying environment may hang.
Then the user has to release all used resources and manually clean the database
in order to get rid of the list of failed environments.

Manual cleaning of the database is unsafe and inconvenient operation. That's
why `abandon` feature has to be provided. It should improve usability and
allows to user to avoid direct database editing. But it is necessary to notify
the user that all resources used by abandoned environment must be also released
manually as previously.


Proposed change
===============

The implementation of this feature consists of three stages:

1) Modify environment-delete API endopoint to have *abandon* feature

The method *delete* must be changed. Depending on the parameter "abandon",
obtained from request data, the method should work differently. This parameter
has a boolean type. If it equals *True* enviroment will be directly removed
from the database without object model cleanup.

2) Provide corresponding changes in the python-muranoclient

Method *delete* of environment manager should be modified. New boolean
parameter *abandon* with default value *False* should be added. The value of
parameter affects the building of url, which is sent to murano-api.

3) Add new button *Abandon* to murano-dashboard

This button should be available with any environment state.

Proposed change doesn't solve problem of deployment process hanging.
Murano-engine may continue to deploy abandoned environment. It is
necessary to find a way how to stop murano-engine in this case.

Alternatives
------------

None

Data model impact
-----------------

None

REST API impact
---------------

**DELETE /environments/<environment_id>?abandon**

*Request*


+----------+----------------------------------+----------------------------------+
| Method   | URI                              | Description                      |
+==========+==================================+==================================+
| DELETE   | /environments/{id}?abandon       | Remove specified environment.    |
+----------+----------------------------------+----------------------------------+


*Parameters:*

* `abandon` - boolean, indicates how to delete environment. *False* is used if
  all resources used by environment must be destroyed; *True* is used when just
  database must be cleaned


*Response*

+----------------+-----------------------------------------------------------+
| Code           | Description                                               |
+================+===========================================================+
| 200            | OK. Environment deleted successfully                      |
+----------------+-----------------------------------------------------------+
| 403            | User is not allowed to delete this resource               |
+----------------+-----------------------------------------------------------+
| 404            | Not found. Specified environment doesn`t exist            |
+----------------+-----------------------------------------------------------+


Versioning impact
-------------------------

Murano dashboard will support only the version of the client, that includes
corresponding changes.

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

New action `Abandon` will be added to murano-dashboard. It will be always
available in the row of other action.

Dialog with warning should appear when user executes action


Implementation
==============

Assignee(s)
-----------

Dmytro Dovbii

Primary assignee:
  <ddovbii@mirantis.com>

Work Items
----------

* Modify method 'delete' of environment API to support two delete modes
* Implement adding 'abandon' parameter to url in 'delete' method of environment
  manager in muranoclient
* Add flag '--abandon' to CLI command 'environment-delete'
* Add new class 'AbandonEnvironment' which provide new button 'Abandon' in
  murano-dashboard

Implementation is acutally completed.


Dependencies
============

None


Testing
=======

Functional tests for murano-dashboard must be updated.
Unit tests should cover API call and CLI client
Tempest tests are out of the scope of this spec.


Documentation Impact
====================

API specification should be updated


References
==========

None

