..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=======================================================
Storing metadata to allow the use of per-secret policy
=======================================================

https://blueprints.launchpad.net/barbican/+spec/add-per-secret-policy

This is a proposal to add per-secret specific policy in Barbican to augment
the generic operation based policy in policy.json. The idea is to store
attributes that help determine whether or not a secret or container could
be accessed (like an include_list of users) per secret.

Oslo policy allows you to access those attributes and pass them to the RBAC
rules engine as a dictionary of target.* attributes, and used in policy
enforcement.

Problem Description
===================

Most recently, Neutron LBaaS tried to figure out how to grant the LBaaS system
user the ability to retrieve secrets (cert containers and secrets) in order to
create load balancer configuration.  The solution that was decided upon was
trusts.  That is, a trust was created on the keystone server for the LBaaS
user for the relevant project ID.

While this mechanism will work, a trust is pretty heavy-handed.  Essentially,
it gives permission to the LBaaS user to impersonate the user for the
relevant project, as long as the trust is valid (which could easily
be forever).  If the LBaaS user were ever hijacked, they would not only be
able to access all the Barbican data, but they would be able to interact
with all the other services as that user.

Indeed, as more projects integrate with Barbican, the requirement for
service users to obtain barbican secrets will come up again and again.
Proliferating trusts throughout the services simply to do something as basic
as retrieving secrets seems like a recipe for disaster.

In addition, recently there have been requests to be able to specify that
some secrets were private.  That is, only the secret creator (the user that
created the secrets) should be able to extract them.  Now, one might argue
that one could create a project for each user - but that seems like a very
burdensome solution.

Both of these problems (and perhaps others) can be solved by using per-secret
access attributes to further refine the policy associated with a secret.  In
policy parlance, this is using attributes of the target (resource being
accessed, in this case, a secret or a container) to further refine policy.

This is something that is done, for instance, in Keystone.  See the rules in
https://github.com/openstack/keystone/blob/master/etc/policy.v3cloudsample.json
which refer to target.  These values are populated in
keystone/common/controller.py (in the definition of protected()).

To make things clearer, I will separate the solution for this second problem
("private secrets") into a separate follow-on spec.

Proposed Change
===============

The spec deals only with the problem posed by LBaaS.  That problem is: how do we
allow access to a secret by a user that is not part of the secret's project.

The proposal is to store access control data with each secret or container that
could be used to augment the existing policy for retrieval on a per-resource basis.

This feature can be broken down in terms of four subproblems:

1. What operations will the proposed ACLs cover, and what will be the default
   policy for each operation?  See the Operations section below.

2. What does the client send into the server when setting the ACLs for a secret
   or container?  See REST API section below.

3. How is this ACL information stored within the database?  See the Database
   section below.

4. How is this data extracted and presented to the Oslo layer to do RBAC enforcement?
   See the Policy section below.

Operations
----------

The ACL permissions here correspond to actions that some user outside of the
creator's project can perform on a secret or container.  These actions include:

get:  For secrets, someone with 'get' permission will be able to retrieve
      secret metadata and payload.  For containers, they will be able to
      retrieve the container (which is essentially all metadata).  Accessing
      secrets referenced within the container will require 'get' permission
      on each secret.  For 'Kilo', get permission on the container will not
      automatically cacade down to the secret.  This is the only action to be
      implemented for K.

delete: Ability to delete a secret or container.  Currently, this permission
      is restricted to a particular role ("creator"?) in the secret's project.
      For Kilo, this will not change.  In particular, its not clear that we would
      ever want someone outside the project to be able to delete a secret or
      container.

write: Ability to modify a project/container.  Currently typed containers and
      secrets are immutable.  Even if they were not, its not clear someone from
      a different project should be able to modify a secret/container.  This
      will not be implemented for K.

change-acl: Ability to change the ACL for a secret.  For K, this will be
     restricted to the secret/container's creator (ie. the user that created the
     secret/container).

REST API impact
---------------

