..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=================================================
Remove the project-id from Barbican resource URIs
=================================================

https://blueprints.launchpad.net/barbican/+spec/api-remove-uri-tenant-id

OpenStack projects appear to be moving away from requiring a project-id in the URI,
as this is redundant information when Keystone authentication is used.
All Barbican resource URIs includes project-id as part of their URI. This need to be
reviewed.

Problem Description
===================

All Barbican resources have a project-id in their URI. This helps to correlate the
resource with the specified tenant when no external authentication mechanism is used.

However, all Barbican deployments would use Keystone to authenticate the API requests.
With Keystone, the project-id information is already obtained when the X-Auth-Token header
from the request is validated. This makes the project-id in the URI redundant.

Need a solution to remove the project-id from the URI.

Proposed Change
===============

1. All Barbican resource URIs will drop project-id from their URI. For example,
   ``/v1/<project-id>/secrets`` *will become* ``/v1/secrets``,
   ``/v1/<project-id>/secrets/<secret-ref>`` *will become* ``/v1/secrets/<secret-ref>``
   and so on.

2. For all successfully authenticated client requests, the project-id (a.k.a tenant-id)
   would be obtained by looking up the authentication context associated with the request.
   If it is not found (unscoped token), the request will be denied with a HTTP 401 error.

3. If no authentication mechanism has been configured with the Barbican deployment, the
   client is expected to pass in a "X-Project-Id" request header, set to the desired project-id.
   If this header is not passed in, the request will be denied with a HTTP 400 error. In other
   words, the client is expected to spoof the data presented to Barbican by
   ``keystone.middleware.auth_token``, which Barbican trusts.

**Backwards compatability note:**

* Once the changes proposed by this spec are in place, callers of the API should no longer
  specify project-id in the URI. Otherwise, the request will be denied with a HTTP 404 error.



Alternatives
------------

None.

Data model impact
-----------------

None.

REST API impact
---------------

All Barbican REST resources will be impacted by this change. However, this change is limited to
dropping the project-id from the uri. All other request parameters (including limit and filter
parameters) will remain the same. If Barbican is configured for unauthenticated request flow,
then ``X-Project-Id`` is required to be passed by the caller.


API
---

Secrets

Create Secret: POST /v1/secrets

Example::

    Request:

        Headers:

        X-Auth-Token:<token>
        Content-Type:application/json

        POST /v1/secrets

        {
          "name": "AES key",
          "expiration": "2014-02-28T19:14:44.180394",
          "algorithm": "aes",
          "bit_length": 256,
          "mode": "cbc",
          "payload": "gF6+lLoF3ohA9aPRpt+6bQ==",
          "payload_content_type": "application/octet-stream",
          "payload_content_encoding": "base64"
        }


    Response:

        Status: 201 Created

        {
            "secret_ref": "http://localhost:9311/v1/secrets/a8957047-16c6-4b05-ac57-8621edd0e9ee"
        }


Two-step secret creation: PUT /v1/secrets/<secret-id>

Example::

    Request:

        Headers:

        X-Auth-Token:<token>
        Content-Type:text/plain

        PUT /v1/secrets/a8957047-16c6-4b05-ac57-8621edd0e9ee

        'mysecret'

    Response:

        Status: 201 Created

        {
            "secret_ref": "http://localhost:9311/v1/secrets/a8957047-16c6-4b05-ac57-8621edd0e9ee"
        }

List Secrets: GET /v1/secrets

