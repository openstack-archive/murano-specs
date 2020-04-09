..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=====================
Agent message signing
=====================


https://blueprints.launchpad.net/murano/+spec/message-signing


Problem description
===================

In order to deploy software, Murano sends scripts, playbooks and other files
to agents that run on each VM created. Murano uses RabbitMQ message broker
and its AMQP protocol to deliver messages to the agents. During first boot
each VM gets its RabbitMQ credentials, the queue name where the messages are
going to appear as well as instructions on where
to put execution results. Typical Murano installation has a dedicated
RabbitMQ instance that is used for such communications.

However, since user management is not included in AMQP protocol, Murano
requires RabbitMQ credentials to be provided in its config file. Hence all
the created VMs share the same credentials and it is only the queue name that
distinguishes one machine from another. Since murano agents steady pull new
commands from the RabbitMQ, if the queue name gets compromised, anyone could
send commands to the VM and take complete control over it. Luckily current AMQP
version does not have a function to list existing queues. However, since
queue name is generally not considered to be a secret information such
function might be added in the future. Also this information might be
exposed by the RabbitMQ plugins installed in the system. For example, there
is a so called Management plugin that can be used for that purpose. Besides,
the queue name can be found in the Heat stack definition for particular
Murano environment and anyone who can access it could take it from there.
Default Heat policy restricts it to users of that tenant plus the admin user.
However, particular OpenStack installations might have it configured
differently. Also even with default policy it is possible for users of a
single project (tenant) to control each other's VMs regardless of their role
in the tenant.


Proposed change
===============

The proposed solution is to implement message signing so that agents can
verify that the messages come from the murano-engine. There is a keypair
that is shared by all the murano-engine instances in the cloud. When murano
creates a VM, it provides it with a public part of the keypair in addition
to currently provided RabbitMQ credentials and queue name. Each message sent
by the murano-engine is going to be signed with the private key. Agents
will verify the signature with the public key they gave and ignore all the
messages that are not proven to be coming from the engine.

Here is how the communication workflow is going to become:

0. Murano engine is provided with the keypair (key file that has both the
   private and the public parts). The default is `~/.ssh/id_rsa` which is a
   standard RSA SSH keypair of the user that is running the murano-engine.
   However, different key might be provided with the newly introduced config
   file setting.

1. When a VM is created, its config that is passed to it via the user-data is
   going to include the public part of the key.

2. Every time murano-engine is about to send message `M` to the agent with
   queue name `Q` it calculates ``sign(Q + M, private_key)`` i.e. the
   signature of the message bytes with prepended queue name in latin1 encoding.
   The queue name is prepended so that the messages sent to one VM could not be
   resent to other VMs. The signature is then put into a custom AMQP message
   header so that the message body does not change.

3. Once the message is received by an gent, it performs
   ``verify(signature, Q + M, public_key)`` and drops the message if the
   verification fails. Since the queue name is different for each agent, it
   will also reject all messages that were signed for other VMs.

For the first version it all the cryptographic parameters are going to be
fixed so that no additional configuration is needed:

* The keypair must be PEM-encoded RSA without password protection
* The padding is PKCS#1 v1.5
* Digest function used for signing is SHA256

Engine-agent compatibility
--------------------------

Since the agent can be baked into the images Murano and updated
independently from the server, Murano must still work in situations where
the updated server talks to the old agents or vice versa (old engine, new
agents).

For the new engine / old agents case the compatibility is retained because
old agents will just ignore the new (and unknown to them) setting in the
config file (the public key) as well as custom AMQP header and just won't
verify the signature.

In the old engine / new agents case the server won't provide the public key
to the agents. Agents won't try to verify the messages in this case.

Thus the message signing is going to help only if both the server and the
agents support it. Otherwise everything is going to work the way it
currently does.

Replay attack prevention
------------------------

Having queue names prepended to the message bodies prevents messages sent to
one VM from being resent to another. However it is still possible to resend
the same message to the same VM later.

In order to prevent it, the message body JSON is going to get additional key
called ``Stamp``. It is a long integer value that is monotonic i.e. the
server guarantees that the number is greater from that of the previous
message. The field is optional (since older engines won't send it), but if
it is present, the agent must ensure that it exceeds the value of the
previously seen messages and ignores the message if it not the case. Murano
engine is going to derive the value from the current timestamp.

Messages with additional field remain compatible with the agents because
they ignore all unknown fields. As a bonus, this feature is going to protect
the agent against messages that for some reason got delivered twice which
can happen in rare cases due to the message was requeued because of
unexpected connectivity loss with RabbitMQ upon its acknowledgement or due
to some RabbitMQ cluster inconsistencies.

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
-------------------------

None


Other end user impact
---------------------

None

Deployer impact
---------------

There should be generated and provisioned for all the engine instances
keypair file. However it is going to be optional in order not to break
existing murano deployment playbooks.

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
  Stan Lagun (slagun)


Work Items
----------

1. Add keypair to the murano-engine config file schema
2. Extract the public key from the keypair and inject it into the generated
   agent config data
3. Implement message signing on the server
4. Implement signature verification in the python agent
5. Implement signature verification in the legacy Windows (PowerShell) C#
   agent
6. Add stamp value to the messages
7. Validate stamp value on the agent (including persistent tracking of the
   previous stamp value) for both agents


Dependencies
============

There are going to be new dependencies on the cryptography python package
that provides all the required crypto functionality. Several such libraries
are already present in the global requirements list.

Testing
=======

New functionality can be fully tested with unit tests.


Documentation Impact
====================

New config setting for the Murano engine should be documented.


References
==========

None
