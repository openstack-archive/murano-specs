..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================
Murano unified logging
======================

https://blueprints.launchpad.net/murano/+spec/unified-style-logging

Rewrite murano logging in unified OpenStack style proposed by
https://blueprints.launchpad.net/nova/+spec/log-guidelines

Problem description
===================

Now log levels and messages in murano are mixed and don't match the OpenStack
logging guideliness.

Proposed change
===============

The good way to unify our log system would be to follow the major guidelines.
Here is a brief description of log levels:

* Debug: Shows everything and is likely not suitable for normal production
  operation due to the sheer size of logs generated (e.g. scripts executions,
  process execution, etc.).
* Info: Usually indicates successful service start/stop, versions and such
  non-error related data. This should include largely positive units of work
  that are accomplished (e.g. service setup, environment create, successful
  application deployment).
* Warning: Indicates that there might be a systemic issue;
  potential predictive failure notice (e.g. package execution problems,
  problems with categories listing).
* Error: An error has occurred and an administrator should research the event
  (e.g. deployment failed, app add failed).
* Critical: An error has occurred and the system might be unstable, anything
  that eliminates part of murano's intended functionality; immediately get
  administrator assistance (e.g. failed to access keystone/database, plugin
  load failed).

As far as murano-dashboard has it own notification system all notifications
should be duplicated at log messages and should follow this spec in the selection
of log level.

Here are examples of log levels depending on environment execution:

* Action execution:

.. code:: python

    LOG.debug('Action:Execute <ActionId: {0}>'.format(action_id))


..

* Environment creation: 

.. code:: python

    LOG.info(_LI('Environments:Create {id} succeed>'.format(id=environment.id)))


..

* Package execution problems

.. code:: python

    LOG.warning(_LW("Class is defined in multiple packages!"))


..

* Environment is not found

.. code:: python

    LOG.error(_LE('Environment {id} is not found').format(id=environment_id))


..

Additional step for our logging system should be usage of pep3101 as unified
format for all our logging messages. As soon as we try to make our code more
readable please use {<smthg>} instead of {0} in log messages.

Alternatives
------------

We need to follow OpenStack guidelines, but if needed we can move plugin logs
to DEBUG level instead of INFO. It should be discussed separately in each case.

Data model impact
-----------------

None

REST API impact
---------------

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

murano-image-elements impact
----------------------------

None

murano-dashboard / Horizon impact
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

* Unify existing logging system
* Unify logging messages
* Synchronize dasboard notifications and log entries
* Add additional logs if needed

Dependencies
============

None

Testing
=======

None

Documentation Impact
====================

None

References
==========

https://blueprints.launchpad.net/nova/+spec/log-guidelines
https://www.python.org/dev/peps/pep-3101/
