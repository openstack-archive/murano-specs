..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=======================
Multiple engine workers
=======================

URL of launchpad blueprint:

https://blueprints.launchpad.net/murano/+spec/multiple-engine-workers

This specification is to implement multiple workers to murano-engine.

Problem description
===================

murano-engine is currently single process. It's a problem for scalability.

Proposed change
===============

Implement multiple workers to murano-engine by oslo.service library.
When starting service, murano-engine forks the number of workers which is
written in configuration file. When a configuration reload is required, restart
service is needed. If child process is killed, murano-engine forks new worker.
Most of the OpenStack projects use oslo.service.

Alternatives
------------

There are some external tools.

Configuration management tool such as Puppet, Chef, Ansible
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Configuration management tool controls services which is managed by init systems
like systemd, upstart. Configuration management tool does not manage processes.

Init systems
~~~~~~~~~~~~
Systemd, upstart are init systems of Unix like systems. Upstart is default init
system until Ubuntu 14.10 or RHEL6. Systemd is default init system of Ubuntu
15.04 or RHEL7, and other Linux systems.

Systemd manages processes based on configuration file. If process spawns child
processes, systemd recognizes child processes properly and manages under single
control group by a configuration file. Systemd can be configured automatically
restart process on crash but does not have ability to spawn process.

Supervisor
~~~~~~~~~~
Supervisor is time-proven process management tool written in python.
Supervisor can control a number of processes on UNIX-like operating systems.
Supervisor starts processes as subprocesses , so can true up/down and can be
configured automatically restart them on a crash.

Supervisor has good function itself, but must managed by init systems and
considered high availability. Supervisor is not working under Python 3. The
whole Openstack and murano are going to support python 3. This is not
appropriate for alternatives.


Combination of configuration management tool and systemd can be altenative for
implementing multiple processes. configuration management tool deploys services,
each service manages single murano-engine process.

Both proposed change and alternative can process restart if process crashed.

Proposed change can manage multi processes easier than alternative. Proposed
change has main process which spawns child processes. If you change the number
of processes, update configuration file and then restart service. On the other
hand, alternative needs to rewrite conf file of configuration managenment tool
and redeploy.

Proposed change can manage control group of murano-engine processes easier than
alternative. Because proposed change can change control group settings by update
a configuration file of systemd. On the other hand, alternative needs to update
each confifiguration file of systemd.

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

The number of murano-engine workers config option is added in
murano.conf under engine section named workers. The default value
is oslo_concurrency.processutils.get_worker_count().

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
  <nakamura-h>

Work Items
----------

* Add the number of murano-engine workers config option to murano.conf
* Replace oslo_service.service.ServiceLauncher with
  oslo_service.service.launch in murano.engine.cmd module.
  Worker parameter of oslo_service.service.launch method takes
  the number of murano-engine workers config option.

Dependencies
============

* oslo_concurrency

Testing
=======

* Unit tests on murano-engine

Documentation Impact
====================

None

References
==========

* `Oslo.service <https://github.com/openstack/oslo.service>`_
* `Puppet <https://github.com/puppetlabs/puppet>`_
* `Chef <https://github.com/chef/chef>`_
* `Ansible <https://github.com/ansible/ansible>`_
* `Upstart <http://upstart.ubuntu.com/>`_
* `Systemd <http://www.freedesktop.org/wiki/Software/systemd/>`_
* `Supervisor <http://supervisord.org/>`_
