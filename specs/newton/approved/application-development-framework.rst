..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=================================
Application Development Framework
=================================

https://blueprints.launchpad.net/murano/+spec/application-development-framework

Build a library of MuranoPL classes to define basic building blocks for the
applications utilizing standard patterns for scaling, load balancing, healing
etc.


Problem description
===================

Currently Murano Applications are not opinionated about the workflow they have
to follow to be scalable, highly available, (self) healable and so on. The only
interface which is declared for a standard murano application defines a single
method called “deploy” which has to implement the complete provisioning
procedure of the app. This approach is cumbersome, requires lots of coding
while most of the code is a boilerplate which could be reused by most the
applications.

Also, the “deploy” workflow is just a beginning of applications’ life in the
cloud: Murano app should define the lifecycle management of the app as well: to
handle the scaling, healing and other special behaviors. Similarly to the
deployment behaviors these kinds of workflow may have some generic steps which
should be made on most of the applications and should not require manual
copying.

It would be beneficial for the developers of Murano Applications to have a set
of base classes defining the standard building blocks for the typical
applications and their life cycle management operations.

So Murano should allow Application Developers to focus on their
application-specific tasks only (e.g. “how to install the software on the
VMs”, “how to configure it to interact with other apps” etc) without the real
need to dive into resource orchestration, server farm configuration and so on.
The application developers’ experience has to be as lightweight as possible:
ideally they should be able to focus on the software configuration tools
(scripts, puppets etc) and completely ignore the MuranoPL if they do not need
to define any custom workflow logic.


Proposed change
===============

It is proposed to implement a set of base classes for the applications and
their components.

Library structure
-----------------

Scaling primitives
~~~~~~~~~~~~~~~~~~

To properly handle various scaling scenarios it is required to have a set of
classes which will be able to group similar resources together, produce new
copies of the same resources or release the existing ones on request.

The following hierarchy of classes is proposed to define this functionality on
the generic level as well as to implement it for the case of the Servers/VMs:


::

 +-------+
 | +-------+
 | | +--------+        +------------------+        +-----------------+
 | | |        |        |                  |        |                 |
 +-+ | Object <--------+ ReplicationGroup +--------> ReplicaProvider |
   +-+        |        |                  |        |                 |
     +--------+        +---+--------------+        +-+--------+------+
                           ^                         ^        ^
                           |                         |        |
                           |      +------------------+-----+  |
                           |      |                        |  |
 +-------+                 |      |  CloneReplicaProvider  |  |
 | +-------+               |      |                        |  |
 | | +----------+          |      +------------------------+  |
 | | |          |          |                                  |
 +-+ | Instance |          |                                  |
   +-+          |          |                                  |
     +----+-----+          |                                  |
          |                |                                  |
    +-----+-------+        |                                  |
    |             |        |                                  |
    | ServerGroup |        |                  +---------------+--+
    |             |        |                  |     Template     |
    +-----^-------+    +---+----------+       |      Server      +--+
          |            |    Server    +------->     Provider     |  |
          +------------+  Replication |       +-----+------------+  +---+
                       |    Group     |             |               |   |
                       +--------------+             +---+---other---+   |
                                                        |               |
                                                        +---------------+




**ReplicationGroup**

    A base class which does the object replication. It holds the collection of
    objects generated in runtime in one of its output properties and contains
    a reference to a ``ReplicaProvider`` object which is used to dynamically
    generate the objects in runtime.

    Input properties of this class include the ``minItems`` and ``maxItems``
    allowing to limit the number of objects it holds in its collection.

    An input-output property ``numItems`` allows to declaratively change the
    set of objects in the collection by setting its size.

    The ``deploy`` method of this class will be used to apply the replica
    settings: it will drop the objects from the collection if their number
    exceeds the specified by the ``numItems`` or generate some new if there
    are not enough of them.


**ReplicaProvider**

    A class to generate the objects for a ``ReplicationGroup``. The base one
    is abstract, its inheritors should implement the abstract
    ``createReplica`` method to create the actual object. The method may
    accept some index parameter to properly parametrize the newly created copy.

    The concrete implementations of this class should define all the input
    properties needed to create new instances of object. Thus the provider
    actually acts as a template of the object it generates.


**CloneReplicaProvider**

    An implementation of ``ReplicaProvider`` capable to create replicas by
    cloning some user-provided object, making use of the ``template()``
    contract.


