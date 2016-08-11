..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===============================================
Validation tool for Murano Application Packages
===============================================

https://blueprints.launchpad.net/murano/+spec/murano-app-validation

Murano doesn't have validation mechanisms for MuranoPL language and
application packages.

This spec proposes a validation toolkit that improves workflow
by making error detection possible at the early stages of application
development lifecycle.

Problem description
===================

Early error detection simplifies application development,
troubleshooting and support.

Currently Murano does not provide mechanisms to perform validation of the
Murano application package before running deployment process.
Thus Murano cannot prevent uploading of invalid package and developer
cannot verify that application package contains valid structure
and code before deployment process.

Many problems with application packages (such as invalid package structure,
errors in code, typos, etc.) can be detected and solved by using
static validation tool.

Proposed change
===============

Introduce a new toolkit for validation MuranoPL language and Murano application
packages.

* Validation toolkit SHOULD be implemented as a standalone Python package
  (library) with CLI interface.
* Validation library will be used to validate Murano packages during loading
  process.
* CLI tool can be used by application developer to perform early checks before
  package is assembled.
* Toolkit should be extensible with custom validators. It will provide API for
  extension purposes.
* Toolkit should be configurable to run list of specified inspections
  or to ignore specified inspections.

Toolkit should implement the following validation suites:
* Manifest validation
* Classes validation
* UI definition validation
Other validators will be implemented later as part of this tool or as external
packages

Manifest validation
-------------------

* ``manifest.yaml`` file should exist and should be a valid YAML document.
* ``manifest.yaml`` structure should respond to manifest structure described in
  :ref:`manifest-structure-section`.
* All files referenced in ``Classes`` section should exist in ``Classes``
  subdirectory and should be valid YAML documents.
* Warning should be reported in case when additional files exist in ``Classes``
  subdirectory but not referenced in the manifest file.
* UI definition file should exist for Application package (MuranoPL =< 1.4)


Class validation
----------------

* Valid classes and methods should check if properties are defined
  correctly and there are no extra keys that are not supported.
* Check if class is in correct namespaces according to manifest and
  class name matches.
* Contract fields should be valid YAQL expressions.
* Methods blocks should be checked for MuranoPL code structure correctness.
  Method blocks may consist only of Expressions, Assignments and Block
  constructs. See `full terms description <http://docs.openstack.org/developer/murano/draft/appdev-guide/murano_pl.html#method-body>`_
  for details.
* (Optional) Detect code that cannot be parsed as YAQL expression but is
  considered to be a valid expression.


UI validation
-------------

* UI definition should contain ``Application`` and ``Forms`` sections
* YAQL expressions in UI definition should be valid
* ``Form`` section should contain only valid forms
  `<http://docs.openstack.org/developer/murano/draft/appdev-guide/murano_packages.html#forms>`_


Errors
------

Error code consists of prefix and error number. Prefix is an upper case
alphabetical sequence. It can be empty for general errors. Errors specific for
some validator should use validator short name as prefix. Error number is an
upper case alphanumerical sequence (one single letter ``E`` or ``W``
following by three digits). Examples: ``E007``, ``MPL:E042`` for errors and
``W777``,  ``UI:W123`` for warnings. Validation tool should have ability to
produce documentation of supported errors and warnings.


Package structure reference
---------------------------

http://docs.openstack.org/developer/murano/draft/appdev-guide/murano_packages.html#package-structure

Package consists of:

* Manifest - File in package root directory named ``manifest.yaml``.
* MuranoPL classes - YAML files located in ``Classes`` directory or its
  subdirectories (optional).
* UI definition - YAML file in ``UI`` directory (optional).
* Resources - files in ``Resources`` directory (optional).
* Logo (optional)
* images.lst file (optional)

.. _manifest-structure-section:

Manifest structure
^^^^^^^^^^^^^^^^^^

**Format**

Application package format. Can be specified as ``<FORMAT>/<VERSION>``,
where format is a format name string (can be ``MuranoPL`` or ``Heat.HOT``).
If format is omitted, ``MuranoPL`` is used by default.

* Required: no
* Default: ``MuranoPL/1.0``
* Format: ``<FORMAT>/<VERSION>``

**Type**

* Required: yes
* Type: Enum(Application, Library)

**Name**

Package human readable name

* Required: no
* Type: String

**FullName**

Package fully qualified name. Should be in lowerCamelCase notation
separated by dots.

* Required: yes
* Type: String

**Version**

Version of package

* Required: no
* Default: ``0.0.0``
* Type: String, should have valid `SemVer format <http://semver.org/>`_

**Description**

* Required: no
* Type: String

**Author**

* Required: no
* Type: String

**Tags**

* Required: no
* Type: List(String)

**Classes**

