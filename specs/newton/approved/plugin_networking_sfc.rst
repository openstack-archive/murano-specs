===============================================
Support for Service Function Chaining in Murano
===============================================

One of the foundations for NFV enabled clouds is to have Networking Service
Function Chaining which provides an ability to define an ordered list of
network services which to form a “chain” of services. This could be used by
f.e. telcos to simplify management of their infrastructure.

Problem description
===================

Service chaining allow to pass traffic from endpoint A to endpoint B thru any
number of additional network service VMs. Those VMs are managed by tenant
admin.
With standard OpenStack networking traffic goes directly from A to B::

    +--------+   +--------+
    |  VM A  |-->|  VM B  |
    +--------+   +--------+

With SFC traffic goes thru additional hops, which can perform any action
(firewall, LB, ...)::

    +------+   +-------+   +-------+   +-------+   +------+
    |      |   |Service|   |Service|   |Service|   |      |
    | VM A |-->| VM  A |-->| VM  B |-->| VM  C |-->| VM B |
    +------+   +-------+   +-------+   +-------+   +------+


Those service VMs are transparent for end-user.

To achieve this, it would require to extend some of the OpenStack components.
Murano is suitable for defining such chains, but currently lacks of necessary features.

SFC use few entities to deliver chaining:

* Port pair

  Represents vNIC (ingress/input, egress/output) attached to service VM.
  Those vNICs will be used to receive (ingress) and send (egress) SFC traffic.

* Port group

  Represents list of port pairs of different VMs with the same service function.

* Flow classifier

  Used to classify network traffic, based on different metrics (e.g. source IP,
  destination IP, source port, destination port, ...).
  Only traffic which matches flow classifier will be handled by SFC.

* Port chain

  Represents a releationship between flow classifiers and port groups.

To be able to use NFV functionality it is needed to have Service Chaining
functionality available in OpenStack.

Neutron does not support Networking SFC out of the box.

Currently Heat does not support configuration for Networking SFC API.

Currently Murano does not support Networking SFC management at all.

If we want to use Murano as a Resource Orchestrator it needs to have support
for configuring Service Chaining.

.. note::

    Networking-SFC caveats:

    * Current SFC implementation is not mature
    * Supports only Open vSwitch
    * API under heavy development
    * Multiple API limitations
    * No ready service VM software (capture SFC traffic, do *thing*, pass SFC
      traffic to next hop)


Proposed change
===============

To solve problems above it is needed to introduce changes in Murano.
Having Murano as a Resource Orchestrator we need to have an ability to
configure Networking SFC from within Murano application templates.
This involves to implement changes in several areas.

Murano will be extended with:

* `Murano plugin`_ that will provide Networking SFC class which will be
  a wrapper for lower level Neutron API.
* `Murano library package`_ that will provide Murano PL classes for
  Networking SFC entities (e.g. PortPair, PortPairGroup, etc.)


Murano plugin
-------------

A python plugin that provides low-level wrappers for Neutron API functions
which are provided by Networking SFC extension. It implements direct method
wrappers for neutronclient library:

.. note:: Interface may change in the future

* ``createPortPair``
* ``deletePortPair``
* ``listPortPairs``
* ``showPortPair``
* ``updatePortPair``
* ``createPortPairGroup``
* ``deletePortPairGroup``
* ``listPortPairGroups``
* ``showPortPairGroup``
* ``updatePortPairGroup``
* ``createFlowClassifier``
* ``deleteFlowClassifier``
* ``listFlowClassifiers``
* ``showFlowClassifier``
* ``updateFlowClassifier``
* ``createPortChain``
* ``deletePortChain``
* ``listPortChains``
* ``showPortChain``
* ``updatePortChain``


Murano library package
----------------------

Application developer should have a possibility to configure Networking SFC
entities from Murano PL classes. We will introduce Murano plugin which will
extend to support configuration of following SFC classes:

* Port Pair
* Port Pair Group
* Flow Classifier
* Port Chain

All above parameters should be provided to successfully configure application
chaining. This should be done in Murano template for meta application
responsible for provisioning whole set of applications. Only mandatory SFC
parameters should be available for editing in Murano application UI.

PortPair class
~~~~~~~~~~~~~~

A Port Pair represents a service function instance. The ingress port and the
egress port of the service function may be specified. If a service function
has one bidirectional port, the ingress port has the same value as the egress
port.

**Properties:**

* ``id`` - Port pair ID
* ``name`` - Port pair name (Default: '')
* ``description`` - Readable description (Default: '')
* ``ingressPort`` - Ingress port (Reference to NeutronPort object)
* ``egressPort`` - Egress port (Reference to NeutronPort object)

.. note:: ingressPort and egressPort need to be changed to references once
          Murano Core Library has support of network objects.

**Methods:**

* ``PortPair.deploy()`` - Create port pair object
* ``PortPair..destroy()`` - Cleanup resources (called implicitly)

**Example:**

