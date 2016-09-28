..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===================================
Metadata Assignment And Propagation
===================================

https://blueprints.launchpad.net/murano/+spec/metadata-assignment-and-propagation

Cloud users and operators may need to assign some arbitrary key-value pairs to
environments and applications deployed by murano to associate some metadata
information with them. If applicable this metadata attributes should be
propagated to OpenStack resources provisioned by Murano.


Problem description
===================

The meta information may describe various aspects of the deployed applications,
for example:

    * Deployed software: names, versions, descriptions and license terms of the
      software components deployed on the VMs;

    * VM-related configuration features, like listening ports or interfaces;

    * Hardware specs of provisioned virtual machines;

    * Guest OS info, like the type of operating system, its family, version
      etc;

    * Information about the deployments purpose: type of the deployed
      environment (development, staging, production), usage purpose, security
      consideration etc;

    * Information about the user who deployed the app or environment: name,
      department, contact information, etc.

This information may be consumed directly by inspecting murano applications
with dashboard or CLI, by indexing these metadata attributes into specialized
search software, such as Searchlight [1] or by building custom reports with
grouping or aggregation by these metadata.

For the latter case it is especially important to propagate the metadata from
the initial objects it is assigned to (application or environment) to the
resource objects spawned by them (VMs, Volumes, Networks etc).


Proposed change
===============

UI change
---------

It is proposed to utilize the functionality provided by Glance Meta Definition
Catalog [2] to define the schemas of the applicable metadata attributes. It is
proposed to add two types of resources for murano applications and murano
environments: ``OS::Murano::Application`` and ``OS::Murano::Environment``
accordingly (ability to add metadata attributes for murano packages is out of
scope for this spec, however in future this may also become possible, so
``OS::Murano::Package`` is likely to be appropriate resource type).

The standard Angular-based Metadata assignment dialog (the one being currently
used to assign metadata to Instances, Images and Volumes) should be extended to
support Murano Dashboard and these new resource types. This extended dialog
will load the metadata definition schemas for attributes which are assignable
to appropriate type of resources and will submit them to Murano API for
persistence once the input is saved.


Murano Object Model change
--------------------------

For the applications it is proposed to persist the metadata attributes in the
object model, in the new block called 'metadata', which should be placed under
the '?' section of each object. For example, the '?' block of Apache
application tagged with a couple of attributes may look like this:

::

    "?": {
         "_26411a1861294160833743e45d0eaad9":  {
  	         "name": "Apache HTTP Server"
	 	  },
	 	 "type": "com.example.apache.ApacheHttpServer",
	 	 "id": "19dfb844-6eee-48bd-826a-c6206d065e85",
	 	 "metadata": {
	 	     "sw_webserver_apache_version": "2.1",
	 	     "sw_webserver_apache_http_port": 80
	 	  }
	  }

Since the ?-headers of objects placed in the environment's object model may be
modified with the existing Murano API no changes in the API is needed.

For the Environments a similar approach may be followed — to store the
metadata attributes under the `/?/metadata` key of the Environment object in
the object model. However an alternative may be considered: since each
Environments have a dedicated entry in murano's database, the metadata
attributes for environments may be stored in database in normalized form. See
the "Alternatives" section below for more details.

Regardless of the chosen approach it is required to introduce a new API call
to assign metadata attributes to the Environment, since the existing API does
not have capabilities to edit the ?-headers of Environment object. The
appropriate change into API is out of scope here and is covered with a separate
spec [3]


MuranoPL Change
---------------

MuranoPL language should be extended with an ability to read the metadata
attributes assigned to any given object. It is proposed to create a global yaql
function named ``metadata()`` for this purpose (similar to ``id()`` and
``name()`` which read appropriate information from the same ?-header).

Murano's serializer/deserializer should be properly updated to load the
metadata attributes and persist them back.


Core Library Change
-------------------

Core library should be extended with an ability to communicate with Glance
Metadata Definition API to query the namespaces applicable to a given resource
type and extract all the names of attributes from those namespaces. This will
allow the resource class to check if the given metadata attribute attached to
the application the class belongs to is applicable for the resource itself. The
binding to the Glance API should be done via a python-backed class
``MetadefBrowser``, while the rest of the logic may be implemented in MuranoPL.

