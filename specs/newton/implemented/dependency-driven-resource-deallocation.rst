..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==================================================
Dependency-driven multi-step resource deallocation
==================================================

https://blueprints.launchpad.net/murano/+spec/dependency-driven-resource-deallocation

Murano components may allocate various kinds of resources: virtual machines,
networks, volumes etc. When these components get removed from the deployment
appropriate resources have to be deallocated. Current implementation of this
process has some significant limitations and flaws.

This specification aims to address these issues and provide a design for the
better resource deallocation / garbage collection system for MuranoPL.

Problem description
===================

In Murano the deallocation of resources is managed by a garbage collection
system (GC). Its present implementation is based on the execution of special
methods called ``.destroy()`` which may be defined in each MuranoPL class and
are intended to contain the custom code to deallocate resources allocated by
the objects of those classes.
These methods get executed when their objects leave object graph, however the
exact order of these executions is currently undefined.

There are two different scenarios when the objects may leave the object graph
thus causing their ``.destroy`` methods to be called:

*   "Offline" changes of the Object Graph, i.e. the changes introduced in the
    serialized version of the object graph via the API. These changes are
    detected by comparing the incoming object graph (the one passed from the
    API for deployment) with the "snapshot" of current environment made after
    the previous deployment has been completed. If some object exists in the
    "snapshot" but is missing in the input graph it is considered to be
    removed. Such objects are deserialized from the snapshot and their
    ``.destroy`` methods are called in the order from deepest nested objects
    towards topmost ones.

*  "Runtime" changes. Some objects may be removed from the object graph during
   deployment: they may be unreferenced or assigned to Runtime properties or
   local variable only. As a result after the deployment completes these
   objects are not serialized neither into the output version of the object
   graph nor into its "snapshot". The next deployments will know nothing about
   their existence so the objects will be lost forever. To recover the
   resources allocated by such unreferenced objects murano analyzes its
   ObjectStore after the execution is complete. Each object which is present in
   the store but is not present in the output version of the object graph is
   considered to be "orphan" and thus its ``.destroy`` method is called. The
   order of these calls is currently undefined: the objects get destroyed based
   on their position in the object store, which is hardly predictable.

Such a design is insufficient for production grade applications which
often require the following scenarios:

* If some object is going to be deleted another object (either owning or just
  referencing it) may need to execute some actions before or after the object
  is deleted.

* When a group of nested or interconnected objects is about to be deleted the
  order in which their destructors should be executed may be different in
  different cases.

* Sometimes the actions being executed during the destruction of an object may
  depend on the fact whether some other object is about to be deleted or not.


    *Example*

    *Consider an Application which consists of a Network component and several
    VM components. All the components are owned by the Application but there
    are no ownership relationships between them. When the Application is going
    to be deleted (i.e. its whole subgraph leaves the environment) all of its
    components are about to be removed as well, and so their ``.destroy``
    methods will be called. Since Network does not own the VM's the order
    of these calls is undefined. Due to various implementation details it may
    be impossible to remove the Network before the VMs which are connected to
    it (e.g. in case when the VM has a mandatory requirement to be always
    connected to at least one network). In this case the ``.destroy`` of a
    Network component should be always called after all the VMs have been
    destroyed.*

Another issue is that murano uses ``Objects`` and ``ObjectsCopy`` objects
to transfer data between deployments. When destruction dependencies are
implemented, the handler will make changes (if any) to objects in
``ObjectsCopy``. Therefore, these changes are not applied during the next
deployment and this should be addressed.

Also, sometimes it can be useful to deallocate resources used by the
unreferenced objects even before the end of deployment on demand from the
application code.

Proposed change
===============

Several improvements have to be made in Murano engine to address the problems
described above.

General concepts
----------------

Destruction dependencies
^^^^^^^^^^^^^^^^^^^^^^^^

There should be a way to establish directional `destruction dependencies`
between Murano objects. If object `Foo` establishes such a dependency on the
object `Bar` then:

* `Foo` will be notified when `Bar` is about to be destroyed. These
  notifications are covered in details in "Multi-step destruction" section
  below.

* If both `Foo` and `Bar` are going to be destroyed in the same garbage
  collection execution, `Bar` will be destroyed before Foo.

