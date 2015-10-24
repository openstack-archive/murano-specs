..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=======================================
Support Berkshelf for Chef applications
=======================================

URL of launchpad blueprint:

https://blueprints.launchpad.net/murano/+spec/support-chef-berkshelf

The goal of this specification is to make possible the use of Berkshelf
(http://berkshelf.com/) to manage Chef cookbooks dependencies when Chef is
used to configure a VM.


Problem description
===================

Currently, Chef is one of the types supported in the Execution plan template.
A Chef provider included in murano-agent gets the Chef cookbooks, generates
the node specifications file and executes chef-solo to configure the VM.

When a cookbook depends on other cookbooks, every transitive dependency must
be made available before executing Chef. That means that the dependencies
metadata for each cookbook must be recursively inspected by the creator of
the Murano package and added to the required files in the Execution plan
template.

Example from murano-apps::

    Scripts:
      executeRecipe:
        Type: Chef
        Version: 1.0.0
        EntryPoint: git::default
        Files:
            -  git: https://github.com/jssjr/git.git
            -  yum-epel: https://github.com/chef-cookbooks/yum-epel.git
            -  yum: https://github.com/chef-cookbooks/yum.git
            -  dmg: https://github.com/opscode-cookbooks/dmg
            -  openssl: https://github.com/opscode-cookbooks/openssl.git
            -  chef-sugar: https://github.com/sethvargo/chef-sugar.git
            -  chef_handler: https://github.com/opscode-cookbooks/chef_handler
            -  windows: https://github.com/opscode-cookbooks/windows
            -  build-essential: https://github.com/opscode-cookbooks/build-essential
        Options:
          captureStdout: true
          captureStderr: true

This list of cookbooks is painful to maintain and error-prone. Furthermore,
a Chef cookbook can require a specific version or a range of versions for its
dependencies. Therefore, the master branch or a git repository does not contain
necessarily the right version of the cookbook. Versions resolution for the
whole dependency graph is not an easy task to execute manually. That is why
using a tool dedicated to this task is useful.

Proposed change
===============

It is proposed to add two new possible options to the Execution plan template.
These options will be available only with the Chef executor (options are
executor-dependent):

* ``useBerkshelf`` (optional, default ``false``)
* ``berksfilePath`` (optional, used only if ``useBerkshelf = true``)

If ``useBerkshelf`` is not set or set to ``false``, the Chef executor will
work as now.

If ``useBerkshelf`` is set to ``true``, the Chef executor will
expect Berkshelf to be present on the VM, and will use it to manage the
cookbooks dependencies.

In that case, the Chef executor will use Berkshelf and the *Berksfile* found on
``berksfilePath`` to *vendor* the package and its dependencies.
Dependency and version resolution is done by Berkshelf using the Berksfile.
The Chef executor will then specify to chef-solo the location of the cookbooks.

If ``berksfilePath`` is not provided, its default value is
``<cookbookName>/Berksfile``, where ``<cookbookName>`` is taken from the
``EntryPoint`` value. For example, if ``EntryPoint = example::default``,
default value for ``berksfilePath`` is ``example/Berksfile``.

A Murano package creator wanting to use Chef with Berkshelf will have to
provide a link to the URL of a repository containing the main cookbook.

Example::

    |__ Classes
    │   |__ Example.yaml
    |__ logo.png
    |__ manifest.yaml
    |__ Resources
    │   |__ DeployExample.template
    |__ UI
    |__ ui.yaml

With ``DeployExample.template``::

    FormatVersion: 2.1.0
    Version: 1.0.0
    Name: Deploy example

    Body: |
      return executeRecipe(args).stdout

    Scripts:
      executeRecipe:
        Type: Chef
        Version: 1.0.0
        EntryPoint: example::default
        Files:
            -  example: https://github.com/cookbook/example.git
        Options:
          captureStdout: true
          captureStderr: true
          useBerkshelf: true

In that example, the *berksfile* is found on ``example/Berksfile``, in the
root directory of the git repository.

Alternatives
------------

Librarian-Chef is another tool to manage cookbooks. But Berkshelf is better
integrated in the Chef ecosystem than Librarian-Chef (Berkshelf is actually
part of ChefDK). Furthermore, Librarian-Chef requires the creation of a
``Cheffile``, whereas Berkshelf uses the ``Berksfile`` already included in
every cookbook.

Data model impact
-----------------

None

REST API impact
---------------

None

Versioning impact
-------------------------

Minor version of ``FormatVersion`` for the Execution plan template should be
incremented (new feature, backwards-compatible). New ``FormatVersion``
should be ``2.2.0``.

Other end user impact
---------------------

None

Deployer impact
---------------

Berkshelf need to be installed in the VMs to be configured.

Developer impact
----------------

None

Murano-dashboard / Horizon impact
---------------------------------

None


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  o-lemasle (Olivier Lemasle <olivier.lemasle@apalia.net>)


Work Items
----------

#. Add Berkshelf support in murano-agent
#. Create DIB elements to include Berkshelf in image
#. Create an example application using Chef with Berkshelf


Dependencies
============

None


Testing
=======

* Unit tests on murano-agent
* Functional tests: deploy an application with Chef template and Chef
  cookbooks dependencies in Murano integration tests
  (``MuranoDeploymentTest``)


Documentation Impact
====================

Information about how to create Execution plan templates with Chef and
Berkshelf will have to be documented.


References
==========

* http://berkshelf.com/