A new MuranoPL class called ``MetadataAware`` should be implemented. This class
will have two methods: ``collectMetadata`` and ``getResourceType``.
``collectMetadata`` method will return all the metadata attributes assigned to
the object itself and all the attributes assigned to the owners of the object,
traversing the ownership tree to the root (i.e. to the Environment). All the
attributes from the traversed objects may be checked for applicability to the
particular resource type. To do this check, the code should call the
``getResourceType`` method which is supposed to return the resource type as
defined in Glance (``OS::Nova::Server`` for VMs, ``OS::Cinder::Volume`` for
volumes etc) and then use the ``MetadefBrowser`` described above to check if
the attribute is applicable to the objects of this type.

The resource-specific classes of the Core Library (``Instance``,
``CinderVolume`` etc) have to extend the ``MetadataAware`` and override the
``getResourceType`` method.


Alternatives
------------

There are a number of alternatives to consider.

First, instead of using Glance Metadef catalog to store the possible attribute
names and constraints, we may assign attributes without any predefined schema.
This will reduce our dependency on other OpenStack components (Glance in this
case), but will introduce extra development burden (since we won't be able to
reuse the existing metadata-assignment dialog) and will prevent us from using
convenient schema-based validators of tags which are made for metadefs. So, it
is proposed to stay with metadef-compliant implementation.

Second option to consider is the storage of metadata attributes on environment
level. Most other OpenStack projects store metadata attributes in normalized
form: they have the separate table mapped to the primary object table as
one-to-many. The approach suggested above uses denormalized storage: the
attributes are stored inside the json-document describing the complete
environment, which is stored as a single value in the database. This is much
easier to implement, also having metadata in object model of environment object
is still needed, so duplicating data storage seems like an extra work.

Third option to consider: not to check for attribute applicability on resource
types. This simplifies the development (no need to create ``MetadefBrowser``
class and much simpler logic in ``MetadataAware``), however this will degrade
user experience, as resources may get inappropriate attributes assigned, which
will be confusing.


Data model impact
-----------------

This proposal adds new fields to the ?-headers of objects in object model. If
the normalized alternative is chosen, it will also introduce extra table in
database and a relationship with the ``Environments`` table. This will require
to create a database migration as well.

REST API impact
---------------

A modification of Murano API is required to allow assigning of metadata
attributes to the Environment objects, however it is out of scope here. See [3]
for details.


Versioning impact
-----------------

Since the proposed change requires to add a new yaql function to the DSL and
modifies the serializer/deserializer logic this requires to bump the MuranoPL
format version.

Other end user impact
---------------------

The users will see new action buttons "Update Metadata" in Murano Dashboard.
This buttons will be available as row actions for Environment Components and
Environments.

Deployer impact
---------------

The proposed change introduces a dependency on Glance API. To configure the
connectivity to this API (endpoint details, encryption etc) a new optional
configuration block should be added to Murano config.

Developer impact
----------------

Developers may implement the ``MetadataAware`` class to add metadata handling
logic for their classes.

Murano-dashboard / Horizon impact
---------------------------------

This change will add a custom version of metadata.service javascript module to
Murano Dashboard. It will have the similar logic as the standard Horizon's
module, but will add the references to murano api's, so the Metadata dialog can
update the attributes assigned to Murano entities.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  ativelkov

Other contributors:
  TBD

Work Items
----------

* Add a rest api handlers for murano-dashboard to handle metadata updates and
  fetching;

* Add a javascript API to this handlers;

* Create a custom version of metadata.service, so the metadata modal dialog may
  utilize murano's javascript APIs to metadata;

* Modify Murano Dashboard to call metadata modal dialog as row handlers for
  Environments and Services;

* Modify Murano API to allow edits of ?-section of Environment objects;

* Add a ``MetadefBrowser`` class to Core Library;

* Implement a ``MetadataAware`` class and extend ``Instance`` and
  ``CinderVolume`` classes with it.


Dependencies
============

* Glance Metadefinition Catalog [2]

* Environment modification API changes [3]

Testing
=======

The integration tests should verify that the attributes are propagated properly

Documentation Impact
====================

The new capabilities to add attributes should be documented in user manual.
The changes in core library and the usage of ``MetadataAware`` class should be
reflected in developers' manual.


References
==========

[1] https://wiki.openstack.org/wiki/Searchlight
[2] http://docs.openstack.org/developer/glance/glancemetadefcatalogapi.html
[3] https://review.openstack.org/#/c/378602
