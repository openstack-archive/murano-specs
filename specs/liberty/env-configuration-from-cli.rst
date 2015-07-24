..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===============================
Configure environments from CLI
===============================

Blueprint for this specification:

https://blueprints.launchpad.net/python-muranoclient/+spec/env-configuration-from-cli

Currently there is no way to configure and/or deploy a murano environment from
the command line. This specification describes how this issue can be addressed
and what steps have to be taken in order to implement this abilities.

Problem description
===================

Currently the only possible way to deploy a murano environment is through
interaction with horizon. CLI client currently lacks commands to add apps to
environment or deploy an environment. This makes murano useless without horizon
and murano-dashboard.
It also means, that scripting and/or automating deployment of murano
apps/environments is currently problematic to say the least.


Proposed change
===============

This spec proposes to add several new CLI commands and modify existing to
allow user to add apps to the env, configure it and send it to deploy.

Command ``environment-create`` currently only supports adding the name of the
environment, while the api call supports setting aditional params, such as
defaultNetworks. ``environment-create`` should be updated to allow setting
custom keys to environment model.

``environment-edit`` command should perform all the editing (adding, deleting,
modifying of existing apps)
This command should be non-interctive, adding an interactive mode is a
good idea as of itself, but is out of scope of this spec.

``environment-edit`` would use jsonpatch format for input (as described by
RFC 6902). This is a well known and powerfull instrument to manipulate json,
and object model is currently stored in json.
This would allow easy addition, modification and deletion of apps.
Editing of environment attributes is not allowed by API, so it's beyond the
scope of this spec.

During command execution, env configuration parameters should be read from
stdin or a file.
Supporting stdin as input source means it should be possible to issue a
command like ``murano environment-edit env_id < app_patch.json``

An example ``app_patch.json`` might look like::


  [
    { "op": "add", "path": "/services/-",
        "value":
          {'?':
            {'id': '2fc3005d-b4f8-420d-9e15-f961b41f49ee',
             'type': 'io.murano.apps.App1'
            },
           'instance':
                {'?': {'id': '53366263-04e9-49d1-829d-50e27158749c',
                       'type': 'io.murano.resources.LinuxMuranoInstance'},
                 'assignFloatingIp': True,
                 'availabilityZone': 'nova',
                 'flavor': 'm1.medium',
                 'image': 'afc1aa61-f623-4c66-bbd8-5359261c5272',
                 'name': 'ilphvibwqyu6h1'  },
           'name': 'App1'
           }
    },
  ]

All operations would only apply to services field, so the `/services/` part of
the path can be ommited.

note::
    This means, that a user has to generate unique ids. This can be a difficult
    task and may hinder scripting capabilities. We might think of a way to
    mitigate this issue, by introducing a way to generate ids. This can be done
    by introducing an optional parameter ``--id-format``. It would accept a
    template for strings to be replaced with generated ids. For example:
    ``--id-format=##id##`` would replace all occurrences of ##id##-1 with fist
    generated id, all occurrences of ##id##-2 with second generated id and all
    occurrences of ##id## with a unique id. uuids should be used for id
    generation.

Command should perform validation of its input. This can be done using
MuranoPL classes and property contracts.
If an error happens during non-interactive mode (for example validation fails
for some field) ``murano`` should return a non-zero status, to facilitate
scipting and error-detecting.

Command ``environment-edit`` should support optional ``--session`` parameter,
that allows user to specify a session for editing an environment.
It should also
warn user in case more than one session is open. If the session parameter is
omitted: last open session for given environment is used.
If there are no open sessions — a new session would be opened.

note::
    This means that to deploy a simple application one has to know jsonpatch
    syntax. We might think about optional `syntax sugar` commands, that would
    allow adding a single app to the environment. These would use predefined
    jsonpatch patch documents and would accept only value to be added to
    /Objects/services/- path.

We should implement session controlling commands during
implementation of this spec, for example:

#. ``session-list`` Show list of open sessions for a given environment.
#. ``session-show`` Show details of currently configured session..
#. ``session-delete`` Discard an open session for a given environment.

In case an error occurs during environment deployment this would allow to
rollback the changes to the previous version of environment.
This should also allow to handle cases, when a single environment is edited
simultaneously by different users.

``environment-action`` command should be implemented to allow performing
actions against an environment. One example of actions is a deploy action, so
``murano environment-action deploy`` should deploy the environment.

Alternatives
------------

Alternatively we can implement an interactive mode, that would mimic current
dashboard behaviour. This would not allow scripting, but would allow us to
build a more user-friendly CLI interface. The dowside is that current UI
definitions are scheduled to be changed and would most likely be replaced in
the forseable future. This means, that most of the work would go to waste.

Data model impact
-----------------

None

REST API impact
---------------

None, CLI should reuse API calls, already used by dashboard

Versioning impact
-------------------------

Since we're adding functionality — None

Other end user impact
---------------------

None

Deployer impact
---------------

It is possible that implementation of this spec would require setting and
reading intermediate environment variables to work correctly with sessions,
during app addition, env deployment.

Overall deployment of murano would not be affected.

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
  <kzaitsev>

Work Items
----------

#. ``environment-edit`` command
   (should create session, check permissions, add
   patch resulting model into ).
#. Input validation for ``envieonment-edit``
   (request packages from API, check ID
   uniqueness, check input field adequacy).
#. (optional) ``--format-id`` option support
#. (optional) syntax sugar command, that would allow easy addition of a package
   to an environment.
#. Session controlling commands.
#. ``environment-action`` command.
#. Shell unit tests.
#. Integration tests.

Dependencies
============

None

Testing
=======

We shall need Unit tests for new commands introduced.

Also, since this change introduces a way to deploy an env from CLI. This means
that integration tests for murano client should be implemented. A typical test
should upload and app, configure a simple environment with 1-2 apps and set
some custom parameters, like access port and optionally deploy the env in
question. This tests should probably take place in murano-ci.


Documentation Impact
====================

New python-muranoclient commands would have to get a proper documentation. It's
also possible, that we would want to document the whole process of deploying an
app or scripting of such a deployment as a separate article in murano
documentattion.


References
==========

* http://jsonpatch.com
* https://tools.ietf.org/html/rfc6902
* https://tools.ietf.org/html/rfc7159
* https://pypi.python.org/pypi/jsonpatch
