..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================
Application policies
====================

https://blueprints.launchpad.net/murano/+spec/application-policies

The spec proposes mechanism that cloud operators can use to impose
constraints on applications and alter application behavior without
making modifications to the source code of the apps.


Problem description
===================

Cloud operators often want to have control over what Murano applications can
or cannot do, impose limitations on environment content or alter applications
behavior somehow in a way that can be applied to a wide range of applications.

Possible policies that operator might want:

* Have a black list of applications (that either has explicit application names
  or all apps in particular namespace) including their inheritors for
  particular tenant

* Make each deployed environment have particular application

* Want particular script to be executed on each VM spawned regardless of the
  image being used by the application

* Want to have quota on the number of instances that can be created by a single
  application

* Do not allow to use large instance flavors for all applications except for
  explicitly approved

* Change application defaults so that particular promoted image will become
  the default value in UI for all applications that work on this OS

* Send notification when number of spawned instances in particular environment
  exceeds quota

* Change default network topology for particular tenant

* Implement custom billing based on the application activities without
  requiring all applications to emit notifications for them.

* Honor policies from the third party policy engine

As you can see all of the policies above are very different in their nature,
require different inputs and different ways to apply them. Also there can
be much more use cases and their variants in order have upfront support for
all of them.

Thus Murano needs a generic way to write policy enforcement rules and actions.


Proposed change
===============

Each policy case consists of two parts: rules specification and code that
applies them.

The code is going to be written as MuranoPL classes that implement required
interface. These classes are called aspects. Each aspect captures and applies
one policy pattern.

Policy rules are going to be the input for those classes (their property
values, similar to our regular object model).

Examples of such input might be:

* Endpoint of the third party policy engine

* Name of the image

* List of YAQL predicates (policy rules)

* Parameters of the largest flavor possible

So each instance of aspect class provided with the values for the rules
parameters can be called a policy as it contains both parts of the policy
definition.

Aspects are provided using the same Murano package format and reside in the
same catalog as applications are. Aspects are going to be located in the
catalog in the packages of the new type ``Aspect``. All aspect classes will be
inherited from the base ``Aspect`` class which should implement some basic
interface for aspects as well as some boilerplate code.

Policy UI workflow
------------------

To use policies, admin should execute the following workflow in UI:

* Import packages with aspect classes (usual package upload).

* Create policies by adding aspects to the list of the ones he wants to apply
  and specifying the policy rules as an input parameters. The same UI workflow
  (including dynamic UI parts) that is used for adding regular Murano
  applications to environments is used here. The difference is that no
  environment is needed.

* Apply aspects for particular tenants or globally. New type of GUI form must
  be created for that but it should be quite simple.

Instances of aspects, a.k.a policies (i.e. their serialized representation) are
going to be created independently from environments and stored in a dedicated
table in the database. Terms "aspect instance" and "policy" can be used
interchangeable further in this spec to make accent on the one or another shade
of the meaning.

In order for aspects to do their job they are going to be provided with
additional capabilities that are not available to the rest of MuranoPL apps.

Aspects event hooking
---------------------

Aspects should have ability to hook themselves to various system events
produced by the engine. These could be:

  * Object model load events

  * Package load events

  * Object creation events that happen before/after new() function is used
    anywhere in the code

  * Garbage collection events

In the initial implementation it is proposed to introduce hooking just to
object creation and package load events. It covers the most important use
cases of policies (changing MuranoPL class code and changing object property
values).

Two new MuranoPL classes need to be added to the Murano core library:

  * ``io.murano.policy.Aspect`` - the basic interface and boilerplate code
    provider for all custom aspects. It has the following methods:

    + ``subscribe(notifier)`` - method which invokes notifier's ``subscribe``
      method passing the calling aspect, event name and aspect's method names
      as arguments. It subscribes aspect to notifications from notifier.
      Subscribes to all events in the base ``Aspect`` class and can be
      overridden by inheritor not to subscribe to some events or to subscribe
      to some event several times with different conditions and handlers.
      This method is called by the engine right after aspects and notifier
      objects initialization

    + *condition methods* - methods which define the condition needed to be
      checked against some item (e.g. initializing object in case of object
      init event, class in case of package load event) to determine whether
      event handler method of the aspect should be called with that item. This
      group of methods include ``objectInitCondition(object)`` and
      ``packageLoadCondition(class)``. ``modelLoadCondition(model)`` can be
      added later. These methods return ``false`` in the base class which means
      no item is handled. It should be overridden by inheritor to specify
      actual condition. For example, ``objectInitCondition`` can check that
      object is the instance of particular class or that it has some property

    + *handling methods* - methods which apply policy rules to some item (e.g.
      initializing object in case of object init event, class in case of
      package load event) when corresponding event occurs. So it is the core
      part of the policy. This group of methods include
      ``handleObjectInit(object)`` and ``handlePackageLoad(class)``.
      ``handleModelLoad(model)`` can be added later. The body of these methods
      is empty in the base class. Other handling methods can be also added and
      used in the inheritor class

  * ``io.murano.policy.Notifier`` - the class to manage notifications of
    the proper aspects about system events. It has one property ``_handlers``
    which holds the dict with event names as keys and lists of event handlers
    as values. Each handler is a dict with aspect object, its condition method
    name and handling method name. ``Notifier`` has the following methods:

    + ``subscribe(subscriber, eventName, methodName, conditionMethodName)`` -
      method to populate notifier's ``_handlers`` dict. It is called by each
      aspect that wants to subscribe to some event

    + ``callHandler(handler, item)`` - method that firstly invokes
      conditionMethod stated in the handler dict, in case it returns ``true``
      for the item, invokes handling method with the item

    + ``onEvent(event, item)`` - method that invokes ``callHandler`` for all
      handlers under the ``event`` key of ``_handlers`` dict. This method is
      invoked by engine when the event occurs