* Required: no
* Type: Dict

  - Keys: String, Fully qualified class name
  - Values: String, Filename relative to ``Classes`` directory

**Require**

Package external requirements.

* Required: no
* Type: Dict

  - Keys: Dependent package ``FullName``
  - Values: ``null`` or string with exact version ('1', '2.3', '4.5.6') or
     valid spec_strings according to `python-semanticversion docs <https://python-semanticversion.readthedocs.io/en/latest/reference.html#semantic_version.SpecItem>`_

**UI**

File with package UI definition

* Required: yes (for package type ``Application``)
* Type: String, Filename relative to root directory
* Default: ``logo.png``

**Logo**

Package logo in png format.

* Required: no
* Type: String, Filename relative to ``UI`` directory
* Default: ``ui.yaml``

**Meta**

* Required: no
* Type: Dict or List(Dict) with metadata information

Class structure
^^^^^^^^^^^^^^^

**Name**

* Required: yes, if more than one class defined in the file
* Type: String
* Format:

  - Matches Python class name format, except leading
    double underscore (``__ClassName``).
  - Matches CamelCase naming style.

**Properties**

* Required: no
* Type: Dict

For more details see :ref:`class-properties-section`.

**Methods**

* Required: no
* Type: Dict

For more details see :ref:`class-methods-section`.

**Extends**

Single or multiple class names.

* Required: no
* Type: String | YAML List

**Namespaces**

* Required: no
* Type: Dict
* Keys: String; Namespace alias
* Values: String; Namespace full qualified name
* Format:

  - Namespace alias is a valid YAQL identifier or ``=``.
  - Namespace FQN is a string divided by dots (``.``).
  - Recommendation is to use lowerCamelCase naming style.

**Import**

* Required: no
* Type: String, Class name, List(Class names)

**Usage**

* Required: no
* Type: Enum(Class, Meta)
* Default: ``Class``

**Meta**

* Required: no
* Type: Dict or List(Dict) with metadata information

MetaClass structure
^^^^^^^^^^^^^^^^^^^

In addition to regular Class structure, MetaClass (Class with Usage: Meta) can
have additional properties:

**Cardinality**

* Required: no
* Type: Enum(One, Many)

**Applies**

* Required: no
* Type: Enum(Package, Type, Method, Property, Argument, All) or List(Enum)

**Inherited**

* Required: no
* Type: boolean

.. _class-properties-section:

Class properties
^^^^^^^^^^^^^^^^

**Contract**

* Required: yes
* Type: YAQL Expression

**Usage**

* Required: no
* Type: Enum

========= ====================
Usage
========= ====================
In        Default
Out
InOut
Const
Runtime
Config    Since version 1.1
Static    Since version 1.2
========= ====================

**Default**

* Required: no
* Type: Any value

**Meta**

* Required: no
* Type: Dict or List(Dict) with metadata information

.. _class-methods-section:

Class methods
^^^^^^^^^^^^^

**Scope**

.. versionadded:: 1.4

* Required: no
* Type: Enum
* Values: Public, Session

**Arguments**

* Required: no
* Type: List(Dict)

For more details see :ref:`methods-argument-section`.

**Body**

* Required:  no
* Type: YAQL Expression | List(YAQL Expression)

**Usage**

* Required: no
* Type: Enum

========= ====================
Usage
========= ====================
Runtime   Default
Static    Since 1.3
Extension Since 1.3
Action    Deprecated since 1.4
========= ====================

**Meta**

* Required: no
* Type: Dict or List(Dict) with metadata information

.. _methods-argument-section:

Method's Argument
^^^^^^^^^^^^^^^^^

* Type: Dict(String, Dict)
* Key: Argument name

**Values:**

* Contract: YAQL Expression
* Default: YAQL Expression
* Usage: Enum added since 1.4

========= ====================
Usage
========= ====================
Standard  Default
VarArgs
KwArgs
========= ====================

**Meta**

* Required: no
* Type: List with metadata information

Extensions
----------

Validation tool should be extensible and provide unified mechanism for builtin
checks and extensions.

Extensions should be implemented as a python package and registered using
entry points.

Extensions are loaded using
`stevedore <http://docs.openstack.org/developer/stevedore/>`_ library.

CLI Usage
---------

::

  Usage: mpl-check [options] <PACKAGE | DIRECTORY>

  Arguments:
      DIRECTORY      Application working directory (e.g. contains
                          manifest.yaml, Classes/, etc.)
      PACKAGE        Compressed application (ZIP-archive)

  Options:
  --ignore=errors    Skip errors and warnings.
  --select=errors    Check only for selected errors and warnings.
  --scope=validators List of validators to execute
  --strict           Consider warnings as an errors.

Alternatives
------------

Implement validation as part of Murano and validate the package
when it is uploaded to Murano.