These dependencies are not related to object ownership relationship or
property-based cross-references: the owner may have a destruction dependency on
its nested object or vise versa; the objects referencing each other may have
some destruction dependency established. Even entirely unrelated objects may
have a destruction dependency between them.

Since the destruction dependencies are directional there is a theoretical
possibility of a circular dependency to exist. In case if two or more objects
form such a circle they will still be notified about pending destruction of
their dependencies, however the order of this notifications - and the
destruction itself - is undefined in this case.

Destruction dependencies are going to be used during all kinds of garbage
collection: pre-deployment ("offline"), on-demand (during deployment) and
post-deployment.

Multi-step destruction
^^^^^^^^^^^^^^^^^^^^^^

Instead of just iterating through all the objects going to be destructed and
calling their ``.destroy`` methods Murano should perform a multi-step garbage
collection according to the following algorithm:

1. Detect all the objects going to be destroyed. It can be done by using
   Python GC to track and collect objects, as described in the
   *Garbage collection executions* section.

2. Sort the list of detected objects using the following comparator: for
   any two objects A and B in the list:

    .. code::

     IF (A owns B) THEN A>B
     ELSE IF (A has-a-destruction-dependency-on B)
         AND (NOT B has-a-destruction-dependency-on A)
     THEN A>B
     ELSE A==B

   where `has-a-destruction-dependency-on` means that the left operand object
   has a destruction dependency (probably transitive) on right operand object,
   `owns` means that the left operand object owns (probably transitively) the
   right operand object.

   The objects which are considered to be equal by the algorithm above can be
   destroyed in parallel.

   The result of the sorting is the dictionary with indexes as keys and
   lists of objects with equal destruction priority as values.

3. Sort the keys of the dictionary in the reversed order of destruction
   priority and for each key start parallel notification about scheduled
   destruction of the objects in the corresponding group. During
   notification, handlers of the objects that have a destruction dependency on
   some "sentenced" object will be invoked.

4. Sort the keys of the dictionary in the direct order of destruction
   priority and for each index in the dictionary start parallel destruction of
   all objects in the corresponding group. Destruction of individual object
   means calling the ``.destroy()`` method of the object if present and
   changing the object's status to "Destroyed" (see below).

   As an environment does not have owners, it will always be in the last
   group of destruction. There is no guarantee that some other objects (for
   example, heat stack) will be alive at the time of its destruction. Thus
   ``io.murano.Environment`` class should not have the ``.destroy()`` method.

Destroyed objects
^^^^^^^^^^^^^^^^^

When an object is being processed by a garbage collector, it means that there
are no live references to it from the objects of the environment. However there
may be cases when the code which handles either the pre-destroy notification
(step 4 above) or the actual ``.destroy`` method re-establishes the references
to the object being destructed, and thus the object remains in the object graph
after the GC is completed. Since the resources may be deallocated at this time
the regular usage of the object is not possible, however if it is assigned to
a property of some another object in the graph it may not always be possible to
just nullify that property since it may cause a contract violation.

To resolve such collisions it is proposed to explicitly mark such destroyed
objects as "destroyed". It means setting object's ``destroyed`` attribute to
``True`` and removing the self-reference from it.
MuranoPL executor will not allow to execute any methods on such objects,
however their properties remain accessible (i.e. readable) so
any runtime information associated with them may be recovered. Destroyed
objects will be serialized with the rest of object graph but the
json-representation of the object will have a special flag in their class
header (the "?" section) to indicate their special status. When deserialized
from json such objects will retain their "destroyed" status, so the method
execution will still be impossible even in subsequent deployments.

When "destroyed" objects are unreferenced from the object graph, their
properties get nullified and they get destroyed automatically by Python GC.

Garbage collection executions
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The multi-step object destruction described above should take place in three
different scenarios:

1. *(currently existing)* Before the deployment, destroys objects which were
   present in the object graph after the previous deployment was finished but
   were not found in the incoming object graph of a new deployment, i.e. the
   ones explicitly removed using the API.

2. *(currently existing)* After the deployment, destroys the objects existing
   in the Object Store but not being a part of the **persistable** object graph
   of current environment, i.e. having no references to them from the
   persistable (In, Out, InOut) properties of the environment or its transitive
   children).

3. *(proposed)* During the deployment, explicitly initiated from MuranoPL code.
   Destroys objects which are not part of the **complete** object graph, i.e.
   having no references to them from any properties of the environment
   (including runtime and private properties) AND not being referenced by local
   variables in any frame of all the green threads of current deployment.