Extended MuranoPL reflection
----------------------------

In addition to standard MuranoPL reflection aspects must be provided with
methods that can modify MuranoPL code model:

  * change property declarations including default values and contracts

  * change methods: wrap it in another method, change argument contracts and
    default values

Also aspect may want to modify the metadata which defines UI layout of the
application. For example, add ``io.murano.metadata.forms.Hidden`` class to
some property along with the inserting some predefined default value for this
property. It will result in hiding the corresponding field in GUI and using
the defined value. This part can be used once the new dynamic UI generation
from schema is introduced.

These capabilities can be written as new MuranoPL methods similar to ordinary
reflection and provided to aspects by registering them to the package context
of ``Aspect`` packages. Thus, other MuranoPL code will be unable to make use
of these methods.

The new methods should include ``setDefault``, ``setContract``, ``addMeta``,
``wrapMethod``.

Also the ability to set property values for other object should be provided to
aspects by setting ``CTX_ALLOW_PROPERTY_WRITES`` flag to ``true`` in the
``Aspect`` packages context.

Changes to engine workflow
~~~~~~~~~~~~~~~~~~~~~~~~~~

With all that, the engine workflow with policies is the following:

#. User adds package to the environment

#. The class schema generation in engine triggers package load event

#. Engine obtains a list of policies that need to be applied for particular
   tenant (aspects and their input parameters)

#. Engine instantiates and initializes all those aspects and a notifier

#. Engine subscribes aspects to notifications from notifier

#. Engine loads the package and invokes ``Notifier``'s method
   ``onEvent(packageLoad)``.

#. ``Notifier`` finds out what aspects want to modify the package and invokes
   their ``handlePackageLoad()`` method.

#. Handler gets a chance to examine the code of the MuranoPL class and make
   necessary modifications using extended reflection capabilities.

# During the model load, objects initializing, garbage collection engine
  instantiates aspects and notifier again and runs the process of
  subscription => notification => modification with these events in a similar
  fashion.

Alternatives
------------

None

Data model impact
-----------------

Model to store policies should look roughly like this:

Table name: policy

+------------------+--------------+------+-----+---------+
| Field            | Type         | Null | Key | Default |
+==================+==============+======+=====+=========+
| created          | datetime     | NO   |     | NULL    |
+------------------+--------------+------+-----+---------+
| updated          | datetime     | NO   |     | NULL    |
+------------------+--------------+------+-----+---------+
| id               | varchar(255) | NO   | PRI | NULL    |
+------------------+--------------+------+-----+---------+
| object_model     | longtext     | YES  |     | NULL    |
+------------------+--------------+------+-----+---------+
| description      | text         | YES  |     | NULL    |
+------------------+--------------+------+-----+---------+

object_model field stores json-serialized representation of the policy
including its rules.

description field contains optional textual information.

Model to store policies assignment to tenants:

Table name: policy_assignment

+------------------+--------------+------+-----+---------+
| Field            | Type         | Null | Key | Default |
+==================+==============+======+=====+=========+
| created          | datetime     | NO   |     | NULL    |
+------------------+--------------+------+-----+---------+
| updated          | datetime     | NO   |     | NULL    |
+------------------+--------------+------+-----+---------+
| policy_id        | varchar(255) | NO   | PRI | NULL    |
+------------------+--------------+------+-----+---------+
| tenant_id        | varchar(36)  | YES  | PRI | NULL    |
+------------------+--------------+------+-----+---------+

NULL value of the tenant_id field means that policy should be applied to all
tenants in the cloud.

REST API impact
---------------

