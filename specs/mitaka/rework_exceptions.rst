..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=========================
Murano exception handling
=========================

https://blueprints.launchpad.net/murano/+spec/murano-exception-handling

Current exception handling in murano project differs from the way
all OpenStack projects handle their exception. Murano have very small list
of its own exceptions and use webob or general exceptions everywhere instead of
store all needed exceptions in one place.

Problem description
===================

* Murano is OpenStack project and should follow community practices.

* Using murano specific exceptions can make debugging easier.

* Now we have 3 different places where exceptions are stored: murano.packages.exceptions,
  murano.dsl.exceptions, murano.common.exception depends on part of murano which
  throws the exception.

Proposed change
===============

It's planned to create 2 new basic exceptions in murano.common.exceptions which
will be parents for others. First one will be general exception and the second
one will be exception for HTTP requests. Exceptions in murano.packages.exceptions
should have general MuranoException as a parent. Exceptions in murano.dsl.exceptions
should be kept as is since dsl is mostly separate part of murano.

These exceptions will be replaced with exceptions inherited from MuranoHTTPException:

* webob.exc.HTTPBadRequest
* webob.exc.HTTPInternalServerError
* webob.exc.HTTPForbidden
* webob.exc.HTTPUnsupportedMediaType
* webob.exc.HTTPUnauthorized
* webob.exc.HTTPForbidden
* webob.exc.HTTPConflict
* webob.exc.HTTPClientError

These exceptions will be replaced with exceptions inherited from MuranoException:


* ValueError
* Exception
* RuntimeError
* NotImplementedError
* SyntaxError
* TypeError
* NameError

All custom exceptions which defined in murano should be moved to one of the general
exceptions files.

Alternatives
------------

Keep everything as it is.

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

End users will see new murano specific exceptions.

Deployer impact
---------------

None

Developer impact
----------------

None

Murano-dashboard / Horizon impact
---------------------------------

New exceptions should be introduced and handled in horizon.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  starodubcena

Work Items
----------

* Create new exception in murano-api, change exception handling where it will be
  needed

* Create new exception in packages, change exception handling where it will be
  needed

* Handle new exceptions in python-muranoclient

* Handle new exception in murano-dashboard


Dependencies
============

None


Testing
=======

This change need a huge refactoring for unit and functional tests.

Documentation Impact
====================

None
