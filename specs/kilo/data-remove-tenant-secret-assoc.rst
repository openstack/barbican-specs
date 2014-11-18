..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Remove the tenant-secret association table
==========================================

https://blueprints.launchpad.net/barbican/+spec/data-remove-tenant-secret-assoc

Currently all secrets stored in Barbican are indirectly associated with
Keystone 'projects' (formerly known as 'tenants') via a tenant-secret
association table. This association was added early in Barbican's lifecycle,
when it seemed that such an association would aid in retrieving different
components of grouped secrets with different RBAC behaviors. However, recent
discussions with OpenStack projects have indicated that this feature would be
better implemented in a different way. Hence this association adds an
additional join on selects with no benefit, and therefore should be removed.


Problem Description
===================

Secrets in Barbican are indirectly related to Keystone 'projects' via a
TenantSecret association table. This association is not utilized for any
features in Barbican currently but was envisioned to allow multiple 'projects'
to access the same secret with different access privileges. The team has since
had numerous discussions with OpenStack projects that wish to integrate with
Barbican, and access control could either be accomplished via Keystone Trusts,
or else via 'user' and 'project' level policy based on metadata configured at
the secret level (see blueprint at [1]_)

.. [1] https://review.openstack.org/#/c/127353/

Hence the TenantSecret association is not required for any envisioned Barbican
use cases and therefore represents a wasteful join for all secret selects.

Proposed Change
===============

This blueprint proposes removing the unused TenantSecret association.

Alternatives
------------

None

Data model impact
-----------------

The TenantSecret table would be removed, along with associated SQLAlchemy
models and repository queries. This schema modification would require a
non-trivial migration for zero downtime, as detailed in the 'Work Items'
section below.


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

Removing the TenantSecret association removes one join from secret-related
queries, and removes Python code needed to create the SQLAlchemy TenantSecret
model. Hence performance and code readability should be improved.

Other deployer impact
---------------------

Migration scripts would need to be run on already existing deployments. There
will be no other impact otherwise.

Developer impact
----------------

None

Implementation
==============

Assignee(s)

Primary assignee:
    john-wood-w

Work Items
----------

The schema modification called for would need to be phased in to avoid breaking
existing deployments with secrets stored using the current TenantSecret
associations.

The first phase would add the new and initially nullable 'tenant_id' FK
relationship to the secret entity, with all new secrets added setting this FK,
and still adding the TenantSecret relationships (until older versions are
rotated out). Secret retrievals would first attempt the simple query by FK, and
fallback to the current TenantSecret join. The following mods would be needed::
* models.py: Add a 'secrets_assoc' relationship in the Tenant model class to
the secrets model, similar to the 'Container' relationship. To the Secret
model, add a 'tenant_id' FK reference similar to the Container model's
'tenant_id' FK reference, but with nullable=True.
* repositories.py: In the SecretRepo
repository class add a query for secrets without the TenantSecret join, but on
errors revert to the existing TenantSecret join query.
* plugin/resources.py: Add setting the new 'tenant_id' FK on secrets to the
'_save_secret()' function.
* store_crypto.py: Add setting the new 'tenant_id' FK on secrets to the
'_store_secret_and_datum()' function.


The second phase would provide an Alembic migration script to set the
'tenant_id' FK on all secrets that don't already have it set. The 'tenant_id'
FK would then be set as nullable=False.


The third phase would remove all logic related to the TenantSecret
relationship. The following mods would be needed::
* models.py: Remove TenantSecret SQLAlchemy model class.
* repositories.py: Remove TenantSecret joins from queries in the SecretRepo
repository class. Remove the ProjectSecretRepo repository class. Remove
'ProjectSecretRepo' from the Repositories class. Remove the
'get_project_secret_repository' function.
* plugin/resources.py: Remove 'new_assoc' lines from the '_save_secret()''
function.
* store_crypto.py: Remove 'new_assoc' lines from the
'_store_secret_and_datum()' function.
* api/test_resources.py: Remove 'project_secret_repo' lines from the
'_test_should_add_new_secret_metadata_without_payload' test method.
* test_repositories.py: Remove all 'TenantSecret' related lines.
* test_store_crypto.py: Remove all 'project_secret_repo' lines.


The fourth phase would provide an Alembic migration script to remove the
TenantSecret relationship from all secrets, and to then drop the TenantSecret
table altogether.


Dependencies
============

None

Testing
=======

The current unit tests will also be modified to have this change reflected upon
them. Migration testing across the different phases would be needed, at least
manually.

Documentation Impact
====================

None

References
==========

None