+---------------------+------------+-----------------------------------------+
| Attribute           | Type       | Description                             |
+=====================+============+=========================================+
| objectModel         | object     | JSON representation of policy           |
+---------------------+------------+-----------------------------------------+
| tenantIds           | array      | Array of the tenants to apply policy to |
+---------------------+------------+-----------------------------------------+

**List policies**

*Request*

+----------+------------------------------+----------------------------------+
| Method   | URI                          | Description                      |
+==========+==============================+==================================+
| GET      | /policies                    | List available policies          |
+----------+------------------------------+----------------------------------+

*Response*

List of policies with their basic properties.

::

    {
        "policies": [
            {
                "created": "2014-05-14T13:02:46",
                "updated": "2014-05-14T13:02:54",
                "id": "2fa5ab704749444bbeafe7991b412c33",
                "description": "Policy to limit possible flavor",
                "tenant_ids": ["726ed856965f43cc8e565bc991fa76c3"]
            },
            {
                "created": "2014-05-14T13:02:51",
                "updated": "2014-05-14T13:02:55",
                "id": "744e44812da84e858946f5d817de4f72",
                "description": "Policy to restrict access to apps",
                "tenant_ids": ["726ed856965f43cc8e565bc991fa76c3"]
            }
        ]
    }

Response codes:

+----------------+-----------------------------------------------------------+
| Code           | Description                                               |
+================+===========================================================+
| 200            | OK. List of policies received successfully                |
+----------------+-----------------------------------------------------------+
| 403            | User is not allowed to browse policies                    |
+----------------+-----------------------------------------------------------+

**Create policy**

*Request*

+----------+------------------------------+----------------------------------+
| Method   | URI                          | Description                      |
+==========+==============================+==================================+
| POST     | /policies                    | Create policy                    |
+----------+------------------------------+----------------------------------+

Body:

::

  {
       "description": "Policy to limit possible flavor",
       "objectModel": {
            "maxFlavor": "m1.medium",
            "?": {
                "type": "com.example.MyAspect",
                "id": "446373ef-03b5-4925-b095-6c56568fa518"
            }
       }
  }

*Response*

::

    {
        "id": "ce373a477f211e187a55404a662f968",
        "description": "Policy to limit possible flavor",
        "created": "2013-11-30T03:23:42Z",
        "updated": "2013-11-30T03:23:44Z",
        "objectModel": {
            "maxFlavor": "m1.medium",
            "?": {
                "type": "com.example.MyAspect",
                "id": "446373ef-03b5-4925-b095-6c56568fa518"
            }
        }
    }

Response codes:

+----------------+-----------------------------------------------------------+
| Code           | Description                                               |
+================+===========================================================+
| 200            | OK. Policy has been created successfully                  |
+----------------+-----------------------------------------------------------+
| 400            | Bad request. Either the format of the body is invalid or  |
|                | parameters doesn't match the contracts                    |
+----------------+-----------------------------------------------------------+
| 403            | User is not allowed to create policies                    |
+----------------+-----------------------------------------------------------+
| 404            | Not found. Specified class doesn't exist or it is not     |
|                | an Aspect                                                 |
+----------------+-----------------------------------------------------------+

**Update policy**

*Request*

+----------+------------------------------+----------------------------------+
| Method   | URI                          | Description                      |
+==========+==============================+==================================+
| PUT      | /policies/<policy_id>        | Update policy                    |
+----------+------------------------------+----------------------------------+

Body:

::

  {
      "description": "Changed description",
      "tenantIds": ["726ed856965f43cc8e565bc991fa76d8"],
      "objectModel": {
            "maxFlavor": "m1.large",
            "?": {
                "type": "com.example.MyAspect",
                "id": "446373ef-03b5-4925-b095-6c56568fa518"
            }
      }
  }

*Response*

::

    {
        "id": "ce373a477f211e187a55404a662f968",
        "description": "Changed description",
        "created": "2013-11-30T03:23:42Z",
        "updated": "2013-11-30T03:39:08Z",
        "tenant_ids": ["726ed856965f43cc8e565bc991fa76d8"],
        "objectModel": {
            "maxFlavor": "m1.large",
            "?": {
                "type": "com.example.MyAspect",
                "id": "446373ef-03b5-4925-b095-6c56568fa518"
            }
        }
    }

Response codes:

+----------------+-----------------------------------------------------------+
| Code           | Description                                               |
+================+===========================================================+
| 200            | OK. Policy has been updated successfully                  |
+----------------+-----------------------------------------------------------+
| 400            | Bad request. Either the format of the body is invalid or  |
|                | parameters doesn't match the contracts                    |
+----------------+-----------------------------------------------------------+
| 403            | User is not allowed to update this policy                 |
+----------------+-----------------------------------------------------------+
| 404            | Not found. Specified policy doesn't exist                 |
+----------------+-----------------------------------------------------------+
| 409            | Policy with specified name already exists                 |
+----------------+-----------------------------------------------------------+