**ServerGroup**

    A class that provides static methods for deployment and releasing
    resources on the group of instances.


**ServerReplicationGroup**

    A subclass of the ``ReplicationGroup`` class and the ``ServerGroup``
    class to replicate the ``Instance`` objects it holds.

    The ``deploy`` method of this group not only generates new instances of
    servers but also deploys them if needed.

**TemplateServerProvider**

    A subclass of ``CloneReplicaProvider`` which is used to produce the objects
    of ``Instance`` class by cloning them with subsequent parameterization of
    the hostnames. May be passed as ``provider`` property to objects of the
    ``ServerReplicationGroup`` class.

**other replica providers**

    Other subclasses of ``ReplicaProvider`` may be created to produce different
    objects of ``Instance`` class and its subclasses depending on particular
    application needs.


Software Components
~~~~~~~~~~~~~~~~~~~

The class to handle the lifecycle of the application is the
``SoftwareComponent`` class which is a subclass of ``Installable`` and
``Configurable``:

::

 +-----------+-+           +-+------------+
 |             |           |              |
 | Installable |           | Configurable |
 |             |           |              |
 +-----------+-+           +-+------------+
             ^               ^
             |               |
             |               |
           +-+---------------+-+
           |                   |
           | SoftwareComponent |
           |                   |
           +-------------------+




The hierarchy of the ``SoftwareComponent`` classes should be used to define the
workflows of different application lifecycles. The general idea is to have the
generic logic in the methods of the base classes and let the derived classes
implement the handlers for the custom logic. The model is event-driven: the
workflow consists of the multiple steps, and most of the steps invoke
appropriate `on%StepName%` methods intended to provide application-specific
logic.

It is proposed to split 'internal' step logic and their 'public' handlers
into separate methods. Technically this is not necessary since the subclass may
always call `super()` to invoke the base logic, but the developers tend to
forget to invoke these super-implementations – so having the logic split into
two parts should improve the developers' experience and simplify the code of
derived classes.

The standard workflows (such as Installation and Configuration) will be defined
by the ``Installable`` and ``Configurable`` classes. The main implementation -
``SoftwareComponent`` will inherit both these classes and will define its
deployment workflow as a sequence of Installation and Configuration flows.
Other future implementations may add new workflow interfaces and mix them in
to change the deployment workflow or add new actions.


Installation workflow consists of the following methods:

::

 +----------------------------------------------------------------------------------------------------------------------+
 | INSTALL                                                                                                              |
 |                                                                                                                      |
 |      +------------------------------+                               +---------------+                                |
 |    +------------------------------+ |                             +---------------+ |                                |
 |  +------------------------------+ | |      +---------------+    +---------------+ | |      +----------------------+  |
 |  |                              | | |      |               |    |               | | |      |                      |  |
 |  |    checkServerIsInstalled    | +-+ +----> beforeInstall +----> installServer | +-+ +----> completeInstallation |  |
 |  |                              +-+        |               |    |               +-+        |                      |  |
 |  +------------------------------+          +------+--------+    +------+--------+          +-----------+----------+  |
 |                                                   |                    |                               |             |
 +----------------------------------------------------------------------------------------------------------------------+
                                                     |                    |                               |
                                                     |                    |                               |
                                                     |                    |                               |
                                                     v                    v                               v
                                               onBeforeInstall      onInstallServer              onCompleteInstallation


**install**
    * **Arguments:** ``serverGroup``
    * **Description:**
      Entry point of the installation workflow.

      Iterates through all the servers of the passed ServerGroup and calls the
      ``checkServerIsInstalled`` method for each of them. If at least one
      of the calls has returned `false` calls a ``beforeInstall`` method. Then,
      for each server which returned `false` as the result of the
      ``checkServerIsInstalled`` calls the ``installServer`` method to do
      the actual software installation.
      After the installation has been completed on all the servers and if at
      least one of the previous calls of ``checkServerIsInstalled``
      returned `false` the method runs the ``completeInstallation`` method.

      If all the calls to ``checkServerIsInstalled`` returned `true`
      this method concludes without calling any others.

