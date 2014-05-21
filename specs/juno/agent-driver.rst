..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============
Agent Driver
============

https://blueprints.launchpad.net/ironic/+spec/agent-driver

Ironic needs a deploy driver to interact with
`Ironic Python Agent <https://wiki.openstack.org/wiki/Ironic-python-agent>`_.

Problem description
===================

Today, Ironic is limited in the tasks that mat be performed on a bare metal
node. Actions like 'update firmware' or 'secure erase disks' are not possible.
This is not due to any flaw in Ironic itself, but rather a limitation of the
deploy ramdisk used by Ironic's PXE driver.

Ironic Python Agent (IPA) is a project to provide a deploy agent for use by
Ironic. This agent is designed to run in a ramdisk and perform maintenance
on bare metal nodes, including hardware configuration and management,
provisioning, and decommissioning of servers. This agent exposes a REST API
that Ironic could call in order to perform various tasks.

Potential use cases:

* It is usually advantageous for a server to be running the latest BIOS
  firmware. Updating firmware is a critical feature in Ironic.

* End users often prefer that their data is erased when a server is released.
  The ability to run secure erase on a bare metal node's disks is another
  critical feature for Ironic.

* End users may have varying workloads they wish to deploy to a bare metal
  node. Some users may wish to run an application on bare metal, where others
  may wish to run a hypervisor. These different workloads may require
  different BIOS configurations, such as switching the VT bit on or off. This
  utility ramdisk enables Ironic to manage these configurations.

* Some users may desire faster boot times for a single bare metal node. The
  utility ramdisk may be used to accomplish this by leaving the node running
  the ramdisk, ready for deployment. This removes one POST cycle compared to
  the PXE driver. Additionally, the ramdisk could write popular images to the
  boot device to remove the time spent writing the image.

* End users may wish to use cloud-init with a configdrive to load data such
  as SSH keys or network configurations. The ramdisk can write a partition
  containing the configdrive to be used by the end user and their image.

Ironic needs a deploy driver that interacts with the agent, rather than the
existing deploy ramdisk.

Proposed change
===============

A full deploy driver that interacts with the agent should be implemented.

The driver will:

* Allow the agent to look up Ironic's UUID for the node it is running on.

* Allow the agent to periodically heartbeat. The driver should use this
  heartbeat to verify that the agent is online.

* Leverage the periodic heartbeat as a callback mechanism.              ============= left off here


A utility ramdisk should be built that runs an agent that exposes a REST API
that Ironic can call to run tasks on the bare metal node. There should be
a reference implementation for building a ramdisk agent that runs this agent.

The work to complete this blueprint consists of the following:



* An agent should be implemented that exposes a REST API, which Ironic may
  call



Here is where you cover the change you propose to make in detail. How do you
propose to solve this problem?

If this is one part of a larger effort make it clear where this piece ends.
In other words, what is the scope of this effort?

Alternatives
------------

What other ways could we do this thing? Has someone else done this thing in
another project? In another language? Why aren't we using those? This doesn't
have to be a full literature review, but it should demonstrate that thought has
been put into why the proposed solution is an appropriate one.

Data model impact
-----------------

Changes which require modifications to the data model often have a wider impact
on the system.  The community often has strong opinions on how the data model
should be evolved, from both a functional and performance perspective. It is
therefore important to capture and gain agreement as early as possible on any
proposed changes to the data model.

Questions which need to be addressed by this section include:

* What new data objects and/or database schema changes is this going to
  require?

* What database migrations will accompany this change?

* How will the initial set of new data objects be generated, for example if you
  need to take into account existing instances, or modify other existing data,
  describe how that will work.

REST API impact
---------------

Each API method which is either added or changed should have the following

* Specification for the method

  * A description of what the method does suitable for use in
    user documentation

  * Method type (POST/PUT/GET/DELETE/PATCH)

  * Normal http response code(s)

  * Expected error http response code(s)

    * A description for each possible error code should be included
      describing semantic errors which can cause it such as
      inconsistent parameters supplied to the method, or when an
      instance is not in an appropriate state for the request to
      succeed. Errors caused by syntactic problems covered by the JSON
      schema defintion do not need to be included.

  * URL for the resource

  * Parameters which can be passed via the url, including data types

  * JSON schema definition for the body data if allowed

  * JSON schema definition for the response data if any

* Example use case including typical API samples for both data supplied
  by the caller and the response

* Discuss any policy changes, and discuss what things a deployer needs to
  think about when defining their policy.

Note that the schema should be defined as restrictively as possible. Parameters
which are required should be marked as such and only under exceptional
circumstances should additional parameters which are not defined in the schema
be permitted.

Use of free-form JSON dicts should only be permitted where necessary to allow
divergence in the drivers. In such case, the drivers must expose the expected
content of the JSON dict and an ability to validate it.

Reuse of existing predefined parameter types is highly encouraged.

Driver API impact
-----------------

Changes which affects the driver API have a direct effect on all drivers, and
often have a wider impact on the system. There are several things to consider
in this section.

* Is it a change to a "core" or "common" API?  

* Can all drivers support it initially, or is it specific to a particular
  vendor's hardware? 

* How will it be tested in the gate and in third-party CI systems?

* If adding a new interface, explain the intended scope of the proposed
  interface, what functionality it enables, why it is needed, and whether it is
  supported by current drivers.

* If adding or changing a method on an existing interface, the impact on
  existing drivers should be explored.

* Will the new interface or method need to be invoked when the hash ring
  rebalances, for example to rebuild local state on a new conductor service?

Nova driver impact
------------------