Example::

    Request:

        Headers:

        X-Auth-Token:<token>

        GET /v1/secrets

    Response:

        Status: 200 Ok

        {
          "secrets": [
            {
              "status": "ACTIVE",
              "updated": "2013-06-28T15:23:30.668641",
              "mode": "cbc",
              "name": "Main Encryption Key",
              "algorithm": "AES",
              "created": "2013-06-28T15:23:30.668619",
              "secret_ref": "http://localhost:9311/v1/secrets/e171bb2d-f14f-433e-84f0-3dfcac7a7311",
              "expiration": "2014-06-28T15:23:30.668619",
              "bit_length": 256,
              "content_types": {
                "default": "application/octet-stream"
              }
            },
            {
              "status": "ACTIVE",
              "updated": "2013-06-28T15:23:32.210474",
              "mode": "cbc",
              "name": "Backup Key",
              "algorithm": "AES",
              "created": "2013-06-28T15:23:32.210467",
              "secret_ref": "http://localhost:9311/v1/secrets/6dba7827-c232-4a2b-8f3d-f523ca3a3f99",
              "expiration": null,
              "bit_length": 256,
              "content_types": {
                "default": "application/octet-stream"
              }
            },
            {
              "status": "ACTIVE",
              "updated": "2013-06-28T15:23:33.092660",
              "mode": null,
              "name": "PostgreSQL admin password",
              "algorithm": null,
              "created": "2013-06-28T15:23:33.092635",
              "secret_ref": "http://localhost:9311/v1/secrets/6dfa448d-c35a-4158-abaf-e4c249efb580",
              "expiration": null,
              "bit_length": null,
              "content_types": {
                "default": "text/plain"
              }
            }
          ],
          "next": "http://localhost:9311/v1/secrets?limit=3&offset=5",
          "previous": "http://localhost:9311/v1/secrets?limit=3&offset=0"
        }


Get individual secret: GET /v1/secrets/<secret-id>

Example::

    Request:

        Headers:

        X-Auth-Token:<token>

        GET /v1/secrets/e171bb2d-f14f-433e-84f0-3dfcac7a7311

    Response:

        Status: 200 Ok

        {
          "status": "ACTIVE",
          "updated": "2013-06-28T15:23:30.668641",
          "mode": "cbc",
          "name": "Main Encryption Key",
          "algorithm": "AES",
          "secret_ref": "http://localhost:9311/v1/secrets/e171bb2d-f14f-433e-84f0-3dfcac7a7311",
          "expiration": "2014-06-28T15:23:30.668619",
          "bit_length": 256,
          "content_types": {
            "default": "application/octet-stream"
          }
        }

Delete individual secret: DELETE /v1/secrets/<secret-id>

Example::

    Request:

        Headers:

        X-Auth-Token:<token>

        DELETE /v1/secrets/e171bb2d-f14f-433e-84f0-3dfcac7a7311

    Response:

        Status: 204 No Content

Create Secret(no authentication) : POST /v1/secrets

Example::

    Request:

        Headers:

        X-Project-Id:<project-id>
        Content-Type:application/json

        POST /v1/secrets

        {
          "name": "AES key",
          "expiration": "2014-02-28T19:14:44.180394",
          "algorithm": "aes",
          "bit_length": 256,
          "mode": "cbc",
          "payload": "gF6+lLoF3ohA9aPRpt+6bQ==",
          "payload_content_type": "application/octet-stream",
          "payload_content_encoding": "base64"
        }


    Response:

        Status: 201 Created

        {
            "secret_ref": "http://localhost:9311/v1/secrets/a8957047-16c6-4b05-ac57-8621edd0e9ee"
        }


Orders

Create Order: POST /v1/orders

Example::

    Request:

        Headers:

        X-Auth-Token:<token>
        Content-Type:application/json

        POST /v1/orders

        {
          "secret": {
            "name": "secretname",
            "algorithm": "AES",
            "bit_length": 256,
            "mode": "cbc",
            "payload_content_type": "application/octet-stream"
          }
        }

    Response:

        Status: 201 Created

        {
            "order_ref": "http://localhost:9311/v1/orders/a8957047-16c6-4b05-ac57-8621edd0e9ee"
        }


Get individual order: GET /v1/orders/<order-id>

Example::

    Request:

        Headers:

        X-Auth-Token:<token>

        GET /v1/orders/f9b633d8-fda5-4be8-b42c-5b2c9280289e

    Response:

        Status: 200 Ok

        {
          "secret": {
            "name": "secretname",
            "algorithm": "aes",
            "bit_length": 256,
            "mode": "cbc",
            "payload_content_type": "application/octet-stream"
          },
          "order_ref": "http://localhost:8080/v1/orders/f9b633d8-fda5-4be8-b42c-5b2c9280289e",
          "secret_ref": "http://localhost:8080/v1/secrets/888b29a4-c7cf-49d0-bfdf-bd9e6f26d718",
          "status": "ERROR",
          "error_status_code": "400 Bad Request",
          "error_reason": "Secret creation issue seen - content-encoding of 'bogus' not supported."
        }

Get list of orders per tenant: GET /v1/orders