To implement scenario 3, a new algorithm is needed. As mentioned in the
*Multi-step destruction* section, Python garbage collector can be used
for that. MuranoPL can make use of the way Python's ``gc.collect`` works.

Python library ``gc`` allows running on-demand garbage collection through the
``gc.collect()`` method and provides access to unreachable objects that the
collector found but cannot free through collection ``gc.garbage`` [2].

To make use of this, there should be an ability to:

* Make object store have weak links only and prevent hard links in other DSL
  places so that only links between objects remain.

* Prevent murano objects that should have been destroyed by Python GC from
  being destroyed.

* Get the list of such objects and destroy them in correct order and notify
  subscribers about destruction.

The prevention can be done by adding ``__del__()`` magic method to the
MuranoObject class and creating cyclic reference in the object to itself.
When ``gc.collect()`` is done, all unreachable objects can be examined and
murano objects owned by current executor can be distinguished among them.

The difference in Python 3.4 and higher is that objects with a ``__del__()``
method don't end up in ``gc.garbage`` anymore and this list should be empty
most of the time [3]. So logic of adding object to GC candidates can be added
directly to ``__del__()``.

It means that in Python versions 3.4 and higher, murano objects will be added
to planned destruction from ``__del__()`` call caused by ``gc.collect()``,
and in versions prior to 3.4, presence of ``__del__()`` along with cyclic
reference to itself will provide adding the object to ``gc.garbage`` list,
and it can be added to candidates for destruction from there.

This logic can be used for garbage collection in all three scenarios
mentioned above.

The resulting list of GC candidates is then destroyed as described in the
*Multi-step destruction* section above.

With this approach, the comparison of Objects and ObjectsCopy is not needed
anymore. Garbage collector works with the same objects on each deployment,
so all changes are saved properly.

Code changes
------------

GC class
^^^^^^^^

A new python-backed Murano class called ``GC`` should be added to the core
library. It should have the following static methods:

* ``collect()`` - initiates garbage collection of unreferenced objects of
  current deployment (see p.3 in "Garbage collection executions" section
  above).

* ``isDestroyed(object)`` - checks if the ``object`` was already destroyed
  during some GC session and thus its methods cannot be called.

* ``isDoomed(object)`` - can be used within the ``.destroy()`` method to
  test if another object is also going to be destroyed.

* ``subscribeDestruction(publisher, subscriber, handler=null)`` - establishes
  a destruction dependency from the ``subscriber`` to the object passed as
  ``publisher``. Method may be called several times, in this case only a single
  destruction dependency will be established, however the same amount of calls
  of ``unsubscribeDestruction`` will be required to remove it.

  ``handler`` argument is optional. If passed it should be the name of an
  instance method defined by the caller class to handle notification of
  ``publisher``'s destruction (see "Multi-step destruction" section above: this
  handler is executed for p. 3.1)

  The following arguments will be passed to the handler method:

  * ``object`` - a target object which is going to be destroyed. It is not
    recommended to persist the reference to this object anywhere. This will not
    prevent the object from being garbage collected but the object will be
    moved to the "destroyed" state which is almost always bad. The option to do
    so is considered to be advanced feature which should not be done unless it
    is absolutely necessary.

* ``unsubscribeDestruction(publisher, subscriber, handler=null)`` - removes
  the destruction dependency from the ``subscriber`` to the object passed as
  ``publisher``. Method may be called several times without any side-effects.
  If ``subscribeDestruction`` was called more than once the same (or more)
  amount of calls to ``unsubscribeDestruction`` is needed to remove the
  dependency.

  ``handler`` argument is optional and must correspond the handler passed
  during subscription if it was provided back then.

Alternatives
------------

Application developers may try to implement their own event-based notification
logic to notify about pending and completed object destructions. However it
will solve only part of the problem: notifications will work properly, but they
will not affect the order in which the objects are destroyed, so the workflows
will be too complicated. Also this alternative will not have the advanced
features proposed in this spec, such as ability to check if some object is
going to be destroyed.

Data model impact
-----------------

None

REST API impact
---------------

None

Versioning impact
-----------------

The proposed change is completely backwards compatible: without explicit
destruction dependencies objects will be collected based on their ownership
relationships, i.e. as it is done in the current implementation.

