..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=====================================
Actions authentication and visibility
=====================================

URL of launchpad blueprint:

https://blueprints.launchpad.net/murano/+spec/actions-authentication-and-visibility

Application actions is a way for external systems to perform action related
procedures such as maintenance, scaling etc. However those 3rd party
systems are incapable to authenticate themselves to Keystone to call this
API securely.


Problem description
===================

Currently in order to call an action caller needs to have a valid Keystone
token or trust. It is not always possible for cases where 3rd party system is
supposed to be such caller.

There can be a 4 types of authentication for application actions:

#. Public actions. Anybody can call them. No authentication required.
   Usually those are the actions without side effect, but if they do
   they do that on environment owner behalf.
#. Actions that require authentication and could be invoked from both Murano
   UI/CLI (considering user has valid OpenStack credentials) and automation
   systems that cannot authenticate to Keystone.
#. Actions that are supposed to be called from automation systems only.
#. Actions that are supposed to be called by admin only and thus require
   valid OpenStack token.

It must be possible to associate actions with one of the categories above
and to handle authentication according to type that is chosen.


Proposed change
===============

It is proposed to implement pre-authenticated URLs for the actions.
We assume that any external system will be able to execute POST query
on a fixed HTTP URI that doesn't require authentication. Thus application
during deployment phase (when it has a valid Keystone token or trust)
may allocate such a secret URI on murano API server than when invoked would
trigger particular action on behalf of the user that owns the application
(environment).

There should be an ReST API call to allocate such URI and there also should
be a MuranoPL method to call it from inside of applications.

This API call works as following:

#. Creates a keystone trust for special service user (specified in config file)
   to perform particular action
#. Creates a record in database that holds trust ID, project ID, environment
   ID, target object ID, action name and action parameters - everything that
   is necessary to call particular action on particular application with
   specified input.
#. Returns URI that has ID of an inserted record.
#. Murano API should be configured to bypass authentication for such URIs.
#. When such URI is accessed, murano API should extract record ID from the URI.
   Then obtain trust ID and action parameters from the database and call
   the actions using trust ID putting it instead of Keystone token into
   object model.
#. Existing API for calling actions should also work without authentication.
   If it is accessed with Keystone token in HTTP header then it must be
   validated in the API code. Otherwise only public actions (those that do
   not require authentication) must be allowed there.
#. Pre-authenticated URIs can be revoked by revoking the trust and then
   deleting the record from database. This should be also possible from within
   MuranoPL code.



Alternatives
------------

None


Data model impact
-----------------

New database table holding:

* Record ID
* Trust ID
* OpenStack project ID
* Murano environment ID
* Target object (application) ID
* Action name
* Action parameters (JSON)


REST API impact
---------------

Several new API endpoints are added:

**POST /environments/<env_id>/actions-auth/<action_id>**

*Request*

+--------+-------------------------------------------------+----------------------------------------------+
| Method | URI                                             | Description                                  |
+========+=================================================+==============================================+
| POST   | /environments/<env_id>/actions-auth/<action_id> | Creates pre-authenticated URI for the action |
+--------+-------------------------------------------------+----------------------------------------------+

::

    {
      "arg1": "value1",
      "arg2": "value2"
    }

*Response*


+----------------+-------------------------------------+
| Code           | Description                         |
+================+=====================================+
| 200            | Authentication URI has been created |
+----------------+-------------------------------------+

::

    {
      "uri": "full pre-authenticated URI",
      "id": "auth_id"
    }


**DELETE /environments/<env_id>/actions-auth/<auth_id>**

*Request*

+-----------+-----------------------------------------------+------------------------------------+
| Method    | URI                                           | Description                        |
+===========+===============================================+====================================+
| DELETE    | /environments/<env_id>/actions-auth/<auth_id> | Revokes authentication for auth_id |
+-----------+-----------------------------------------------+------------------------------------+

*Response*

+----------------+----------------------------+
| Code           | Description                |
+================+============================+
| 200            | Authentication was revoked |
+----------------+----------------------------+


**POST /environments/<env_id>/actions/pre-auth/<auth_id>**

*Request*

+--------+---------------------------------------------------+----------------------------------------------------+
| Method | URI                                               | Description                                        |
+========+===================================================+====================================================+
| POST   | /environments/<env_id>/actions/pre-auth/<auth_id> | Calls action by its auth_id. If Keystoke header is |
|        |                                                   | provided in HTTP readers it takes presendence over |
|        |                                                   | trust_id.                                          |
+--------+---------------------------------------------------+----------------------------------------------------+

*Response*

+----------------+----------------------------+
| Code           | Description                |
+================+============================+
| 200            | Action was executed        |
+----------------+----------------------------+


Versioning impact
-----------------

None


Other end user impact
---------------------

None

Deployer impact
---------------

Service user credentials should be configured in murano.conf.
Also base part of the URI need to be there as well so that API server
would know load balancer IP it stands behind. Keystone v3 should be enabled.


Developer impact
----------------

None


Murano-dashboard / Horizon impact
---------------------------------

There should be a way for the user to create pre-auth URI from Murano
dashboard.


Implementation
==============

Assignee(s)
-----------

  Stan Lagun <slagun@mirantis.com>

Work Items
----------

#. Define a markup for action visibility in MuranoPL.
#. Improve object model serialization to include action visibility.
#. Create database migration and model code to work with new table.
#. Add support for missing config parameters.
#. Create API endpoint to create pre-authenticated URI.
#. Create API endpoint to revoke pre-authenticated URI.
#. Create API endpoint to invoke pre-authenticated URI.
#. Configure authentication in paste.ini.
#. Improve python-muranoclient with support for new endpoints
#. Implement MuranoPL functions to create/revoke pre-authenticated URIs for
   particular action. Function should return existing Action API endpoint
   for public and token-only actions.
#. Provide the same capabilities in Murano dashboard.


Dependencies
============

None

Testing
=======

Develop sample application that would deploy some 3rd part system
and provide it with pre-authenticated URI that it could call periodically.

Observe action invocation fact by its side-effects (logs etc.)


Documentation Impact
====================

New ReST and MuranoPL APIs need to be documented.


References
==========

None