Example::

    Request:

        Headers:

        X-Auth-Token:<token>

        GET /v1/orders


    Response:

        Status: 200 Ok

        {
          "orders": [
            {
              "status": "ACTIVE",
              "secret_ref": "http://localhost:9311/v1/secrets/bf2b33d5-5347-4afb-9009-b4597f415b7f",
              "updated": "2013-06-28T18:29:37.058718",
              "created": "2013-06-28T18:29:36.001750",
              "secret": {
                "name": "secretname",
                "algorithm": "aes",
                "bit_length": 256,
                "mode": "cbc",
                "payload_content_type": "application/octet-stream"
              },
              "order_ref": "http://localhost:9311/v1/orders/3100078a-6ab1-4c3f-ab9f-295938c91733"
            },
            {
              "status": "ACTIVE",
              "secret_ref": "http://localhost:9311/v1/secrets/fa71b143-f10e-4f7a-aa82-cc292dc33eb5",
              "updated": "2013-06-28T18:29:37.058718",
              "created": "2013-06-28T18:29:36.001750",
              "secret": {
                "name": "secretname",
                "algorithm": "aes",
                "bit_length": 256,
                "mode": "cbc",
                "payload_content_type": "application/octet-stream"
              },
              "order_ref": "http://localhost:9311/v1/orders/30b3758a-7b8e-4f2c-b9f0-f590c6f8cc6d"
            }
          ]
        }

Delete individual order: DELETE /v1/orders/<order-id>

Example::

    Request:

        Headers:

        X-Auth-Token:<token>

        DELETE /v1/orders/e171bb2d-f14f-433e-84f0-3dfcac7a7311

    Response:

        Status: 204 No Content



Containers

Create Container: POST /v1/containers

Example::

    Request:

        Headers:

        X-Auth-Token:<token>
        Content-Type:application/json

        POST /v1/containers

        {
          "name": "container name",
          "type": "rsa",
          "secret_refs": [
            {
               "name": "private_key",
               "secret_ref":"http://localhost:9311/v1/secrets/05a47308-d045-43d6-bfe3-1dbcd0c3a97b"
            },
            {
               "name": "public_key",
               "secret_ref":"http://localhost:9311/v1/secrets/05a47308-d045-43d6-bfe3-1dbcd0c3a97b"
            },
            {
               "name": "private_key_passphrase",
               "secret_ref":"http://localhost:9311/v1/secrets/05a47308-d045-43d6-bfe3-1dbcd0c3a97b"
            }
          ]
        }


    Response:

        Status: 201 Created

        {
            "container_ref": "http://localhost:9311/v1/containers/a8957047-16c6-4b05-ac57-8621edd0e9ee"
        }


Get individual container: GET /v1/containers/<container-id>

Example::

    Request:

        Headers:

        X-Auth-Token:<token>

        GET /v1/containers/f9b633d8-fda5-4be8-b42c-5b2c9280289e

    Response:

        Status: 200 Ok

        {
           "name":"rsa container",
           "secret_refs":[
              {
                 "secret_ref":"http://localhost:9311/v1/secrets/059805d5-b400-47da-abc5-cae7286d3ede",
                 "name":"private_key_passphrase"
              },
              {
                 "secret_ref":"http://localhost:9311/v1/secrets/28704f0f-3273-40d4-bc40-4de2691135ea",
                 "name":"private_key"
              },
              {
                 "secret_ref":"http://localhost:9311/v1/secrets/29d89344-10ad-4f92-8aa2-adebaf7556ee",
                 "name":"public_key"
              }
           ],
           "container_ref":"http://localhost:9311/v1/containers/888b29a4-c7cf-49d0-bfdf-bd9e6f26d718",
           "type":"rsa"
        }

Get list of containers per tenant: GET /v1/containers