Chances are, if this change affects the REST or Driver APIs, it will also
affect the Nova driver in some way. Questions which need to be addressed in
this section include:

* What is the impact on Nova?

* If this change is enabling new functionality exposed via Nova, this section
  should cite the relevant components within other Nova drivers that alraedy
  implement this.

* Ironic and Nova services must be upgradable independently. If the change is
  affecting existing functionality of the nova.virt.ironic driver, how will an
  upgrade be performe? How will it be tested?

Security impact
---------------

Describe any potential security impact on the system.  Some of the items to
consider include:

* Does this change touch sensitive data such as tokens, keys, or credentials?

* Does this change affect the accessibility of hardware managed by Ironic?

* Does this change alter the API in a way that may impact security, such as
  a new way to access sensitive information or a new way to login?

* Does this change involve cryptography or hashing?

* Does this change require the use of sudo or any elevated privileges?

* Does this change involve using or parsing user-provided data? This could
  be directly at the API level or indirectly such as changes to a cache layer.

* Can this change enable a resource exhaustion attack, such as allowing a
  single API interaction to consume significant server resources? Some examples
  of this include launching subprocesses for each connection, or entity
  expansion attacks in XML.

For more detailed guidance, please see the OpenStack Security Guidelines as
a reference (https://wiki.openstack.org/wiki/Security/Guidelines).  These
guidelines are a work in progress and are designed to help you identify
security best practices.  For further information, feel free to reach out
to the OpenStack Security Group at openstack-security@lists.openstack.org.

Other end user impact
---------------------

Aside from the API, are there other ways a user will interact with this
feature?

* Does this change have an impact on python-ironicclient? What does the user
  interface there look like?

* Will this require changes in the Horizon panel?

Scalability impact
------------------

Describe any potential scalability impact on the system, for example any
increase in network, RPC, or database traffic, or whether the feature
requires synchronization across multiple services.

Examples of things to consider here include:

* Additional network calls to internal or external services.

* Additional disk or network traffic that will be required by the feature.

* Any change in the number of physical nodes which can be managed by each
  conductor service.

Performance Impact
------------------

Describe any potential performance impact on the system, for example
how often will new code be called, and is there a major change to the calling
pattern of existing code.

Examples of things to consider here include:

* A periodic task might look like a small addition, but all periodic tasks run
  in a single thread so a periodic task that takes a long time to run will
  have an effect on the timing of other periodic tasks.

* A small change in a utility function or a commonly used decorator can have a
  large impacts on performance.

* Calls which result in a database queries (whether direct or via conductor)
  can have a profound impact on performance when called in critical sections of
  the code.

* Will the change include any TaskManager locking, and if so what
  considerations are there on holding the lock?

* How will the new code be affected if the hash ring rebalances while it is
  running?

Other deployer impact
---------------------

Discuss things that will affect how you deploy and configure OpenStack
that have not already been mentioned, such as:

* What config options are being added? Should they be more generic than
  proposed (for example a flag that other hypervisor drivers might want to
  implement as well)? Are the default values ones which will work well in
  real deployments?

* Is this a change that takes immediate effect after its merged, or is it
  something that has to be explicitly enabled?

* If this change is a new binary, how would it be deployed?

* Please state anything that those doing continuous deployment, or those
  upgrading from the previous release, need to be aware of. Also describe
  any plans to deprecate configuration values or features.  For example, if we
  change the directory name that instances are stored in, how do we handle
  instance directories created before the change landed?  Do we move them?  Do
  we have a special case in the code? Do we assume that the operator will
  recreate all the instances in their cloud?

Developer impact
----------------

Discuss things that will affect other developers working on OpenStack,
such as:

* If the blueprint proposes a change to the driver API, discussion of how
  other drivers would implement the feature is required.


Implementation
==============

Assignee(s)
-----------

Who is leading the writing of the code? Or is this a blueprint where you're
throwing it out there to see who picks it up?

If more than one person is working on the implementation, please designate the
primary author and contact.

Primary assignee:
  <launchpad-id or None>

Other contributors:
  <launchpad-id or None>

Work Items
----------

Work items or tasks -- break the feature up into the things that need to be
done to implement it. Those parts might end up being done by different people,
but we're mostly trying to understand the timeline for implementation.


Dependencies
============

* Include specific references to specs and/or blueprints in ironic, or in other
  projects, that this one either depends on or is related to.

* If this requires functionality of another project that is not currently used
  by Ironic, document that fact.

* Does this feature require any new library dependencies or code otherwise not
  included in OpenStack? Or does it depend on a specific version of library?

* Does this feature target specific hardware? If so, is it a common standard
  (eg IPMI) or a vendor-specific implementation (eg iLO)?

Testing
=======

Please discuss how the change will be tested. We especially want to know what
tempest tests will be added. It is assumed that unit test coverage will be
added so that doesn't need to be mentioned explicitly, but discussion of why
you think unit tests are sufficient and we don't need to add more tempest
tests would need to be included.

Is this untestable in gate given current limitations (specific hardware /
software configurations available)? If so, are there mitigation plans (3rd
party testing, gate enhancements, etc)?


Documentation Impact
====================

What is the impact on the docs team of this change? Some changes might require
donating resources to the docs team to have the documentation updated. Don't
repeat details discussed above, but please reference them here.


References
==========

Please add any useful references here. You are not required to have any
reference. Moreover, this specification should still make sense when your
references are unavailable. Examples of what you could include are:

* Links to mailing list or IRC discussions

* Links to notes from a summit session

* Links to relevant research, if appropriate

* Related specifications as appropriate (e.g.  if it's an EC2 thing, link the
  EC2 docs)

* Anything else you feel it is worthwhile to refer to
