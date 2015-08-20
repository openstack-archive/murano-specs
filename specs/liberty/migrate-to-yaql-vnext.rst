..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=====================
Migration to yaql 1.0
=====================

https://blueprints.launchpad.net/murano/+spec/migrate-to-yaql-vnext

yaql 1.0 has a long list of improvements over yaql 0.2 used by Murano.
But it is not a drop-in replacement for yaql 0.2 and requires to perform
serious changes to Murano in order to switch to it.

-------------------
Problem description
-------------------

While yaql 1.0 is more or less similar in terms of query language to that in
v0.2 it is very different in how its functionality exposed to a hosting
project. Most notable differences are:

* syntax for Python function declaration for yaql is different from that in
  yaql 0.2.
* yaql 0.2 had so called #resolve() function. When it encountered function
  call expression (say ``foo()``) it tried to call ``#resolve()`` so that it
  will do the function resolution. If #resolve was not present in context of
  failed to resolve function standard yaql 0.2 resolution procedure was used.
  Murano used #resolve to catch attempts to call ``$obj.something()`` where obj
  is a MuranoObject because it is impossible to register all possible methods
  of all possible objects in advance. yaql 1.0 doesn't have neither
  ``#resolve()`` nor anything similar to it.
* yaql 1.0 doesn't have tuples syntax any more. In v0.2 ``foo(a => b)`` meant
  ``foo(tuple(a, b))``. In v1.0 ``=>`` syntax means keyword argument value.
  Murano used to convert tuples to keyword arguments when calling MuranoPL
  methods.
* in yaql 0.2 parser was a global object so that you could say yaql.parse().
  In v1.0 parser itself is customizable and need to be constructed by the
  hosting project so you need a parser instance (called YaqlEngine in v1.0)
  to parse expressions.
* yaql 1.0 doesn't have ``#validate()`` function used to implement ``:`` in
  ns:TypeName syntax. yaql 1.0 also doesn't have ':' operator but has a way
  to define custom operators that can be used instead.
* there is a great overlap between MuranoPL and yaql 1.0. yaql have a rich
  and extensible framework to decorate function parameters to specify their
  type and requirements, to inject hidden parameters and to control parameter
  laziness. Not only it used for validation but also for function overload
  resolution. In MuranoPL there are argument contracts serving similar purpose.
  However their implementation is completely independent from each other as for
  the whole function invocation process. As a result for MuranoPL methods that
  are writen in Python (system classes) it is impossible neither contracts nor
  yaql specs, have additional injected parameters and so on.


---------------
Proposed change
---------------

In Murano Engine:
=================

* Create and store in a global variable YaqlEngine instance that will be used
  through the application to parse expressions. Update all the places where
  parsing takes place to use that object
* extend list of yaql 1.0 parameters specifications (smart-types) with
  additional that will validate parameter value using contract expression.
* generate yaql 1.0 FunctionDefinitions from MuranoPL yaql methods using
  introduced smart-type. As a result all MuranoPL methods will be directly
  exposed as yaql functions. Actual function payload will still point to
  MuranoDslExecutor code to perform correct context switching, locking and
  logging
* do similar thing for native MuranoPL methods (those that are written in
  Python). Python methods could use any yaql parameter declarations and full
  potential of yaql 1.0 specs. So now yaql functions, Python methods and YAML
  methods will all be unified as yaql functions/methods with full
  interoperability. For native methods there will be 2 FunctionDefinition:
  one is the interface stub that gets registered in yaql context and calls
  executor.invoke_method() under the hood. And another one for a native method
  itself to call methods via yaql mechanics (and apply all the smart-type
  specs) rather that via custom reflection-based implementation that is found
  in dsl code. For YAML methods method.body will still remain the dictionary
  as it is now
* implement ``#operator_.`` (``.`` operator) in such a way that for expressions
  of a form ``$obj.foo()`` where ``$obj`` is a MuranoObject it will discover
  all possible methods of ``$obj``'s type (via added MuranoClass functionality)
  and continue with ``foo(sender=$obj)`` execution in a child context with all
  ``$obj``'s method registered in it. This is a way to dynamically register
  functions in context overcoming absence of yaql 0.2's ``#resolve()``
