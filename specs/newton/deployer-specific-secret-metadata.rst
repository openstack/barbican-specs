..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================================
Deployer Specific Metadata for Secrets
======================================

Blueprint:
https://blueprints.launchpad.net/barbican/+spec/deployer-specific-secret-metadata


Problem Description
===================

Deployers(Service Admins) may require the ability to add additional data to
a Barbican Secret, which users cannot access/modify. The data must be immutable
for to the user, but a deployer should be able to change it. Secrets contain
immutable metadata such as `created`, `updated`, `algorithm`, `etc.`. The
deployer may want to add some data which is also immutable for the user. For
example, the deployer may want to store specifics such as the `region` where
the secret is located.

Currently only user metadata can be used, and it is not immutable to a user[1].

Proposed Change
===============

The proposed change will be to add a new attribute to Barbican
Secrets in order for deployer meta-data to be stored. A new API
endpoint must also be created for the manipulation of the metadata.

Alternatives
------------
* Lock usage of metadata and have database admins add keys and values.

Data model impact
-----------------

A new table will be created called `secret-deployer-metadata` with `secret_id`,
`key`and `value` columns.

At the database level a limit will also be placed on the size of the
`metadata`.

.. note::

    Values will all be stored as Strings.

REST API impact
---------------

Each current API call for Secrets will now have `deployer-metadata` as an
added attribute. If no `deployer-metadata` is provided then an empty dictionary will be
initialized.

The following will be added to the REST API:

Get deployer-metadata from a secret::

    GET /v1/secrets/{uuid}/deployer-metadata
    Headers:
        Accept: application/json
        X-Project-Id: {project_id}

    200 OK

    {
        'deployer-metadata': {
            'description': 'contains the AES key',
            'geolocation': '12.3456, -98.7654'
        }
    }

Create/Update deployer-metadata for a secret::

    PUT /v1/secrets/{uuid}/deployer-metadata
    Headers:
        Accept: application/json
        X-Project-Id: {project_id}

    Content:
    {
        'deployer-metadata': {
            'description': 'contains the AES key',
            'geolocation': '12.3456, -98.7654'
        }
    }

    200 OK

    {
        'deployer-metadata': {
            'description': 'contains the AES key',
            'geolocation': '12.3456, -98.7654'
        }
    }

.. note::

    Only Create/Update deployer-metadata will be needed. To remove the metadata
    a user can perform the PUT with an empty dict. If a partial model is
    sent then the whole metadata will be changed to the partial model which
    has been sent. Values that exist in the data model but not in the PUT
    will be deleted.

The following will be added to the REST API in order to address individual
user metadata items:

Create an individual metadata item in a secret::

    POST /v1/secrets/{uuid}/deployer-metadata
    Headers:
        Accept: application/json
        X-Project-Id: {project_id}

    Content:
    {
      "key": "access-limit",
      "value": 11
    }

    201 Created

    Secret Metadata Location: http://example.com:9311/v1/secrets/{uuid}/deployer-metadata/access-limit
    {
        "key": "access-limit",
        "value": 11
    }

.. note::

    If the item already exists then a 409 Conflict error
    code will be returned.

Update an individual metadata item in a secret::

    PUT /v1/secrets/{uuid}/deployer-metadata/access-limit
    Headers:
        Accept: application/json
        X-Project-Id: {project_id}

    Content:
    {
      "key": "access-limit",
      "value": 11
    }

    200 OK

    {
      "key": "access-limit",
      "value": 11
    }

.. note::

    access-limit must already have been created if not a 404 error code will
    be returned.

Get an individual metadata item in a secret::

    GET /v1/secrets/{uuid}/deployer-metadata/access-limit
    Headers:
        Accept: application/json
        X-Project-Id: {project_id}

    200 OK

    {
        "key": "access-limit",
        "value": 0
    }

.. note::

    If the `access-limit` key does not exist then a 404 error code will
    be returned.

Delete an individual metadata item in a secret::

    DELETE /v1/secrets/{uuid}/deployer-metadata/access-limit
    Headers:
        X-Project-Id: {project_id}

    204 No Content

.. note::

  If the `access-limit` key does not exist then a 404 error code will
  be returned.

Security impact
---------------

ACLs and Policy must be setup for the new API calls listed above.

Barbican's policy.json will now include the following:

* "secret-deployer-meta:get": "rule:service_admin"
* "secret-deployer-meta:post": "rule:service_admin"
* "secret-deployer-meta:put": "rule:service_admin"
* "secret-deployer-meta:delete": "rule:service_admin"


Notifications & Audit Impact
----------------------------

If supported, adding/modifying secret-deployer-metadata should be audit events.


Other end user impact
---------------------

None

Performance Impact
------------------

Minimal:

A new table will be added to the database. It will include new alembic
scripts to create the new table and it's associations.


Other deployer impact
---------------------

Deployer will now have the ability to store secret specific metadata that may
be consumed by an application.

Developer impact
----------------

None


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  diazjf

Other contributors:
  None


Work Items
----------

Phase 1: Database alterations
Phase 2: Current and New API alterations and Tests
Phase 3: Documentation

Dependencies
============

None

Testing
=======

Unit tests must be written for internal component testing. Functional tests must
be written for testing this new feature as a whole.

Documentation Impact
====================

Barbican API must be updated to include these changes.

References
==========
[1] https://github.com/openstack/barbican-specs/blob/master/specs/mitaka/add-user-metadata.rst
