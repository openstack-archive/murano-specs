..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==============
MuranoPL forms
==============

https://blueprints.launchpad.net/murano/+spec/muranopl-forms

Murano has a YAML-based DSL to describe the UI form required by the
application. However it is completely independent from the application itself.
So the changes made to the application properties doesn't affect the form.
This specification is aimed to make UI definition be driven by the application
code and metadata rather than by a separate language.


Problem description
===================

Current UI definition format and the way Murano handles it comes with a bunch
of problems that are there by design and cannot be solved without making some
significant changes. Most notable problems are:

* It is monolithic. UI form defines all the inputs required not just by the
  application but for all its object model subtree except for the other
  application references. Thus it is impossible to have UI forms for
  individual components. Neither it is possible for a user to choose the
  desired implementation of those components without turning them into
  applications (and putting in a separate packages).

* It contains yet another yaql-based DSL that defines the mapping between
  input fields and application (and inner component) properties. However
  this mapping is not bound to the property contracts. So any change made to
  the properties or their contracts will not affect the form but only its
  validation that happens upon deployment after the form is no longer visible
  to the user.

* Because the input fields do not directly correlate to application properties
  it is not possible to get the reverse mappings from object model to the
  input fields. Because of that once the form is closed it is impossible to
  edit entered values because they already transformed to some other structure
  that corresponds to application's object model and we don't have any UI
  definition for that structure.

* It also means that each property needs to be described in several places in
  different syntax thus duplicating the work. It is also easy to fail or get
  out of sync with the code if one changes the contract but forgets to update
  the UI definition or vice versa.

* It is one more DSL to learn. One more DSL invented in Murano and not covered
  by any standard.

* Its object model template part (yaql mappings) use functions that are not
  available in MuranoPL and vice versa. And its syntax has its own number of
  problems like inability to define an object and reference it from several
  places.

* It requires the client be aware of several OpenStack entities like images,
  flavors etc. For example murano-dashboard talks to nova to get the list of
  available flavors. This brings two additional problems:

  a. It cannot be used for application that is not targeting OpenStack or
     targeting OpenStack cloud other than the one used to run the dashboard.

  b. It is not extensible. Each new selector requires special type in UI
     definition that must be know to the client.

* Applications cannot dynamically control content of the form. For example it
  is not possible to populate values for a drop-down list if those values are
  not constants that are known in advance.

* Neither murano-dashboard nor the UI definition syntax support non-scalar
  properties (dictionaries, lists etc.). Thus it is easy to come up with a
  class design that is very useful but yet cannot be shown in UI.

* There can be only one UI definition form per package.

Proposed change
===============

Solution overview
-----------------

Proposed solution is to implement two major paradigm changes:

* Switch from the current UI definition format to an auto generated form
  definitions that are not supposed to be written manually. Form generator is
  going to accept MuranoPL class as an input and produce json-schema based
  output that has all the required information to both present the form to a
  user and do a client-side validation of the input.

* Make that UI schemas be per-class. There will be a new API call to request
  UI definition for particular class or even particular method of that class
  rather than for a package.

As a result the UI workflow to add new application is going to be like this:

#. User wants to add application x.y.z identified by the application package
   FQN.

#. Client asks for UI definition for that package using existing API call
   (the one that returns UI definition in existing "old" format).
   If the call is successful and UI definition was obtained then the rest of
   workflow is remained as it is now.

#. Otherwise the client (dashboard) requests UI schema for the class x.y.z
   using new API call.

#. API sends murano-engine request to generate UI schema for the class x.y.z.

#. Engine loads class definition from appropriate package including attached
   meta-values.

#. Taking class property declarations (contracts) and attached meta-values
   as an input, engine generates json-schema that has everything needed to
   present the UI form and do most of the client-side validation.

#. The same is done for all model builder methods (see below) using the same
   algorithm but using their arguments instead of class properties as an input.

#. List of schemas is sent back to the API and, in turn, returned to the
   caller (client).

#. The client decides which of the schemas to use (usually by asking the user).
   It can always go with default schema for the class or take advantage of
   one of the schemas provided by model builder methods.

#. Using the extended json schema client generates UI form. The form is going
   to have 1-1 mapping between input controls and class properties or method
   arguments.

#. After the form was filled it is validated on the client side using given
   json schema (and its optional murano-specific extensions) and then submitted
   to the API.

#. If the client opted to go with one of the model builder schemas the form
   output is sent to the engine in attempt to call the model builder method
   using the form values as an argument values. Model builder will then
   return modified object model that is sent back to the client (via API).
   Then the client can continue with the form edit using default schema or
   apply additional model builder method.

#. Otherwise the constructed object model is inserted into environment
   definition into API's database.

To change the application settings later, the same workflow is applied. The
only difference is that it doesn't try to use the old UI definition approach
but instead immediately requests new UI schema for the type specified in the
object model.


Schema generation
-----------------

