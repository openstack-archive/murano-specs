..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

================================================
Middleware for external (non-OpenStack) requests
================================================

https://blueprints.launchpad.net/murano/+spec/external-request-middleware

When request is coming to murano (or any other service) from outside of OpenStack
it has no OpenStack-specific requests headers at all. So, it's hard to use
standart middlewares and authentication methods for exteranal requests.

Problem description
===================

Now we need to recreate keystoneclient and get authentication information for each request
which comes from outside of OpenStack using basic auth credentials. This can be look
better if we can use standart keystone middleware for external service requests,
but we don't have enough info (at least token) in the external requests.

Now you can see this behaviour in murano service broker for Cloud Foundry.

Proposed change
===============

Create a new middleware which will handle external requests and add X-Auth-Token
header. This should be enough for keystone middleware and murano context middleware.
Middleware should be added to all external services adaptors (now we have only
service broker). It's not recommended to add it directly to murano-api because
it can be real security issue.

Alternatives
------------

Take everything as it is.

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

None

Deployer impact
---------------

None

Developer impact
----------------

Developers which will create adapters for external service shouldn't worry about
how it will authenticate and work with murano. They can simply add this middleware
to their applications.

Murano-dashboard / Horizon impact
---------------------------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  starodubcevna

Work Items
----------

* Implement external requests middleware

Dependencies
============

None

Testing
=======

Now this can be tested in a bunch of functional tests for Cloud Foundry service
broker.

Documentation Impact
====================

None
