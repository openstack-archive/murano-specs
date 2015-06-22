..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Support TOSCA definitions for applications
==========================================

URL of launchpad blueprint:

https://blueprints.launchpad.net/murano/+spec/support-tosca-format

TOSCA is a standard developed under OASIS foundation. It is aimed to cover
definition of complex enterprise applications which might consist of different
loosely coupled components. Components can be tied by using requirements and
capabilities which are part of TOSCA standard. As TOSCA covers most of the
Applications aspects (excluding application compilation and build) it should
be straightforward to support TOSCA format in murano. TOSCA is adopted by
enterprises so it will allow the openstack ecosystem to integrate with
enterprise IT applications.

The suggested approach for adding TOSCA support to murano is to leverage
the heat-translator project, which is an openstack project. Heat-translator is
a command line tool that takes non-heat templates (e.g., TOSCA-based templates)
as an input and produces a heat orchestration template (HOT) which can then be
deployed by heat. The heat-translator functionality is imported as a library.

The benefit of using heat-translator is that it is not TOSCA specific, and
designed to be easily extensible to support any other specification language.
And this flexibility will be transferred to murano as well.


Problem description
===================

Murano currently does not support TOSCA and TOSCA-based applications. With
the growing popularity of TOSCA for describing application orchestrations
it would be nice if murano supports TOSCA-based orchestration templates.

