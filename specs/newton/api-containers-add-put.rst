..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===================================================
Add Support For Mutable Generic Containers Resource
===================================================

The URL of the associated launchpad blueprint:

https://blueprints.launchpad.net/barbican/+spec/api-containers-add-put

The original intent of the 'generic' container type was to support arbitrary
secrets to be collected and referenced, similar to a file system folder. Hence
it makes sense for an already-created 'generic' container to support changing
this collection after the fact. This blueprint details how this API would look.


Problem Description
===================

Currently Barbican allows a 'generic'-typed container to be created via a POST
call, but once that container is created it is not possible to modify it to
change which secrets it contains. The 'generic' container was intended to be a
convenient way to collect related secrets and as such should allow for
modification after it is created.

For example continuous integration and deployment systems might wish to collect
all secrets for a given environment such as database passwords and access
tokens, for use by automation scripts when servers are created in the
environment later. It would be convenient if each environment could have its
own Barbican container that automation scripts could reference to retrieve
secrets (by their fixed key names) as part of booting up servers in that
environment. If secrets are later rotated or updated, the secret references
for a given key name could be updated in the container, with no need to update
the automation scripts.

Attempts to update containers of other types will be rejected.


Proposed Change
===============

This blueprint calls for adding new sub-resources to generic containers.
The API impact section details what these calls look like relative to clients.
The actual service implementation is straightforward though, allowing clients
to provide additional secrets or delete individual secrets by sending a POST or
DELETE request to the sub-resources in a 'generic' container.

Alternatives
------------

A previously-accepted but not implemented version of this blueprint called
for adding PUT support for containers whereby clients must specify all secrets
to be held in the container. For example when adding a secret to an existing
container the PUT body would have to list all existing secrets in addition to
the new secret. However, during the Newton cycle we discussed that this could
be error-prone due to potential race conditions.

Utilize a PATCH call to provide partial updates to containers. This adds
complexity to API processing, especially if JSONPatch [1] is used. If this
feature is deemed necessary, that should be proposed as a separate blueprint.

Data model impact
-----------------

None.

REST API impact
---------------

The following documentation specifies the proposed container sub-resource
calls, used to add or remove a secret from  an existing 'generic'-type
container.

The access policy for this call is similar to the container resource's POST
call, so users with the Barbican 'admin' or 'creator' role can modify
containers.

POST /v1/containers/{container_uuid}/secrets
############################################

Add a secret to an existing container.  This is only supported on generic
containers.

Request Attributes
******************

+------------+--------+------------------------------------------------------------+
| Name       | Type   | Description                                                |
+============+========+============================================================+
| name       | string | (optional) Human readable name for identifying your secret |
|            |        | within the container.                                      |
+------------+--------+------------------------------------------------------------+
| secret_ref | uri    | (required) Full URI reference to an existing secret.       |
+------------+--------+------------------------------------------------------------+

Request:
********

.. code-block:: none

    POST /v1/containers/{container_uuid}/secrets
    Headers:
        X-Project-Id: {project_id}

    Content:
    {
        "name": "private_key",
        "secret_ref": "https://{barbican_host}/v1/secrets/{secret_uuid}"
    }

Response:
*********

.. code-block:: none

    {
        "container_ref": "https://{barbican_host}/v1/containers/{container_uuid}"
    }

Note that the requesting 'container_uuid' is the same as that provided in the
response.


HTTP Status Codes
*****************

In general, error codes produced by the containers POST call pertain here as
well, especially in regards to the secret references that can be provided.

+------+-----------------------------------------------------------------------------+
| Code | Description                                                                 |
+======+=============================================================================+
| 201  | Successful update of the container                                          |
+------+-----------------------------------------------------------------------------+
| 400  | Missing secret_ref                                                          |
+------+-----------------------------------------------------------------------------+
| 401  | Invalid X-Auth-Token or the token doesn't have permissions to this resource |
+------+-----------------------------------------------------------------------------+

DELETE /v1/containers/{container_uuid}/secrets
##############################################

Remove a secret from a container.  This is only supported on generic
containers.

Request:
********

.. code-block:: none

   DELETE /v1/containers/{container_uuid}/secrets
   Headers:
       X-Project-Id: {project_id}

   Content:
   {
       "name": "private key",
       "secret_ref": "https://{barbican_host}/v1/secrets/{secret_uuid}"
   }

Response:
*********

.. code-block:: none

   204 No Content

HTTP Status Codes
*****************

+------+-----------------------------------------------------------------------------+
| Code | Description                                                                 |
+======+=============================================================================+
| 204  | Successful removal of the secret from the container.                        |
+------+-----------------------------------------------------------------------------+
| 400  | Missing secret_ref                                                          |
+------+-----------------------------------------------------------------------------+
| 401  | Invalid X-Auth-Token or the token doesn't have permissions to this resource |
+------+-----------------------------------------------------------------------------+
| 404  | Specified secret_ref is not found in the container.                         |
+------+-----------------------------------------------------------------------------+

Alternative
***********

Alternatively, we could define the removal of a secret as a call to a resource
that includes the secret_uuid as part of the URI, for example:

.. code-block:: none

   DELETE /v1/containers/{container_uuid}/secrets/{secret_uuid}

However, this would require the client to parse out UUIDs from secret URIs to
be able to construct the correct URI for deletion.  Because of this reason
the DELETE with a body described above should be implemented instead.


Security impact
---------------

The proposed change does not pose a security impact because (1) it does not
alter underlying secrets or their access restrictions, and (2) only 'generic'-
type containers can be modified hence sensitive grouped secrets such as
'certificate'- and 'asymmetric'-type containers remain immutable. Please also
see the Performance Impact section for advice on rate limiting calls such as
the one proposed in this blueprint to avoid denial of service attacks.


Notifications & Audit Impact
----------------------------

The proposed new POST and DELETE container request should be logged and audited
the same as any other REST call to Barbican, and therefore does not need to be
called out specifically in this blueprint.


Other end user impact
---------------------

The Barbican Python client needs to be modified as well.


Performance Impact
------------------

As no cryptographic operations are needed for this blueprint, with only
database references to secrets changed, the proposal should have minimal
impact on performance. However, the call itself is not quota limited, so
deployers might wish to utilize a rate limiting application in front of their
Barbican API nodes, such as Repose [2].


Other deployer impact
---------------------

No configuration or dependency changes are required to utilize the proposed
operation, but as mentioned in the Performance Impact section above if rate
limiting is deployed, the PUT operation on the containers resource should be
limited as well to avoid denial of service attacks.


Developer impact
----------------

None.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  TBD

Other contributors:
  TBD

Work Items
----------

The following work items are required to implement this blueprint:

1) Update container controllers to add the new POST and DELETE sub-resources.

2) Add a new policy entry to `etc/policy.json` for the new operations

3) Add a positive test to modify a 'generic'-type container, and a negative
   test prove that a non-'generic'-type container cannot be modified.

4) Add Barbican Python client support for the new feature.

5) Add sphinx documentation for the new POST and DELETE actions.


Dependencies
============

None.

Testing
=======

See the Work Items section above for details on what to test. No special 3rd
party systems are required to test the proposed functionality.


Documentation Impact
====================

See the Work Items section above for details on what to document.


References
==========

[1] http://jsonpatch.com/
[2] http://openrepose.org/
