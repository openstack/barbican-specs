..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

================
Secret Consumers
================

https://storyboard.openstack.org/#!/story/2005770

This spec proposes an addition to the Barbican Secrets API to allow
other OpenStack projects to add references to individual Secrets when
those secrets are being used by them.

This spec also proposes a change to both the Python and CLI clients in
python-barbicanclient in how they handle the deletion of secrets.
Clients would be changed such that deleting a secret will result in an
error when they are still being consumed by another project unless a `force`
parameter is provided.

This spec is part of a larger effort to provide Encrypted Images
to OpenStack clouds.

Problem Description
===================

Other OpenStack projects would like to make use of an end user's secrets
e.g. A Secret that contains an encryption key for Image Encryption.
But there is currently no way for those projects to let the user know
that they are using the Secret.  This lack of awareness may lead to errors
if the user deletes a Secret that is still in use by other projects.

On the other hand, users should be allowed to delete secrets whenever they
want, so a Secret being used by other projects should not prevent deletion.

Proposed Change
===============

Add a new API to Secrets to register Secret Consumers (similar, but not
identical to the Containers Consumer API [1]).

With this new API, other OpenStack projects would register as a consumer
of a secret by sending a request to Barbican.  Barbican stores the service
type of the requesting service, as well as both the resource type and
resource ID of the resource that is using the Secret.

See REST API Impact below for details of the API changes.

Clients to barbican would change the semantics for deleting secrets by
returning an error when trying to delete a secret if that secret has one
or more consumers.  Clients will also accept an additional boolean parameter
to delete a secret regardless of how many consumers it has.

See Python and Command Line Client Impact below for details of the client
changes.

Alternatives
------------

One alternative would be to implement Secret Consumers just like Container
Consumers, which uses a URL instead of the consuming entity type and ID.

Another alternative approach that was considered was to have each project
clone the secret when they need to use it.  This alternative has some
downsides, however.  For one, an end user may not be able to delete
those copies.

Data model impact
-----------------

A new model and associated data table will need to be added. For example,
a new class SecretConsumerMetadatum with a secret_consumer_metadata table.

The new class will have references to both the secret_id as well as the
project_id which owns the secret.

REST API impact
---------------

POST /v1/secrets/{secret_id}/consumers
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Add a new resource as a consumer to a secret.

Body Parameters
+++++++++++++++

+---------------------+--------+--------------------------------------------------------+
| Name                | Type   | Description                                            |
+---------------------+--------+--------------------------------------------------------+
| service             | string | Consumer's OpenStack service type as shown in          |
|                     |        | https://service-types.openstack.org/service-types.json |
+---------------------+--------+--------------------------------------------------------+
| resource_type       | string | Name of the resource type using the secret             |
| (or resource_path?) |        | e.g. "images"  or "lbaas/loadbalancers"                |
+---------------------+--------+--------------------------------------------------------+
| resource_id         | string | Unique identifier for the resource using this secret.  |
+---------------------+--------+--------------------------------------------------------+

Barbican will consider the resource_id to be a unique consumer.  This assumes
that resource_id is a UUID, and that duplicate IDs for different projects
is not likely to ever happen in a single cloud.

resource_type should be meaningful to the individual projects, and should
be used to identify the resource in the consuming service.  For example,
Glance could use "images" as the value of the resource type to indicate that
the resrouce_id refers to an image.

Request
+++++++

    POST /v1/secrets/{secret_id}/consumers
    Headers:
        X-Auth-Token: {token}
        X-Content-Type: application/json

    {
        "service": "image",
        "resource_type": "images",
        "resource_id": "{image_id}"
    }

Responses
+++++++++

+------+--------------------------------------------------------------------+
| Code | Description                                                        |
+======+====================================================================+
|  200 | OK                                                                 |
+------+--------------------------------------------------------------------+
|  401 | Unauthorized - X-Auth-Token is invalid                             |
+------+--------------------------------------------------------------------+
|  403 | Forbidden - X-Auth-Token is valid, but the associated project does |
|      |             not have the appropriate role/scope                    |
+------+--------------------------------------------------------------------+

GET /v1/secrets/{secret_id}/consumers
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

List consumers for a particular Secret.

Parameters
++++++++++

+---------+---------+---------+-------------------------------------------------+
| Name    | Type    | Default | Description                                     |
+=========+=========+=========+=================================================+
| offset  | integer |       0 | Offset to start consumer response               |
+---------+---------+---------+-------------------------------------------------+
| limit   | integer |      10 | Number of consumer entries returned in response |
+---------+---------+---------+-------------------------------------------------+
| service |  string | None    | Filter by service type                          |
+---------+---------+---------+-------------------------------------------------+

Request
+++++++

    GET /v1/secrets/{secret_id}/consumers
    Headers:
        X-Auth-Token: {token}

OK Response
+++++++++++

    200 OK

    {
        "total": 1,
        "consumers": [
            {
                "service": "image",
                "resource_type": "images",
                "resource_id" : "{image_id}"
            }
        ]
    }