.. code-block:: yaml

    Namespaces:
    =: com.example
    netsfc: io.murano.extensions.networkingSfc

    Name: DemoApp
    Properties:
    ...
    portPair: $.class(netsfc:PortPair).notNull()

    Methods:
        deploy:
            Body:
            - ...
            - $.portPair.deploy()


PortPairGroup class
~~~~~~~~~~~~~~~~~~~

Inside each "port-pair-group", there could be one or more port-pairs.
Multiple port-pairs may be included in a "port-pair-group" to allow the
specification of a set of functionally equivalent SFs that can be be used for
load distribution, i.e., the "port-pair" option may be repeated for multiple
port-pairs of functionally equivalent SFs.

**Properties:**

* ``id`` - Port pair group ID
* ``name`` - Readable name (Default: '')
* ``description`` - Readable description (Default: '')
* ``portPairs`` - List of PortPair class objects.

**Methods:**

See `PortPair class`_

Example:

.. code-block:: yaml

    Name: DemoApp
    Properties:
        ...
        portPairs: [$.class(netsfc:PortPair).notNull()]
    Methods:
        deploy:
        Body:
        - ...
        - $.portPairGroup: new(netsfc:PortPairGroup, portPairs => $.portPairs)
        - $.portPairGroup.deploy()

FlowClassifier class
~~~~~~~~~~~~~~~~~~~~

Flow classifiers are used to select the traffic that can access the chain.
Traffic that matches any flow classifier will be directed to the first port
in the chain. The flow classifier will be a generic independent module and
may be used by other projects like FW, QOS, etc.
A flow classifier cannot be part of two different port-chains otherwise
ambiguity will arise as to which chain path that flow's packets should go.
A check will be made to ensure no ambiguity. But multiple flow classifiers
can be associated with a port chain since multiple different types of flows
can request the same service treatment path.

**Properties:**

* ``name`` - Classifier name
* ``ethertype`` - IPv4 (default) / IPv6
* ``protocol`` - IP protocol
* ``sourcePortRange`` - Source protocol port; port number or tuple of min and
  max ports
* ``destinationPortRange`` - Destination protocol port; port number or tuple of
  min and max ports
* ``sourceIpPrefix``
* ``destinationIpPrefix``
* ``logicalSourcePort`` - neutron source port (required)
* ``logicalDestinationPort`` - neutron destination port (required)
* ``l7Parameters`` - dictionary of L7 parameters (not used)

**Methods:**

See `PortPair class`_

PortChain class
~~~~~~~~~~~~~~~

A Port Chain is a directional service chain. The first port of the first
port-pair is the head of the service chain. The second port of the last
port-pair is the tail of the service chain. A bidirectional service chain would
be composed of two unidirectional Port Chains.

**Properties:**

* ``id`` - Port chain ID
* ``name`` - Readable name
* ``description`` - Readable description
* ``portPairGroups`` - List of PortPair class objects
* ``flowClassifiers`` - List of FlowClassifier class objects

**Methods:**

See `PortPair class`_

Delivering Murano extensions
----------------------------

Murano uses stevedore [0] library to support plugins. Basically stevedore
plugin is a python package with special metadata (plugin entry points).
It can be installed as python package (via ``pip``) or distribution specific
package (e.g. DEB, RPM, etc.). Murano plugins provides new classes
that are referenced in the plugin metadata.

Murano library package is a simple **zip** archive which contains
``metadata.yaml`` file and Murano PL classes.
It can be installed from file using ``murano`` command-line tool
or from the repository.

Alternatives
------------

Extend Heat to support Networking SFC.
Instead of accessing Neutron API directly from Murano plugins, Murano can
manage Networking SFC through Heat resources. It will require creating Heat
extension which will delegate SFC configuration from Heat templates.
Heat extension should provide set of resources which will implement.

Networking SFC entities:

* Port Chain
* Port Pair Group
* Port Pair
* Flow Classifier

Heat can be extended by adding new resources using plugins. It provides mapping
between heat resources and python classes. It will implement hooks on heat
events like resource create, update etc.

**PROS:**

* Resource management delegated to Heat. Heat can be used for resource cleanup
* Heat can provide some helpful transactional capabilities
* Cloud operator will be able to use SFC w/ and wo/ Murano

**CONS:**

* Security-related implications for customers, who wants to keep vanilla
  OpenStack installation.

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

End user will be able to use networking SFC from Murano templates.

Networking SFC will introduce additional delay in RTT (round-trip-time).
This mean some applications prone to network delays can stop working.

Deployer impact
---------------

To use SFC with Murano cloud operator needs to install and enable SFC Murano
plugin.

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

Work Items
----------

* Create Murano Networking SFC plugin
* Create Murano library package
* Create Murano test application
* Implement support of multiple Neutron ports to Murano Core Library
* Tests

Dependencies
============

Murano plugin depends on networking-sfc package, which should be installed.

Testing
=======

Proper functional tests should be implemented.

Documentation Impact
====================

User will be informed how to enable and start using networking SFC inside
Murano templates.

References
==========

[0] - http://docs.openstack.org/developer/stevedore/