* configure yaql engine to have additional operator ``:`` with a highest
  priority. Expressions of a form ns:Type should be resolved in place using
  current Namespaces specification and resulting type name will be used to
  obtain MuranoClass from class loader. As a result of operation return
  special object (MuranoClassReference) that will wrap MuranoClass and could
  be used to reference type in YAML MuranoPL code without providing access
  to MuranoClass internals. That can be used later to implement static methods
  with ``ns:Type.foo()`` syntax
* add one more yaql smart-type so that it will be possible to declare that
  function requires type reference. This smart type should accept both strings
  and MuranoClassReference objects. new() and class() functions should be
  implemented with this smart-type. This will allow transparency between
  string-based syntax new('io.murano.MyClass') and new(ns:MyClass) since
  ns:MyClass will not be a string anymore
* change dsl's and engine's yaql functions to have new declaration syntax.
  Remove functions that are already present in yaql 1.0 standard library.
* use yaql 1.0 legacy mode for backward compatibility with existing
  MuranoPL applications


In python-muranoclient:
=======================

* create global YaqlEngine and update code to use it (and new yaql module
  structure)
* update YaqlExpression code to use that global parser object and new yaql's
  module structure


In Murano Dashboard:
====================

* make dashboard either use parser object from updated python-muranoclient or
  create one for itself
* update yaql functions that were written for dynamicUI forms to have new
  declaration syntax
* because dashboard uses expression serialization (pickling) and yaql 1.0
  expressions are not serializable because of a outer AST expression contains
  reference to YaqlEngine used to create it and YaqlEngine cannot be made
  serializable there is a need for custom pickling code that will make pickle
  not to try to serialize YaqlEngine instance
* use yaql 1.0 legacy mode for backward compatibility with existing
  MuranoPL applications. In the future we can switch to non-legacy mode on
  newer form format versions.


------------
Alternatives
------------

Try to fix some of the bugs in yaql 0.2 and not to switch to v1.0

Data model impact
=================

None

REST API impact
===============

None

Versioning impact
=================

yaql migration should not break existing applications so there is no versioning
impact here. However it may have an impact on how plugins/system classes for
MuranoPL are written. But since versioning is not implemented yet there is
nothing to increment and backward compatibility will be broken for those
classes.

Also this yaql legacy mode may be switched off for next engine version and
still be turned on for old applications

Other end user impact
=====================

Because new yaql is not 100% compatible with yaql 0.2 there is still a
possibility that some applications will break even with compatibility mode
turned on. For example there might be bugs in applications that remained
unnoticed with yaql 0.2 but cause exception to be raised with v1.0 or
application might have relied on yaql 0.2 buggy behavior that is not present
anymore.

Deployer impact
===============

None

Developer impact
================

Plugins will have to be updated to use yaql-style method declarations. But
because yaql can infer many things automatically changes are going to be
minor and will not require much efforts to do. However without any
modifications old plugins will likely to break after migration will be made.

Murano-dashboard / Horizon impact
=================================

dashboard code need to be updated to use new yaql. This will not affect user
experience

--------------
Implementation
--------------

Assignee(s)
===========

Primary assignee:
  Stan Lagun <slagun>

Other contributors:
  Ekaterina Chernova <efedorova>

Work Items
==========

Bullets from "proposed changes" section may be used as a work items directly

------------
Dependencies
------------

This spec depends on yaql 1.0 be released and the latest changes in it.

-------
Testing
-------

Because applications can break they should be re-tested. When incompatibility
found it is better to try to fix it in engine's code rather that in application
to capture similar cases in other applications.

--------------------
Documentation Impact
--------------------

Documentation for how to write plugins/system classes need to be updated.
Most of the newly introduced features to MuranoPL are direct consequence
of new yaql features and should be documented in yaql's scope rather than
copying it to MuranoPL documentation.

----------
References
----------

None