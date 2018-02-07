..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==================================================================
Adding per-secret policy to allow the storing of private secrets
==================================================================

https://blueprints.launchpad.net/barbican/+spec/add-per-secret-policy

Problem Description
-------------------

This is a companion spec to the add-per-secret-policy spec.  That spec proposed
a mechanism for allowing users outside the secrets project to access a
secret/container.  This mechanism involved storing access control data
in the database for each secret/container.

This spec solves a related problem.  Currently, secrets are accessible by all
users with the relevant role in the secret creator's project.  That means that
all members of the same project can access all secrets stored for that project.

Recently there have been requests to be able to designate "private secrets".
These are secrets that only the secret creator (the user that created the secrets)
would be able to extract.

Now, it is possible to create a project for each user that would like to store
private secrets.  This seems to be a very burdensome solution, with large admin
overhead in managing all the required groups and roles.

Proposed Change
---------------

The proposal is to store additional access data with each secret/container.  This
data ("creator_only" with a value of True/False) will be stored in the SecretACL
and ContainerACL data tables as a separate column.  The details are in the Data
Model Impact section below.

When a secret/container is accessed, the creator_only information and any
whitelist information is passed up to the policy layer through target.* variables.
Access is then determined by evaluating policy rules that use these parameters.
Proposed policy rules are specified in the Policy section below.

Finally, the REST API will need to be augmented to allow users to specify and
change the creator_only status of a secret/container.

Operations
----------

Many different operations are restricted when creator_only = True:

get:  If creator_only is true, only the secret/container's creator will
      be able to access the secret.  Restricting access to a container though
      does NOT cascade the restriction down to the constituent secrets.

      Note that this does not affect whitelisted users that have explicitly been
      allowed to get the secret/container.

delete: If creator_only is true, the secret/container can only be deleted by
      the creator.

write:  If creator_only is true, the secret/container can only be modified by
      the creator.

change-acl: By default, this can only be modified by the creator.

list:  If creator_only is true, only the creator will be able to list the secret.
       This means that the link to the secret will not show up if a non-creator
       accesses GET /secrets

Data model impact
-----------------

As per the per-secret spec, secrets, containers and orders tables will have a column
added to store the creator of the secret.  This would be populated by the
creator's user_id.

A boolean column ("creator_only") will need to be added to the ContainerACL
and SecretACL tables.  The fields for the ACL tables will be as follows:

    secret_id: foreign key to Secrets table
    operation: (in this case, get, write, delete, list)
    users: string, list of whitelisted users for the specified operation
    creator_only: True/False

When a secret is designated as "creator_only", several fields will be created/
modified::

    secret_id : get : users, True
    secret_id:  write: None, True
    secret_id:  delete: None, True
    secret_id:  list:  None, True

Undesignating a secret as "creator-only" would either modify/delete the above
entries.

REST API impact
---------------

* A new parameter ("creator-only"), which would take the values "True" and
  "False" will be added to the ACL definition.  It will default to False.
  Accordingly, the ACL definition will look like::

    {
        "get": {
            "users": ["some_user",
                      "another_user"],
        },
        "creator-only": "true"
    }

  If creator-only is set to true, the fields mentioned in the section above would
  be created.

* As described in the per-secret spec, the ACL could be specified as an optional
  acl parameter when creating the secret with a POST request.  It can also be
  retrieved or modified through a GET or POST ::

      POST <host>/v1/secrets/<secret-UUID>/acl
      GET  <host>/v1/secrets/<secret-UUID>/acl

Policy
------

When a container or secret is accessed, the ACL data is retrieved from the
database and provided to the RBAC layer as target.* attributes.
For reading a secret, for example, we could pass in target.user_whitelist and
target.creator_only.

The policy rule would then look something like::

    (can_read_shared and user_in_user_whitelist) OR 
    ((current_resource_permissions and not creator_only) OR user_is_creator)

The (current_resource_permissions) part is basically that the user has the relevant
role in the secret creator's project.

For deletes, modifications etc., the rule is much simpler because we do not
need to account for the whitelists.  Delete, for example, will likely look
something like this::

    ((current_resource_permissions and not creator_only) OR user_is_creator)


Alternatives
------------

As mentioned before, for private secrets, we could create a group for each
user.  Other than being cumbersome, this will entail a maintenance load on
system administrators to keep track of new and removed users.

Security impact
---------------

This improves security and usability in the stack as a whole by allowing users
to specify private secrets.

Notifications & Audit Impact
----------------------------

None.

Other end user impact
---------------------

python-barbicanclient will need to be updated to provide an interface to
populate the extra parameters.

Performance Impact
------------------

Accessing a secret/container will require two database calls: one to get
secret/container whitelist as part of the RBAC engine's rules
enforcement, and one to actually get the secret.

These two database accesses are logically separate as the first process is
controlled by middleware, and the second by Pecan. We might be able to utilize
the same SQLAlchemy transaction, or else cache that secret entity data for the
controllers to work with, so this might be moot.

Other deployer impact
---------------------

None.

Developer impact
----------------

None.

Implementation
==============

Assignee(s)

Primary assignee:
    alee
    rm_work

Work Items
----------

* Add new field to the database tables, and new parameters/calls to the REST API.

* Add logic to parse the data and store the  data in the database.

* Add logic to retrieve the data from the database and provide to RBAC layer as
  target.* attributes.

* Modify policy rules based on these target.* attributes.  It may be necessary to
  extend oslo policy here to account for the new boolean flag.  Any changes will
  need to be communicated clearly to deployers as it is not guaranteed that they
  will deploy with default policy files.

Dependencies
============

None

Testing
=======

The current unit tests will also be modified to have this change reflected upon
them.

Documentation Impact
====================

Barbican docs and API docs will need to be changed.

References
==========

* Earlier blueprint with similar ideas.
  https://blueprints.launchpad.net/barbican/+spec/secret-isolation-at-user-level
