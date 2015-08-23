..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Implement Cloud Foundry Service Broker API
==========================================

https://blueprints.launchpad.net/murano/+spec/cloudfoundry-api-support

Cloud Foundry is PaaS which supports full lifecycle from initial development,
through all testing stages, to deployment. Most well known Cloud Foundry
flavours is Cloud Foundry OSS, Pivotal Cloud Foundry and Pivotal Web Services.

If we implement Cloud Foundry Service Broker API in murano, murano apps will
be available at Cloud Foundry as services. So Cloud Foundry users will be
granted an ability to operate murano applications.

Problem description
===================

Typical scenario of Cloud Foundry and murano collaboration will look like:

1. While configuring murano enable murano service broker in config file. Provide
   host and port for it.
2. Add murano Service Broker in Cloud Foundry and grant access to murano apps for
   Cloud Foundry organization.
3. Provision murano app as Cloud Foundry service instance using Cloud Foundry tools.

Proposed change
===============

We need to write Cloud Foundry Service Broker implementation for murano. One of
the parts of this implementation should be some mapping function between Cloud
Foundry and OpenStack resources. Now it's planned to map Cloud Foundry
Organizations to OpenStack tenants and Cloud Foundry spaces to murano
environments. So, all tenant users will be granted with the privileges based on
their existing roles in OpenStack tenant. Each Cloud Foundry space will be linked
wih murano environment. The parameters which is needed to murano for successful
application deployment will store in service object section parameters. The
Service Broker itself will parse them as soon as Cloud Foundry can't do it.
It will be parsed during Provision request. The request body will look like that:

.. code-block:: javascript

    {
      "service_id":        "service-guid-here",
      "plan_id":           "plan-guid-here",
      "organization_guid": "org-guid-here",
      "space_guid":        "space-guid-here",
      "parameters":{

                    "parameter1": 1,
                    "parameter2": "value"

                   }

    }

It's planned to setup a Service Broker as a separate service. Additional
options should be added to the configs. Also we want to use Cloud Foundry
experimental asynchronous operations[3].

Alternatives
------------

None

Data model impact
-----------------

None

REST API impact
---------------

No existing API of murano service is going to be changed. New service will be
created implementing the standard Cloud Foundry Service Broker API. Its API
methods are as follows:

**GET /v2/catalog**

*Request*


+----------+----------------------------------+----------------------------------+
| Method   | URI                              | Description                      |
+==========+==================================+==================================+
| GET      | /v2/catalog                      | List all available apps          |
+----------+----------------------------------+----------------------------------+


*Parameters:*

* None

*Response*

+----------------+-----------------------------------------------------------+
| Code           | Description                                               |
+================+===========================================================+
| 200            | OK. The expected resource body as below                   |
+----------------+-----------------------------------------------------------+

::

    {
     "services": [{
        "id": "service-guid-here",
        "name": "mysql",
        "description": "A MySQL-compatible relational database",
        "bindable": true,
        "plans": [{
          "id": "plan1-guid-here",
          "name": "small",
          "description": "A small shared database with 100mb storage quota and 10 connections"
        },{
          "id": "plan2-guid-here",
          "name": "large",
          "description": "A large dedicated database with 10GB storage quota, 512MB of RAM, and 100 connections",
          "free": false
        }],
        "dashboard_client": {
          "id": "client-id-1",
          "secret": "secret-1",
          "redirect_uri": "https://dashboard.service.com"
        }
      }]
    }

**PUT /v2/service_instances/:instance_id?accepts_incomplete=true**

*Request*

+----------+-----------------------------------------------------------+-------------------------------------------+
| Method   | URI                                                       | Description                               |
+==========+===========================================================+===========================================+
| PUT      | /v2/service_instances/:instance_id?accepts_incomplete=true| Create new service resources for developer|
+----------+-----------------------------------------------------------+-------------------------------------------+

::

    {
      "service_id":        "service-guid-here",
      "plan_id":           "plan-guid-here",
      "organization_guid": "org-guid-here",
      "space_guid":        "space-guid-here"
    }

*Response*

+----------------+------------------------------------------------------------+
| Code           | Description                                                |
+================+============================================================+
|                | OK. May be returned if the service instance already exists |
| 200            | and the requested parameters are identical to the existing |
|                | service instance.                                          |
+----------------+------------------------------------------------------------+
| 202            | Accepted. Service instance creation is in progress.        |
+----------------+------------------------------------------------------------+
| 409            | Conflict. Should be returned if the requested service      |
|                | instance already exists. The expected response body is “{}”|
+----------------+------------------------------------------------------------+
| 422            | Should be returned if the request did not include          |
|                | ?accepts_incomplete=true                                   |
+----------------+------------------------------------------------------------+

::

    {
     "dashboard_url": "http://example-dashboard.com/9189kdfsk0vfnku"
    }

**PATCH /v2/service_instances/:instance_id?accepts_incomplete=true**

*Request*

+----------+-----------------------------------------------------------+----------------------------------+
| Method   | URI                                                       | Description                      |
+==========+===========================================================+==================================+
| PATCH    | /v2/service_instances/:instance_id?accepts_incomplete=true| Update existing service instance |
+----------+-----------------------------------------------------------+----------------------------------+

