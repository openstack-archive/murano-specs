..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==============
Static actions
==============

https://blueprints.launchpad.net/murano/+spec/static-actions

Currently Murano provides a way to invoke MuranoPL methods through the
actions API. However it doesn't cover all the use cases where one may want
to expose some MuranoPL code. This specification further improves the idea
of actions.


Problem description
===================

Existing actions approach has a number of limitations:

* Only instance methods can be exposed as actions. In other words you need
  to have a class instance in order to call a method on it. And instance means
  it has to be a part of object model. This makes impossible to expose MuranoPL
  code that doesn't operate on object model or even produces it like factory
  methods;

* Actions bring large overhead and complexity for simple methods. For each
  action call a database record is created and never deleted then. All action
  results are persisted as well;

* No two actions in the same environment can be called simultaneously.

All of above is not a problem for regular application actions that usually
involve deployment steps and can take long time to complete. However it makes
actions a bad choice in case when one just wants to expose some application
code to outside to keep all of his code in MuranoPL in single place.


Proposed change
===============

In order to overcome above issues it is proposed to add ability to expose and
call static MuranoPL methods through new API and engine RPC calls. Because
the methods are static they are not bound to any object model (environment)
and thus can be called simultaneously.

Also because now there are two types of action methods (static and instance)
the fact that method needs to be exposed cannot be expressed with the Usage
attribute of the method (static methods already have `Usage: Static`).
We could introduce yet another usage (e.g. `StaticAction`). But because action
means that regular MuranoPL method need to be exposed to outside and it has
much more to do with a method visibility rather then its usage it is proposed
to introduce another method keyword to declare its scope.

Thus current syntax for the action declaration

::

  methodName:
    Usage: Action

will become

::

  methodName:
    Scope: Public

and static actions will be declared as

::

  methodName:
    Usage: Static
    Scope: Public

`Scope` attribute must be one of the two possible values:

* `Session` - regular method that is accessible from anywhere in the current
  execution session. This is the default if the attribute is omitted;

* `Public` - accessible anywhere, both within the session and from outside
  through the API call (i.e. it is an action in current terminology).

In the future new scopes might be added to further restrict the access
to the method (e.g. `Package`, `Class`)

For backward compatibility `Usage: Action` should remain valid and acceptable
but be deprecated for future format versions. An exception must be raised if
both `Usage: Action` and `Scope: Session` are specified.


Synchronous and asynchronous actions
------------------------------------

Current instance actions are always asynchronous and this is achieved by means
of API with action result being persisted into database. For static methods
that doesn't usually do any deployment and much more suitable for quick
request-response interactions that do not modify object model this would be
an overkill that greatly reduces their performance. However if we make
static actions synchronous this will cause confusion because we get two
completely different usage pattern for method that differ only by the fact
that one is static and another is not. This is especially confusing due to the
fact that one can easily imagine synchronous instance actions and asynchronous
static ones. Thus it is proposed to:

#. On MuranoPL side all actions are synchronous i.e. they are plain methods
   that return the result;

#. RPC calls for action execution are always asynchronous. I.e. the call to
   invoke the method and call to notify API on the method result are two
   different calls rather than single request-response. However the request
   may indicate that the response must be sent to specific API instance to
   make synchronization on the API without database persistence possible.

#. On python-muranoclient side all actions are asynchronous in terms that you
   get a Future object from which one can get result later as well as check for
   its readiness. For synchronous call the result becomes available immediately.
   However from the user's perspective all methods are asynchronous.
   Current python-muranoclient method for action execution returns action ID
   that can be later provided to another method rather that a Future object.
   For consistency it is proposed to change that method to return Future
   instead. For backward compatibility the method that accepted action ID
   must accept task Future too.

#. It is the API who decides if the method is going to be executed
   synchronously or not. In the former case API does RPC request to the engine
   without creating database records and instead waits for response that
   must be sent to that API instance. As soon as result is available it is
   returned to the caller (python-muranoclient). For now the API is going
   to do synchronous execution for static actions and asynchronous for
   instance actions. However in the future we may consider making it
   configurable or introduce some level of automation when API can wait for
   some time and then switch to asynchronous mode (create database record
   and so on) if there is still no result.


