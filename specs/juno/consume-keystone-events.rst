..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================================
Consume Keystone Project Delete Events
======================================

https://blueprints.launchpad.net/barbican/+spec/consume-keystone-events

In most of openstack deployments, barbican is expected to be integrated with
keystone for its resource authorization needs. Keystone and barbican share
keystone project as a common entity reference. Which means, project needs to
exist in keystone before barbican resources (e.g. secrets and containers) can
be added for that project. But when project is deleted in keystone, there is
no action taken by barbican to reflect the change and it keep maintaining that
project dangling resource references as-is. This blueprint is addressing
related issues in this area.


Problem Description
===================

Barbican resources are managed at keystone project level. As part of barbican
resource creation, barbican tenant (a.k.a. project) is mapped to keystone
project and resource is associated with the tenant. Over time, many more
resources are added to the tenant. Now when project is deleted from keystone,
currently barbican stays unaware of that change and keeps its resources as-is.
Those project resources are now invalid and are kind of dangling references to
a non existing project.

* As barbican feature set expands, the amount of such tenant resources will
  grow over time which can negatively impact performance of db operations.

* Hardware Security Modules (HSM) has a upper limit on the number of key
  encryption keys it can store. These numbers are correlated to number of
  projects in barbican. Without project status awareness, related invalid keys
  cleanup cannot be done to reclaim space for new keys in HSM.

* In a corner case of PKI token service usage, for a small time window,
  unexpired PKI token(s) can access a deleted project resource as it has yet
  to fetch and process related revocation events from keystone. By making
  barbican aware of deleted projects, even those PKI token would not have
  access to barbican resources. Just to be clear, in general PKI token would
  not have this issue if revocation events are fetched pretty frequently.


Proposed Change
===============

Keystone publishes notifications whenever a project is created, updated and
deleted. Proposal is to add notification listener in barbican to consume
keystone public events.

Keystone event payload has project id as resource_info and barbican event
listener logic will propagate it to make changes on resources associated with
that project.

Following is keystone sample notification on project delete::

   Message: ctxt = [{}], metadata = [{'timestamp': u'2014-06-12 00:20:03.584997', 'message_id': u'1ade0b2b-1584-48b9-a026-64bd06659baf'}]
   Message: severity = [info], publisher_id = [identity.arunkant-uws], event_type = [identity.project.deleted]
   Message: payload = [{u'resource_info': u'00ac7ea2a1a3486284c8e2af27b7bc9e'}]

Event listener will use oslo messaging library which provides a abstraction to
allow usage of different type of messaging transport.

* Deletion of a keystone project will invoke delete on related secret(s),
  container(s), tenant entity. Currently entity delete is a soft delete as
  specific fields are marked deleted but record remains in datastore. In the
  event that a project is disabled, Keystone will invalidate the tokens for
  that project and the secrets owned by that project will be inaccessible.
  The secrets owned by Barbican won't be soft deleted but they will be
  unreachable.

* Keystone sends various notification events based on its own resource types
  (e.g. user, project, role etc.) and type of operation (e.g. created,
  updated, deleted etc.). This proposal scope is to act on keystone project
  delete event *only*.

* Barbican will add a queue to exchange to listen keystone notifications.
  Queue specific attributes e.g. exchange name and type, queue name, binding
  pattern etc. are going to be configurable in barbican configuration. Some of
  these attribute values need to align with keystone notification
  configuration.

* Keystone currently publishes notification only with ``info`` severity. The
  binding pattern of ``notifications.*`` will handle that.

* Barbican queue is going to be durable to survive broker restart.

* This feature is going to be configurable and not enabled by default similar
  to keystone authentication via pipeline configuration.

* Notifications are acknowledged only when barbican listener has processed
  event information. Auto-acknoledgement is going to be ``false``.

* Barbican connection to message queue is authenticated using username and
  password. These credentials are going to be defined as part of barbican
  configuration.

* Depending on SSL support available in oslo messaging for a specific
  transport provider, related configuration is going to be added in barbican
  configuration.

* In initial version, there is no trust established between messaging queue
  and barbican instance(s). KDS (Key distribution service, Kite) may help
  around this in future once other services start using it for this kind of
  functionality.

* Changes on resource side will be committed only when complete event
  processing is done otherwise change will be rolled back.

* Once such event support is available in barbican, those events can be
  propagated to barbican's external sub-components if it makes sense. One
  example of such sub-comonent can be clean up of key encrpyion keys in HSM
  (when this feature is added in future).

Alternatives
------------

There can be other ways to get keystone project information like polling model
or accessing keystone datastore directly but there are not scalable and may
introduce tight coupling between services. Events based asynchornous apporach
is better as keystone notifications are primarily intended for other openstack
services to take action based on events e.g. cleanup of their resources.

Data model impact
-----------------

There is no data model impact as we are going to utilize existing entity soft
delete functionality.

REST API impact
---------------

None

Security impact
---------------

A keystone public event does not contain sensitive data by itself. Event data
has information about operation and resource id.

* As implication of those events will result in deleting barbican resources
  so clear logging of events and action taken will be needed.

* Once auditing support is added in barbican, the project delete event will
  be one type of audit activity.

* Message queue connection information needs to be secured similar to other
  external resources connections e.g. database.

Notifications & Audit Impact
----------------------------

A new notification listener is added. Related auditing information is going to
be added to system.

Other end user impact
---------------------

None

Performance Impact
------------------

There should not be performance impact other than new messgae handling server
is added on same host system.

* As result of this change, barbican number of connections to db system may
  increase depending on load of actionable events. Generally the number of
  such events will be quite less considering keystone projects are likely not
  to deleted regularly.

* Depending on keystone event activity, a deployer can choose to enable
  notification listener on some of barbican instances assuming there is pool
  of barbican instances in a setup.

Other deployer impact
---------------------

* This feature needs to be enabled as default configuration will have it
  disabled.

* Related notification listener configuration needs to be configured as per
  deployer's existing messaging infrastructure setup.

* A message handling server is added, as a new process, to transport
  notification events from queue to barbican.

Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Arun Kant <arun.kant@hp.com>

Other contributors:
  ??

Work Items
----------

* Need to add notification listener configuration

* Implement notification listener and message handling server using oslo
  messaging packages.

* Identify actionable events and process to make changes on barbican
  resources.

* Implement these actions as a unit and rollback in case of a processing
  error. One option, needs to be investigated, is to do all operations within
  a SQLAlchemy session.

* If missing, add checks in order and other resource API so that create and
  update of resource is not allowed for deleted tenants (a.k.a. keystone
  projects)


Dependencies
============

None


Testing
=======

Add any itegration test provided needed messaging support is available.


Documentation Impact
====================

There will be additional documentation around notification listener
configuration. Possibly a new option similar to following link

https://github.com/cloudkeep/barbican/wiki/Barbican-Options:-authentication-with-Keystone


References
==========

* http://docs.openstack.org/developer/oslo.messaging/notification_listener.html

* http://docs.openstack.org/developer/oslo.messaging/index.html

* http://docs.openstack.org/developer/oslo.messaging/notification_listener.html#oslo.messaging.MessageHandlingServer
