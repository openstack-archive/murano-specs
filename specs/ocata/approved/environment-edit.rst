..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==================================================
Capability to edit existing environment properties
==================================================

https://blueprints.launchpad.net/murano/+spec/environment-edit

The spec proposes mechanism for users to edit any details of an existing
environment, for example edit its regions, set home region, change the
networks or any other possible input field.

Problem description
===================

Current Murano API and python-muranoclient only allow to specify its home
region and default network during the environment creation. These parameters
cannot be seen and cannot be updated by the user after the environment
creation. Users may want to examine and change these or other environment
properties after it is created. Adding a mechanism to apply custom JSON-patch
to environment object model would provide this capability and add a great
flexibility in managing environments.


Proposed change
===============

Changes in API
--------------

It is proposed to create a new API endpoint
`/environments/<env_id>/model` and two calls to it using GET and PATCH
requests.

**GET** request provides environment object model where all environment
properties can be seen. See the `REST API impact` section for the example of
such output. If session is specified in the request and it is not in deploying
state, response gives modified environment model from the session, otherwise
actual environment model.

**PATCH** request allows to perform custom change to the environment object
model using JSON-patch object. It requires session to be specified by request
and saves changes to environment model in the specified session.

JSON-patch is a valid JSON that contains a list of changes to be applied to
the current object. Each change contains a dictionary with three keys: ``op``,
``path`` and ``value``. ``op`` (operation) can be one of the three values:
`add`, `replace` or remove`. *See RFC 6902 [1] for details*.

To indicate that body of the request is JSON-patch and validate correct
format of the body, a new media type should be added -
`application/env-model-json-patch`. During the validation in the middleware,
the following conditions must be checked against the passed object:

* It is a valid JSON-patch object (already implemented for
  `application/murano-packages-json-patch` media type and should be reused).
  Otherwise `400 Bad Request` response is given.

* Not all kinds of operations can be applied to all sections of the
  environment model. The list of allowed operations is the following:

  + defaultNetworks: replace
  + name: replace
  + region: replace
  + regions: add, replace, remove
  + services: add, replace, remove
  + ?: add, replace, remove

  Attempt to perform the restricted change will result in the `403 Forbidden`
  response.

* Types of the environment properties and possible values are validated by the
  appropriate JSON-schema. Otherwise `400 Bad Request` response is given.

Changes in python-muranoclient
------------------------------

To support the new API calls, corresponding python-muranoclient methods for
environments are needed: ``get_model`` and ``update_model`` respectively.

New CLI command that displays the result of ``get_model`` method:

.. code-block:: shell

   murano environment-model-show <ENV_ID> [--path <PATH>] [--session-id <SESSION_ID>]

<PATH> allows to get a specific section of the model, for example
`defaultNetworks`, `region` or `?` or any of the subsections. It defaults to
'/' which means getting the whole model. Specifying <SESSION_ID> allows to get
the state of the environment with pending changes, while omitting it gives
the actual state.

New CLI command that allows to update environment model with a custom patch:

.. code-block:: shell

   murano environment-model-edit <ENV_ID> <FILE> --session-id <SESSION_ID>

The command calls ``update_model`` method with data (JSON-patch) provided in
the <FILE> file. Changes are to be saved in session <SESSION_ID>.

Alternatives
------------

Separate CLI command can be created for updating each environment property.
It will be somewhat easier for users because file with JSON-patch will not be
needed. On the other hand, it will require creating a bunch of commands
instead of one generic and necessity to add new commands in future for the
possible new properties.

Data model impact
-----------------

None

REST API impact
---------------

**Get environment model**

+----------+-------------------------------------+------------------------+--------------------------+
| Method   | URI                                 | Header                 | Description              |
+==========+=====================================+========================+==========================+
| GET      | /environments/<env_id>/model/<path> | X-Configuration-Session| Get an Environment model |
|          |                                     | (optional)             |                          |
+----------+-------------------------------------+------------------------+--------------------------+

Specifying <path> allows to get a specific section of the model, for example
`defaultNetworks`, `region` or `?` or any of the subsections.

*Response*

**Content-Type**
  application/json

::

    {
        "defaultNetworks": {
            "environment": {
                "internalNetworkName": "net_two",
                "?": {
                    "type": "io.murano.resources.ExistingNeutronNetwork",
                    "id": "594e94fcfe4c48ef8f9b55edb3b9f177"
                }
            },
            "flat": null
        },
        "region": "RegionTwo",
        "name": "new_env",
        "regions": {
            "": {
                "defaultNetworks": {
                    "environment": {
                        "autoUplink": true,
                        "name": "new_env-network",
                        "externalRouterId": null,
                        "dnsNameservers": [],
                        "autogenerateSubnet": true,
                        "subnetCidr": null,
                        "openstackId": null,
                        "?": {
                            "dependencies": {
                                "onDestruction": [{
                                    "subscriber": "c80e33dd67a44f489b2f04818b72f404",
                                    "handler": null
                                }]
                            },
                            "type": "io.murano.resources.NeutronNetwork/0.0.0@io.murano",
                            "id": "e145b50623c04a68956e3e656a0568d3",
                            "name": null
                        },
                        "regionName": "RegionOne"
                    },
                    "flat": null
                },
                "name": "RegionOne",
                "?": {
                    "type": "io.murano.CloudRegion/0.0.0@io.murano",
                    "id": "c80e33dd67a44f489b2f04818b72f404",
                    "name": null
                }
            },
            "RegionOne": "c80e33dd67a44f489b2f04818b72f404",
            "RegionTwo": {
                "defaultNetworks": {
                    "environment": {
                        "autoUplink": true,
                        "name": "new_env-network",
                        "externalRouterId": "e449bdd5-228c-4747-a925-18cda80fbd6b",
                        "dnsNameservers": ["8.8.8.8"],
                        "autogenerateSubnet": true,
                        "subnetCidr": "10.0.198.0/24",
                        "openstackId": "00a695c1-60ff-42ec-acb9-b916165413da",
                        "?": {
                            "dependencies": {
                                "onDestruction": [{
                                    "subscriber": "f8cb28d147914850978edb35eca156e1",
                                    "handler": null
                                }]
                            },
                            "type": "io.murano.resources.NeutronNetwork/0.0.0@io.murano",
                            "id": "72d2c13c600247c98e09e2e3c1cd9d70",
                            "name": null
                        },
                        "regionName": "RegionTwo"
                    },
                    "flat": null
                },
                "name": "RegionTwo",
                "?": {
                    "type": "io.murano.CloudRegion/0.0.0@io.murano",
                    "id": "f8cb28d147914850978edb35eca156e1",
                    "name": null
                }
            }
        },
        services: []
        "?": {
            "type": "io.murano.Environment/0.0.0@io.murano",
            "_actions": {
                "f7f22c174070455c9cafc59391402bdc_deploy": {
                    "enabled": true,
                    "name": "deploy",
                    "title": "deploy"
                }
            },
            "id": "f7f22c174070455c9cafc59391402bdc",
            "name": null
        }
    }

+----------------+-----------------------------------------------------------+
| Code           | Description                                               |
+================+===========================================================+
| 200            | Environment model received successfully                   |
+----------------+-----------------------------------------------------------+
| 403            | User is not authorized to access environment              |
+----------------+-----------------------------------------------------------+
| 404            | Environment is not found or specified section of the      |
|                | model does not exist                                      |
+----------------+-----------------------------------------------------------+

**Update environment model**

*Request*

+----------+--------------------------------+------------------------+-----------------------------+
| Method   | URI                            | Header                 | Description                 |
+==========+================================+========================+=============================+
| PATCH    | /environments/<env_id>/model/  | X-Configuration-Session| Update an Environment model |
+----------+--------------------------------+------------------------+-----------------------------+

* **Content-Type**
  application/env-model-json-patch

* **Example**

::

   [{
     "op": "replace",
     "path": "/defaultNetworks/flat",
     "value": true
   }]

*Response*

**Content-Type**
  application/json

See GET request response.

+----------------+-----------------------------------------------------------+
| Code           | Description                                               |
+================+===========================================================+
| 200            | Environment is edited successfully                        |
+----------------+-----------------------------------------------------------+
| 400            | Body format is invalid                                    |
+----------------+-----------------------------------------------------------+
| 403            | User is not authorized to access environment or specified |
|                | operation is forbidden for the given property             |
+----------------+-----------------------------------------------------------+
| 404            | Environment is not found or specified section of the      |
|                | model does not exist                                      |
+----------------+-----------------------------------------------------------+

Versioning impact
-----------------

None

Other end user impact
---------------------

Users will get new CLI commands to examine and edit environment object model.

Deployer impact
---------------

None

Developer impact
----------------

None

Murano-dashboard / Horizon impact
---------------------------------

New python-muranoclient method can be used for providing environment edit
capability from GUI, but this perspective is out of scope of this spec.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  vakovalchuk


Work Items
----------

* Implement new API call to get object model of environment
* Implement new API calls to update the object model of environment
* Implement new python-muranoclient methods to support new API calls
* Add shell command that allows to display environment object model
* Add shell command that allows to apply JSON-patch to the environment model
* Add the same commands to the OpenStack client shell


Dependencies
============

None


Testing
=======

* API calls should be tested by unit tests and tempest tests
* CLI commands should be tested by unit tests


Documentation Impact
====================

* API calls should be described in the Murano API specification
* CLI commands should be described in the end user guide


References
==========

[1] https://tools.ietf.org/html/rfc6902
