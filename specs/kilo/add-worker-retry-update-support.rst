..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===========================================
Add worker retry and future updates support
===========================================

Launchpad blueprint:
https://blueprints.launchpad.net/barbican/+spec/add-worker-retry-update-support

The Barbican worker processes need a means to support retrying failed yet
recoverable tasks (such as when remote systems are unavailable) and for
handling updates for long-running order processes such as certificate
generation. This blueprint defines the requirements for this retry and update
processing, and proposes an implementation to add this feature.

Problem Description
===================
Barbican manages asynchronous tasks, such as generating secrets, via datastore
tracking entities such as orders (currently the only tracking entity in
Barbican). These entities have a status field that tracks their state, starting
with PENDING for new entities, and moving to either ACTIVE or ERROR states for
successful or unsuccessful termination of the asynchronous task respectively.

Barbican worker processes implement these asynchronous tasks, as depicted on
this wiki page: https://github.com/cloudkeep/barbican/wiki/Architecture

As shown in the diagram, a typical deployment can include multiple worker
processes operating in parallel off a tasking queue. The queue invokes task
methods on the worker processes via RPC. In some cases, these invoked tasks
require the entity (eg. order) to stay PENDING, either to allow for follow on
processing in the future or else to retry processing due to a temporary
blocking condition (eg. remote service is not available at this time).

The following are requirements for retrying tasks in the future and thus
keeping the tracking entity in the PENDING state::

    R-1) Barbican needs to support extended workflow processes whereby an entity
         might be PENDING for a long time, requiring periodic status checks to
         see if the workflow is completed

    R-2) Barbican needs to support re-attempting an RPC task at some point in
         the future if dependent services are temporarily unavailable

Note that this blueprint does not handle concurrent updates made to the
same entity, say to perform a periodic status check on an order and also apply
client updates to that same order. This will be addressed in a future
blueprint.

Note also that this blueprint does not handle entities that are 'stuck' in the
PENDING state because of lost messages in the queue or workers that crash while
processing an entity. This will also be addressed in a future blueprint.

In addition, the following non-functional requirements are needed in the final
implementation::

    NF-1) To keep entity state consistent, only one worker can work on an
          entity or manage retrying tasks at a time.

    NF-2) For resilience of the worker cluster:

        a) Any worker process (of a cluster of workers) should be able to
           handle retrying entities independently of other worker processes,
           even if these worker processes are intermittently available.

        b) If a worker comes back online after going down, it should be able to
           start processing retry tasks again, without need to synchronize with
           other workers.

    NF-3) In the default standalone Barbican implementation, it should be
          possible to demonstrate the periodic status check feature via the
          SimpleCertificatePlugin class in
          barbican.plugin.simple_certificate-Manager.py.

The following assumptions are made::

    A-1) Accurate retry times are not required:

        a) For example, if a task is to be retried in 5 minutes, it would be
           acceptable if the task was actually retried after more than 5
           minutes. For SSL certificate workflows, where some certificate types
           can take days to process, such retry delays would not be
           significant.

        b) Relaxed retry schedules allow for more granular retry checking
           intervals, and to allow for delays due to excessive tasks in queues
           during busy times.

        c) Excessive delays in retry times from expected could indicate that
           worker nodes are overloaded. This blueprint does not address
           this issue, deferring to deployment monitoring and scaling
           processes.

Proposed Change
===============
This blueprint proposes that for requirements R-1 and R-2, the plugins used by
worker tasks (such as the certificate plugin) determine if tasks should be
retried and at what time in the future. If plugins determine that a task
should be retried, then these tasks will be scheduled for a future retry
attempt.

To implement this scheduling process, this blueprint proposes using the Oslo
periodic task feature, described here:

https://docs.openstack.org/developer/oslo-incubator/api/openstack.common.periodic_task.html

A working example implementation with an older code base is shown here:

