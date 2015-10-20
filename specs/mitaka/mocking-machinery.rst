..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

================================================
Mocking machinery for MuranoPL testing framework
================================================

https://blueprints.launchpad.net/murano/+spec/mocking-machinery

Currently we have separate executor, that allows to run MuranoPL code in
testing mode without dashboard. But the main purpose of this executor is to
provide 'mock' availability. This specification describes how this can be done.

Problem description
===================

It's not possible to provide unit-testing without mocking machinery.
The whole idea of murano simulation mode was in deployment imitation.

During the deployment of murano applications, a lot of external objects,
that are not connected to the app itself are involved, such as *murano agent*,
*networks*, *heat stacks* and etc.

All of the classes that should be mocked can be divided into:

* python classes
* yaml classes
* dependent applications

Mocks can really help with new application development (especially compound
ones).

Mock implementation will enable:

* Real deployment simulation

* Run several deployments with different parameters

Actually, list of use-cases is unlimited, since user will be allowed to provide
any alternative function implementation to speed up the development.

Proposed change
===============

Test-runner executor should have new context manager, that will take into
consideration specified mocks. Mocks will be specified with new global YAQL
function.
This will be first step to enable mocking. Later improvements will be added and
described in a separate specification.

Alternatives
------------

None

Data model impact
-----------------

None

REST API impact
---------------

None

Versioning impact
-----------------

None. Will be not available in the previous murano versions.

Other end user impact
---------------------

None

Deployer impact
---------------

None

Developer impact
----------------

None, only new opportunity for developers

Murano-dashboard / Horizon impact
---------------------------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  <efedorova@mirantis.com>

Work Items
----------

#. Create MockContextManager.

   Context manager is responsible for providing a valid context with
   corresponding attributes and functions for the current object.
   Modified context manager will be used for mock implementation. It will be
   inherited from the original one and will be called ``MockContextManager``.
   It will store instructions which objects or classes are needed to be
   replaced with mocks and return context with mock definition instead of the
   original function. For that purpose several dictionaries should be used:
   one for mapping objects with mocks and other one for class mapping.

   If mock for the current object exist, new linked context
   (``murano.dsl.linked_context.LinkedContext``) will be returned. It links
   existing context with the new context, where mock definition is presented.
   So if mocked function will be called, context will contain two
   definitions with that function: mock and original one, but mock will have
   higher priority.

   If there is no mock for the current object or class, existing context
   will be returned. If there is no existing context, *None* will be returned.

#. Add global YAQL functions (it will be available only in test-runner mode) to
   set scope of mocking and mock definition. It will be called ``inject`` and
   will have several declarations for different purposes:

   * ``inject`` for set up mock for *class* or *object*, where mock definition
     is a *name of the test class method*

     * `def inject(target, target_method, mock_object, mock_name)`

   * ``inject`` for set up mock for *a class* or *object*, where mock
      definition is a *YAQL expression*

      * `def inject(target, target_method, yaql_expr)`


   Description:

     * *target*: MuranoPL class name (namespaces can be used or full class name
       in quotes) or MuranoPL object

     * *target_method*: Method name to mock in target

     * *mock_object*: Object, where mock definition is contained

     * *mock_name* Name of method, where mock definition is contained

     * *yaql_expr*: YAQL expression, parameters are allowed

   So user is allowed to specify concrete method to use instead of original,
   or to provide YAQL expression from which new function will be composed.

   Advantages of defining mock with YAQL expression:

   * Simplicity

     Thus, if you need your methods to return different constants, you can
     return it inline instead of creating different methods for each constant.

   * Restricted context

     By default Local variables are not seen in the mock function scope, but
     it's possible to specify which variables to pass to the expression.


#. Add `withOriginal` YAQL function

   * `withOriginal(a => $x, b => $y)`

     YAQL function, registered in the mock context.
     Allows to pass values from the original context to the mock context,
     where mock function is executed. Suppose, we have ``$x: 2`` in the mock
     function and ``$x: 1`` in the original function. We can not just combine
     the contexts since we want to use both 'x' variables and it will be
     unclear which. With new YAQL function original variables can be passed to
     a mock context with configurable name. So original context need to be
     saved in advanced and ``withOriginal`` function should have access to it.

   * `originalMethod()`

     YAQL function, registered in the context of inject function.
     Calls the original method and can be used in a mock function.


#. Add `OneOf` Smart type

    In new function for injection mock parameter can be class or object.
    So we need to accept one of those. This new type will check function
    parameter for belonging to one of type in the provided list.

Dependencies
============

None


Testing
=======

New code should be 100% covered by unit tests.


Documentation Impact
====================

Separate documentation for the whole test-runner and mock machinery will
be provided.


References
==========

None