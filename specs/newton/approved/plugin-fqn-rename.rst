..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================================
Change the naming scheme for MuranoPL plugins
=============================================

https://blueprints.launchpad.net/murano/+spec/plugin-fqn-rename

Current naming scheme for classes defined in MuranoPL extension plugins is not
consistent with the naming scheme for regular MuranoPL classes and packages.

This spec addresses this by changing the scheme and proposing a fallback logic
which will provide a backward compatibility for the existing MuranoPL
applications targeting old plugins.



Problem description
===================

MuranoPL classes and packages should have globally unique FQNs to prevent
potential name collisions when environment combines packages provided by
different developers. Recommended approach here is to start the name of a
package with a reversed Internet domain name given that the domain is
controlled by the developer of the package or class (such approach is inspired
by package naming rules of Java, see [1] for more details)

The examples of such class names may include:

 * ``com.example.Foo`` - for demo applications and packages

 * ``org.openstack.projectName.Foo`` - for applications and packages developed
   and maintained by the teams of official Openstack projects

 * ``com.companyname.Foo`` - for applications and packages developed and
   maintained by a third party controlling the "companyname.com" domain name

 * ``io.murano.Foo`` - for applications and packages developed and maintained
   by the core murano team as part of the murano project. So, ``io.murano`` is
   a preferred alias for longer ``org.openstack.murano`` FQN prefix.

However, currently this approach is not working for the classes defined by
MuranoPL extension plugins, since all the names of these classes have to start
with ``io.murano.extension`` prefix, as this is the requirement of the current
plugin management system. This breaks the naming convention and introduces the
possibility of name collision for plugin-defined classes.


Proposed change
===============

Current plugin system is based on stevedore library [2]. So we are utilizing
the concepts of `namespace` and `entry point name`. The namespace value
identifies the consumer of the plugin, the MuranoPL class plugins are
identified by the ``io.murano.extension`` stevedore namespace. Entry-point
names are the suffixes of the declared class' FQNs: the plugin loader forms the
actual FQN of the class by concatenating the stevedore namespace with entry
point name. For example, Murano's demo plugin [3] (providing Glance API
connectivity) declares the following stevedore-based extension:

::

 [entry_points]
 io.murano.extensions =
    mirantis.example.Glance = murano_exampleplugin:GlanceClient

The developer's provided entry-point name is ``mirantis.example.Glance`` here,
and the current plugin loader will concatenate it with the namespace name thus
resulting in class ``io.murano.extensions.mirantis.example.Glance`` being
loaded by the class loader.

It is proposed to get rid of the concatenation: the stevedore's namespace name
should be used **only** to identify the type of the plugin, while the FQNs of
its classes should be defined only by the names of the entry points. So, when
this change is implemented the example above will load a class with an FQN
``mirantis.example.Glance``, without the ``io.murano.extensions`` prefix.

After this change is implemented the developers of the plugin will be able to
follow the regular naming rules of Murano. So, our example plugin will have to
be renamed to ``com.example.mirantis.Glance`` etc - and it will not be part of
the ``io.murano`` space, as this plugin is just an example and is not supposed
to be part of the core murano.


Backwards compatibility
-----------------------

The proposed change will effectively rename all the classes defined in the
existing plugins by removing the ``io.murano.extensions`` prefix from their
names. These may (and will) break all the MuranoPL code which relies on these
plugins.

To prevent this there should be a fallback logic implemented for the
class loader. When a class being loaded by name is not found and the name
starts with the ``io.murano.extensions`` prefix, there should be a second
attempt to load it by a shortened name without a prefix. If such class is found
and is loaded, a warning message should be logged to indicate that the plugin
is outdated and has to be updated with appropriate naming schema.

note::
    This backwards-compatibility mode may be considered a way to deprecate the
    old plugin naming scheme. In several releases this fallback logic may be
    removed completely.


Alternatives
------------

We may postpone the change of the naming scheme till the next version of the
plugin system is introduced, which will some different format of plugins and
their content etc.

However postponing is bad, since the plugin development for murano is already
active, and having inconsistently named classes makes a bad example in the
community. Waiting for another cycle or two may greatly increase the number of
such bad examples.



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


Security impact
---------------

Because of the proposed change it will be now possible to define classes having
the names colliding with the Core Library and thus overriding the behavior of
the standard murano classes.

This should not be allowed unless the deployer explicitly allows that in the
configuration.


Deployer impact
---------------

Deployers will have to monitor the logs of Murano Engine for the Warning
messages produced by the loading of outdated plugins in the backwards
compatibility mode.

Developer impact
----------------

Plugin developers will have to update their plugin according the the correct
naming scheme.

Application developers using the plugins will have to rename the class usages
in their MuranoPL code to use the new names.


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
  ksnihyr


Work Items
----------

* Modify the plugin loader

* Add backwards compatibility mode to the class loader

* Update existing plugins to utilize the proper naming schema

* Update the documentation


Dependencies
============

None


Testing
=======

No new testing needed. Existing plugin-loading tests should be updated so no
warnings are displayed.


Documentation Impact
====================

Plugin development manual has to be updated.


References
==========

#. https://docs.oracle.com/javase/tutorial/java/package/namingpkgs.html
#. http://docs.openstack.org/developer/stevedore/
#. http://git.openstack.org/cgit/openstack/murano/tree/contrib/plugins/murano_exampleplugin/setup.cfg?h=stable/mitaka
