..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================================
Add auditing capability via CADF based notification events
==========================================================

https://blueprints.launchpad.net/barbican/+spec/audit-cadf-events

This proposal is to add auditing capability in Barbican using CADF (Cloud Audit
Data Federation) specification. The idea is to identify auditable attributes and
construct audit data record as per CADF event specification and provide delivery
option via OpenStack notification framework to interested services and consumers.

DMTF CADF standard provides auditing capabilities for compliance with security,
operational and business processes. See References section in end for more details
around CADF specification.

Problem Description
===================

Large number of enterprise faces challenge when moving to OpenStack cloud.
And most of these challenges have nothing to do with OpenStack service capabilities.
Instead, the challenges are in terms of auditing and monitoring their workload and
data in accordance with their strict corporate, industry or regional policies and
compliance requirements (FIPS-140-2, PCI-DSS, SoX, ISO 27017 etc).
Barbican, being a security component in a cloud, has similar auditing requirement.

In addition, Barbican has this concept of soft deletes where it keeps resources in
database even after they are marked deleted and are no longer used in live systems.
The idea is to have a sense of audit trail of its resources state. This approach
does not provide much value in terms of audit other than when it was marked deleted.

Proposed Change
===============

To add auditing capability in Barbican, proposal is to leverage a existing standard
instead of creating own semantics and audit data model. In OpenStack ecosystem,
number of services (Ceilometer, Keystone) are using CADF based event data model to
report activities on its resources. Using CADF standard for auditing will allow
consistent reporting across services and will allow customers to use common audit
tools and processes for their audit data.

The CADF model can answer critical questions about activity or event happening on
rest resources in a normative manner using CADF's seven W's of auditing (What,
When, Who, On What, Where, From Where, To Where). See Auditing Event Details section
below.

The overhead of adding audit data interaction can be minimized by publishing
audit event as OpenStack notifications where event publisher does not have to wait
for response/ acknowledgment. This way audit event delivery is decoupled and can
still benefit from messaging infrastructure durability and delivery guarantees.

Auditing Event Details
-----------------------

Audit event data is constructed using who initiated request, request outcome, resource
operated on, resource identifier and event type information. Following is sample
of seven W\'s of auditing values for Barbican rest resources.

  +-------------+-------------+----------------------+------------------------------+
  | W Component | Resource    | CADF                 | Possible                     |
  |             |             | Properties           | Values                       |
  +=============+=============+======================+==============================+
  |  What       | POST/PUT/   | event.type           |activity/monitor/control      |
  |             | DELETE      | event.outcome        |success/failure/pending       |
  |             | v1/secrets  |                      |                              |
  +-------------+-------------+----------------------+------------------------------+
  |  When       |             | event.eventTime      |{event generation timestamp}  |
  +-------------+-------------+----------------------+------------------------------+
  |  Who        |             | initiator.id         |{token user id}               |
  |             |             | initiator.type       |service/security/account/user |
  |             |             | initiator.project_id |{scoped token project id}     |
  +-------------+-------------+----------------------+------------------------------+
  |  From Where |             | initiator.host       |environemnt REMOTE_ADDR       |
  |             |             | initiator.agent      |environment HTTP_USER_AGENT   |
  +-------------+-------------+----------------------+------------------------------+
  |  On What    |             | target.id            |resource id (secret/container)|
  |             |             | target.type          |data/keymgr/secret            |
  +-------------+-------------+----------------------+------------------------------+
  |  Where      |             | observer.id          |target                        |
  |             |             | observer.type        |service/keymgr                |
  +-------------+-------------+----------------------+------------------------------+
  |  To Where   |             |                      |                              |
  +-------------+-------------+----------------------+------------------------------+

CADF extension properties (mentioned below) can be used to capture domain specific
data available as part of rest request. Currently there is no plan to add these
properties as this may need per resource awareness in common decorator logic.

- event.attachments (structured or unstructured data)
- event.tags (domain specific identifiers/ classifications)


**pyCADF changes**

Creation of audit event requires oslo pyCADF library. pyCADF library needs to be
updated to reflect Barbican specific resource types. As per pyCADF contributors,
either use existing types or create very few new generic types. Following Barbican
specific resource types are going to be added in pyCADF resource taxonomy.

#. data/keymgr/secret

