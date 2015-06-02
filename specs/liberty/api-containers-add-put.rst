..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===================================================
Add PUT Support For the Generic Containers Resource
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

This blueprint calls for adding PUT support for containers. The API impact
section details what this call looks like relative to clients. The actual
service implementation is straightforward though, allowing clients to provide
a revised list of key-names to secret refs that are then updated for a given
'generic' container.

Alternatives
------------

Utilize a PATCH call to provide partial updates to containers. This adds
complexity to API processing, especially if JSONPatch [1] is used. If this
feature is deemed necessary, that should be proposed as a separate blueprint.

Data model impact
-----------------

None.

REST API impact
---------------

The following documentation specifies the proposed container PUT call, used to
entirely replace an existing 'generic'-type container.

The access policy for this call is similar to the container resource's POST
call, so users with the Barbican 'admin' or 'creator' role can modify
containers. This blueprint would also call for adding a new 'write' access
control list policy group, with users added to this group allowed to also
modify the container.


PUT /v1/containers/{container_uuid}
###################################

Replace an existing container

The uploaded container information replaces the existing container's. This
operation is only supported for 'generic'-type containers.

Request Attributes
******************

+-------------+--------+-----------------------------------------------------------+
| Name        | Type   | Description                                               |
+=============+========+===========================================================+
| name        | string | (optional) Human readable name for identifying your       |
|             |        | container                                                 |
+-------------+--------+-----------------------------------------------------------+
| type        | string | Type of container, defaults to 'generic'. As only generic |
|             |        | container types would be modifiable via this proposal,    |
|             |        | providing a a value here other than 'generic' would fail  |
+-------------+--------+-----------------------------------------------------------+
| secret_refs | list   | A list of dictionaries containing references to secrets,  |
|             |        | which entirely replace the existing references.           |
+-------------+--------+-----------------------------------------------------------+

Request:
********

.. code-block:: none

    PUT /v1/containers/{container_uuid}
    Headers:
        X-Project-Id: {project_id}

    Content:
    {
        "type": "generic",
        "name": "container name",
        "secret_refs": [
            {
                "name": "private_key",
                "secret_ref": "https://{barbican_host}/v1/secrets/{secret_uuid}"
            }
        ]
    }


Response:
*********

.. code-block:: json

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
| 401  | Invalid X-Auth-Token or the token doesn't have permissions to this resource |
+------+-----------------------------------------------------------------------------+

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

The proposed new PUT container request should be logged and audited the same
as any other REST call to Barbican, and therefore does not need to be called
out specifically in this blueprint.

Other end user impact
---------------------

The Barbican Python client needs to be modified as well.

Performance Impact
------------------

As no cryptographic operations are needed for this blueprint, with only
database references to secrets changed, the proposal should have minimal
impact on performance. However, the call itself if not quota limited, so
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

1) Override the 'update()' method on the container class
   `barbican.model.models.Container` to update from the provided dict of
   provided update attributes, including a list of key/secret dicts to update
   on the container.

2) Add `on_put()` method to the containers controller class
   `barbican.api.controllers.containers.ContainerController`.

3) Add a new policy entry to `etc/policy.json` for `container:put`.

4) Add a positive test to modify a 'generic'-type container, and a negative
   test prove that a non-'generic'-type container cannot be modified.

5) Add Barbican Python client support for the new feature.

6) Add sphinx documentation for the new PUT action.


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
