..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===========================
Fully Asynchronous REST API
===========================

https://blueprints.launchpad.net/ironic/+spec/fully-async-api

Ironic's REST API should not block on any RPC calls to the conductor service.

Problem description
===================

Today, all of Ironic's REST API requests block on the RPC layer, because all
RPC methods are invoked as "call" rather than "cast". Methods currently
block on RPC calls for the following reasons:

* blocking simply so that the conductor can lock the node prior to updating
  the database;
* blocking on confirmation that a long-lived action (eg, deploy) will be
  started;
* blocking on data transfer, eg. because the API must fetch information held
  in the memory state of a conductor, or which the conductor must fetch
  from the physical hardware;

Proposed change
===============

Calls that merely touch the database will ...
XXX: How does nova handle instance-update if n-cond is offline?
This includes the following methods:

* update_node
* destroy_node
* change_node_maintenance_mode
* update_port

Calls that merely wait for confirmation that a conductor received the request
and is capable of starting the requested action will be converted to "cast".
Any preconditions which can be checked in the API will be done within the API,
rather than the conductor. The conductor will merely log a NOTICE for
unservicable requests. The conductor will not reject requests due to
NodeLocked, but will be queued up locally and processed in the order received.
This includes the following methods:

* change_node_power_state
* do_node_deploy
* do_node_tear_down
* set_console_mode

Calls that require fetching information from a conductor will instead fetch it
from the database. This will require data model changes, and may require new
periodic task(s) to keep the data model in sync, such that the API service can
reliably return the requested information without an RPC call.  This includes
the following methods:

* validate_driver_interfaces
* get_console_information

Calls that require the conductor to perform some action in order to determine
the requested information will need a periodic task to keep the data model in
sync. This includes the following methods:

* validation of driver ManagementInterfaces


Alternatives
------------

XXX: investigate how other projects do this and ensure we are doing something
reasonably similar.

Data model impact
-----------------

XXX: determine data model impact to cache state more accurately in the database

REST API impact
---------------

The goal of this change is to have no material impact on the REST API data
structures while making it non-blocking on RPC calls.

Driver API impact
-----------------

This change should have no impact on the Driver API.

Nova driver impact
------------------

This change should have no impact on the Nova driver API.

Security impact
---------------

There is currently a DoS vulnerability whereby any authenticated user can
issue a sufficient number of requests to:

  GET /v1/nodes/NNN/validate

which will block on calling down to driver.*.validate(), including, if the
node uses the IPMIToolPowerDriver, tickling the BMC. This can have two
negative effects:

* it can starve the ironic-conductor processes out of RPC workers, as the
  workers are waiting for IPMI to complete, causing further REST API requests
  to timeout or fail due to having no available workers on the other side of
  the RPC bus.
* it can actually DoS the BMC, causing it to crash.

Both of these are possible even with a relatively low rate of requests.

This change will resolve this DoS vulnerability.

Other end user impact
---------------------

None.

Scalability impact
------------------

This should have a direct and very positive impact on the overall scalability
of Ironic. By decoupling the REST API from the conductor service, API requests
will be serviced faster and with less resources, and the conductor services'
main thread will have significantly fewer synchronous RPC requests to handle.

Performance Impact
------------------

Aside from the aforementioned scalability impact, this should not have any
impact on the performance of the ironic-conductor process.

There will be minor additional load on the database as a result of the API
service requesting information from the DB, instead of from the conductor.

Other deployer impact
---------------------

None.

Developer impact
----------------

None.


Implementation
==============

Assignee(s)
-----------

XXX

Work Items
----------

XXX


Dependencies
============

XXX: Is there a dependent relationship between this change and properly
versioning RPC calls? Should we do one before the other?


Testing
=======

Only minor changes to unit tests will be necessary, and these are also
sufficient.


Documentation Impact
====================

None.


References
==========

XXX: Add references to nova/compute/rpcapi.py and when cast vs call are used.