https://github.com/cloudkeep/barbican/blob/verify-resource/barbican/queue/server.py#L174

Each worker node could then execute a periodic task service, that invokes a
method on a scheduled basis (configurable, say every 15 seconds). This method
would then query which tasks need to be retried (say if current time >=
retry time), and for each one issue a retry task message to the queue. Once
tasks are enqueued, this method would remove the retry records from the retry
list. Eventually the queue would invoke workers to implement these retry tasks.

To provide a means to evaluate the retry feature in standalone Barbican per
NF-3, the SimpleCertificatePlugin class in
barbican.plugin.simple_certificate_manager.py would be modified to have the
issue_certificate_request() method return a retry time of 5 seconds
(configurable). The check_certificate_status() method would then return a
successful execution to terminate the order in the ACTIVE state.

This blueprint proposes adding two entities to the data model: OrderRetryTask
and EntityLock.

The OrderRetryTask entity would manage which tasks need to be retried on which
entities, and would have the following attributes::

    1) id: Primary key for this record

    2) order_id: FK to the order record the retry task is intended for

    3) retry_task: The RPC method to invoke for the retry. This method could be
                   a different method than the current one, such as to support
                   a SSL certificate plugin checking for certificate updates
                   after initiating the certificate process

    4) retry_at: The timestamp at or after which to retry the task

    5) retry_args: A list of args to send to the retry_task. This list includes
                   the entity ID, so no need for an entity FK in this entity

    6) retry_kwargs: A JSON-ified dict of the kwargs to send to retry_task

    7) retry_count: A count of how many times this task has been retried

New retry records would be added for tasks that need to be retried in the
future, as determined by the plugin as part of workflow processing. The next
periodic task method invocation would then send this task to the queue for
another worker to implement later.

The EntityLock entity would manage which worker is allowed to delete from the
OrderRetryTask table, since per NF-1 above only one worker should be able to
delete from this table. This entity would have the following attributes::

    1) entity_to_lock: The name of the entity to lock ('OrderRetryTask' here).
                       This would be a primary key.

    2) worker_host_name: The host name of the worker that has the
                         OrderRetryTask entity 'locked'.

    3) created_at: When this table was locked.

This entity would only have zero or one records. So the periodic method above
would execute the following pseudo code::

    Start SQLAlchemy session/transaction
    try:
        Attempt to insert a new record into the EntityLock table
        session.commit()
    except:
        session.rollback()
        Handle 'stuck' locks (see paragraph below)
        return

    try:
        Query for retry tasks
        Send retry tasks to the queue
        Remove enqueued retry tasks from OrderRetryTask table
        session.commit()
    except:
        session.rollback()
    finally:
        Remove record from EntityLock table
        Clear SQLAlchemy session/transaction

Lock tables can be problematic if the locking process crashes without removing
the locks. The overall time a worker holds on to a lock should be brief
however, so the lock attempt rollback process above should check for and remove
a stale lock based on the 'created_at' time on the lock.

To separate coding concerns, it makes sense to implement this process in a
separate Oslo 'service' server process, similar to the `Keystone listener
approach <https://github.com/openstack/barbican/blob/master/barbican/queue/keystone_listener.py#L130>`_
This service would only run the Oslo periodic task method, to perform the retry
updating process. If the method failed to operate, say due to another worker
locking resource, it could just return/exit. The next periodic call would then
start the process again.

Alternatives
------------
Rather than having each worker process manage retrying tasks, a separate node
could be designated to manage these retries. This would eliminate the need for
the EntityLock entity. However, this approach would require configuring yet
another node in the Barbican network, adding to deployment complexity. This
manager node would also be a single point of failure for managing retry tasks.

Data model impact
-----------------
As mentioned above, two new entities would be required. No migrations would be
needed.

REST API impact
---------------
None

Security impact
---------------
None

Notifications & Audit Impact
----------------------------
None

Other end user impact
---------------------
None