Pros:

  - Easier to implement.

Cons:

  - Suitable for Murano itself, but not for developers.
  - Integration with CI which requires execution of functional
    tests and is slower than static validation.
  - Cannot be integrated with text editors or IDEs.

Data model impact
-----------------

None

REST API impact
---------------

Murano API for upload packages should use this tool to do package validation
before store it. If package is not valid, HTTP error with code 400 should be
returned to user with list of check errors in description.


Versioning impact
-----------------

None

Other end user impact
---------------------

* Murano application package developer will be able to validate package
  during development process.

Deployer impact
---------------

None

Developer impact
----------------

Application developer will be able to validate application code
during development process without importing package to Murano
and running deployment process or test suite.

Murano-dashboard / Horizon impact
---------------------------------

Error message is displayed in Horizon when trying to upload
malformed package.

Implementation
==============

Application class
-----------------

Application class provides an entry point to the checker.
It is responsible for configuration, extensions loading and validation
execution.

Application options are registered using oslo.config

**Methods:**

* __init__(self, options)
* ``load_plugins(self)`` - Finds validators from installed plugins
* ``prepare_check_suit(self, loaded_package, validator_list)`` - Collect
  prepared checks from all validators in validator_list against specified
  package. Returns list with prepared checks.
* ``list_errors(self, filters)``- Lists errors/warnings exposed by all available
  validators

Validator classes
-----------------

Validator class represents set for specific checks and logic to register them.

**Classes**

::

  BaseValidator
    +-- PackageValidator
    +-- ManifestValidator
    +-- UiValidator
    +-- ClassValidator

**Attributes**

* ``short_name`` short name for validator. Used as prefix for validator specific
  errors

**Methods**

* ``__init__(loaded_package)`` - Init validator for specific format
* ``run(self)`` - Runs validator against specified suit of files or
  whole package if file_suit=None (for Package validator)

Checkers
--------

Checker is a function that performs checking logic. It takes data from args
and yields error if it is not valid. Checkers should not contain logic for
data preparation. Checker can be reused multiple times with different options
in a single suit of checks. Checker can be wrapped with custom filters
(i.e. manifest version), so it is called only for matching conditions.

Exceptions
----------

Class ``CheckerError`` contains information including error code, message,
file name and position in that file. Additionally it can include
code snippet, if it's available. For YAML data it's available using
``yaml.error.Mark`` class. Otherwise code snippet can be generated during
error formatting.

**Attributes**

* code - Error code (Usually single letter and three digits)
* message - Human readable error reason
* filename - File name relative to package root
* line - Line number (starts from 0, optional)
* column - Column number (starts from 0, optional)
* source - One line code snippet with highlighted column (optional)

Package loaders
---------------

Package loaders provide interface to Murano packages in multiple formats.
Currently packages can be loaded from directory, single file or zip archive
package.

::

  PackageLoader
   +-- DirectoryLoader
   +-- FileLoader
   +-- ZipLoader

**Methods**

* ``try_load(cls, path, *options)`` - [classmethod] Tries to load package, if it
  can be loaded returns instance of loaded package, otherwise None.

Loaded package
^^^^^^^^^^^^^^

Loaded package class introduces a general interface for access to packages of
different format.

**Methods**

* ``get_file(self, name)`` - File wrapper if file exists, None otherwise
* ``format(self)`` - Ru
* ``exists(self, name)`` - Returns true if file exists in the package.
* ``list(self, subdir=None)`` - Returns list of all files in the package
  (or in it's subdirectory).
* ``search_for(self, regex)`` - Return iterator of files fit to regex

File wrapper allows to access to raw file context with method ``raw()`` and
to parsed dict with method ``parsed()``

Formatters
----------

Formatter classes help to represent error report in various ways.
Errors can be printed to ``stdout``/``stderr`` using text representation,
which suits developer or written to file in ``JSON``/``YAML`` format.


Assignee(s)
-----------

Primary assignee:
  Alexander Saprykin <cutwater>

Other contributors:
  Krzysztof Szukielojc <kszukielojc>
  Sergey Slipushenko <sslypushenko>

Work Items
----------

#. Develop core framework with support of extensions.
#. Implement manifest and package structure checks.
#. Implement Murano PL classes checks.
#. Implement UI definition file validation.
#. Implement YAQL checks.
#. Integrate package validation with murano API.
#. Develop CLI tool based on core framework
#. Create documentation.

Dependencies
============

* `PyYAML <http://pyyaml.org/>`_
* `stevedore <http://docs.openstack.org/developer/stevedore/>`_

Testing
=======

Package should include unit and functional tests.

Documentation Impact
====================

Documentation for the library should include:

* Usage information that describes how to use the application.
* API reference.
* Plugin developer documentation.

References
==========

None