**checkServerNeedsInstallation**
    * **Arguments:** ``server``
    * **Description:** checks if the given server requires a (re)deployment of
      the software component. By default checks for the value of the attribute
      `installed` of the instance.

      May be overridden by subclasses to provide some better logic (e.g. the
      app developer may provide code to check if the given software is
      pre-installed on the image which was provisioned on the VM)

**beforeInstall**
    * **Arguments:** ``servers``, ``serverGroup``
    * **Description:**
      Reports the beginning of installation process and calls the public event
      handler ``onBeforeInstall``.

**onBeforeInstall**
    * **Arguments:** ``servers``, ``serverGroup``
    * **Description:** Public handler of the `beforeInstall` event. Empty in
      the base class, may be overridden in subclasses if some custom pre-install
      logic needs to be executed.

**installServer**
    * **Arguments:** ``server``, ``serverGroup``
    * **Description:** Does the actual software deployment on a given server by
      calling an ``onInstallServer`` public event handler. If the installation
      completes successfully sets the `installed` attribute of the server to
      `true`, reports successful installation and returns `null`. If an
      exception encountered during the invocation of ``onInstallServer``, the
      method handles that exception, reports a warning and returns the server.
      The return value of the method indicates to the ``install`` method how
      many failures encountered in total during the installation and with what
      servers.

**onInstallServer**
    * **Arguments:** ``server``, ``serverGroup``
    * **Description:** an event-handler method which is called by the
      ``installServer`` method when the actual software deployment is needed.
      Is  empty in the base class. The implementations should override it with
      custom logic to deploy the actual software bits.

**completeInstallation**
    * **Arguments:** ``servers``, ``serverGroup``, ``failedServers``
    * **Description:** is executed after all the ``installServer`` methods were
      called. Checks for the number of errors reported during the installation:
      if it is greater than some pre-configurable threshold an exception is
      risen to interrupt the deployment workflow. Otherwise the method calls an
      ``onCompleteInstallation`` event handler and then reports a successful
      completion of the installation workflow.

**onCompleteInstallation**
    * **Arguments:** ``servers``, ``serverGroup``, ``failedServers``
    * **Description:** an event-handler method which is called by the
      ``completeInstallation`` method when the component installation is about
      to be completed.

      Default implementation is empty. Inheritors may implement this method to
      add some final handling, reporting etc.


Configuration workflow consists of the following methods:

::

 +----------------------------------------------------------------------------------------------------------------------+
 | CONFIGURATION                                                                                                        |
 |               +-----------------+                                                                                    |
 |               |                 |                                                                                    |
 |               |          +---------------+                          +-----------------+                              |
 |               |        +---------------+ |                        +-----------------+ |                              |
 |  +------------v--+   +---------------+ | |   +--------------+   +-----------------+ | |   +-----------------------+  |
 |  |               |   |               | | |   |              |   |                 | | |   |                       |  |
 |  | checkCluster\ +---> checkServer\  | +-+---> preConfigure +---> configureServer | +-+---> completeConfiguration |  |
 |  | IsConfigured  |   | IsConfigured  +-+     |              |   |                 +-+     |                       |  |
 |  +------------+--+   +---------------+       +------+-------+   +--------+--------+       +-----------+-----------+  |
 |               |                                     |                    |                            |              |
 |               |                                     |                    |                            |              |
 |    +----------v----------+                          |                    |                            |              |
 |    |                     |                          |                    |                            |              |
 |    | getConfigurationKey |                          |                    |                            |              |
 |    |                     |                          |                    |                            |              |
 |    +---------------------+                          |                    |                            |              |
 |                                                     |                    |                            |              |
 +----------------------------------------------------------------------------------------------------------------------+
                                                       |                    |                            |
                                                       |                    |                            |
                                                       v                    v                            v
                                                 onPreConfigure     onConfigureServer          onCompleteConfiguration


**configure**
    * **Arguments:** ``serverGroup``
    * **Description:**
      Entry point of the configuration workflow.

      Calls a ``checkClusterIsConfigured`` method. If the call returns `true`,
      workflow exits without any further action. Otherwise for each server in
      the ``serverGroup`` it calls ``checkServerIsConfigured`` method and gets
      the list of servers that need reconfiguration. The ``preConfigure``
      method is called with that list. At the end calls the
      ``completeConfiguration`` method.