Schema generation is a special service provided by the API (through the RPC
call to murano-engine) that takes a class FQN (including version or version
spec) and optional method name and produces json-schema compliant JSON. The
schema may have extra attributes for information that cannot be expressed by
standard json-schema attributes alone (for example field order).

If no method was provided the generation code takes class declaration and tries
to produce json-schema records by running YAQL expression contracts for its
properties in a special yaql context where all contract functions are
redefined to produce json-schema rather than validate the input. Thus it
creates a different implementation of the same contract syntax. In addition
the generation code may take into account number of well-known Meta classes
from the core library (to be added) and alter the schema if corresponding Meta
is attached to the properties. Meta values can specify things that cannot be
taken from contracts alone. For example it can be a property description text.

For the check() contract that accepts arbitrary yaql expression the analysis
of the expression AST is performed:

#. If the expression matches EXPR1 and EXPR2 and ... and EXPRN pattern it is
   split into several validations (as if it was written as
   ``check(EXPR1).check(EXPR2)...check(EXPRN))``.

#. Each of (sub)expressions is checked against supported patterns that can
   be translated to json-schema:
   * comparison of len($) for a string len
   * regex match function
   * number comparison
   * `in` operator that can be converted to enum

All expressions that cannot be translated with this algorithm are ignored and
the value will not be validated on the client side.

If the property/argument has a `Default` specifier it is translated to
`default` schema property.

For the class() the generated schema type is going to be `muranoObject`
with the following attributes:

   * `muranoType` - FQN of the base class;
   * `version` - version or version-spec that should be used to obtain
     `muranoType` (as seen in the manifest);
   * `owned` which can be `true`, `false` or null (or missing which means the
     same as null). `true` means that the object must be owned by the parent
     (thus ID of existing object in object model cannot be provided here),
     `false` means that only existing object can be referenced and `null`
     means that both options are valid.

Custom json-schema type is needed in order to retain reference semantics that
is it is an object which real type needs to be looked up in the catalog rather
than some plain dictionary or string. Client must understand `muranoObject`
type and know how to get list of valid type inheritors.

When `check()` contract is used to validate MuranoObject value (i.e. the
result of `class()` contract) it may put some constraints on that object's
properties or properties of some inner objects. In this case the schema
generator can emit additional `context` attribute to the property schema.
`context` is set to json-schema for the nested object (or its children).
Upon the input of a referenced object the client should check it against
all the `context` schemas up in the object model tree.

By default the generation algorithm produces the schema for the class and
for each model builder method available.

If the method name was provided to the engine command the same algorithm is
applied to that method only and its arguments are used where the class
properties would be used otherwise. In this case methods can be any methods
that can be invoked by the API which currently are actions and model builder
methods.


Model builders
--------------

Model builders are special MuranoPL methods that take a class definition
(in an object model format dictionary form) and number of optional arguments
and return modified object model.

Model builders are used to simplify object model generation using the template
obtained from its parameters. When generating json schema from such methods
their first parameter (which is current object model) is skipped.

In order for a method to be considered a model builder it must have the
following properties:

#. It must be static (`Usage: Static`)

#. In must have the public scope (`Scope: Public`. I.e. it must be a static
   action. See https://blueprints.launchpad.net/murano/+spec/static-actions for
   more information on static actions)

#. It must have `io.murano.metadata.ModelBuilder` `Meta` applied to it.
   This is a marker class that is going to be introduced to the core library
   to distinguish model builders from other static actions.

Caller uses static actions API to invoke the builder and obtain a generated
object model snippet.

UI hints
--------

In addition to the information that can be obtained from contracts some
additional information is needed to produce correct representation for
the property or argument. This information is provided by meta-classes
that need to be introduced to the core library:

`io.murano.metadata.Title`: title of an entity. Can be applied to anything.
The value is in `text` property of a meta-class. Upon schema generation
it is translated to `title` schema key. If no meta is attached then the
property/argument name is used as a title.

`io.murano.metadata.Description`: description of an entity. Can be applied to
anything. The value is in `text` property of the meta-class. Upon schema
generation it is translated to `description` schema key.

`io.murano.metadata.HelpText`: help text of an entity. Can be applied to
anything. The value is in `text` property of the meta-class. Upon schema
generation it is translated to `helpText` schema key.

`io.murano.metadata.forms.Hidden`: marks property or argument to be invisible.
Upon schema generation `"visible": false` is produced.

