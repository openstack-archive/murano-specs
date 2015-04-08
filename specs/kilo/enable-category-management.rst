..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================================================
Provide opportunity to manage application categories
====================================================

https://blueprints.launchpad.net/murano/+spec/enable-category-management

Murano is an application catalog, where new applications can be easily added.
Those applications may belong to a category, that is not on predefined list.

Also, some categories may not be needed, and user can delete such kind
of categories.
All operations should be available only for admin users.

Problem description
===================

Category management should be provided during:

 * Packages uploading in dashboard;

 * Packages modifying in dashboard;

 * Packages uploading or modifying via command line;

Adding new category:

 * Category name may contain spaces and other special characters;

 * Category name limit is equal to 80 characters;

 * Only admin users can add new category;

Deleting category:

 * Category may be deleted only if no packages belong to this category;
 * Only admin users can delete category;


Proposed change
===============

Changes required to support this feature:

 * Add new panel named 'Categories' under 'Manage' section;
   It should be available only for admin users.
   This panel should represent table with categories.
   The table will contain 'name' column and 'assigned packages' column.
   "Add category" will be a table action and "Delete Category" will be
   row action.
   Delete button should be hidden for those categories connected to
   the packages.

 * Provide corresponding methods in the python-muranoclient;
   Category manager will be added to v1 module;

Alternatives
------------

Admin can edit a database to add or delete categories.

Data model impact
-----------------

None

REST API impact
---------------

**GET /catalog/categories**

The previous call /catalog/packages/categories is will still be valid to
support backward compatibility.

*Request*

+----------+----------------------------------+----------------------------------+
| Method   | URI                              | Description                      |
+==========+==================================+==================================+
| Get      | /catalog/categories              | Get list of existing categories  |
+----------+----------------------------------+----------------------------------+


*Response*

::

    {"categories": [
            "id": "3dd486b1e26f40ac8f35416b63f52042",
            "updated": "2014-12-26T13:57:04",
            "name": "Web",
            "created": "2014-12-26T13:57:04",
            "package_count": 0
            },
            {
            "id": "k67gf67654f095gf89hjj87y56g98965v",
            "updated": "2014-12-26T13:57:04",
            "name": "Databases",
            "created": "2014-12-26T13:57:04",
             "package_count": 0
            }]
    }

**GET /catalog/categories/<category_id>**



*Request*

+----------+-----------------------------------+----------------------------------+
| Method   | URI                               | Description                      |
+==========+===================================+==================================+
| Get      | /catalog/categories/<category_id> | Get category detail              |
+----------+-----------------------------------+----------------------------------+

*Parameters*

* `category_id` - category ID, required

*Response*

::

    {
        "id": "0420045dce7445fabae7e5e61fff9e2f",
        "updated": "2014-12-26T13:57:04",
        "packages": [
            "Apache HTTP Server",
            "Apache Tomcat",
            "PostgreSQL"
        ],
        "name": "Web",
        "created": "2014-12-26T13:57:04"

    }


+----------------+-----------------------------------------------------------+
| Code           | Description                                               |
+================+===========================================================+
| 200            | OK. Category retrieved successfully                       |
+----------------+-----------------------------------------------------------+
| 401            | User is not authorized to access this session             |
+----------------+-----------------------------------------------------------+
| 404            | Not found. Specified category doesn`t exist               |
+----------------+-----------------------------------------------------------+


**POST /catalog/categories**

+----------------------+------------+--------------------------------------------------------+
| Attribute            | Type       | Description                                            |
+======================+============+========================================================+
| name                 | string     | Category name                                          |
+----------------------+------------+--------------------------------------------------------+

*Request*

+----------+----------------------------------+----------------------------------+
| Method   | URI                              | Description                      |
+==========+==================================+==================================+
| POST     | /catalog/categories              | Create new category              |
+----------+----------------------------------+----------------------------------+

*Content-Type*
  application/json

*Example*
   {"name": "category_name"}

*Response*

::

    {
        "id": "ce373a477f211e187a55404a662f968",
        "name": "category_name",
        "created": "2013-11-30T03:23:42Z",
        "updated": "2013-11-30T03:23:44Z",
    }


+----------------+-----------------------------------------------------------+
| Code           | Description                                               |
+================+===========================================================+
| 200            | OK. Category created successfully                         |
+----------------+-----------------------------------------------------------+
| 401            | User is not authorized to access this session             |
+----------------+-----------------------------------------------------------+
| 403            | Forbidden. Category with specified name already exist     |
+----------------+-----------------------------------------------------------+



**DELETE /catalog/categories**

*Request*

+----------+-----------------------------------+-----------------------------------+
| Method   | URI                               | Description                       |
+==========+===================================+===================================+
| DELETE   | /catalog/categories/<category_id> | Delete category with specified id |
+----------+-----------------------------------+-----------------------------------+

*Parameters:*

* `category_id` - category ID, required

*Response*

+----------------+-----------------------------------------------------------+
| Code           | Description                                               |
+================+===========================================================+
| 200            | OK. Category deleted successfully                         |
+----------------+-----------------------------------------------------------+
| 401            | User is not authorized to access this session             |
+----------------+-----------------------------------------------------------+
| 404            | Not found. Specified category doesn`t exist               |
+----------------+-----------------------------------------------------------+
| 403            | Forbidden. Category with specified name is assigned to    |
|                | the package, presented in the catalog. Only empty         |
|                | categories can be removed                                 |
+----------------+-----------------------------------------------------------+


Versioning impact
-----------------

Murano dashboard will support only the version of the client, that includes
corresponding changes in the client. 'Categories' panel will not work
with the old murano version, but application catalog and package management
will work fine.

Other end user impact
---------------------

None

Murano-dashboard / Horizon impact
---------------------------------

Category management will be available in dashboard.
Areas to be changed: (described in sections above)

* Manage section will have new panel;


Deployer impact
---------------

None

Developer impact
----------------

None


Implementation
==============

Assignee(s)
-----------

Ekaterina Chernova

Primary assignee:
  <efedorova@mirantis.com>

Work Items
----------

* Introduce 2 additional calls in API

* Update API specification

* Provide these calls in python-muranoclient

* Implement changes in dashboard

* Enable CLI to manage categories

Dependencies
============

None

Testing
=======

New tests should be added in dashboard integration tests


Documentation Impact
====================

API specification should be updated.
All changes are already represented here, just need to copy.


References
==========

None