Example::

    Request:

        Headers:

        X-Auth-Token:<token>

        GET /v1/containers


    Response:

        Status: 200 Ok

        {
           "total":42,
           "containers":[
              {
                 "status":"ACTIVE",
                 "updated":"2014-02-11T18:05:58.909411",
                 "name":"generic container_updated",
                 "secret_refs":[
                    {
                       "secret_id":"123",
                       "name":"private_key"
                    },
                    {
                       "secret_id":"321",
                       "name":"public_key"
                    },
                    {
                       "secret_id":"456",
                       "name":"private_key_passphrase"
                    }
                 ],
                 "created":"2014-02-11T18:05:58.909403",
                 "container_ref":"http://localhost:9311/v1/containers/d4e06015-4f6e-4626-ac3d-4ece6621f96d",
                 "type":"rsa"
              },
              {
                 "status":"ACTIVE",
                 "updated":"2014-02-11T18:08:58.160557",
                 "name":"generic container_updated",
                 "secret_refs":[
                    {
                       "secret_id":"321",
                       "name":"public_key"
                    },
                    {
                       "secret_id":"456",
                       "name":"private_key_passphrase"
                    }
                 ],
                 "created":"2014-02-11T18:08:58.160551",
                 "container_ref":"http://localhost:9311/v1/containers/bb24fa61-0b5f-4d40-8990-846e95cd7b12",
                 "type":"rsa"
              },
              {
                 "status":"ACTIVE",
                 "updated":"2014-02-11T18:25:58.198072",
                 "name":"generic container_updated",
                 "secret_refs":[
                    {
                       "secret_id":"1df433d6-c2d4-480d-90fb-0bfd9c5da3dd",
                       "name":"private_key"
                    },
                    {
                       "secret_id":"321",
                       "name":"public_key"
                    },
                    {
                       "secret_id":"456",
                       "name":"private_key_passphrase"
                    }
                 ],
                 "created":"2014-02-11T18:25:58.198063",
                 "container_ref":"http://localhost:9311/v1/containers/38f58696-5013-4bd6-ab2b-fbea41dc957a",
                 "type":"rsa"
              },
              {
                 "status":"ACTIVE",
                 "updated":"2014-02-11T18:44:06.296957",
                 "name":"generic container_updated",
                 "secret_refs":[
                    {
                       "secret_id":"1df433d6-c2d4-480d-90fb-0bfd9c5da3dd",
                       "name":"private_key"
                    },
                    {
                       "secret_id":"321",
                       "name":"public_key"
                    },
                    {
                       "secret_id":"456",
                       "name":"private_key_passphrase"
                    }
                 ],
                 "created":"2014-02-11T18:44:06.296947",
                 "container_ref":"http://localhost:9311/v1/containers/a8d1adfd-0d36-4eb0-8762-99787eb4a7ff",
                 "type":"rsa"
              }
           ],
           "next":"http://localhost:9311/v1/containers?limit=10&offset=10"
        }

Delete individual container: DELETE /v1/containers/<container-id>

Example::

    Request:

        Headers:

        X-Auth-Token:<token>

        DELETE /v1/containers/e171bb2d-f14f-433e-84f0-3dfcac7a7311

    Response:

        Status: 204 No Content


Security impact
---------------

* This change will require that all requests have a valid keystone token which will
  be used for authorization.
* Requests without the token will be considered as unauthenticated requests.


Notifications & Audit Impact
----------------------------

None.

Other end user impact
---------------------

* Barbican python client will need to be modified to accommodate API changes.
  This work could happen in parallel to the server side tasks.

Performance Impact
------------------

None.

Other deployer impact
---------------------

None.

Developer impact
----------------

None.


Implementation
==============

Assignee(s)
-----------

Venkat Sundaram (tsv)

Work Items
----------

* Modify PecanAPI routing mechanism and remove logic that parses project-id from
  the URI (**tsv**)

* Enhance the enforce_rback decorator for the controllers to get the keystone_id
  from the authentication context if it is not passed in(**tsv**)

* Update the barbican/common/util module method hostname_for_refs to strip
  project-id from the URI links returned in response body(**tsv**)

* Modify barbican client and replace all tenant-id references from the URI it
  calls (**tsv**)

* Enhance the tests (**tsv**)


Dependencies
============

None.

Testing
=======

* Update unit tests to drop project-id from uri.
* Tempest tests need to be added for functional testing


Documentation Impact
====================

* Existing documentation has to be updated to remove the project-id from the uri.
* Explain how to specify project-id when no authentication mechanism is used.


References
==========

- https://blueprints.launchpad.net/barbican/+spec/api-remove-uri-tenant-id
- https://review.openstack.org/#/c/92491/4/api-docs/keystore-api-v1-remove-tenant-from-uri-and-more.md