One standard for defining TOSCA-based application packages is CSAR
(http://tinyurl.com/tosca-yaml-csar). It is a compressed file that includes the
main application definition template along with the supporting templates and
files that are referenced in the main template. There is also a required
TOSCA.meta file in the CSAR that provides metadata about the package.

This specification addresses adding support for CSAR-based application packages
in murano application catalog. It can simply be expanded to other formats and
standards that are, or will be, supported by heat-translator.


Proposed change
===============

In order to support the TOSCA CSAR package format there are a couple of use
cases that need to be implemented in murano:

*package-create / package-import*
These commands are currently used to create application definition archives
from HOT templates (diagram below).

::

                  +------------------+
                  |      Murano      |
                  +--------+---------+
                           |
    package-create(HOT)    |
  +---------------------> +++
                          | |
  <---------------------+ +++
  application definition   |
    archive (HOT ADA)      |
                           |
                           |
 package-import (HOT ADA)  |
  +---------------------> +++
                          | |
                          | | +-----+ add component
                          | |       | to catalog
                          | | <-----+ along with
                          | |         its HOT
  <---------------------+ +++         template
            OK             |
                           |
                           +

Murano should be able to create similar application definition archives from
templates specified in TOSCA or any other language. Diagram below shows how
this can be achieved for TOSCA CSAR packages.

Note that the TOSCA parsing functionality has been taken out of heat-translator
and put into its stand-alone own OpenStack project, called tosca-parser.
Heat-translator relies on tosca-parser in translating TOSCA specifications.
Both projects are released as PyPI packages.

::

                    +------------------+      +------------------+
                    |      Murano      |      |   Tosca Parser   |
                    +--------+---------+      +---------+--------+
                             |                          |
    package-create(CSAR)(1)  |                          |
    +---------------------> +++                         |
                            | |     validate(CSAR)      |
                            | | +--------------------> +++
                            | |                        | |
                            | | <--------------------+ +++
                            | |          OK             |
                            | |                         |
                            | | +-----+ extract         |
                            | |       | metadata        |
                            | | <-----+ from CSAR       |
                            | |                         |
    <---------------------+ +++                         |
     application definition  |                          |
       archive (CSAR ADA)    |                          |
                             |                          |
                             |                          |
  package-import(CSAR ADA)   |                          |
    +---------------------> +++                         |
                            | |   validate(CSAR)(2)     |
                            | | +--------------------> +++
                            | |                        | |
                            | | <--------------------+ +++
                            | |           OK            |
                            | |                         |
                            | | +-----+ add component   |
                            | |       | to catalog      |
                            | | <-----+ along with      |
    <---------------------+ +++         its CSAR        |
              OK             |          package         |
                             |                          |
                             +                          +

(1) The package format is provided to the package-create command so murano
knows it has to process a TOSCA CSAR file. This can be done using a 'format'
parameter. Also, target platforms are specified using another parameter so
murano can build a proper UI field accordingly. To guarantee backward
compatibility if no format input is provided 'HOT' is assumed as default.

(2) Re-validation is required to cover cases where the application archive
is manually created or modified.


*Deploying an environment*
For HOT-based packages murano deploys one heat stack per component. Even if
two components are in the same environment they are deployed as their own
stack and independent of each other (diagram below)

::

                               +----------------+                     +----------------+
                               |     Murano     |                     |      Heat      |
                               +--------+-------+                     +--------+-------+
               deploy env               |                                      |
 +-----------------------------------> +++                                     |
                                       | |                                     |
    +-------+------------------------------------------------------------------------+
    | loop  |                          | |                                     |     |
    +-------+     +------+---------------------------------------------------------+ |
    |[for each    | opt  |             | |                                     |   | |
    |component    +------+             | |       deploy component's HOT        |   | |
    |in env]      |[if component       | | +--------------------------------> +++  | |
    |             |is not already      | |                                    | |  | |
    |             |deployed]           | | <--------------------------------+ +++  | |
    |             |                    | |         deployed stack info         |   | |
    |             |                    | |                                     |   | |
    |             +----------------------------------------------------------------+ |
    |                                  | |                                     |     |
    +--------------------------------------------------------------------------------+
                                       | |                                     |
    +-------+------------------------------------------------------------------------+
    | loop  |                          | |                                     |     |
    +-------+                          | |      remove component's stack       |     |
    |[for each previously              | | +--------------------------------> +++    |
    |deployed component                | |                                    | |    |
    |removed from env]                 | | <--------------------------------+ +++    |
    |                                  | |                 OK                  |     |
    |                                  | |                                     |     |
    +--------------------------------------------------------------------------------+
                                       | |                                     |
 <-----------------------------------+ +++                                     |
                   OK                   +                                      +

A similar behavior can be achieved by introducing TOSCA or other non-HOT
templates. A TOSCA-based package is stored with its TOSCA template(s) (and not
the corresponding HOT template). It may or may not need translation depending
on the platform it is deployed to. Murano passes the CSAR file to
heat-translator. Heat-translator translates the CSAR file if translation is
required for deployment, and then deploys it to the target platform.

Similar to how it is done for HOT templates, proper UI forms will be created
using MuranoPL so users can specify input parameters for the TOSCA template.

The CSAR archive will appear inside the murano package archive in the same
compressed format. This is because murano does not need to process the
internals of the CSAR archive, and the vision is that any piece of information
required from the CSAR will be acquired through tosca-parser. MuranoPL cannot
access the root of the murano archive, and therefore one of the following
options will be implemented:

1. Similar to how attached files to a HOT template are handled put the CSAR
   into Resources subfolder at package creation time.
#. Put the CSAR in the root folder and move it to Resources when processing the
   unzipped package.

::

                     +--------------+                +-----------------+       +------------------+
                     |    Murano    |                | Heat Translator |       | Target Platform  |
                     +--------+-----+                +---------+-------+       +---------+--------+
          deploy env          |                                |                         |
 +-------------------------> +++                               |                         |
                             | |                               |                         |
  +-------+---------------------------------------------------------------------------------------+
  | loop  |                  | |                               |                         |        |
  +-------+  +------+-------------------------------------------------------------------------+   |
  |[for each | opt  |        | |                               |                         |    |   |
  |CSAR      +------+        | |  deploy CSAR (platform) (1)   |                         |    |   |
  |component |[if component  | | +--------------------------> +-+                        |    |   |
  |in env]   |is not already | |                              | | +---+ translate if     |    |   |
  |          |deployed]      | |                              | |     | deployment       |    |   |
  |          |               | |                              | | <---+ requires         |    |   |
  |          |               | |                              | |       translation      |    |   |
  |          |               | |                              | |                        |    |   |
  |          |               | |                              | |          deploy        |    |   |
  |          |               | |                              | | +-------------------> +-+   |   |
  |          |               | |                              | |                       | |   |   |
  |          |               | |                              | | <-------------------+ +-+   |   |
  |          |               | |                              | |     deployment info    |    |   |
  |          |               | | <--------------------------+ +-+                        |    |   |
  |          |               | |   deployment confirmation     |                         |    |   |
  |          |               | |                               |                         |    |   |
  |          +--------------------------------------------------------------------------------+   |
  |                          | |                               |                         |        |
  +--------------------------------------------------------------------------------------------- -+
                             | |                               |                         |
                             | |                               |                         |
  +-------+---------------------------------------------------------------------------------------+
  | loop  |                  | |                               |                         |        |
  +-------+                  | |     remove deployed CSAR      |                         |        |
  |[for each previously      | | +--------------------------> +++        remove          |        |
  |deployed CSAR             | |                              | | +-------------------> +++       |
  |removed from env]         | |                              | |                       | |       |
  |                          | |                              | | <-------------------+ +++       |
  |                          | |                              | |          OK            |        |
  |                          | | <--------------------------+ +++                        |        |
  |                          | |     removal confirmation      |                         |        |
  |                          | |                               |                         |        |
  +-----------------------------------------------------------------------------------------------+
                             | |                               |                         |
 <-------------------------+ +-+                               |                         |
               OK             +                                +                         +

(1) Necessary bindings for the integration of murano and heat-translator will
be implemented similar to existing bindings for HOT.


Alternatives
------------

* An alternative to explicitly providing the format for package-create is
  enhancing package-create so it can auto-detect the format of the input
  template and act accordingly. This enhancement can be implemented via a
  separate blueprint.
* If by the time the deployment functionality is implemented heat-translator
  does not yet support deployment, murano's deployment through heat will be
  used.


Data model impact
-----------------

None


REST API impact
---------------

None


Versioning impact
-------------------------

This should not have any negative impact on versioning. Older versions of
murano would still not support TOSCA-based templates. And the new version
would work with HOT-based templates the same way as before.


Other end user impact
---------------------

There should be no user impact. Users who are creating HOT-based packages would
do it the same way as before. Users who are creating TOSCA-based packages would
do it in a similar way. Application packages can be added to environments for
deployment the same way as before too.


Deployer impact
---------------

None


Developer impact
----------------

None


Murano-dashboard / Horizon impact
---------------------------------

Murano would need to represent TOSCA CSAR-based components using a specific
logo, to distinguish them from HOT-based components. This applies to other
specification formats that may come on board in the future.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  vahidhashemian (I would be happy to take the lead on this, but I am hoping
  I could leverage the existing knowledge on murano development by having a
  co-contributor who knows murano code well).

Other contributors:
  TBD


Work Items
----------

In order to support the above use cases murano needs to add functionality for
the pieces where it communicates with heat-translator. This is a break-down of
work items (each work item may be broken further into several patches for
implementation):

1. Enhance package-create command to accept CSAR format, validate the CSAR
   package by calling tosca-parser, extracting metadata that comes as part of
   CSAR, and eventually building the application definition archive for the
   given CSAR (that includes the CSAR file too).
#. Enhance package-import command to accept CSAR-based application definition
   archives, re-validate them using tosca-parser, and import them to murano
   application catalog.
#. Enhance environment definition by creating a UI form for CSAR-based
   applications, asking user for their input parameters and target platform
   when adding them to an environment, and storing user-provided inputs along
   with the added CSAR-based application.
#. Enhance environment deployment and send CSAR-based components to
   heat-translator for, potential translation, and deployment to the user
   specified target platform. Also, add support for removal of a deployed CSAR
   component when it is removed from a re-deployed environment.


Dependencies
============

Aside from what needs to be implemented inside murano project, heat-translator
would need to implement a couple of new functionalities:

* Tosca-parser currently does not support full validation of TOSCA templates.
  Workaround is to call tosca-parser and report errors, if any. To do so, an
  option has to be added so it does not expect input parameters (and just
  translates to HOT instead of translation and deployment) in case the TOSCA
  template requires inputs (`related bug
  <https://bugs.launchpad.net/heat-translator/+bug/1464024>`_).
* Tosca-parser needs to fully support translation of CSAR packages to HOT.
* Heat-translator needs to enhance its API and add a 'deploy' functionally
  using which it would deploy the given application specification to the
  platform of choice.


Testing
=======

Implementation work items above requires a number of overall unit tests each of
which may be broken into smaller unit tests for implementation:

1. Verify package-create for TOSCA CSAR files.
#. Verify package-import for TOSCA CSAR packages.
#. Verify UI form creation for TOSCA CSAR packages.
#. Verify deployment and removal of TOSCA CSAR packages.


Documentation Impact
====================

How TOSCA CSAR packages can be added to catalog and deployed should be
documented, perhaps with some examples.


References
==========

* `Mailing List Discussion
  <http://lists.openstack.org/pipermail/openstack-dev/2015-June/065424.html>`_
* `TOSCA YAML CSAR Format <http://tinyurl.com/tosca-yaml-csar>`_
* `Heat-Translator on GitHub <https://github.com/openstack/heat-translator>`_
* `Tosca-Parser on GitHub <https://github.com/openstack/tosca-parser>`_