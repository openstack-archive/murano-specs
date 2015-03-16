..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Environment Template Catalogue
==========================================


https://blueprints.launchpad.net/murano/+spec/blueprint-template


Problem description
===================

One of powerful Murano use cases is deploying compound applications, composed by set of
different layers such as DB, web server and others, which implies the deployment of
software not just in an unique VM but in several ones. The environment template is the specification of
the set of VMs plus the applications to be installed on top of. The user can define
environment templates from scratch or reuse and customize them (e.g. including keypairs).
Environment templates not only cover instantiation, but also the specification of such templates,
store them in a catalogue and clone them from the abstract template catalog. In this way,
an abstract environment template catalogue as well as a environment template one will be stored in the
database, storing templates shared among all users and individual ones.


Proposed change
===============

In order to fullfill this functionality, a new entity can be introduced in the MURANO
database. It is the Murano environment-template, which contains the specification about what
is going to be deployed in terms of virtual resources and application information, and it can be deployed
on top of Openstack by translating it into environment. This environment template can be created, deleted,
modified and customized by the users. In fact, it can be instantiate as many times as the user wants.
For instance, the user wants to have different deployments from the same environment template: one for
testing and another for production.

The environment template is composed by a set of services/applications with the software to be installed together their
properties to work with. This software can be instantiate over an virtual service.

