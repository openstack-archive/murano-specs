..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===========================
Public Environment Template
===========================

URL of launchpad blueprint:

https://blueprints.launchpad.net/murano/+spec/abstract-env-template

Continuing with the environment template catalog blueprint, which allowed
to store complex architecture for applications, an abstract environment
template catalog can be included.
It involves the existence of an environment template catalogue independent
on any user, which can be customized by anyone to include their concrete
information (e.g. keypair).
Concretely, it involves all the operations to manage environment templates
(creation, deletion, updating), plus the customization done by the user.
At the end, the customized environment template can be launched and converted
into an environment


Problem description
===================

MURANO environment template catalogue is composed by environment templates
which belong only to its owner. Thus, it is not possible to share or
reuse environment templates among tenants, since all are privates. The
idea is to have the possibility to have public templates beside private one,
which can be reused and customized by other tenants. In this
way the abstract environment template catalogue will be composed by all public
environment template from all tenants.


Proposed change
===============

Including public environment templates in MURANO requires to modify the
environment template entity in the model to include a property "is_public",
which describes whether the template is public or private. In addition, it is
required to extend the API to allow for obtaining all public environment
templates from all tenants, or customizing a concrete environment template
from another tenant.

Alternatives
------------

None

Data model impact
-----------------

A new property will be include in the environment template model.

  environment-template :-
    murano:property(temp_id, "is_public", bool)


REST API impact
---------------
The inclusion of the environment-template entity will imply the extension of
the API for the environment-template creation, deletion, updating and
translate into the environment.

**GET /templates?is_public=true/false**

*Request*

+----------+-----------------------------+-----------------------------------+
| Method   | URI                         | Description                       |
+==========+=============================+===================================+
| GET      |/templates?is_public         | Get all public/privates template  |
|          |                             | from the tenant or all tenants.   |
+----------+-----------------------------+-----------------------------------+

*Parameters:*

+----------------+-----------------------------------------------------------+
| Parameter      | Description                                               |
+================+===========================================================+
| `is_public`    |boolean, indicates whether public environment templates are|
|                |listed or not.                                             |
+----------------+-----------------------------------------------------------+


Concretely, it is possible to have the following cases:
* GET /templates?is_public=true. All public templates from all tenants will be
returned.
* GET /templates?is_public=false. All private templates from current tenant
will be returned.
* GET /templates. All templates from current tenant plus all public templates
from all tenants will be returned.


* **Content-Type**
  application/json

*Response*

::

    [{
       "updated": "2015-01-26T09:12:53",
       "name": "env_template_name1",
       "created": "2015-01-26T09:12:51",
       "tenant_id": "00000000000000000000000000000001",
       "version": 0,
       "is_public": true,
       "id": "aa9033ca7ce245fca10e38e1c8c4bbf7",
    }
    {
       "updated": "2015-01-26T09:12:55",
       "name": "env_template_name2",
       "created": "2015-01-26T09:12:51",
       "tenant_id": "00000000000000000000000000000082",
       "version": 0,
       "is_public": true,
       "id": "aa9033ca7ce245fca10easdfasdfadsfaasdf",
    }]



+----------------+-----------------------------------------------------------+
| Code           | Description                                               |
+================+===========================================================+
| 200            | OK. Environment Templates obtained successfully           |
+----------------+-----------------------------------------------------------+
| 401            | User is not authorized to perform the operation           |
+----------------+-----------------------------------------------------------+


**POST /templates/{env-temp-id}/clone**

*Request*

+----------+------------------------------+-----------------------------------+
| Method   | URI                          | Description                       |
+==========+==============================+===================================+
| POST     |/templates/{env-temp-id}/clone|It clones a template from one      |
|          |                              |tenant to another, changing its    |
|          |                              |name, its tenant-id and its public |
|          |                              |availability if required.          |
+----------+------------------------------+-----------------------------------+

*Parameters:*

* `env-temp-id` - environment template ID, required

*Example Payload*
::

    {
        'name': 'cloned_env_template_name'
    }

*Content-Type*
  application/json

*Response*

::

    {
       "updated": "2015-01-26T09:12:51",
       "name": "cloned_env_template_name",
       "created": "2015-01-26T09:12:51",
       "tenant_id": "00000000000000000000000000000001",
       "version": 0,
       "is_public": false,
       "id": "aa9033ca7ce245fca10e38e1c8c4bbf7",
    }



+----------------+-----------------------------------------------------------+
| Code           | Description                                               |
+================+===========================================================+
| 200            | OK. Environment Template cloned  successfully             |
+----------------+-----------------------------------------------------------+
| 401            | User is not authorized to perform the operation           |
+----------------+-----------------------------------------------------------+
| 404            | The environment template does not exit                    |
+----------------+-----------------------------------------------------------+
| 409            | Conflict. The environment template name already exists    |
+----------------+-----------------------------------------------------------+


Versioning impact
-------------------------

None

Other end user impact
---------------------

As well as a change in the API to include this new entity, the
python-muranoclient will be changed for including the environment template
clone.
* env-template-clone   Clone the environment template.


Deployer impact
---------------

None

Developer impact
----------------

None

Murano-dashboard / Horizon impact
---------------------------------

New views will be required for including the public template catalog and the
clone functionality.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  hmunfru

Other contributors:
  jesuspg


Work Items
----------
1.  Including the is_public in the model
#.  Extension of the environment template API for the is_public parameter.
#.  Extension of index operation and creation of clone functionality in API.
#.  Adding functional tests.

Dependencies
============


Testing
=======

Unit and functional tests should be implemented.

Documentation Impact
====================

Environment template documentation should be included.


References
==========

