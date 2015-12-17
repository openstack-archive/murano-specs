..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================
Engine Package Cache
====================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/murano/+spec/murano-engine-package-cache

This specification provides means to improve caching of packages to fasten
consecutive deployments and action executions, that use the same package.


Problem description
===================

When murano-engine performs a deployment or executes an action on a deployed
environment it needs access to the code of the package. Currently it downloads
a package from API for each deployment and stores it in a local cache for the
duration of the deployment, but deletes it after the deployment has finished.

This means, that a consecutive deployment of the environment that uses the same
package would have to download it again. Same is true for actions. This puts an
undesired strain on murano-api and network, as the same package is downloaded
many times even if it didn't change
Currently murano-engine stores packages on disc, using package-id as cache key.
This further means that if two environments are using the same package the
first to finish would delete the cache, leaving the second environment in an
unexpected state. This can potentially lead to unexpected and hard to pinpoint
bugs.

Among other things the cache system should be able to satisfy and correctly
work around the following corner cases:

#. Several different versions of single package may be used simultaneously by
   parallel deployments/actions.
#. Package may be deleted or re-uploaded to API (with or without
   version change) while there are ongoing deployments that use previous
   package version.
#. Deployment/action may require to download the package that is currently
   being downloaded for another deployment/action.
#. Packages might need to be eventually invalidated and space constrained.

Generally there are 2 situation when a package cache needs to be invalidated:

#. A new package of the same version and FQN has been updated, meaning old one
   is no longer available, but might still be used by an ongoing deployment
#. A package has not been used for long time (but may be exactly the same as
   the most recent package in the API)


Proposed change
===============

Proposed change includes modifying current cache mechanism to allow it to
persist the packages on disk and not clean them up after deployment/action has
ended.

#. Change cache storage directory path from {id} to {fqn}/{version}/{id}
   This would allow to easily detect outdated versions of the same package.
   Listing {fqn}/{version} directory *before* asking API for an id would give
   engine exact list of packages that are in cache and should be invalidated
#. Whenever engine starts executing a task it would acquire shared *usage*
   lock on 2 levels: eventlet-based lock (to synchronise tasks from the
   current execution task with other execution tasks) and file-system based
   lock, using flock or similar primitives (to synchronise use of package
   cache between different murano-engine processes). The lock should be
   released after the package is no longer required.
#. If the package is not available in cache execution task would attempt to
   acquire exclusive *download* lock on 2 levels (eventlet/file-based), thus
   allowing only one download per id at a time. The lock should be released
   after the download is finished.
#. The task/process, downloading the package would be the one responsible
   for deleting outdated versions of the same package.
   To do so it would acquire exclusive
   *usage* lock on 2 levels for the packages it wishes to delete as part of
   cleanup. This would ensure, that ongoing deployments would not be affected
   by the cleanup.

Usage lock has to be acquired before download lock. Otherwise there is a
race condition where 2 versions of the same package were downloaded within
a very short interval of time by 2 tasks, and the newer version has finished
downloading before the first one did (for example the newer package is
significantly smaller) and engine started cleanup. This could lead to
newer version being the first to acquire the exclusive *usage* lock and
cleaning the package, that is in use.
(Alternatively we could have used just one *usage* lock and upgrade it from
shared to exclusive if the operation would be available for all types of
locks. Unfortunately this is not true for flock/fcntl based locks in
linux/bsd system. While the upgrade/downgrade operation is available it is
not guaranteed to be atomic, therefore we risk a race condition. See
respective man pages for more clarifications. Therefore it's not an option)

If the process crashes or is killed all the
locks held by it would be released. This is ensured by nature of flock/fcntl
locks and is obviously true for eventlet-based locks. Therefore new process
would not get deadlocked by those locks.


Limitations
-----------

This spec only aims to address the problem of invalidating package cache of
packages, that are outdated.
Solving the problem of invalidating of packages, that have not been used in a
while requires a separate cache-manager, process/service, which in turn would
require additional more fine grained locks to be implemented and it doesn't
look like a real problem right now as the size of the packages is relatively
small and the number of packages in a typical murano installation is not that
large to consume all of the space on the server.
However we should be aware of this situation and probably work on that solution
some time in the future.

Alternatives
------------

One of the alternatives would be to add HTTP headers for caching control, for
example If-Modified-Since. While this is a good idea as of itself it would
impact murano-api, python-muranoclient and murano-engine, thus making it a lot
harder to implement and test.

Instead of having 2-layer locking we could implement spin-locks around file
locks, which doesn't look like a good idea though.

Data model impact
-----------------

None

REST API impact
---------------

None

Versioning impact
-----------------

None

Other end user impact
---------------------

None

Deployer impact
---------------

Since package cache would persist on disk — it would be possible to drain disk
resources by it. This is still possible to do today by creating a really large
amount of simultaneous deployments, although in the current situation the
package cache would be eventually deleted, and space would be reclaimed.
If we believe that this is a serious security flaw — we need to implement cache
invalidation/caps of max cache storage before M release

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
  kzaitsev

Work Items
----------

#. Implement caching mechanisms.
#. Implement unit/functional tests.

Dependencies
============

None

Testing
=======

Looks like unit tests would be enough for proper coverage, although
functional tests against race conditions might be benefitial.

Documentation Impact
====================

Usual docs update required

References
==========

* `FreeBSD flock man page
  <https://www.freebsd.org/cgi/man.cgi?query=flock&sektion=2>`_
* `Linux flock man page <http://linux.die.net/man/2/flock>`_