* A new parameter ("acl") will be added to POST for secrets and containers.
  The data in this field has the following format::

      {
          "get": {
                  "users": ["some_user_id",
                            "another_user_id"],
                 }
      }

  The data in those fields will be placed directly into the ContainerACL and
  SecretACL tables below.

  There has been some discussion of adding parameters for whitelisted groups
  and projects to the acl.  This requires additional discussion because groups
  are not specified in the keystone token, and roles should also be defined
  within projects.  As there is no current need for these parameters, we will
  defer adding them till L+.

* In addition, a new endpoint will be provided to allow a user to change the
  ACL on a secret or container::

      POST <host>/v1/secrets/<secret-UUID>/acl

  where the body would be an ACL in the same format as above.  This endpoint
  would be limited to uid == <uid of creator> by default.

* ACL data will not be returned by default with a GET request.  Rather, a new
  route will be provided to retrieve the ACL:

      GET /secrets/<secret_uuid>/acl

  which would return the ACL as shown above.  This endpoint would be limited to
  uid: <creator>.

* When orders are created, the Barbican server will need to keep track of the
  order's creator to ensure that the order's creator has permissions to access
  the secrets.

  For example, when generating a certificate using the stored key mechanism, the
  asynchronous worker will need to access a container and the private key
  referenced within that container.  This should only be allowed if the order's
  creator has access to the relevant resources.

Data model impact
-----------------

For secrets and containers, we will need add a column to store the creator
of the secret.  This would be populated by the creator's user_id.  It may
also be useful to store the creator's project_id, given that we plan to
remove the secret/project table.

We will also need to add column to store the creator of an Order, so that any
secrets/containers that are generated when an Order is processed will be
stamped with their creator.

We also need to create two new tables: ContainerACL and SecretACL, in which
to store ACL data.  The fields for the SecretACL table will be as follows:

    secret_id: foreign key to Secrets table
    action: currently, only "read" will be implemented
    users: string, list of whitelisted users for specified action

In L+, we may decide to add columns to allow for whitelisted groups or projects.
For now, only users are whitelisted.

Policy
------

When a container or secret is accessed, the ACL data is retrieved from the
database and provided to the RBAC layer as target.* attributes.  For getting
a secret, for example, we could pass in target.user_whitelist.

The policy rule would then look something like::

    (can_read_shared and user_in_user_whitelist) OR current_resource_permissions

The current_resource_permissions part is basically that the user has the relevant
role in the secret creator's project.

We may need to extend (and upstream) oslo-policy to allow lists to be processed.


Alternatives
------------

We could use trusts, but they are, as we described before, very heavy handed.
All in all, this is something that is simply required in Barbican.

Security impact
---------------

This improves security in the stack as a whole by eliminating the
requirement for a trust, and rather, providing granular per-secret
permissions.

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
secret/container include)list as part of the RBAC engine's rules
enforcement, and one to actually get the secret/container.

Other deployer impact
---------------------

Neutron and other service users that will be accessing secrets in Barbican
will be able to do so using the new mechanism, rather than a trust.

Developer impact
----------------

Changes will likely need to be made in the neutron code to take advantage of
this mechanism.  The current mechanism of using trusts should still work.

Implementation
==============

Assignee(s)

Primary assignee:
    alee
    rm_work

Work Items
----------

* Add new fields to the database tables, and new parameters to the REST API.

* Add logic to stamp secrets/containers created through the REST API or through
  Orders with their creator.

* Add logic to retrieve ACL data from the database and provide to RBAC layer as
  target.* attributes.

* Modify policy rules based on these target.* attributes.  It may be necessary to
  extend oslo policy here to process lists correctly.

* Modify neutron code/barbican client to take advantage of this new mechanism.

Dependencies
============

None

Testing
=======

The current unit tests will also be modified to have this change reflected upon
them.

Documentation Impact
====================

Neutron and Barbican docs will need to be changed.

References
==========

* Diagram showing Neutron LBaaS detailed flow. http://goo.gl/Wc8oIj

* Earlier blueprint with similar ideas.
  https://blueprints.launchpad.net/barbican/+spec/secret-isolation-at-user-level
