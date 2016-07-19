..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================
Clearwater Murano application
=============================

https://blueprints.launchpad.net/murano-apps/+spec/clearwater-vims-implementation

Problem description
===================

It must be possible to deploy and manage lifecycle of Clearwater vIMS.
Clearwater vIMS consists of several components:

* Bono (Edge Proxy)
* Sprout (SIP Router)
* Homestead (HSS Cache)
* Homer (XDMS storage)
* Ellis (provisioning portal)
* Ralf (CTF)

Additionally following DNS server application should be written:
* BIND

Clearwater deployment diagram
::

                           +----------+
                           |          |
                           |   Ellis  |
                    HTTP   |          |
                    +------+          +-----+
                    |      |          |     | HTTP
                    |      +----------+     |
                    |                       |
              +-----+-----+           +-----+-----+       +------------------+
              |           |           |           |       |                  |
              |  Homer    |           | Homestead |       |Client SIP software
              |           | XCAP  SIP |           |       |                  |
              |           +--+     +--+           |       +---------------+--+
              |           |  |     |  |           |                       ^
              +-----------+  |     |  +-----------+                       |
                             |     |                                      |
                             |     |                                      |
    +------------+         +-+-----+----+          +------------+         |
    |            |  HTTP   |            |   SIP    |            |    SIP  |
    |    Ralf    +---------+   Sprout   +----------+    Bono    +---------+
    |            |         |            |          |            |
    |            |         |            |          |            |
    |            |         |            |          |            |
    +------+-----+         +------------+          +---+--------+
           |                                           |
           |                    HTTP                   |
           +-------------------------------------------+

More details about Clearwater components may be found at http://www.projectclearwater.org

Proposed changes
================
Separate application should be written for each of Clearwater components.
One more application should be written for aggregation of all components
(Clearwater main application).

Main application input parameters:

* OS image for deploy
* user keypair name
* zone name
* DNSSEC key

This parameters should be passed to child components.

Components input parameters:

* Flavour for instances
* Count of instances

Main application workflow

#. Instantiate child applications
#. Create common networks and security group
#. Call child applications `deploy()` method

Components have to be deployed in following order:

#. DNS application
#. One application with initial
   `etcd <https://coreos.com/etcd>`_ cluster node (Ellis preferred)
#. Rest of applications


Components common workflow

#. Find main application for common properties retrieving
#. Create security group for application
#. Create instances
#. Create deploy scripts from templates
#. Run deploy scripts


For components (except DNS server) following actions must be provided:

* Scale Up - add additional node to cluster
* Scale Down - remove node from cluster

Both actions can be implemented in two steps:

* add/remove node to/from component object model
* update appropriate DNS records


Alternatives
------------

* `OpenStack Heat templates for Clearwater deployments <https://github.com/Metaswitch/clearwater-heat>`_
* `Cloudify TOSCA application <https://github.com/Orange-OpenSource/opnfv-cloudify-clearwater>`_


Data model impact
-----------------

None


REST API impact
---------------

None


Versioning impact
-----------------

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

  Konstantin <ksnihyr@mirantis.com>

Work Items
----------

#. Create Clearwater application prototype
#. Extend Clearwater application functionality using current Core Library
#. Add Scalable Application Framework support



Dependencies
============

#. `Scalable Application Framework <https://blueprints.launchpad.net/murano/+spec/scalable-application-framework>`_

Testing
=======

#. Clearwater live tests can be used for testing installation.

Documentation Impact
====================

None


References
==========

* `Project Clearwater Home <http://www.projectclearwater.org>`_
* `Clearwater documentation <http://clearwater.readthedocs.io/en/stable/>`_
* `Clearwater heat template <https://github.com/skolekonov/clearwater-heat/tree/mitaka>`_
* `Clearwater live tests <https://github.com/Metaswitch/clearwater-live-test>`_
