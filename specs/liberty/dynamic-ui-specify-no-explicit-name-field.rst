..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode


============================================================
Remove name field from fields and object model in dynamic UI
============================================================

https://blueprints.launchpad.net/murano/+spec/dynamic-ui-specify-no-explicit-name-field

Now name is a required parameter in every single form definition.
But murano-engine doesn't know anything about this parameter. It mostly used
for dymanic UI purposes. So we can insert this field automatically in every
form.


Problem description
===================

* 'name' property has no built-in predefined meaning for MuranoPL classes
  or applications.

  Now all unknown parameters are ignored, but the error will be spawned and
  all applications will be invalid in the near future.
  To prevent global failure this change is suggested.


Proposed change
===============

Add automatic field inserting into the first form and store 'name' field value
in object header:

* Dynamic UI version will be increased to 2.1;

* 'name' property is not required in MuranoPL class definition anymore.
  If user still have 'name' property in class definition, he should supply
  corresponding field in UI definition. It will have no special meaning for
  murano dashboard.

* New field will be inserted to the first form if Dymanic UI version is higher
  or equal to 2.1;

* If Dymanic UI version is higher or equal to 2.1 'name' field value will be
  placed to the object header ('?'), in "done" method of application creation
  wizard;



* In get-mode, application 'name' parameter will be checked in:

  * Primary under '?' parameter;
  * Secondary in the object model root;

* Add new YAQL function to dashboard's YAQL to be used by dynamic UI.
  The function will be called 'name' and will allow to use
  automatically-inserted name field in object model description.


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
-----------------

This change introduces new version of dynamic UI form definition.
Only backward compatibility will be supported: new dashboard can work with old engine.

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

This spec is supposed to be implemented in murano-dashboard only.
User will see new field for adding app name.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  <efedorova@mirantis.com>


Work Items
----------

* Implement automatic 'name' field insertion if version satisfies the requirements;
* Put 'name' field value to the object header area;
* Use 'name' attribute from object header or inside object modal root
  in environments table;
* Implement new YAQL function, which returns applications name.


Dependencies
============

This change is a dependency for a future changes in engine


Testing
=======

CI tests should be updated and catch all errors.

Documentation Impact
====================

Dynamic UI form definition examples should be updated


References
==========

None
