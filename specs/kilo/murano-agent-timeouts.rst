..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==================================
Add timeouts to murano-agent calls
==================================

https://blueprints.launchpad.net/murano/+spec/murano-agent-timeouts

Now there is no way to be sure that the agent successfully started execution
on a VM. Also there is no control of the execution time of scripts on agent.
This process should be more controllable. It can be done by adding timeouts
in Murano engine.


Problem description
===================

* During the agent`s work there could be some problems with execution of scripts and
  VM may hang. In this case user will wait indefinitely without knowing
  what's happened.

* Currently there is no feedback from agent so, it`s impossible to determine
  whether agent is ready to  accept execution plans or not.

It is proposed to provide mechanism of timeouts which solves these problems.

Proposed change
===============

1) First of all add timeout to method *call* of agent on engine-side.

Optional parameter *timeout* will be added to method *call* of class *Agent*
in agent.py. This parameter is the time in seconds with default value *600*.
Developer can set up custom value during developing apps for example in this
way::

- $.instance.agent.call($template, $resources, 300)

If the agent plan execution time exceeds the limit, it will be terminated.

2) Add method *waitReady* in Agent class.

Method will be added to *Agent* class in agent.py. It has optional parameter
*timeout* with default value *100*. *waitReady* creates test plan with trivial
body::

  template = {'Body': 'return', 'FormatVersion': '2.0.0', 'Scripts': {}}

and sends this plan once to agent by method *call*. It can be used by developer
to stop deployment before sending template of app if agent is inaccessible as
follows::

- $.instance.agent.waitReady()
- $.instance.agent.call($template, $resources)

If the agent test plan execution time exceeds the time limit,
*TimeoutException* will be raised and deployment terminates. *TimeoutException*
will be created in murano/common/exceptions.py.

3) Add new method *isReady* in Agent class.

This method will simply call the *waitReady*. Method *isReady* returns:

  * *True*, if test plan is executed on time;

  * *False*, if the agent plan execution time exceeds the
    limit.

  * and raise *PolicyViolationException*, which will be created in
    murano/common/exceptions.py, if the agent disabled by the server.

The method can be used during development Murano-applications. For example,
developer can check if the agent is running and ready to execute templates
before sending the execution plan of application:

::

    - If: $.instance.agent.isReady()
      Then:
        - $._environment.reporter.report($this, 'Murano Agent is ready')

The message in above example will be reported to Murano Dashboard.

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

None

Murano-dashboard / Horizon impact
---------------------------------

None


Implementation
==============

Assignee(s)
-----------

Primary assignee:lk
  ddovbii

Work Items
----------

The change is simple and can be done in one Gerrit patch. Implementation is
acutally completed.

Dependencies
============

None

Testing
=======

Unit and integration tests must be done.


Documentation Impact
====================

MuranoPL specification should be updated.


References
==========

None

