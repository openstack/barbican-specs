..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================================================================
Replace the concept of tenants in the code-base in favor of projects
====================================================================

https://blueprints.launchpad.net/barbican/+spec/replace-concept-of-tenants-for-projects

To enforce a uniform and consistent code-base, it is also necessary to make use
of concepts in a consistent way. As a notorious example of this, the incidental
usage of the concepts 'tenant' and 'project' throughout the code-base. The
proposition provided in this blueprint is to finally replace one concept for
the other, and thus end this lack of uniformity.

Problem Description
===================

Even though, it has been talked of following the steps taken by the Keystone
team of starting to enforce the usage of the concept of 'projects' instead of
tenants; there are still 580 occurrences of the word 'tenant' in the Barbican
code-base alone (At the time that this blueprint was written), as opposed to
the 103 occurrences of the word 'project'. This shows lack of uniformity, which
degrades the consistency throughout the code-base.

Proposed Change
===============

Thus, to address the aforementioned issue, I propose to replace every instance
of 'tenants' from the code-base, and enforce the usage of 'projects'. This is
to, not only have a consistent code-base, but also have consistency towards
other projects such as Keystone.

Alternatives
------------

Leave the code-base as it is.

Data model impact
-----------------

There will be impacts in the data model, since the concept of 'tenants' is
actually being used there. Thus, the Tenant and the TenantSecret classes will
be replaced with a Project and ProjectSecret classes respectively. On the other
hand, tables containing columns with the word 'tenant' will also need to be
replaced, and with those, migration scripts will need to be added
(the TenantSecret, Tenant, EncryptedDatum, KEKDatum, Order and Container
classes/tables will be affected).

On the other hand, keystone_id will also be replaced from the Tenant (which
will be called Project) in favor of calling it 'project_id', since this was
initially named due to the confusion as to whether it should be 'tenant_id' or
'project_id'. The proper business logic around this which involves changing how
the context is filled, and how it is passed to the controllers will also
reflect this change. So, in summary, 'keystone_id' will also be replaced by
'project_id'

REST API impact
---------------

None

Security impact
---------------

None

Notifications & Audit Impact
----------------------------

If the word 'tenant' is being used while emitting notifications, this will be
changed to 'project'. Other than that, there will not be any extra impact.

Other end user impact
---------------------

There are certain arguments taken by the python-barbicanclient that will change
due to this.

Performance Impact
------------------

None

Other deployer impact
---------------------

Migration scripts will need to be run on already existing deployments. There
will be no other impact otherwise.

Developer impact
----------------

Developers hopefully will stop using the term 'tenant' in barbican.

Implementation
==============

Assignee(s)

Primary assignee:
    juan-osorio-robles

Work Items
----------

* Replace all trivial instances in the code-base

* Replace instances in the data model (write the proper migration scripts)

* Replace all instances in the python-barbicanclient

Dependencies
============

None

Testing
=======

The current unit tests will also be modified to have this change reflected upon
them.

Documentation Impact
====================

If there exist instances of the term in the wiki, these should be changed also.

References
==========

None