::

    {
     "plan_id": "plan_guid_here"
    }

*Response*

+----------------+------------------------------------------------------------+
| Code           | Description                                                |
+================+============================================================+
| 200            | Return if only the new plan matches the old one completely |
+----------------+------------------------------------------------------------+
| 202            | Accepted. Service instance update is in progress.          |
+----------------+------------------------------------------------------------+
| 422            | Should be returned if the request did not include          |
|                | ?accepts_incomplete=true                                   |
+----------------+------------------------------------------------------------+

**DELETE /v2/service_instances/:instance_id?accepts_incomplete=true**

+----------+-----------------------------------------------------------+-----------------------------------+
| Method   | URI                                                       | Description                       |
+==========+===========================================================+===================================+
| DELETE   | /v2/service_instances/:instance_id?accepts_incomplete=true| Delete all resources create during|
|          |                                                           | the provision.                    |
+----------+-----------------------------------------------------------+-----------------------------------+

*Response*

+----------+--------------------------------------------------+
| Code     | Description                                      |
+==========+==================================================+
| 202      | Accepted. Service instance deletion in progress. |
+----------+--------------------------------------------------+
| 410      | Returned if service does not exist               |
+----------+--------------------------------------------------+
| 422      | Should be returned if the request did not include|
|          | ?accepts_incomplete=true                         |
+----------+--------------------------------------------------+

**PUT /v2/service_instances/:instance_id/service_bindings/:binding_id**

*Request*

+----------+----------------------------------------------------------------+----------------------------------+
| Method   | URI                                                            | Description                      |
+==========+================================================================+==================================+
| PUT      | /v2/service_instances/:instance_id/service_bindings/:binding_id| Bind service                     |
+----------+----------------------------------------------------------------+----------------------------------+

::

    {
     "plan_id": "plan_guid_here",
     "service_id": "service_guid_here",
     "app_guid": "app_guid_here"
    }

*Response*

+----------------+------------------------------------------------------------------+
| Code           | Description                                                      |
+================+==================================================================+
| 201            | Binding has been created. The expected response body is below.   |
+----------------+------------------------------------------------------------------+
| 200            | May be returned if the binding already exists and the requested  |
|                | parameters are identical to the existing binding. The expected   |
|                | response body is below.                                          |
+----------------+------------------------------------------------------------------+
| 409            | Should be returned if the requested binding already exists. The  |
|                | expected response. body is `{}`, though the description field can|
|                | be used to return a user-factorin error message.                 |
+----------------+------------------------------------------------------------------+

::

    {
      "credentials": {
        "uri": "mysql://mysqluser:pass@mysqlhost:3306/dbname",
        "username": "mysqluser",
        "password": "pass",
        "host": "mysqlhost",
        "port": 3306,
        "database": "dbname"
      }
    }

**DELETE /v2/service_instances/:instance_id/service_bindings/:binding_id**

*Request*

+----------+----------------------------------------------------------------+----------------------------------+
| Method   | URI                                                            | Description                      |
+==========+================================================================+==================================+
| DELETE   | /v2/service_instances/:instance_id/service_bindings/:binding_id| Unbind service                   |
+----------+----------------------------------------------------------------+----------------------------------+

*Response*

+----------+-----------------------------------+
| Code     | Description                       |
+==========+===================================+
| 200      | Binding was deleted               |
+----------+-----------------------------------+
| 410      | Returned if binding does not exist|
+----------+-----------------------------------+

**GET /v2/service_instances/:instance_id/last_operation**

*Request*

+----------+--------------------------------------------------+-------------------------------+
| Method   | URI                                              | Description                   |
+==========+==================================================+===============================+
| GET      | /v2/service_instances/:instance_id/last_operation| Polling status of the last 202|
|          |                                                  | operation                     |
+----------+--------------------------------------------------+-------------------------------+

*Response*

+----------+--------------------------------------------------------+
| Code     | Description                                            |
+==========+========================================================+
| 200      | OK                                                     |
+----------+--------------------------------------------------------+
| 410      | GONE. Appropriate only for asynchronous delete requests|
|          | Cloud Foundry will consider this response a success and|
|          | remove the resource from its database.                 |
+----------+--------------------------------------------------------+

::

    {
      "state": "in progress",
      "description": "Creating service (10% complete)."
    }


Versioning impact
-------------------------

None

Other end user impact
---------------------

None

Deployer impact
---------------

Service Broker should be deployed and enabled in the murano config.

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
  starodubcevna

Work Items
----------

Changes can be split to this parts:

* Implement the stub of Service Broker itself. Add needed config opts and starting point.

* Implement basic Cloud Foundry API calls such as list and provision. Also on this step we
  should add murano specific API calls.

* Series of extensions for Cloud Foundry API support:
   * Add update and deprovision API calls
   * Add bind/unbind API calls

Dependencies
============

None

Testing
=======

Unit tests should cover new API calls.

Documentation Impact
====================

Document "Murano and Cloud Foundry HowTo". It should be step by step guide for
Cloud Foundry and murano cooperation.


References
==========

[1] https://youtu.be/ezq9P1WN2LY
[2] http://docs.cloudfoundry.org/services/api.html
[3] https://docs.cloudfoundry.org/services/asynchronous-operations.html