`io.murano.metadata.Position`: position of the property/argument within the
form. It has two properties:

 * `index`: integer by which all of the fields are sorted before rendering.
   it doesn't have to be consecutive. If the inherited field has the same
   index as the field from the generated class then inherited one goes first.
   For this matter property indexes might be re-enumerated upon schema
   generation to the consecutive unique indexes. This property is translated
   to `formIndex` schema key. If no position specified then the field will be
   placed in the list of unordered fields (probably sorted by their title).

 * `section': section name for the field. If not provided then it will be
   automatically placed in default section for all such fields.
   Section name is represented as `formSection` key in the schema of each
   field. Additional attributes for the section with that name can be found
   in the root schema for the type/method.

`io.murano.metadata.forms.Section`: specifies form sections for the class.
Can be multiple times applied either to the class or to the method.
It has the following properties:

  * `name`: name of the section that is going to be used in
    `io.murano.metadata.Position` instances.

  * `title`: title of the section. In no title provided it is assumed to be
     equal to the section mame.

  * `index`: index of the section in a section list (e.g. tab number).
    Similar to `Position` indexes those numbers doesn't have to be consecutive
    and only used to sort sections within the form.

Child classes may redefine sections inherited from their parent classes by
re-declaring section with the same name.
Sections are translated to the

::

  "formSections": {
    "mySectionName1": {
       "title": "text1",
       "index": 0
    },
    "mySectionName2": {
       "title": "text2",
       "index": 1
    }
  }

entries in the root schema of the type or method.


Alternatives
------------

Instead of switching to json-schema we could generate UI definition in existing
(or improved) UI definition format.

Data model impact
-----------------

None

REST API impact
---------------


**GET /schemas**

Execute static MuranoPL method. Method must have a Public scope.

*Request*

+---------+--------------------------------+-------------------------------------+
| Method  | URI                            | Description                         |
+=========+================================+=====================================+
| GET     | /schemas/{className}           | Obtain json-schema for class        |
+---------+--------------------------------+-------------------------------------+
| GET     | /schemas/{className}/{methods} | Obtain json-schema for class method |
+---------+--------------------------------+-------------------------------------+

Parameters:

* `className`: name of the class

* `classVersion`: version or version spec for the class. Optional. If not
  provided then '=0' is assumed

* `packageName`: optional FQN of the package. If provided the class will only
  looked up there instead of full catalog.

* `methods`: model builder method name or list of names which schemas are
  requested. Empty string indicates schema of the class rather than of one of
  its methods. If the parameter is absent then all the schemas (both class and
  model builders) are returned.


*Response*

::

  {
    "": {
      # json-schema for the class
    },
    "myModelBuilder1": {
      # json-schema for the myModelBuilder1 method
    }
  }


HTTP codes:

+----------------+-----------------------------------------------------------+
| Code           | Description                                               |
+================+===========================================================+
| 200            | OK. Schema was generated successfully                     |
+----------------+-----------------------------------------------------------+
| 400            | Bad request.                                              |
+----------------+-----------------------------------------------------------+
| 401            | User is not allowed to access the schema                  |
+----------------+-----------------------------------------------------------+
| 404            | Not found. Specified class or doesn't exist               |
+----------------+-----------------------------------------------------------+


Versioning impact
-----------------

If we introduce more capabilities to the contracts then a new FormatVersion
should be introduced.

New murano-dashboard/python-client could still talk to an older API service
that lacks new API call and uses old UI definition alone in this case.

New approach is backward compatible so existing applications will still work.


Other end user impact
---------------------

A new python-muranoclient with a method to obtain json-schema for the class
will be required in order to take advantage on the MuranoPL forms.


Deployer impact
---------------

Maximum number of class implementations need to be specified in murano.conf
file. However it is going to have a reasonable default value.


Developer impact
----------------

In order to have rich GUI, an application developer will have to decorate all
his properties with lots of Meta values. Otherwise if no UI definition file
was provided the UI form may present input fields in a random order with a
labels set to a property name which is not very user friendly.

We should design a way to extract all visual hints into a separate per-class
file to separate them from the application code. This is a subject for another
spec.


Murano-dashboard / Horizon impact
---------------------------------

murano-dashboard should present user with the form constructed from a json
schema. The schema would contain all the required visual hints in the extra
attributes (not defined by the json-schema standard).

The dashboard may either construct the form on its own or use 3rd party
libraries that are capable to generate UI from the schema. In the later case
dashboard might become responsible for generating form definition - a structure
describing visual aspects of the form that is provided in addition to the
schema. Such structure might be produced by extracting required information
from extra attributes of the schema. The dashboard might also split it into
several forms/schemas in order to have a wizard UI rather than a single form.


Implementation
==============

Assignee(s)
-----------


Primary assignee:
  Stan Lagun <istalker2>

Work Items
----------

* Create new API method;

* Write python-muranoclient method for new API call;

* Implement RPC method in murano-engine that will do schema generation;

* Write json-schema generator for the class/method;

* Define all the mentioned meta-classes and enhance schema generator to make
  use of them.

Dependencies
============

* https://blueprints.launchpad.net/murano/+spec/static-actions

Testing
=======

All the path from the MuranoPL code to the rendered UI form can be tested
by the unit tests. Transformation from MuranoPL to json-schema can be tested
independently from the one from json-schema to the form definition or even
HTML layout.


Documentation Impact
====================

The following need to be documented:
* The new UI workflow.

* json-schema specification link along with description of extra fields added
  by Murano;

* All changes made to the contracts;

* Standard Meta classes that can be used for visual hints in MuranoPL;

* Model builder methods documentation;

* Developers guide.


References
==========

* http://json-schema.org

* https://blueprints.launchpad.net/murano/+spec/static-actions