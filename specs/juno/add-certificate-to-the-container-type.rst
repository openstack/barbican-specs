..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

================================================
Add certificate to container type
================================================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/barbican/+spec/add-certificate-to-the-container-type

Open Stack LBaaS is planning to use barbican for their certificate use case.
We need to add 'certificate' to the container type option.

Problem Description
===================

Barbican's Container resource is capable of storing SSL certificate along with
its private_key and passphrase. Current implementation of container only supports
RSA and Generic type, due to this limitation containers holding certificate can
not be distinguishable.


Proposed Change
===============

Add the `certificate` option for containers. This augments the the current type attribute on the containers resource and data model to accept `certificate` as a valid option.

Alternatives
------------

None

Data model impact
-----------------

The `Container` schema will be modified to have `certificate` in
the container types constraint check.

REST API impact
---------------

The REST request and response structure for container may contain
`certificate` in type field.

* Example 01 - Container request with certificate type::

    {
      "name": "my-cert",
      "type": "certificate",
      "secret_refs": [
        {
           "name": "certificate",
           "secret_ref":"http://localhost:9311/v1/b885a8320e4f48c69ddffb0364eeef36/secrets/46332930-8e52-4c40-b069-cc39ca65a221"
        },
        {
           "name": "intermediates",
           "secret_ref":"http://localhost:9311/v1/b885a8320e4f48c69ddffb0364eeef36/secrets/21f2234a-f65c-11e3-8791-002564955ea1"
        },
        {

           "name": "private_key",
           "secret_ref":"http://localhost:9311/v1/b885a8320e4f48c69ddffb0364eeef36/secrets/094e76ed-85e0-49b1-b6ce-6bde3cf6571c"
        },
        {
           "name": "private_key_passphrase",
           "secret_ref":"http://localhost:9311/v1/b885a8320e4f48c69ddffb0364eeef36/secrets/da5ccd5a-ef0f-4cb3-8d8f-dd12308b3109"
        }
      ]
    }

* Example 02 - Container response with certificate type::

    {
        "status": "ACTIVE",
        "updated": "2014-06-10 17:21:37.477461",
        "name": "my-cert",
        "secret_refs": [
            {
                "secret_ref": "http://localhost:9311/v1/b885a8320e4f48c69ddffb0364eeef36/secrets/579d0ffe-a40c-47a6-b0c6-8978a441f661",
                "name": "private_key_passphrase"
            },
            {
                "secret_ref": "http://localhost:9311/v1/b885a8320e4f48c69ddffb0364eeef36/secrets/93688bf3-5101-47de-bdd5-46e52186038f",
                "name": "certificate"
            },
            {
               "name": "intermediates",
               "secret_ref":"http://localhost:9311/v1/b885a8320e4f48c69ddffb0364eeef36/secrets/21f2234a-f65c-11e3-8791-002564955ea1"
            },
            {

                "secret_ref": "http://localhost:9311/v1/b885a8320e4f48c69ddffb0364eeef36/secrets/a7a61669-c6fb-4375-9577-11744f4a88f7",
                "name": "private_key"
            }
        ],
        "created": "2014-06-10 17:21:37.477448",
        "container_ref": "http://localhost:9311/v1/b885a8320e4f48c69ddffb0364eeef36/containers/dc252a7d-600f-49a3-9df4-7bca35aa366d",
        "type": "certificate"
    }


Security impact
---------------

None

Notifications & Audit Impact
----------------------------

None

Performance Impact
------------------

None

Other deployer impact
---------------------

None

Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee: atiwari (arvind-tiwari)


Work Items
----------

* CR to address data migration.
* CR to add certificate type in container.

Dependencies
============

None

Testing
=======

New unit test will be required to test this feature.

Documentation Impact
====================

Documents has to be enhanced to add certificate in container type.

    https://github.com/cloudkeep/barbican/wiki/Application-Programming-Interface#containers-resource

References
==========

https://blueprints.launchpad.net/barbican/+spec/add-certificate-to-the-container-type