#. data/keymgr/container

#. data/keymgr/order

#. data/keymgr  - general placeholder for all other remaining resourcs.

#. service/keymgr

Above types are going to be added in pyCADF taxonomy as mentioned in link below.

* pyCADF resource taxonomy
  https://github.com/openstack/pycadf/blob/master/pycadf/cadftaxonomy.py#L111

Audit Event Generation
----------------------

For API requests, audit event are going to be generated using audit middleware approach.
This middleware is now available as part of keystonemiddleware (> 1.5) library. This
library has Keystone middleware which is used by Barbican for Keystone token validation.

Audit middleware is going to create two events per Barbican REST API invocation. One with
information extracted from request data and the second correlated one with request outcome
(response). The mapping information is going to be managed via barbican specific configuration
file which is going to be similar to files described in following pycadf samples link.
https://github.com/openstack/pycadf/tree/master/etc/pycadf

For asynchronous task processing workers, audit event is constructed using task related
data. It can be be implemented as a decorator which can be added to each task common
methods (handle_processing, handle_success and handle_error) or can be added on base task
process method. This audit event is going to be published as notification to same queue
which is used by audit middleware as well.


Audit Event Delivery
--------------------

Internally audit middleware uses an oslo messaging based notifier to publish CADF
events to configured messaging infrastructure. Audit decorators will use similar
approach.

Audit middleware is added to Barbican request pipeline via additional filter in
`barbican-api-paste.ini` where related audit mapping file path is defined. Delivery of
audit event via decorator needs to be configurable as developer boxes may not necessarily
have the needed setup. By default, audit delivery as notification is going to disabled.

The oslo messaging framework supports publish of this audit data to messaging queue via
'messagingv2' driver or can be written to log files via its 'log' driver.


Alternatives
------------

We could rely on Barbican existing logging but that does not provide a complete
and consistent picture of service audit data. Non-standard logging means that cloud
provider need service specific audit tools to aggregate and analyze the logs.

Security impact
---------------

This improves security in the stack by providing audit capability. This will
help in addressing some of compliance requirements expected from Barbican service.

Notifications & Audit Impact
----------------------------

Will have additional notification capability to publish audit events.

Other end user impact
---------------------

None

Performance Impact
------------------

Audit events are published as notifications to queue and does not have to wait for
response/acknowledgment so overall associated overhead should be very minimal.
When notifications are written to log files, related overhead should still be low and
can be comparable to logic of adding 2 log statements.

Other deployer impact
---------------------

To enable audit event delivery via notifications, deployer will need to change
default configuration.

Developer impact
----------------

None.

Implementation
==============

Assignee(s)

Primary assignee:
    arunkant

Work Items
----------

* Add new blueprint and update oslo pyCADF library.

* Define pipeline filter with audit map configuration file barbican_api_audit_map.conf

* Add new decorator to create CADF event data for asynchronous worker processing logic.
  Add decorator to related worker task methods.

* Add ability to turn on audit event generation. By default it needs to be off.


Dependencies
============

* pyCADF library (with barbican specific updated taxonomy).

Testing
=======

The new unit tests are going to be added for middleware and order processing flow.
For middleware unit tests, configuration override is needed for paste and api ini files.
Oslo test driver https://github.com/openstack/oslo.messaging/blob/master/oslo_messaging/notify/_impl_test.py
can be used to verify message content in addition to mock patched methods approach.


Documentation Impact
====================

CADF event usage should be documented. For order processing flow, document what audit
related changes, if any, are needed to add audit support for new order types.

References
==========

* CADF Working Group. http://www.dmtf.org/standards/cadf

* CADF representation for OpenStack
  http://www.dmtf.org/sites/default/files/standards/documents/DSP2038_1.0.0.pdf
 
* Audit Middleware
  https://docs.openstack.org/developer/keystonemiddleware/audit.html

* PyCADF developer docs
  https://docs.openstack.org/developer/pycadf/

* CADF event model and taxonomies
  https://wiki.openstack.org/w/images/e/e1/Introduction_to_Cloud_Auditing_using_CADF_Event_Model_and_Taxonomy_2013-10-22.pdf

* Ceilometer CADF support
  https://wiki.openstack.org/wiki/Ceilometer/blueprints/support-standard-audit-formats
