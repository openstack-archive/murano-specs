..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================
Simple Software Configuration
=============================

https://blueprints.launchpad.net/murano/+spec/simple-software-configuration

The purpose is to add to Murano core-library new functionality allowing to
simplify the process of software configuration.

Problem description
===================

At the moment a developer of murano applications often has difficulties with
configuration of software on the instance. They are related with fact that
developer always has to create execution plans even in situations when some
short scripts must be executed on VM.

For example, developer wants to run mysql service. In this case he should:

* Prepare template file for execution plan

* Prepare script containing only something like this:

..

  service mysql-server start

..

* Describe in code of class the sending of plan to the murano agent

It is proposed to update the murano core-library to allow to user make express
software configuration instructions without writing explicit configuration
scripts and execution plans for Murano Agent in order to develop applications
faster, and have concise and clear workflows.

Proposed change
===============

The extension of library will support only Linux instances.

The new class **io.murano.configuration.Linux** will be added. In initial
implementation this class has two methods:

::

  runCommand:
    Arguments:
      - agent:
          Contract: $.class(sys:Agent)
      - command:
          Contract: $.string().notNull()
      - helpText:
          Contract: $.string()

* **runCommand** method sends specified string containing CLI command to
  murano-agent and then it will be executed.

  Arguments:

   * *agent* - instance of murano-agent
   * *command* - string with CLI command
   * *helpTest* - string (optional), description for logging. Will be used as
     name of execution plan. If it is *Null* value of *command* will be used.

::

  putFile:
    Arguments:
      - agent:
          Contract: $.class(sys:Agent)
      - fileContent:
          Contract: $.string().notNull()
      - path:
          Contract: $.string().notNull()
      - helpText:
          Contract: $.string()


* **putFile** method takes content of file and writes it to specified path
  on VM

  Arguments:

   * *agent* - instance of murano-agent
   * *fileContent* - string, content of file
   * *path* - string, path for writing
   * *helpTest* - string (optional), description for logging. Will be used as
     name of execution plan. If it is *Null*, value of *path* will be used.

The both methods actually use the same procedure of sending the execution plans
to the agent and require corresponding templates for that, but hide from a
developer this routine.

**Example of usage**

The next example describes how new feature can be used. This code demonstrates
workflow of method in WordPress application, which used for re-configuration
of database settings.

::

  changeDatabaseConnection:
    Arguments:
      - dbHost:
          Contract: $.string().notNull()
      - dbName:
          Contract: $.string().notNull()
      - dbUser:
          Contract: $.string().notNull()
      - dbPassword:
          Contract: $.string().notNull()
    Body:
      - $resources: new(sys:Resources)
      # Creating instance of Linux class
      - $linux: new(conf:Linux)
      # First we need to stop server. 'runCommand' can be used here
      - $linux.runCommand($.instance.agent, 'service apache2 stop')
      # Creating a dictionary for replacement
      - $configReplacements:
          "%DB_HOST%": $dbHost
          "%DB_NAME%": $dbName
          "%DB_USER%": $dbUser
          "%DB_PASS%": $dbPassword
      # Making a replacement. `wp-config.php' is included to package
      - $confFileContent: $resources.string('wp-config.php').replace($configReplacements)
      # Putting ready content to necessary path on VM
      - $linux.putFile($.instance.agent, $confFileContent, '/var/www/html/wordpress/wp-config.php')
      # Now we can start Apache again
      - $linux.runCommand($.instance.agent, 'service apache2 start')

Alternatives
------------

Instead of using the common procedure with creating execution plans and
communication with murano-agent some software configuration resources of heat
probably can be used. During updating of library it can be used in the future.


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

Application developers will be able to use new functionality in their apps.
Existing apps will not be affected.

Murano-dashboard / Horizon impact
---------------------------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  ddovbii

Work Items
----------

* Create new class *io.murano.configuration.Linux*
* Implement methods *putFile* and *runCommand*
* Update Murano PL docs

Dependencies
============

None

References
==========

None