Other Responses
+++++++++++++++

+------+--------------------------------------------------------------------+
| Code | Description                                                        |
+======+====================================================================+
|  401 | Unauthorized - X-Auth-Token is invalid                             |
+------+--------------------------------------------------------------------+
|  403 | Forbidden - X-Auth-Token is valid, but the associated project does |
|      |             not have the appropriate role/scope                    |
+------+--------------------------------------------------------------------+

DELETE /v1/secrets/{secret_id}/consumers/{resource_id}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Delete a consumer.  ie. The resource is being deleted and it longer needs
to access this secret.

Request
+++++++

     DELETE v1/secrets/{secret_id}/consumers/{resource_id}

Responses
+++++++++

+------+--------------------------------------------------------------------+
| Code | Description                                                        |
+======+====================================================================+
|  200 | OK                                                                 |
+------+--------------------------------------------------------------------+
|  401 | Unauthorized - X-Auth-Token is invalid                             |
+------+--------------------------------------------------------------------+
|  403 | Forbidden - X-Auth-Token is valid, but the associated project does |
|      |             not have the appropriate role/scope                    |
+------+--------------------------------------------------------------------+
|  404 | Not Found - Consumer record for given resource_id was not found.   |
+------+--------------------------------------------------------------------+

Security impact
---------------

Because the consumers are stored in the database, there is the possibility
that a bad actor could add many consumers to try to fill the database disk
space.  Secret Consumers should be limited to the same quota as Container
Consumers to mitigate this risk. For example:

    [quota]
    quota_consumers=10000

Would limit both Container Consumers and Secret Consumers to a maximum
of 10,000 consumers each for both a single Container or a single Secret.

Notifications & Audit Impact
----------------------------

The new API endpoints should be audited as usual.

Python and Command Line Client Impact
-------------------------------------

The Secret class in python-barbicanclient should be updated to add new
methods such as:

    class Secret(...):
        ...

        def add_consumer(self, service_type, resource_type, resource_id):
            ...

        def remove_consumer(self, service_type, resource_type, resource_id):
            ...

Both methods should raise appropriate exceptions when the API returns an error.
Additionally, the Secret.delete() method should be updated to take a new *force*
parameter and throw an exception when delete() is called with force=False,
and the secret still has consumers:

    class Secret(...):
        ...

        def delete(self, force=False):
            ...

The CLI client should be changed to add new consumer options, such as:

    openstack secret consumer add --service-type=image --resource-type=image \
        --resource-id=XXXX-XXXX-XXXX-XXXX

    openstack secret consumer remove --service-type=image --resource-type=image \
        --resource-id=XXXX-XXXX-XXXX-XXXX

The secret delete command should be changed to take a *--force* parameter:

    openstack secret delete --force {secret_uuid}

This command should return an error when a secret has one or more consumers
and the --force flag is not used:

    openstack secret delete {secret_uuid_with_consumers}
    ERROR: Secret has one or more consumers.  Use --force to delete anyway.

These changes will require a new Major version for python-barbicanclient
because the default --force=False option could cause some scripts to break in
certain scenarios where secrets are currently being deleted that do have
consumers associated with them.

Other end user impact
---------------------

Currently there is no other impact to the end user other than the CLI changes
listed above.  In the future, when a barbican-ui for Horizon is developed,
it should use the consumers to present confirmation dialogs to the user
when deleting Secrets which have consumers.

It should be noted that Deleting Secrets in the Barbican REST API
has not changed, and a client using the API directly will be able to delete
a secret regardless of the presence of consumers.

Performance Impact
------------------

Deleting secrets using the CLI or the Python client will be affected as we
will likely need to perform additional requests to the API to get the list of
consumers for a secret before sending a DELETE request.

Other deployer impact
---------------------

When python-barbican changes are merged, some automation scripts that use
secret deletion may break if the secrets being deleted have consumers.

Any automation scripts should be updated to use the --force flag if needed.

Developer impact
----------------

Developers of other projects that want to make use of this feature will
need to use python-barbicanclient to integrate with the Key Manager service.

Implementation
==============

Assignee(s)

Primary assignee:
  Douglas Mendizábal (Freenode: redrobot) <dmendiza@redhat.com>

Other contributors:
  Moisés Guimarães (Freenode: moguimar) <moguimar@redhat.com>

Work Items
----------

* Implement Model changes and database migration
* Implement API changes
* Implement python-barbicanclient changes (both python client and CLI)

Dependencies
============

None.

Testing
=======

Tempest test cases should be added to test adding/removing Secret Consumers
using a service-user that is not barbican.

Documentation Impact
====================

All API changes should be documented in the API reference, as well as the
API Guide.

References
==========

[1] Container Consumers API:
https://docs.openstack.org/barbican/stein/api/reference/consumers.html

Barbican Train PTG Etherpad:
https://etherpad.openstack.org/p/barbican-train-ptg