Alternatives
============

* Use actions instead for cases where static methods will be more appropriate;

* Declare static actions with yet another Usage value or make Usage be a list
  ([Static, Action])

* Use Meta to declare actions


Data model impact
=================

None

REST API impact
===============

**POST /actions**

Execute static MuranoPL method. Method must have a Public scope.

*Request*

+----------+----------------------------------+----------------------------------+
| Method   | URI                              | Description                      |
+==========+==================================+==================================+
| POST     | /actions                         | Execute static action            |
+----------+----------------------------------+----------------------------------+

Body:

::

  {
      "className": "my.class.fqn",
      "methodName": "myMethod",
      "packageName": "optional.package.fqn",
      "classVersion": "1.2.3",
      "parameters": {
          "arg1": "value1",
          "arg2": "value2"
      }
   }

packageName is optional. If provided the class must be in that package.

classVersion is optional. If provided it may contain either full SemVer version
or version spec (e.g. ">1.2"). If omitted "=0" is assumed.

*Response*


JSON-serialized method response or exception is returned.

HTTP codes:

+----------------+-----------------------------------------------------------+
| Code           | Description                                               |
+================+===========================================================+
| 200            | OK. Action was executed successfully                      |
+----------------+-----------------------------------------------------------+
| 400            | Bad request. Either the format of the body is invalid or  |
|                | parameters doesn't match the contracts                    |
+----------------+-----------------------------------------------------------+
| 403            | User is not allowed to execute the action                 |
+----------------+-----------------------------------------------------------+
| 404            | Not found. Specified class or method doesn't exist        |
|                | or not exposed                                            |
+----------------+-----------------------------------------------------------+
| 503            | Unhandled exception in the action                         |
+----------------+-----------------------------------------------------------+


Versioning impact
=================

None

Other end user impact
=====================

python-muranoclient must be updated to provide method for the new API call.

Deployer impact
===============

None

Developer impact
================

Existing MuranoPL applications shouldn't be affected by the change.
For the next package format version old syntax of action declaration
should still work but produce a deprecation warning.


Murano-dashboard / Horizon impact
=================================

None


Implementation
==============

Assignee(s)
-----------

Stan Lagun <istalker2>

Work Items
----------

* Update DSL to make use Scope method attribute

* Update code that inspects if method is an action for the new syntax
  (including object model serializer)

* Implement new RPC call in murano-engine

* Implement API call

* Implement method in python-muranoclient


Dependencies
============

None

Testing
=======

RPC call implementation can be tested by unit tests alone.
However to test the new API several tempest tests need to be added:

* Test that executes sample MuranoPL static action (hello world)

* Test that executes method that throws an exception

* Test that executes method with invalid parameters

Documentation Impact
====================

* Documentation for the actions need to be updated

* Developer's guide need to be extended with static actions info

* REST API specification need to be updated with new call info


References
==========

None


Versioning impact
-----------------

None

Other end user impact
---------------------

python-muranoclient must be updated to provide method for the new API call.

Deployer impact
---------------

None

Developer impact
----------------

Existing MuranoPL applications shouldn't be affected by the change.
For the next package format version old syntax of action declaration
should still work but produce a deprecation warning.


Murano-dashboard / Horizon impact
---------------------------------

None


Implementation
==============

Assignee(s)
-----------


Stan Lagun <istalker2>


Work Items
----------

* Update DSL to make use Scope method attribute

* Update code that inspects if method is an action for the new syntax
  (including object model serializer)

* Implement new RPC call in murano-engine

* Implement API call

* Implement method in python-muranoclient


Dependencies
============

None

Testing
=======

RPC call implementation can be tested by unit tests alone.
However to test the new API several tempest tests need to be added:

* Test that executes sample MuranoPL static action (hello world)

* Test that executes method that throws an exception

* Test that executes method with invalid parameters

Documentation Impact
====================

* Documentation for the actions need to be updated

* Developer's guide need to be extended with static actions info

* REST API specification need to be updated with new call info


References
==========

None