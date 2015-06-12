..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=======================================================
Add List of Group-IDs to ACL for Secrets and Containers
=======================================================

The URL to the launchpad blueprint is here:

https://blueprints.launchpad.net/barbican/+spec/api-acl-add-group-list

The current access control list (ACL) approach in Barbican only allows for
adding user IDs for access to a given secret or container. This blueprint
proposes allowing group IDs to be added to ACLs to accommodate users within
specified groups access to secrets/containers as well. Adding group support to
ACLs would support LDAP group based access to secrets/containers. This
blueprint depends on the approval of a Keystone blueprint [1].


Problem Description
===================

Organizations with many users could find managing access to Barbican secrets at
the per-user level via the current ACL approach a significant effort.
Likewise for engineering systems that wish to manage access to secrets for
many devices. An approach to simplify access configuration would be to manage
secret and container access at the user group level, as proposed by this
blueprint.


Proposed Change
===============

The proposed approach is to add a new one-to-many reference on the current
`SecretACL` entity called `SecretACLGroup` which would be very similar to the
`SecretACLUser` entity, except with a `group_id` attribute that refers to
a desired group-ID to allow secret access to. Likewise a new
`ContainerACLGroup` entity would be added and referenced via the current
`ContainerACL` entity.

In the `barbican.api.controllers`'s `acls.py` module, a new 'groups' element
within each ACL operation would be added, and handled similarly to the 'users'
element now.

Alternatives
------------

None.

Data model impact
-----------------

The 'Proposed Change' section calls for adding two new entities:
`SecretACLGroup` and `ContainerACLGroup`, with additional one-to-many
references from the existing `SecretACL` and `ContainerACL` entities
respectively. No initial records or migrations are needed.

REST API impact
---------------

The API will be changed to add a 'groups' element to the current ACL JSON
message.

Hence the response from `GET /v1/secrets/{uuid}/acl` would become::

     HTTP/1.1 200 OK
     {
       "read":{
         "updated":"2015-05-12T20:08:47.644264",
         "created":"2015-05-12T19:23:44.019168",
         "users":[
           {user_id1},
           {user_id2},
           .....
         ],
         "groups":[
           {group_id1},
           {group_id2},
           .....
         ],
         "project-access":{project-access-flag}
       }
     }

The following sentence is proposed to be added to the description for the
`PUT /v1/secrets/{uuid}/acl` call:

    To delete existing groups from an ACL definition, pass an empty list [] for
    `groups`.

Also the following row is proposed added to the `Attributes` section:

+----------------------------+----------+-----------------------------------------------+----------+
| Attribute Name             | Type     | Description                                   | Default  |
+============================+==========+===============================================+==========+
| groups                     | [string] | (optional) List of group ids. This needs to   | []       |
|                            |          | be group ids as returned by Keystone.         |          |
+----------------------------+----------+-----------------------------------------------+----------+

Also the following should replace the 'Body' element in the 'Request/Response'
section::

    Body:
    {
      "read":{
        "users":[
          {user_id1},
          {user_id2},
          .....
        ],
        "groups":[
          {group_id1},
          {group_id2},
          .....
        ],
        "project-access":{project-access-flag}
      }
    }


In addition to the secrets resource API changes, the response from
`GET /v1/containers/{uuid}/acl` would become::

     HTTP/1.1 200 OK
     {
       "read":{
         "updated":"2015-05-12T20:08:47.644264",
         "created":"2015-05-12T19:23:44.019168",
         "users":[
           {user_id1},
           {user_id2},
           .....
         ],
         "groups":[
           {group_id1},
           {group_id2},
           .....
         ],
         "project-access":{project-access-flag}
       }
     }

The following sentence is proposed to be added to the description for the
`PUT /v1/containers/{uuid}/acl` call:

    To delete existing groups from an ACL definition, pass an empty list [] for
    `groups`.

Also the following row is proposed added to the `Attributes` section:

+----------------------------+----------+-----------------------------------------------+----------+
| Attribute Name             | Type     | Description                                   | Default  |
+============================+==========+===============================================+==========+
| groups                     | [string] | (optional) List of group ids. This needs to   | []       |
|                            |          | be group ids as returned by Keystone.         |          |
+----------------------------+----------+-----------------------------------------------+----------+

Also the following should replace the 'Body' element in the 'Request/Response'
section::

    Body:
    {
      "read":{
        "users":[
          {user_id1},
          {user_id2},
          .....
        ],
        "groups":[
          {group_id1},
          {group_id2},
          .....
        ],
        "project-access":{project-access-flag}
      }
    }


Security impact
---------------

The proposed change provides another optional mechanism to access secrets and
containers, via a user's group ID information.

Notifications & Audit Impact
----------------------------

No additional auditing beyond that planned for resource access should be
required.

Other end user impact
---------------------

Support for adding group IDs to the ACL for a secret or container should be
added to the Barbican Python client as well.

Performance Impact
------------------

For each retrieve of a secret or container, an additional query for the group
ACL information will be required.

Other deployer impact
---------------------

No configuration is required. Changes should be applied in the sequence
provided in the 'Work Items' section.

Developer impact
----------------

None.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  TBD

Work Items
----------

The following work sequence is proposed for a zero-down-time deployment:

1. Create a CR to modify the Sphinx documentation to include the new 'groups'
   API information, marked as 'Work in Progress'

2. Create a CR to add the data model entities and relationships described in the
   'Data model impact' section above, ensuring that current state code still
   operates correctly after the data model updates are applied

3. Create CR(s) to add the SQLAlchemy `barbican.model`'s `models.py` classes
   and relationships to match the above data model changes

4. Create CR(s) to add the new 'groups' list to the 'acls' controller module
   as detailed above, along with unit tests

5. Add CR(s) to add functional testing as detailed below

6. Add CR(s) to add 'groups' ACL support to the Barbican Python client


Dependencies
============

This blueprint depends on Keystone middleware providing the group ID
information for a authenticated token via the `X-Group-Ids` HTTP header. A
pending blueprint to add this capability to Keystone is available here [1].


Testing
=======

The current functional tests for ACL will be augmented to also test out the new
'groups' list.


Documentation Impact
====================

The current documentation for ACL will be augmented to add the 'groups'
information as detailed in the 'API impact' section above.


References
==========

[1] https://review.openstack.org/#/c/188564/
