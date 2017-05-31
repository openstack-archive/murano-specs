..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============================
Encrypt Murano PL Properties
============================

https://blueprints.launchpad.net/murano/+spec/allow-encrypting-of-muranopl-properties

Currently the object model for an application in Murano is stored in the Murano
database as plain text. This can pose a security risk for applications whose
object model contains passwords or other sensitive data.


Problem description
===================

Many Murano applications request input data from users, which in most cases is
entered via the murano dashboard. Currently all input from users is stored in
database in plain text. In the case of a MySQL [#]_ app for example, the
password that the user enters will be stored in plain text in both the
``session`` and ``environment`` database tables. This info is available both to
the cloud admin, other users in the same project, and also to an attacker
should the database become compromised.


Proposed change
===============

This spec proposes to add a two new yaql functions; `encrypt_data` and
`decrypt_data`. The former will be processed at the dashboard stage, as it will
be applied to input fields presented to the user. `decrypt_data` will be
processed in the engine during app execution.

These functions will make use of Castellan [#]_ which is a generic
key manager interface for OpenStack. Castellan is designed to be used with a
variety of secret storage backends, the most common being Barbican [#]_. The
Castellan key manager is already being used by the Nova and Cinder project to
name two [#]_.  Because Barbican is the default storage backend, Barbican and
Castellan will be referred to interchangeably for the rest of this document.

Alternatives
------------

Some questions have been raised as to the usefulness of Barbican as a whole
[#]_. Some community members are concerned with adding it as "yet another
dependency" to their project, while others have concerns about its integration
with Keystone with regards to secret storage.

With regards to dependency concerns, these functions will be optional to Murano.
If Castellan is not configured, an error message will be shown in Horizon
informing the user to contact their administrator if they wish to use apps
requiring encryption. The use of Castellan also helps here, as it gives
operators flexibility in which secret backend they wish to use. It should be
noted however that Castellan does not appear to currently provide any "dummy"
backend drivers [#]_.

With regards to security, the argument that Barbican is the common solution for
secret storage in OpenStack seems reasonable; security is something best left
to specialists in the field.  If a flaw is discovered in Barbican, it can be
patched once by an operator and all projects using it will automatically
benefit.

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

Deployers wishing to use the new encryption functionality will be required to
deploy a key manager such as Barbican (it is worth noting Barbican may already
be available in some cloud environments in which case there would be minimal
impact).

The functionality will be made optional via a configuration item, hence
there will be no impact for deployers who don't wish to take advantage of this
feature.

Developer impact
----------------

Developers should be made aware of these functions via documentation, and also
how to use them.

Murano-dashboard / Horizon impact
---------------------------------

Will need to be updated to understand calls to `encrypt_data` in forms.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  paul.bourke@oracle.com

Other contributors:
  None

Work Items
----------

* Integrate Castellan into murano-dashboard.

* Add a `encrypt_data` function to the dashboard to make use of the above.

* Integrate Castellan into the murano engine.

* Add a `decrypt_data` function to the dashboard to make use of the above.

* Testing.


Dependencies
============

* Castellan

* Barbican


Testing
=======

* Appropriate unit and functional tests will need to be added.

* Potentially the tempest tests could be updated for full end-to-end testing
  with Barbican.


Documentation Impact
====================

Deployment docs will need info on Barbican configuration for Murano and
murano-dashboard. MuranoPL Docs will also be needed on the new functions.


References
==========

.. [#] https://github.com/openstack/murano-apps/blob/master/MySQL/package/UI/ui.yaml
.. [#] https://github.com/openstack/castellan
.. [#] https://github.com/openstack/barbican
.. [#] https://review.openstack.org/#/c/247561/
.. [#] http://lists.openstack.org/pipermail/openstack-dev/2017-January/110192.html
.. [#] https://github.com/openstack/castellan/tree/stable/ocata/castellan/key_manager

* Cinder discussion around their alternative insecure key manager for Castellan:
  http://lists.openstack.org/pipermail/openstack-dev/2016-January/083241.html

* Murano discussion around implementation details for this blueprint:
  http://lists.openstack.org/pipermail/openstack-dev/2017-June/118290.html