In this case the workflow for the creation and the instantiation of the environment template will imply:
1.- Creation of the environment template (including application information)
2.- Transformation of the environment template into the environment (creation of the environment, session and
adding applications to the environment
3.- Deploy the environment on top of Openstack.

The environment template structure and information will be similar to the environment, not including some hardware information, like
default network, or virtual server name. Mainly, the environment template information will contain:
- template name, the name for the template. In case, it is not provided, the request will not be valid.
- services, the application information. For each service it will include information about the applications to be
installed (like tomcat), including application properties like tomcat port. In addition, in case applied, the information about
the virtual server (instance) will be incorporated like keyname, flavor, image and so on. The following lines show
an environment template example.

::

 {
        "name": "env_template_name",
        "services": [
        {
            "instance": {
                "assignFloatingIp": "true",
                "keyname": "mykeyname",
                "image": "cloud-fedora-v3",
                "flavor": "m1.medium",
                "?": {
                    "type": "io.murano.resources.LinuxMuranoInstance",
                    "id": "ef984a74-29a4-45c0-b1dc-2ab9f075732e"
                    }
                },
                "name": "tomcat",
                "port": "8080",
                "?": {
                    "type": "io.murano.apps.apache.Tomcat",
                    "id": "54cea43d-5970-4c73-b9ac-fea656f3c722"
                }
            }
        ]
    }

Alternatives
------------

None

Data model impact
-----------------

A environment template entity will be introduced in the MURANO object model. This implies its existence
in the database, the inclusion of Template services and the extension of the API. The template entity
can consist on:

  template :-
    murano:property(temp_id, "created", datetime)
    murano:property(temp_id, "updated", datetime)
    murano:property(temp_id, "id", ID)
    murano:property(temp_id, "name", varchar)
    murano:property(temp_id, "tenant-id", varchar)
    murano:property(temp_id, "version", bigint)
    murano:property(temp_id, "description", text)
    murano:property(temp_id, "networking", text)


REST API impact
---------------
The inclusion of the environment-template entity will imply the extension of the API for the environment-template
creation, deletion, updating and translate into the environment.

**POST /templates**

*Request*

+----------+--------------------------------+--------------------------------------+
| Method   | URI                            | Description                          |
+==========+================================+======================================+
| POST     | /templates                     | Create a new environment template    |
+----------+--------------------------------+--------------------------------------+

*Content-Type*
  application/json

*Example Payload*
This template description can be composed just by the environment template name, or it can
include the description of all services/applications to be deployed.
1.- Just the template
::

    {
        'name': 'env_template_name'
    }

2.- Specification of all the services

::

    {
        "name": "env_template_name",
        "services": [
        {
            "instance": {
                "assignFloatingIp": "true",
                "keyname": "mykeyname",
                "image": "cloud-fedora-v3",
                "flavor": "m1.medium",
                "?": {
                    "type": "io.murano.resources.LinuxMuranoInstance",
                    "id": "ef984a74-29a4-45c0-b1dc-2ab9f075732e"
                },
                "name": "orion",
                "port": "8080",
                "?": {
                    "type": "io.murano.apps.apache.Tomcat",
                    "id": "54cea43d-5970-4c73-b9ac-fea656f3c722"
                }
            }
        ]
    }

*Response*

::


    {
       "updated": "2015-01-26T09:12:51",
       "networking":
       {
       },
       "name": "template_name",
       "created": "2015-01-26T09:12:51",
       "tenant_id": "00000000000000000000000000000001",
       "version": 0,
       "id": "aa9033ca7ce245fca10e38e1c8c4bbf7",
    }



+—————-----------+-----------------------------------------------------------+
| Code           | Description                                               |
+================+===========================================================+
| 200            | OK. Environment Template created successfully             |
+----------------+-----------------------------------------------------------+
| 401            | User is not authorized to access this session             |
+----------------+-----------------------------------------------------------+
| 409            | The environment template already exists                   |
+----------------+-----------------------------------------------------------+

**GET /templates/{env-temp-id}**

*Request*

+----------+--------------------------------+-------------------------------------------------+
| Method   | URI                            | Description                                     |
+==========+================================+=================================================+
| GET      | /templates/{env-temp-id}       | Obtains the enviroment template information     |
+----------+--------------------------------+-------------------------------------------------+

*Parameters:*

* `env-temp-id` - environment template ID, required

*Content-Type*
  application/json

*Response*

::


    {
       "updated": "2015-01-26T09:12:51",
       "networking":
       {
       },
       "name": "template_name",
       "created": "2015-01-26T09:12:51",
       "tenant_id": "00000000000000000000000000000001",
       "version": 0,
       "id": "aa9033ca7ce245fca10e38e1c8c4bbf7",
    }



+—————-----------+-----------------------------------------------------------+
| Code           | Description                                               |
+================+===========================================================+
| 200            | OK. Environment Template created successfully             |
+----------------+-----------------------------------------------------------+
| 401            | User is not authorized to access this session             |
+----------------+-----------------------------------------------------------+
| 404            | The environment template does not exit                    |
+----------------+-----------------------------------------------------------+


**DELETE /templates/{env-temp-id}**

*Request*

+----------+-----------------------------------+-----------------------------------+
| Method   | URI                               | Description                       |
+==========+===================================+===================================+
| DELETE   | /templates/<env-temp-id>          | Delete the template id            |
+----------+-----------------------------------+-----------------------------------+

*Parameters:*

* `env-temp_id` - environment template ID, required


*Response*

+----------------+-----------------------------------------------------------+
| Code           | Description                                               |
+================+===========================================================+
| 200            | OK. Environment Template deleted successfully             |
+----------------+-----------------------------------------------------------+
| 401            | User is not authorized to access this session             |
+----------------+-----------------------------------------------------------+
| 404            | Not found. Specified environment template doesn`t exist   |
+----------------+-----------------------------------------------------------+


**POST /templates/{template-id}/services**

*Request*

+----------+------------------------------------+----------------------------------+
| Method   | URI                                | Description                      |
+==========+====================================+==================================+
| POST     | /templates/{env-temp-id}/services  | Create a new application         |
+----------+------------------------------------+----------------------------------+

*Parameters:*

* `env-temp-id` - The environment-template id, required
* payload - the service description

*Content-Type*
  application/json

*Example*

::

    {
        "instance": {
            "assignFloatingIp": "true",
            "keyname": "mykeyname",
            "image": "cloud-fedora-v3",
            "flavor": "m1.medium",
            "?": {
                "type": "io.murano.resources.LinuxMuranoInstance",
                "id": "ef984a74-29a4-45c0-b1dc-2ab9f075732e"
            }
        },
        "name": "orion",
        "port": "8080",
        "?": {
            "type": "io.murano.apps.apache.Tomcat",
            "id": "54cea43d-5970-4c73-b9ac-fea656f3c722"
        }
    }

*Response*

::


    {
       "instance":
       {
           "assignFloatingIp": "true",
           "keyname": "mykeyname",
           "image": "cloud-fedora-v3",
           "flavor": "m1.medium",
           "?":
           {
               "type": "io.murano.resources.LinuxMuranoInstance",
               "id": "ef984a74-29a4-45c0-b1dc-2ab9f075732e"
           }
       },
       "name": "orion",
       "?":
       {
           "type": "io.murano.apps.apache.Tomcat",
           "id": "54cea43d-5970-4c73-b9ac-fea656f3c722"
       },
       "port": "8080"
    }



+----------------+-----------------------------------------------------------+
| Code           | Description                                               |
+================+===========================================================+
| 200            | OK. Application added successfully                        |
+----------------+-----------------------------------------------------------+
| 401            | User is not authorized to access this session             |
+----------------+-----------------------------------------------------------+
| 404            | The environment template does not exit                    |
+----------------+-----------------------------------------------------------+


**GET /templates/{env-temp-id}/services***
Request*

+----------+-------------------------------------+-----------------------------------+
| Method   | URI                                 | Description                       |
+==========+====================================+====================================+
| GET      | /templates/{env-temp-id}/services   | It obtains the service description|
+----------+-------------------------------------+-----------------------------------+

*Parameters:*

* `env-temp-id` - The environment template ID, required

*Content-Type*
  application/json

*Response*

::

    [
       {
           "instance":
           {
               "assignFloatingIp": "true",
               "keyname": "mykeyname",
               "image": "cloud-fedora-v3",
               "flavor": "m1.medium",
               "?":
               {
                   "type": "io.murano.resources.LinuxMuranoInstance",
                   "id": "ef984a74-29a4-45c0-b1dc-2ab9f075732e"
               }
           },
           "name": "tomcat",
           "?":
           {
               "type": "io.murano.apps.apache.Tomcat",
               "id": "54cea43d-5970-4c73-b9ac-fea656f3c722"
           },
           "port": "8080"
       },
       {
           "instance": "ef984a74-29a4-45c0-b1dc-2ab9f075732e",
           "password": "XXX",
           "name": "mysql",
           "?":
           {
               "type": "io.murano.apps.database.MySQL",
               "id": "54cea43d-5970-4c73-b9ac-fea656f3c722"
           }
       }
    ]


+----------------+-----------------------------------------------------------+
| Code           | Description                                               |
+================+===========================================================+
| 200            | OK. Tier created successfully                             |
+----------------+-----------------------------------------------------------+
| 401            | User is not authorized to access this session             |
+----------------+-----------------------------------------------------------+
| 404            | The environment template does not exit                    |
+----------------+-----------------------------------------------------------+


**POST /templates/{env-temp-id}/create-environment**

*Request*

+----------+--------------------------------------------+--------------------------------------+
| Method   | URI                                        | Description                          |
+==========+================================+==================================================+
| POST     | /templates/{env-temp-id}/create-environment| Create an environment                |
+----------+--------------------------------------------+--------------------------------------+


*Parameters:*

* `env-temp-id` - The environment template ID, required

*Payload:*

* 'environment name': The environment name to be created.


*Content-Type*
  application/json

*Example*

::

    {
        'name': 'environment_name'
    }

*Response*

::

    {
       "environment_id": "aa90fadfafca10e38e1c8c4bbf7",
        "name": "environment_name",
        "created": "2015-01-26T09:12:51",
        "tenant_id": "00000000000000000000000000000001",
        "version": 0,
        "session_id": "adf4dadfaa9033ca7ce245fca10e38e1c8c4bbf7",
    }

+—————-----------+-----------------------------------------------------------+
| Code           | Description                                               |
+================+===========================================================+
| 200            | OK. Environment template created successfully             |
+----------------+-----------------------------------------------------------+
| 401            | User is not authorized to access this session             |
+----------------+-----------------------------------------------------------+
| 404            | The environment template does not exit                    |
+----------------+-----------------------------------------------------------+
| 409            | The environment already exists                            |
+----------------+-----------------------------------------------------------+

Versioning impact
-------------------------

Murano client will change to include this new functionality.

Other end user impact
---------------------

As well as a change in the API to include this new entity, the python-muranoclient will
be changed for including the environment template operations.
* env-template-create  Create an environment template.
* env-template-delete  Delete an environment template.
* env-template-list    List the environment templates.
* env-template-rename  Rename an environment template.
* env-template-show    Show the information of the environment template
* env-template-add-app Add an application to the environment template
* env-template-create-environment It creates an environment from the environment template description

Deployer impact
---------------

None

Developer impact
----------------

None

Murano-dashboard / Horizon impact
---------------------------------

New views will be required for including the environment template functionality


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  hmunfru

Other contributors:
  jesuspg
  TBC

Work Items
----------

1.- Including the environment template entity in database
2.- Extension of the API for environment template catalogue
3.- Generation of environment from template operation
4.- Implement the changes in murano CLI


Dependencies
============


Testing
=======

TBD

Documentation Impact
====================

Environment template documentation should be included.


References
==========

https://etherpad.openstack.org/p/GLLAQ0m1H7
