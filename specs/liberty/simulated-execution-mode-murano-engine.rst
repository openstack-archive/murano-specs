..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Simulated Execution Mode For Murano Engine
==========================================

https://blueprints.launchpad.net/murano/+spec/simulated-execution-mode-murano-engine

Problem description
===================

As an Application Developer I'd like to execute my workflows without actual
deployment and interaction with murano-dashboard in order to verify my workflow
before actual deployment by those increasing my application development
speed.

Proposed change
===============

Verifying application packages should be simple and fast.
User doesn't have to re-upload package and add app to the environment on every
change.

* Allow application author to validate his application using unit-tests

* Those tests will be put to the application package to allow anyone to test
  this app at any time

* Tests will look like regular unit tests. Testing framework, witch will
  run unit-tests will support commands, that will allow to load test package
  from directory, to call class methods and configure deployments parameters.
  That will make deployment test run easier. Also, test writer may run
  deployment several times, examine and compare results with different
  parameters

* Tests should be able to produce complete object model with some parameters
  of deployment:

  * environment attributes (such as tokens) (or overwriting values, defined in
    config);

  * mock the methods of system classes which include various kinds of external
    communications. Dependent applications, system resources and various API
    clients and also be mocked. It should be allowed to specify a returned
    value. There would be separate specification for mocking, where the
    details will be described.

Test-case prototype may look like that:

::

 Namespaces:
    =: io.murano.apps.foo.tests
    sys: io.murano.system
    pckg: io.murano.apps.foo

 Extends: io.murano.tests.TestFixture

 Name: FooTest

 Methods:
    initialize:
        Body:
            # - $.appJson: new(sys:Resources).json('foo-test-object-model.json')
            - $.appJson:
                - ?:
                    id: 123
                    type: io.murano.apps.foo.FooApp
                  name: my-foo-obj
                  instance:
                      ?:
                          type: io.murano.resources.Instance
                          id: 42
                      ...

    setUp:
        Body:
            - $.env: $.createEnvironment($.appJson)   # creates an instance of std:Environment
            - $.myApp: $.env.applications.where($.name='my-foo-obj').first()
            - mock($.myApp.instance, "deploy", "mockInstanceDeploy", $this)
            - mock(res:Instance, deploy, "mockInstanceDeploy", $this)


    testFooApp:
        Body:
            - $.env.deploy()
            - $.assertEqual(true, $.myApp.getAttr('deployed'))

    tearDown:
        Body:

    mockInstanceDeploy:
        Arguments:
            - mockContext
        Body:
            - Return:
               # heat template



Alternatives
------------

Provide one CLI  command, that will mock creation of VMs and other things and
returns the deployment result.

Cons:
Impossible to verify deployments, where execution plan returns a value, which
is used in future app workflow. Compare results of several deployments would be
inconvenient Real VM’s can’t be

Data model impact
-----------------

None

REST API impact
---------------

None

Versioning impact
-----------------

Tests will be placed to a package, so manifest version need to be updated.
This functionality should be described in a separate spec.
For now, there will be no impact on project itself.

Other end user impact
---------------------

None

Deployer impact
---------------

None

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
  <efedorova@mirantis.com>

Work Items
----------

#. Add ‘simulation’ mode (new entry-point) to Murano Engine, where
packages would be uploaded from the provided path there would be no
interconnection with RabbitMQ

#. Make changes to the class-loader, located in engine, to not use API.
   Separate spec is provided for this change (https://review.openstack.org/#/c/198745/).

#. Implement testing framework, written in MuranoPL that will include the
   classes, described below. The structure would be taken from python unittest
   module. Framework will include test-runner

#. Implement mock support.

#. Define what is needed to change in MuranoPL itself


Testing framework may contain the following classes and methods.
This are base classes for simple testing framework.

* ``TestCase`` class


+------------------+-------------------------------------------------------------+
| Method           | Description                                                 |
+==================+=============================================================+
| setUp()          | Method called immediately before calling the test method.   |
+------------------+-------------------------------------------------------------+
| tearDown()       | Method called immediately after the test method has been    |
|                  | called and the result recorded.                             |
+------------------+-------------------------------------------------------------+
| run(result=None) | Run the test, collecting the result into the test result    |
|                  | object passed as result.                                    |
+------------------+-------------------------------------------------------------+
| assert...        | Different asserts (assertEqual, assertNotEqual, assertTrue, |
|                  | assertFalse).                                               |
+------------------+-------------------------------------------------------------+

* ``TestResult`` class: This class is used to compile information about which tests have succeeded and which have failed.

+--------------+-------------------------------------------------------------+
| Attribute    | Description                                                 |
+==============+=============================================================+
| errors       | A list containing 2-tuples of TestCase instances and strings|
|              | holding formatted tracebacks. Each tuple represents a test  |
|              | which raised an unexpected exception.                       |
+--------------+-------------------------------------------------------------+
| failures     | A list containing 2-tuples of TestCase instances and strings|
|              | holding formatted tracebacks. Each tuple represents a test  |
|              | where a failure was explicitly signalled using the          |
|              | TestCase.assert*() methods.                                 |
+--------------+-------------------------------------------------------------+
| testsRun     | The total number of tests run so far.                       |
+--------------+-------------------------------------------------------------+

* ``TestRunner(stream=sys.stderr, descriptions=True, verbosity=1)`` A basic test runner
  implementation which prints results on standard error.
  Has *run* method, witch executes the given test case. Also stores the execution result.


For the fist time test may be run only one by one. Later we can add ``TestSuite`` class and
``TestLoader`` class:
* ``TestLoader`` class is responsible for loading tests according to various criteria
and returning them wrapped in a TestSuite (or TestSuite if will add this class).

+--------------------------------------+----------------------------------------------------+
| Methods                              | Description                                        |
+======================================+====================================================+
| loadTestsFromTestCase(testCaseClass) | Return a suite of all tests cases contained in the |
|                                      | TestCase-derived testCaseClass.                    |
+--------------------------------------+----------------------------------------------------+

#. Implement simple mocking machinery

All mockes are separated into NonCallable and Callable mocks

``Mock`` class

Public methods

+-----------------------------+-----------------------------------------------------------------+
| Methods                     | Description                                                     |
+=============================+=================================================================+
| start()                     | Activate a patch, returning any created mock.                   |
+-----------------------------+-----------------------------------------------------------------+
| stop()                      | Stop an active patch.                                           |
+-----------------------------+-----------------------------------------------------------------+
| patch(target)               | The `target` is patched with a `new` object. `target` should be |
|                             | a string in the form `package.module.ClassName`.                |
+-----------------------------+-----------------------------------------------------------------+
| attach_mock(mock, attribute)| Attach a mock as an attribute of this one, replacing its name   |
|                             | and parent                                                      |
+-----------------------------+-----------------------------------------------------------------+
| configure_mock(kwargs)      | Set attributes on the mock through keyword arguments            |
+-----------------------------+-----------------------------------------------------------------+

Private methods:

initialize, __call__, _patch, __enter__, __exit__

Dependencies
============

None

Testing
=======

None

Documentation Impact
====================

New testing framework will be documented from scratch.

References
==========

Discussions in IRC will be provided