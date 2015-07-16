..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=================================
Add support for heat environments
=================================

https://blueprints.launchpad.net/murano/+spec/add-support-for-heat-environments-and-files

Add support for using Heat environments, when saving and deploying Heat
templates.


Problem description
===================

Today there is no option to create stacks neither with an environment nor with
nested stacks via Murano. Only basic stacks are supported.

Use cases for deploying a stack with an environment:

* Supporting resource registry
* Supporting saving 'test' and 'production' parameter profiles for the same
  template


Proposed change
===============
The change described here will focus on supporting Heat environments. Another
spec was submitted for Heat files.

In this spec no UI changes are included. A UI part that can be added in future
specs, and in any case is out of the scope of this spec.

Allowing Heat environments to be saved and deployed as part of a Murano Heat
package. Adding environments to a package should be optional. Heat environment
files should be placed in the package under '/Resources/HotEnvironments' When a
user request to add a Heat-generated package to a Murano environment, he should
have an option to specify one of the hot-environment-files located at
'/Resources/HotEnvironments' in the package, and this heat environment file
should be sent to Heat with the template during Murano environment deployment.
If no heat environment file was specified, the template will be deployed
without a Heat environment.
The stage where the Heat environment for the deployment should be specified
is when adding a package to a Murano environment as part of a configuration
session.
The heat environment should be referenced by name.

In the future, there can be an API to list the hot environments of a package.
This might be done by using the existing API of getting the UI of a package.

Alternatives
------------

* The package structure can be different.
* Words that can replace the word environments to reference heat environments:
  configurations, profiles, settings. The problem is that in Heat they use the
  word environments.
* Specifying the heat environment to be deployed with the Heat template as
  part of a Murano environment when deploying the environment (instead of
  specifying it when adding a package to an environment). If we will wait to
  this point, we will have to give a map of packages and the environments to be
  deployed with them. this alternative requires more validations.

Data model impact
-----------------

new data objects:

* hotEnvironment - This parameter will be passed as part of the
  /environments/{env-id}/services POST API request body. The value of this
  parameter will be an environment file name.
* templateParameters - All heat parameters that were passed in the root of the
  /environments/{env-id}/services POST API request body, will be moved under
  this property.

REST API impact
---------------
None

Versioning impact
-----------------

None

Other end user impact
---------------------

User will now have the option to add heat environments and additional files to
the package, and have them deployed with the package as part of the Murano
environment. The Heat environment will be deployed when it is requested in the
relevant API, while the additional files will always be deployed.

At this point there will be no change in the python-muranoclient. If the user
will wish to add environments or additional files to a heat generated package.
He can edit the package and continue via API. If the user only added additional
files, he can continue via python-muranoclient/UI as well.

Deployer impact
---------------

None

Developer impact
----------------

None

Murano-dashboard / Horizon impact
---------------------------------

New features must not break UI. So since it was proposed to move the user
defined parameters from the root of the request body to the property
"templateParameters", the UI must change the way it build the request to add  a
package to a Murano environment.

The new feature for sending a hot environment with the template will be exposed
in the UI the following way:
The user will be able to choose an environment file from a drop-down list
during the same wizard actions he uses to enter heat template parameters.


Implementation
==============

new data objects:
* envs
* env_name

envs will be generated when parsing the package, so if there is nothing in the
package, the new data objects will be empty. If an API to list the environments
will be implemented an empty list will return, same as in other Murano APIs.

A new object called envs of type dictionary will be added in HeatStack class
in the init method. It can be referenced from inside HeatStack by using
self._envs before/when sending the create request to Heat. This object will be
initialized to be empty.

A new method called setEnvs will be added in HeatStack class. It will allow to
enter a value into _envs just like setParameters allows to enter values into
_parameters.

env_name will take the value passed by the user in the parameter
heatEnvironment. It will be initialized and passed to class HeatStack the same
way parameters are passed today.

When sending the create stack request to heat the selected environment content
should be passed. If all is configured and passed correctly the environment
file content can be accessed from class HeatStack by using the next command:
self._envs[self._env_name]

A new method called _translate_envs will be add to HotPackage class. it will
get a path to the envs directory and will return a dictionary of environments
locations and files values. It will be in the next format:
environmentRelativePathStartingHeatEnvironmentsDirectory -> stringFileContent
For example if there is an environment with a full path of
/Resources/HeatEnvironments/my_heat_env.yaml and content of:
"heat_template_version: 2013-05-23\n\nparameters:\n" and it
is the only file in the folder, than this will be the dictionary returned:
{"my_heat_env.yaml": "heat_template_version: 2013-05-23\n\nparameters:\n"}

A very similar function was proposed for the Heat files feature. There will be
a reuse of the code there, if it will be implemented first.

Assignee(s)
-----------

Primary assignee:
  michal-gershenzon

Other contributors:
  noa-koffman

Work Items
----------

* Add support for adding a package to a Murano environment with a Heat
  environment specified in the request.
* Add support for Heat environments when deploying a Murano environment. If
  a Heat environment is saved in the session for the package, it should be
  parsed and send with the template, when deploying a Murano environment.
* make sure that when ui generate a POST request for API:
  /environments/{env-id}/services the user defined parameters are located under
  templateParameters in the request body.


Dependencies
============

None


Testing
=======

Unit tests should cover API calls changes:

* Test sending a Heat environment when adding a package to a Murano environment
  (positive and negative).
* Test that the request for Heat is build correctly with Heat environment


Documentation Impact
====================

None


References
==========

* http://docs.openstack.org/developer/heat/template_guide/environment.html