..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=====================================
Add User Metadata to Barbican Secrets
=====================================

Blueprint:

https://blueprints.launchpad.net/barbican/+spec/add-user-metadata

Client Blueprint:

https://blueprints.launchpad.net/python-barbicanclient/+spec/add-user-metadata

Currently the only unique information that can be generated for a Barbican
Secret by a user is ``name``. This blueprint will add a new field to each
Barbican Secret to allow a user to add their own metadata to the Secret.


Problem Description
===================

Users may require to add additional data to a Barbican Secret. A user may
want to add a description of the Secret as well as other necessary data
such as geolocation, rate, allowed time-access, etc. This will allow user
applications/services to check the user-metadata for certain fields in order
to allow/disallow an action to Barbican.


Proposed Change
===============

The proposed change will be to add a new attribute to Barbican
Secrets in order for user meta-data to be stored. A new API
endpoint must also be created for the manipulation of the metadata.


Alternatives
------------

Currently there are no alternatives, since user is not able to add
any unique information other than to the attribute ``name``.

Data model impact
-----------------

Secrets will now have an additional parameter called `metadata`.
`metadata` will be a dictionary in which a user can add a key and
a value.

A limit must also be set as to how much meta-data a user can add to a
secret. `quota_secret_meta` will be added to the Barbican config and set to
`-1`, which allows for unlimited metadata.

At the database level a limit will also be placed on the size of the
`metadata`.

A new table will be created called `secret-metadata` with `secret_id`, `key`
and `value` columns.

This has the following benefits:

1. Searching Benefits
The value column can be used to search and the search can stop once an invalid
character is found. Indexing on keys can also be done to make searching for
keys with a certain tag faster.

2. Extensibility
If we wish to add new information, such as created-timestamp,
updated-timestamp, etc. we can simply add a new column. This is cleaner
than adding some other system.

3. Data Consistency
Restrictions can be done at the database level, such as length of the
key/value strings. Number of metadata items can be limited by adding
column restrictions. This will provide good measures for data integrity, and
strengthen security.

.. note::

    Values will all be stored as Strings.


REST API impact
---------------

Each Current API call for Secrets will now have `metadata` as an added
attribute. If no `metadata` is provided then an empty dictionary will be
initialized.

::

    POST /v1/secrets
    Headers:
        Content-Type: application/json
        X-Project-Id: {project_id}

    Content:
    {
        "name": "AES key",
        "expiration": "2015-12-28T19:14:44.180394",
        "algorithm": "aes",
        "bit_length": 256,
        "mode": "cbc",
        "payload": "YmVlcg==",
        "payload_content_type": "application/octet-stream",
        "payload_content_encoding": "base64"
        "metadata": {
          'description': 'contains the AES key',
          'geolocation': '12.3456, -98.7654'
        }
    }

    201 Created

    {
        "secret_ref": "https://{barbican_host}/v1/secrets/{uuid}"
    }


::

    GET /v1/secrets/{uuid}
    Headers:
        Accept: application/json
        X-Project-Id: {project_id}

    200 OK

    {
        "status": "ACTIVE",
        "created": "2015-03-23T20:46:51.650515",
        "updated": "2015-03-23T20:46:51.654116",
        "expiration": "2015-12-28T19:14:44.180394",
        "algorithm": "aes",
        "bit_length": 256,
        "mode": "cbc",
        "name": "AES key",
        "secret_ref": "https://{barbican_host}/v1/secrets/{secret_uuid}",
        "secret_type": "opaque",
        "content_types": {
            "default": "application/octet-stream"
        }
        "metadata": {
          'description': 'contains the AES key',
          'geolocation': '12.3456, -98.7654'
        }
    }


.. note::

    If no `metadata` is input on the POST then `metadata` will
    not be shown.

The following will be added to the REST API:

Get user-metadata from a secret::

    GET /v1/secrets/{uuid}/metadata
    Headers:
        Accept: application/json
        X-Project-Id: {project_id}

    200 OK

    {
        'metadata': {
            'description': 'contains the AES key',
            'geolocation': '12.3456, -98.7654'
        }
    }

Create/Update user-metadata for a secret::

    PUT /v1/secrets/{uuid}/metadata
    Headers:
        Accept: application/json
        X-Project-Id: {project_id}

    Content:
    {
        'metadata': {
            'description': 'contains the AES key',
            'geolocation': '12.3456, -98.7654'
        }
    }

    200 OK

    {
        'metadata': {
            'description': 'contains the AES key',
            'geolocation': '12.3456, -98.7654'
        }
    }

.. note::

    Only Create/Update user-metadata will be needed. To remove the metadata
    a user can perform the PUT with an empty dict. If a partial model is
    sent then the whole metadata will be changed to the partial model which
    has been sent. Values that exist in the data model but not in the PUT
    will be deleted.

The following will be added to the REST API in order to address individual
user metadata items:

Create an individual metadata item in a secret::

    POST /v1/secrets/{uuid}/metadata
    Headers:
        Accept: application/json
        X-Project-Id: {project_id}

    Content:
    {
      "key": "access-limit",
      "value": 11
    }

    201 Created

    Secret Metadata Location: http://example.com:9311/v1/secrets/{uuid}/metadata/access-limit
    {
        "key": "access-llimit",
        "value": 11
    }

.. note::

    If the item already exists then a 409 Conflict error
    code will be returned.

Update an individual metadata item in a secret::

    PUT /v1/secrets/{uuid}/metadata/access-limit
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

    GET /v1/secrets/{uuid}/metadata/access-limit
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

    DELETE /v1/secrets/{uuid}/metadata/access-limit
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

* "secret-meta:get": "rule:secret_non_private_read or rule:secret_project_creator
  or rule:secret_project_admin or rule:secret_acl_read"
* "secret-meta:post": "rule:admin_or_creator and rule:secret_project_match"
* "secret-meta:put": "rule:admin_or_creator and rule:secret_project_match"
* "secret-meta:delete": "rule:admin_or_creator and rule:secret_project_match"


Notifications & Audit Impact
----------------------------

If supported, adding/modifying secret-metadata should be audit events.

Python and Command Line Client Impact
-------------------------------------

The following commands will need to be added to the CLI:

Secrets::

    Secret Metadata Update
    Secret Metadatum Update

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

Deployer will now need to know to set `quota_secret_meta` if they wish users to
be able to add less `metadata` keys to a secret.

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
Phase 3: Add Functionality to CLI

Dependencies
============

None

Testing
=======

Unit tests must be written for internal component testing. Functional tests must
be written for testing this new feature as a whole.

Documentation Impact
====================

Barbican API and Barbican Client must be updated to include these changes.

References
==========

https://blueprints.launchpad.net/barbican/+spec/add-user-metadata
https://blueprints.launchpad.net/python-barbicanclient/+spec/add-user-metadata
