..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==============
Policy in code
==============

`bp policy-in-code <https://blueprints.launchpad.net/murano/+spec/policy-in-code>`_

Deployers currently have to maintain policy files regardless if they change
the default policy provided from murano. They also have to trace through
code in order to determine the purpose behind each policy operation in the
policy file. Maintaining the policy files and tracing through code to provide
context to the operations in the policy files is cumbersome and error-prone.
By moving policy into code, we can leverage tooling to make maintenace easier
for deployers, provide a centralized location for default policy values,
and more effectively implement self-documenting policies.

Problem Description
===================

Today policy exists in a file that deployers are expected to maintain in their
deployment. If a deployer needs to change the default policy rules for an
operation, they have to make those changes and continuously check to make sure
conflicts are resolved with each new release of the policy file. This is
cumbersome to maintain, even if a deployment is only using the default policy.
Deployers must also trace through code to identify where certain policies
are enforced, which is also cumbersome.

Proposed Change
===============

The proposed solution is to check policy into the code base and register it
using the oslo.policy library. This is very similar to how projects register
and use configuration options using oslo.config. If policies are provided in a
policy file on disk, those policies will be registered instead of the in-code
default. This provides a way for deployers to override the defaults provided
in the in-code policies.

The registration will need two pieces of data:

1. The operation, e.g. "create_environment" or "list_environments_all_tenants"
2. The rule, e.g. "role:admin" or "rule:default"

Descriptions can also be provided in the registered policy object that help
describe the operation and the rule or role that is required to execute it,
in addition to which API endpoints enforce the policy operation. This
description can be used when generating sample policy files from registered
rules, as well as help operators to better understand policy enforcement
in murano.

This is the exact same approach
`nova <http://specs.openstack.org/openstack/nova-specs/specs/newton/implemented/policy-in-code.html>`_
and
`keystone <http://specs.openstack.org/openstack/keystone-specs/specs/keystone/pike/policy-in-code.html>`_
used to codify policy.

The following are benefits from the approach:

* There is no longer a need to maintain a policy file in tree.
* A tool can be written to auto-generate a policy.json file from the default
  policy operations in code.
* It will be easier for operators to understand the intent of the policy
  operations and where they are enforced in the system.
* It will be easier to provide a description of each policy much like we do
  configuartion options. This will ensure that the policies are well-documented
  and maintained.

Alternatives
------------

An alternative approach was to pull policy into murano as an official
resource. This would still require some sort of policy override ablility for
deployments that do not wish to deploy the default.

Security Impact
---------------

None.

Notifications Impact
--------------------

None.

Other End User Impact
---------------------

None. Policy will continue to be evaluated and enforced like it does today.

Performance Impact
------------------

The performance impact of moving policy in code should be minimal. If the
deployment doesn't have a policy file on disk, the service will not have to
fetch it. Instead the default will be registered and used from within code. In
the event the deployment is using policy overrides, the combination of the two
approaches might cause some performance impact compared to defaults in code,
but the overall impact should be negligible.

Other Deployer Impact
---------------------

If a deployer already makes modifications to the default policy file, they
will have to continue maintaining those changes. For deployers who modify a
subset or none of the policy entries, they can essentially remove their policy
file, or the policies that are the default. The end result should be a policy
file that purely consists of overrides the deployer wishes to enforce.

Another deployer impact is that deployers no longer need to double-check they
are protecting all new operations by manually inspecting policy files across
releases. Instead, they can be notified about new policies available in a
release via release notes and then choose to either use the well-documented
defaults or to override them in the policy file.

Developer Impact
----------------

Any policies added to the code should be registered before they are used. There
should also be checks and tests added that make sure new policy entries are
accompanied with a release note.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Felipe Monteiro (felipemonteiro)

Work Items
----------

* Investigate the process for adding oslo.policy into keystone's policy.
* Gradually move policy checks from ``policy.json`` into oslo.policy objects.
  This can be done incrementally and should remove the check from
  ``policy.json``.
* Change genconfig tox environment to generate sample policy.json file.
* Update documentation.
* Remove the murano policy file from devstack and murano.

Dependencies
============

None.

Documentation Impact
====================

Documentation for deployers about the policy file will be updated to mention
that only policies which differ from the default will need to be included.

References
==========

* `nova specification <http://specs.openstack.org/openstack/nova-specs/specs/newton/implemented/policy-in-code.html>`_
* `keystone specification <http://specs.openstack.org/openstack/keystone-specs/specs/keystone/pike/policy-in-code.html>`_