**checkClusterIsConfigured**
    * **Arguments:** ``serverGroup``
    * **Description:**
      Has to return `true` if the configuration (i.e. the values of input
      properties) of the component has not been changed since it was last
      deployed on the given server group. Default implementation calls the
      ``getConfigurationKey`` method and compares the returned result with a
      value of `configuration` attribute of ``serverGroup``. If the results
      match returns `true` otherwise `false`.

**getConfigurationKey**
    * **Arguments:** None
    * **Description:**
      Should return some values describing the configuration state of the
      component. This state is used to track the changes of the configuration
      by the ``checkClusterIsConfigured`` and ``checkServerIsConfigured``
      methods.

      Default implementation returns a synthetic value which gets updated on
      every environment redeployment. Thus the subsequent calls of the
      ``configure`` method on the same server group during the same deployment
      will not cause the reconfiguration, while the calls on the next
      deployment will reapply the configuration again.

      The inheritors may redefine this to include the actual values of the
      configuration properties, so the configuration is reapplied only if the
      appropriate input properties are changed.

**checkServerIsConfigured**
    * **Arguments:** ``server``, ``serverGroup``
    * **Description:**
      It is called to check if the particular server of the server group has
      to be reconfigured thus providing more precise control compared to
      cluster-wide ``checkClusterIsConfigured``.

      Default implementation calls the ``getConfigurationKey`` method and
      compares the returned result with a value of `configuration` attribute
      of the server. If the results match returns `true` otherwise `false`.

      This method gets called only if the ``checkClusterIsConfigured`` method
      returned `false` for the whole server group.

**preConfigure**
    * **Arguments:** ``servers``, ``serverGroup``
    * **Description:**
      Reports the beginning of configuration process and calls the public
      event handler ``onPreConfigure``. This method is called once per the
      server group and only if the changes in configuration are detected.

**onPreConfigure**
    * **Arguments:** ``servers``, ``serverGroup``
    * **Description:**
      Public event-handler which is called by the ``preConfigure`` method
      when the (re)configuration of the component is required.

      Default implementation is empty. Inheritors may implement this method to
      set various kinds of cluster-wide states or output properties which may
      be of use at later stages of the workflow.

**configureServer**
    * **Arguments:** ``server``, ``serverGroup``
    * **Description:**
      Does the actual software configuration on a given server by calling the
      ``onConfigureServer`` public event handler. If the configuration
      completes successfully calls the ``getConfigurationKey`` method and sets
      the `configuration` attribute of the server to resulting value thus
      saving the configuration applied to a given server. Returns `null` to
      indicate successful configuration.

      If an exception encountered during the invocation of
      ``onConfigureServer``, the method will handle that exception, report a
      warning and return the current server to signal its failure to the
      ``configure`` method.

**onConfigureServer**
    * **Arguments:** ``server``, ``serverGroup``
    * **Description:**
      An event-handler method which is called by the ``configureServer``
      method when the actual software configuration is needed. It is empty in
      the base class. The implementations should override it with custom logic
      to apply the actual software configuration on a given server.

**completeConfiguration**
    * **Arguments:** ``servers``, ``serverGroup``, ``failedServers``
    * **Description:**
      It is executed after all the ``configureServer`` methods were called.
      Checks for the number of errors reported during the configuration: if it
      is greater than set by some pre-configured threshold, an exception is
      risen to interrupt the deployment workflow. Otherwise the method calls
      an ``onCompleteConfiguration`` event handler, calls the
      ``getConfigurationKey`` method and sets the `configuration` attribute of
      the server group to resulting value and then reports successful
      completion of the configuration workflow.

**onCompleteConfiguration**
    * **Arguments:** ``servers``, ``serverGroup``, ``failedServers``
    * **Description:**
      The event-handler method which is called by the ``completeConfiguration``
      method when the component configuration is finished at all the servers.

      Default implementation is empty. Inheritors may implement this method to
      add some final handling, reporting etc.


Uninstallation workflow consists of the following methods:

::

 +-----------------------------------------------------------------------------------+
 | UNINSTALL                                                                         |
 |                                                                                   |
 |                                +----------------+                                 |
 |                             +-----------------+ |                                 |
 |  +-----------------+      +-----------------+ | |      +------------------------+ |
 |  |                 |      |                 | | |      |                        | |
 |  | beforeUninstall +------> uninstallServer | +-+------> completeUninstallation | |
 |  |                 |      |                 +-+        |                        | |
 |  +-------+---------+      +--------+--------+          +-----------+------------+ |
 |          |                         |                               |              |
 |          |                         |                               |              |
 +-----------------------------------------------------------------------------------+
            |                         |                               |
            v                         v                               v
    onBeforeUninstall          onUninstallServer           onCompleteUninstallation