Performance Impact
------------------
The addition of a periodic task to identify task to be retried presents an
extra load on the worker nodes (assuming they are co-located processes to the
normal worker processing, as expected). However, this process does not perform
the retry work, but rather issues tasks into the queue to then evenly
distribute back to the worker processes. Hence the additional load on a given
worker should be minimal.

This proposal includes utilizing locks to deal with concurrency concerns
across the multiple worker nodes that could be handling retry tasks. This can
result in two performance impacts: (1) multiple workers might fight to grab
the lock simultaneously leading to degraded performance for the workers that
fail to grab the lock, and (2) a lock could become 'stuck' if a worker holding
the lock crashes.

Regarding (1), locks are only utilized on the worker nodes involved in
processing asynchronous tasks which are not time sensitive. Also, the time the
lock is utilized will be very brief, just long enough to perform a query for
retry tasks and to send those tasks to queue for follow on processing. In
addition the periodic process of each worker node handles these retry tasks,
so if the deployment of worker nodes is staggered the retry processes should
not conflict. Another option is to randomly dither the periodic interval (eg.
30 seconds +- 5 seconds) so that worker nodes are less likely to conflict with
each other.

Regarding concern (2) about 'stuck' locks, since the conditions which involve
locks are either long-running orders that can suffer delays until locks are
restored, or else are (hopefully) rare conditions when resources aren't
available, this condition should not be critical to resolve. The proposal does
however suggest a means to remove stuck locks utilizing their created-at
times.

Other deployer impact
---------------------
The Barbican configuration file will need a configuration parameter to
periodically run the retry-query process, called 'schedule_period_seconds',
with a default value of 15 seconds. This parameter would be placed in a new
'[scheduler]' group.

A configuration parameter called 'retry_lock_timeout_seconds' would be used to
release 'stuck' locks on the retry tasks table, as described in the 'Proposed
Change' section above. This parameter would also be added to the '[scheduler]'
group.

A configuration parameter called 'delay_before_update_seconds' would be used to
configure the amount of time the SimpleCertificatePlugin delays from
initiating a demo certificate order to the time the update certificate method
is invoked. This parameter would be placed in a new '[simple_certificate]'
group.

These configurations would be applied and utilized once the revised code base
is deployed.

Developer impact
----------------
None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  john-wood-w

Other contributors:
  Chelsea Winfree

Work Items
----------

1) Add data model entities and unit tests, for OrderRetryTask and EntityLock

2) Add logic to SimpleCertificatePlugin per the Approach section, to allow demonstration of retry feature

3) Modify barbican.tasks.certificate_resources.py's _schedule_retry_task to add retry records into OrderRetryTask table

4) Add Oslo periodic task support

5) Implement periodic method, that performs the query for tasks that need to be retried

6) Implement workers sending retry RPC messages back to the queue...see note below

7) Add new scripts to launch the Oslo periodic task called bin/barbican-task-scheduler.py and .sh, similar to bin/barbican-keystone-listener.py and .sh

8) Add to the Barbican Devstack gate functional tests a test of the new retry feature via the SimpleCertificatePlugin logic added above

9) Add logic to handle expired locks on the OrderRetryTask table

Note that for #6, the 'queue' and 'tasks' packages have to be modified somewhat
to allow the server logic to send messages to the queue via the client logic,
mainly to break circular dependencies. Again, see the example `here <https://github.com/cloudkeep/barbican/tree/verify-resource/barbican>`_
for a working example of this server/client/retry processing.

Dependencies
============
None

Testing
=======
In addition to planned unit testing, the functional Tempest-based tests in the
Barbican repository would be augmented to add a test of the new retry feature
for the default certificate plugin.

Documentation Impact
====================
Developer guides will need to updated, to include the additional periodic retry
process detailed above. Deployment guides will need to be updated to specify
that a new process needs to executed (for the bin/barbican-task-scheduler.sh
process).

References
==========
None
