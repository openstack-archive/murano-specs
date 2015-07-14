..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

========================
Logging API for MuranoPL
========================

https://blueprints.launchpad.net/murano/+spec/logging-api-for-muranopl

The purpose is to add an ability to log actions while developing MuranoPL
applications

Problem description
===================

It is good practice to log basic stages during application execution.
Logging API should make debugging and troubleshooting processes easier.

Proposed change
===============

Implemention key points

1) New MuranoPL class `io.murano.system.Logger`

    Class `Logger` will be part of the MuranoPL core library.

    This class should contain basic logging functionality.
    Usage example in MuranoPL::

        $.log: logger('logger_name')
        $.log.info('message: {0}', 'checkpoint')

    that code should be equivalent of python code::

        from oslo_log import log as logging
        LOG = logging.getLogger('logger_name')
        LOG.info(_LI('message: {0}').format('checkpoint'))

    Others logging methods are `Logger.debug`, `Logger.info`, `Logger.warning`,
    `Logger.error`, `Logger.critical`. Each method corresponds to the oslo.log
    logging level.

    There is also separate method for exception stack trace output described
    below.

2) Exceptions logging

    Method `Logger.exception` intended to log a stack trace::

         $.log.exception(exc, 'Something bad happened: {0}', 'Oops!')

   `exception` method uses the same log level as `Logger.error`.

3) New global MuranoPL function `logger()`

    Call of the function `logger('logger_name')` returns new instance of the
    `io.murano.system.Logger`. If function with the same `logger_name` was
    called previously then the same logger instance should be returned instead of
    building new one.

4) Logging configuration

    Logging should use standard Murano Engine logging subsystem.

    Application itself cannot set logging settings at runtime.

    All of appenders, formatters an etc. may be configured via standard way
    as others loggers in Murano Engine.

    Prefix `applications` should be added to each logger name which created by
    application. As example, logger named `active_directory` in the MuranoPL
    should be identified as `applications.active_directory` at the python side
    and in the system config. This is for separation loggers used by
    applications and engine system loggers. Also it will allow us to specify
    settings for the application loggers separately from others loggers.

5) Default configuration

    All logs written in the one file by default. Log rotation should be used,
    so maximum size of logs is limited.

6) Logging naming conventions

    A note about the logging naming conventions should be added to the MuranoPL
    tutorial.

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

Deployer will get more information about application execution stages and additional
tool for more precise troubleshooting

Developer impact
----------------

Logging API will allow developer to debug application in a more effective manner
getting information from logs.

Murano-dashboard / Horizon impact
---------------------------------

None

Implementation
==============

Assignee(s)
-----------

Alexey Khivin

Primary assignee:
  <akhivin@mirantis.com>

Work Items
----------

* Create new MuranoPL class `io.murano.system.Logger`
* Create new global MuranoPL function `logger()`
* Create method `Logger.exception`
* Add new section for logging parameters into the Murano Engine config
* Describe naming conventions for loggers in the Murano docs

Dependencies
============

None


Testing
=======

Functional tests for MuranoPL must be updated.

Documentation Impact
====================

MuranoPL


References
==========

None