**uninstall**
    * **Arguments:** ``ServerGroup``
    * **Description:**
      Entry point of the uninstallation workflow.

      Iterates through all the servers of the passed ServerGroup, for each of
      them checks the presence of the `installed_at_%serverId%` attribute.
      If at least one attribute is present calls a ``beforeUninstall`` method
      once, and then calls an ``uninstallServer`` method for each server which
      has the attribute. If at least one method was called, calls an
      ``afterUninstall`` method at the end.

**beforeUninstall**
    * **Arguments:** ``ServerGroup``
    * **Description:** reports the beginning of uninstalling process and then
      calls an ``onBeforeUninstall`` public event handler.

**onBeforeUninstall**
    * **Arguments:** ``ServerGroup``
    * **Description:** Public handler of the `beforeUninstall` event. Empty in
      the base class, may be overridden in subclasses if some custom pre
      uninstall logic needs to be executed.

**uninstallServer**
    * **Arguments:** ``Server``
    * **Description:** does the actual software removal on a given server by
      calling an ``onUninstallServer`` public event handler. If the removal
      completes successfully clear the `installed_at_%serverId%` attribute of
      the component's attribute storage to indicate that the software component
      is no longer installed on that particular machine.
      If an exception was encountered during the invocation of
      ``onUninstallServer`` the method will handle that exception, report a
      warning and increment the error counter for the particular deployment.

**onUninstallServer**
    * **Arguments:** ``Server``
    * **Description:** an event-handler method which is called by the
      ``uninstallServer`` method when the actual software removal is needed.
      Is  empty in the base class. The implementations should override it with
      custom logic to uninstall the actual software bits.

**completeUninstallation**
    * **Arguments:** ``ServerGroup``
    * **Description:** is executed after all the ``uninstallServer`` methods
      were called. Checks for the number of errors reported during the
      uninstalling: if it is greater than some pre-configurable threshold an
      exception is risen to interrupt the uninstalling workflow. Otherwise the
      method calls an ``onCompleteUninstallation`` event handler and then
      reports a successful completion of the uninstalling workflow.

**onCompleteUninstallation**
    * **Arguments:** ``ServerGroup``
    * **Description:** an event-handler method which is called by the
      ``completeUninstallation`` method when the component removal is done on
      all the servers.

      Default implementation is empty. Inheritors may implement this method to
      add some final handling, reporting etc.


Alternatives
------------

The only alternative is to let application developers to write their own code
for these common tasks. We don't completely drop this alternative, since the
developers are not forced to use the framework and may still continue having
the applications which do not inherit its base classes.

Data model impact
-----------------

None

REST API impact
---------------

None


Versioning impact
-----------------

The first implementation of this spec will utilize the existing version of the
core library. Subsequent ones - redefining the hierarchy of base resource
classes - will need to increment the major version of the core library.

Other end user impact
---------------------

End users should not notice the difference between apps written using the
proposed framework and regular ones.


Deployer impact
---------------

The Framework's library package should be deployed in the target catalog for
other applications to use it.


Developer impact
----------------

This is all about simplifying the life of application developer. Developers
will have to learn the classes and patterns to utilize the benefits of the
framework.

Murano-dashboard / Horizon impact
---------------------------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  ativelkov

Other contributors:
    tbd

Work Items
----------

* Implement the base ``Server`` class and a stub for its core-library-compatible
  inheritor.

* Implement scalable primitives (ReplicationGroup, ReplicaProvider and their
  inheritors)

* Implement a Event/Notification layer in MuranoPL

* Implement ``SoftwareComponent`` class with its standard
  Install/Configure/Uninstall workflows.

* Implement base application classes binding ``SoftwareComponent`` objects to
  ``ReplicationGroup`` objects producing Servers.

Dependencies
============

None

Testing
=======

All the new MuranoPL classes should be covered by test-runner based tests.

Documentation Impact
====================

The framework should be well documented so the package developers have a
reliable and up-to-date source of information.

References
==========

None
