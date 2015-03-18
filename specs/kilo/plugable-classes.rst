..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================================
Plugable pythonic classes for Murano
====================================

https://blueprints.launchpad.net/murano/+spec/plugable-classes

One of the key features of Murano is extensibility, and we need to push this
feature even further and give our customers a way to extend Murano with new
functions (e.g. support for F5 BigIP API) in a drag-n-drop manner. This spec
proposes a solution which will add this extensibility option.


Problem description
===================

Currently all the functionality which is available to user is limited to the
features of MuranoPL language which just provides data transformation
capabilities and flow control primitives. The language itself does not contain
any functions for I/O operations, hardware access or interaction with host
operating system, other OpenStack or third-party services. This is an
intentional design feature: MuranoPL code is provided by users and cannot be
always trusted. All the external communications and low-level interactions are
done via python code which is bundled with Murano Engine and is accessible to
MuranoPL code via MuranoPL wrappers. Any interactions which are not supported
by that Python classes are impossible.

Some deployment scenarios may need to extend this set of allowed low-level
interactions. They may include some customer-specific logic, custom software
bindings etc, so trying to bundle all of them into the standard Murano Engine
classes is not a good idea. Instead, there should be a way to dynamically
add extra interactions to any existing deployment of Murano without modifying
its core components but rather with installing some plugin-like components.

Installing these plugins is supposed to be a maintainer-only operation,
requiring administrative access to nodes running Murano services. It is
supposed that the maintainer is always aware about the contents of the plugins
and is able to verify them from security, performance and other sensible points
of view.

Proposed change
===============

It is proposed to implement each extension as independent Python Package built
using setuptools library. Each package should define one or more entry-point in
a specific namespace (``io.murano.extensions`` is suggested). Each of this
entry- points should export a class, which may be registered as MuranoPL class
when the engine loads.


Each package should be installed on Murano nodes into the same Python
environment with Murano engine service.

Murano will get a PluginLoader class which will utilize stevedore library [1]
to discover classes registered as entry-points in ``io.murano.extensions``
namespace.

Murano Engine will use PluginLoader to register all the loaded plugins in its
class loader (i.e. will call ``import_class`` with a class imported from the
plugin as a parameter). As the result, the classes will become available for
the MuranoPL code being executed in the Engine.

To prevent potential name collisions, MuranoPL names for the loaded classes
will be assigned automatically: the name will consist of the namespace
(``io.murano.extensions`` as suggested above) and the name of entry-point.
To guarantee this naming rule the imported classes should not define their
MuranoPL names on their own (i.e they should not have
``@murano_class.classname`` decorators or other code which modifies their
``_murano_class_name`` field). If they do, the PluginLoader will discard
that information and will log a warning message.

As the entry-point name will eventually become a name of MuranoPL class, the
PluginLoader will validate it accordingly.

As neither stevedore nor setuptools enforce any uniqueness constraints on the
entry-point names (i.e. several packages may define entry-points with the same
name within a same namespace, and all of them will be correctly loaded by
stevedore), then this enforcement should also be done by the Murano's Plugin
Loader. If two or more plugin packages attempt to register classes with the
same endpoint name, then a warning will be logged and no classes from all the
conflicting packages will be loaded.

PluginLoader will also ensure that objects being exported in these entry-points
are indeed classes, and will check if they define a classmethod called
``init_plugin``. If such method exists, the PluginLoader will execute it before
loading it.

The plugins which are already installed in the environment may be prevented
from being loaded by a configuration option. This new option called
``enabled_plugins`` will be added to ``murano.conf``. If it has its default
value ``None`` or is omitted from the config, there will be no restriction on
the plugins which are being loaded (any plugin registered within the
environment will be loaded). If it exists and is not None, then it is expected
to contain a list of names of the packages from which the plugins will be
loaded. If the package is not mentioned there, all its endpoints will be
ignored and no classes from it will be imported. Empty value of
``enabled_plugins`` will mean that no plugins may be loaded and only bundled
system classes are accessible from the MuranoPL code.

The ``enabled_plugins`` setting will be implemented using
``EnabledExtensionManager`` class of the stevedore library [2], so the disabled
plugins will be excluded from entry-point name analysis. Thus if there are
plugins which define conflicting entry-point names, then the conflict may be
resolved with this setting instead of uninstalling the plugin from the
environment.

Currently stevedore is unable to load packages which were installed after the
start of the current process. So, in current proposal it is required to restart
Murano services after plugin package is installed, removed or upgraded and
after the changing of ``enabled_plugins`` value in configuration file.
As the restart of the services is not a good thing for production solutions, it
may be a good idea to design a "graceful restart" solution which will make the
service to stop listening for incoming requests, finish its current tasks and
then exit and restart, loading the updated configuration and plugins. However
such solution is out of scope of the current spec and is left for future
blueprints.


Alternatives
------------

Instead of using stevedore to discover and load the plugins, some home-made
solution may be invented to load Python modules from some directory. This
solution may have its benefits (e.g. it does not require restarts to load new
plugins), however stevedore is currently a de-facto standard for building
plugable solutions in Openstack, so it is suggested to use it.

Data model impact
-----------------

This proposal does not affect data model.

REST API impact
---------------

N/A


Versioning impact
-----------------

N/A

Other end user impact
---------------------

N/A

Deployer impact
---------------

The change itself does not have any immediate impact on deployer: a new
configuration option is optional and has meaningful default. However
registering new plugins will require to restart the Services, which may bring
up some concerns in production environments.


Developer impact
----------------

Developers who build their own plugins should be aware about setuptools entry-
points and should inherit their exported classes from
`murano.dsl.murano_object.MuranoObject`.

Murano-dashboard / Horizon impact
---------------------------------

No immediate changes required.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  ativelkov


Work Items
----------

* Implement the PluginLoader class
* Modify MuranoEngine to register plugin-imported classes in class loader.


Dependencies
============

This requires stevedore library as a dependency. It is already part of
OpenStack Global Requirements, so no problems are expected.


Testing
=======

The unit-tests have to cover PluginLoader class using the make_test_instance
method of stevedore.

Separated tests should cover API method call.

Tempest tests are out of the scope of this spec.

Documentation Impact
====================

There should be created a "Plugin developer's Manual" which will describe the
process of plugin package creation.


References
==========

[1] http://docs.openstack.org/developer/stevedore/
[2] http://docs.openstack.org/developer/stevedore/managers.html#enabledextensionmanager