The packages containing classes which explicitly call the methods of ``GC``
should have package format of at least 1.4 to prevent their execution on older
versions of Murano which do not have this feature.

Other end user impact
---------------------

None

Deployer impact
---------------

None

Developer impact
----------------

Developers will get the new MuranoPL-based API to manage resource deallocation
lifecycle. If they do not want to use it they don't need to do anything.


Murano-dashboard / Horizon impact
---------------------------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  ativelkov

Other contributors:
  Stan Lagun <istalker2>
  starodubcevna

Work Items
----------

* Implement a system to define and use destruction dependencies in runtime.

* Introduce changes to MuranoObject class to keep track of "destroyed"
  object status.

* Modify the serializer / deserializer to properly persist the value of the
  "destroyed object" flag.

* Implement collecting unreferenced murano objects utilizing Python ``gc``
  library

* Implement sorting algorithms to arrange objects-to-be-destroyed based on
  criteria defined in p.2 of "Multi-step destruction" section above.

* Implement multi-step destruction workflow.

* Implement ``GC`` class to bind all the above.

* Create test-runner-based tests to cover all the test scenarios.

* Document the new features.


Dependencies
============

The development of this feature will enable Application Development Framework
[1] to address resource deallocation problems during application uninstall.

Testing
=======

Tests should be written for test-runner to cover various scenarios of resource
deallocation.

Runtime garbage collection
--------------------------

There should be test cases covering that:

* objects assigned to persistent (Input, Output, InputOutput) properties (both
  locally-declared and inherited) of objects reachable from the current roots
  are NOT garbage collected;

* objects assigned to transient (Runtime and undeclared) properties (both
  locally-declared and inherited) of objects reachable from the current roots
  are NOT garbage collected; target properties should be both locally-declared
  and inherited;

* objects assigned to static properties of various classes are NOT garbage
  collected;

* objects passed to python-backed objects and unreferenced in MuranoPL are NOT
  garbage collected unless their MuranoObjectInterface proxies are unreferenced
  / GC'ed in python;

* objects assigned to local variables of the current execution frame (i.e.
  variables of the current method and all the caller methods in call stack)
  including method arguments are NOT garbage collected;

* single unreferenced objects ARE garbage collected;

* graphs of interconnected objects having no references from non-collected
  objects ARE garbage collected;

* objects passed to python-backed objects and unreferenced in both MuranoPL and
  python ARE garbage collected;

* garbage collector correctly processes stack-frame objects from green-threads
  other than the one it is executed from

Destruction dependency resolution order
---------------------------------------

There should be test cases covering that:

* if some child object has a destruction dependency on its parent, the parent
  gets destroyed before the child;

* if some parent object has a destruction dependency on its child, the child
  gets destroyed before the parent;

* if some objects not being the part of some ownership hierarchy have some
  destruction dependency, the dependency-object is destroyed before the
  dependent one;

* if some objects have circular destruction dependency they are all destroyed
  (the order is not enforced by the test);

Destruction events
------------------

Given the base scenario of object A having a destruction dependency on object B
and B being GC'ed, there should be tests covering that:

* the right order of events occurs (B scheduled for destruction -> A is
  notified about planned B's destruction -> B's ``.destroy()`` method is
  called -> B gets destroyed);

* A may prevent B's destruction by establishing a reference on B in the
  handler;

* A may establish more than 1 destruction dependency on B and still be
  notified just once;

* A may remove the destruction dependency and not get notified on B's
  destruction;

* If A established N destruction dependencies and then removed them M times,
  (N>M) then notifications are still delivered;

* If A established N destruction dependencies and then removed them M times,
  (N<=M) then notifications are not delivered;

* B may establish a destruction dependency on itself thus subscribing to
  appropriate notifications;

* ``isDoomed`` and ``isDestroyed`` methods return appropriate values when
  called by A for B in appropriate event handlers.

Documentation Impact
====================

Developers documentation should be updated to describe the new ``GC`` class and
its methods, as well as the design guidelines for application developers to
follow to utilize the new capability.

References
==========

[1] https://github.com/openstack/murano-specs/blob/master/specs/newton/approved/application-development-framework.rst
[2] https://docs.python.org/2/library/gc.html
[3] https://docs.python.org/3/library/gc.html