**Get policy details**

*Request*

+----------+------------------------------+----------------------------------+
| Method   | URI                          | Description                      |
+==========+==============================+==================================+
| GET      | /policies/<policy_id>        | Get policy details               |
+----------+------------------------------+----------------------------------+

*Response*

::

    {
        "id": "ce373a477f211e187a55404a662f968",
        "description": "Policy description",
        "created": "2013-11-30T03:23:42Z",
        "updated": "2013-11-30T03:39:08Z",
        "tenant_ids": ["726ed856965f43cc8e565bc991fa76d8"],
        "objectModel": {
            "maxFlavor": "m1.medium",
            "?": {
                "type": "com.example.MyAspect",
                "id": "446373ef-03b5-4925-b095-6c56568fa518"
            }
        }
    }

Response codes:

+----------------+-----------------------------------------------------------+
| Code           | Description                                               |
+================+===========================================================+
| 200            | OK. Policy details have been received successfully        |
+----------------+-----------------------------------------------------------+
| 403            | User is not allowed to look up policies                   |
+----------------+-----------------------------------------------------------+
| 404            | Not found. Specified policy doesn't exist                 |
+----------------+-----------------------------------------------------------+

**Delete policy**

*Request*

+----------+------------------------------+----------------------------------+
| Method   | URI                          | Description                      |
+==========+==============================+==================================+
| DELETE   | /policies/<policy_id>        | Remove policy                    |
+----------+------------------------------+----------------------------------+

*Response*

::

    {
        "id": "ce373a477f211e187a55404a662f968",
        "description": "Policy description",
        "created": "2013-11-30T03:23:42Z",
        "updated": "2013-11-30T03:39:08Z",
        "tenant_ids": ["726ed856965f43cc8e565bc991fa76d8"],
        "objectModel": {
            "maxFlavor": "m1.medium",
            "?": {
                "type": "com.example.MyAspect",
                "id": "446373ef-03b5-4925-b095-6c56568fa518"
            }
        }
    }

Response codes:

+----------------+-----------------------------------------------------------+
| Code           | Description                                               |
+================+===========================================================+
| 200            | OK. Policy has been removed successfully                  |
+----------------+-----------------------------------------------------------+
| 403            | User is not allowed to remove this policy                 |
+----------------+-----------------------------------------------------------+
| 404            | Not found. Specified policy doesn't exist                 |
+----------------+-----------------------------------------------------------+

Versioning impact
-----------------

None

Other end user impact
---------------------

python-muranoclient needs to be extended with support of new REST API calls.

Deployer impact
---------------

Deployers will need to check the logs and UI messages informing that some
parameters of deployment were changed by the policies, or for the reasons why
certain applications can not be deployed etc.

Developer impact
----------------

Business logic of the murano applications should not be affected by the
change. Policies will have the ability to modify its code on the fly during the
execution.

Murano-dashboard / Horizon impact
---------------------------------

The following changes need to be made in the GUI:

* Form to instantiate and initialize aspect with the rules (create policy)

* Form to map policies to tenants

* Page to display the list of policies

* Page to display each policy details

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  vakovalchuk

Work Items
----------

* Implement join points in engine where aspects can subscribe for events and
  apply changes (object init and packages load during the first phase
  of implementation, model load, garbage collection later).

* Create MuranoPL class ``Aspect`` that defines the basic interface for custom
  aspects and place it to the core library.

* Create MuranoPL class ``Notifier`` that manages notifying process and place
  it to the core library.

* Implement extended MuranoPL reflection capabilities and make it available
  only to aspects.

* Create database model to store policies and model to store their assignment
  to tenants.

* Implement API for CRUD operations for the aspect instances and their
  assignment to tenants.

* Create python-muranoclient methods to support new API.

* Implement UI workflow to create policies and apply it to tenants.

* Create demo packages with example policies.

Dependencies
============

* https://blueprints.launchpad.net/murano/+spec/dependency-driven-resource-deallocation

* https://blueprints.launchpad.net/murano/+spec/schema-driven-ui

Testing
=======

* DSL unit tests for extended MuranoPL reflection

* Testrunner-based tests for the example policies functionality

* Tempest tests for the new REST API calls

* Unit and functional tests for the new python-muranoclient methods

* Selenium tests for the new GUI workflow

Documentation Impact
====================

* New functionality must be properly documented

* REST API specification need to be updated with new calls info


References
==========

* https://blueprints.launchpad.net/murano/+spec/dependency-driven-resource-deallocation

* https://blueprints.launchpad.net/murano/+spec/schema-driven-ui
