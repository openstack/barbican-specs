..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Add quota support for Barbican resources
==========================================

https://blueprints.launchpad.net/barbican/+spec/quota-support-on-barbican-resources

Barbican REST API doesn't impose any upper limit on the number of resources
allowed per project. This could result in resource explosion. This blueprint
proposes a way to specify and enforce quotas with projects. Quotas are
operational limits so that cloud resources are optimized.


Problem Description
===================

Here are few scenarios that could impact the normal functioning of the
Barbican server:

* A client could place requests for several thousands of orders to create
  secrets for a single project. This might overwhelm the Barbican server
  both in terms of processing time and disk space consumed

* If a buggy client script runs amok and attempts to create a generic
  type container with no associated secret, it could quickly fill-up
  the Barbican database!

* If a user creates a large number of projects and further creates a
  large number of Barbican resources per these projects, that could
  impact other genuine users

Note: The last point could be avoided if keystone could enforce a
upper limit on the number of projects that an user could create. Hence
this is treated as a lower priority for Barbican for now. This spec
aims to implement project level quotas first. A later spec is expected
to add support for user level quotas.

This is similar to the quota enforcement done by nova and cinder
services.


Proposed Change
===============

Introduce quotas for all Barbican resources. The quotas should have
unlimited values as defaults, to ensure backwards compatibility with
the current code that supports no quotas. The following resources will
have quota support:

* secrets
* orders
* containers
* transport_keys
* consumers

*Note:*

This proposal is a simpler subset of the quota implementation done
by Nova and Cinder. Barbican does not have any reservable resources
and so the quotas are much simpler in that there is no usage tracking and
reservations. Also, project-user level quota enforcement is not covered
by this spec.

***Enforcing quotas:***

Barbican API controllers will be updated with quota logic for the
create resource methods. Once implemented, the quota check will work
as follows for resource creation requests:

1. Get the quotas for the project retrieved from the auth context.
   If per-project quotas have not been setup, use default quotas
2. Get a count of the resources for the context project
   Note: Currently secrets that expire or are removed are not hard removed
   from the database, but rather are soft deleted, so they still are using
   resources within Barbican and would be counted against the project's
   quota. However, this is deployment dependent. A deployment could have a
   process that hard removes such resources after a period of time
3. If the count equals or exceeds the quotas, reject the request with
   the following error message:
   HTTP 403 Forbidden
   {"error": "Quota exceeded for <project_id>. Only <count> <resource>s
   are allowed"
   }
4. Continue with the resource creation


Update the Barbican config file to include the following section for
quota limits:

::

    # ====================== Quota Options ===============================

    [quotas]
    # For each resource, the default maximum number that can be used for
    # a project is set below.  This value can be overridden for each
    # project through the API.  A negative value means no limit.  A zero
    # value effectively disables creation of the resource.

    # default number of secrets allowed per project
    quota_secrets = -1

    # default number of orders allowed per project
    quota_orders = -1

    # default number of containers allowed per project
    quota_containers = -1

    # default number of transport_keys allowed per project
    quota_transport_keys = -1

    # default number of consumers allowed per project
    quota_consumers = -1


A number >=0 for the quota_<value> indicates the max
limit for that resource and a negative value means unlimited.  A value
of zero indicates a maximum of zero instances of that resource, effectively
disabling creation of that entity.  To ensure backwards compatibility, the
provided default for the defaults will be -1 (unlimited) for each resource.

While these generic quotas apply to all projects, there will
also be support to define and enforce quotas per project.
The priority in which the quotas are enforced is then:
[per project quotas] => [default quotas]

where,
"per project quotas" - these quotas are directly associated with a particular
project id. Any changes to these quotas will impact only that project.

"default quotas" - if no per project quotas are specified, the
default quotas are used. These would typically be the values supplied in the
config file.

The default quotas are stored in the config file (as shown above) but
per-project quotas are stored in db.

A REST API for Barbican administrators for the quota CRUD operations will be
implemented as well. Non-admin users will be provided with a REST API to get
their own effective quotas for various resources. Keystone RBAC checks will
be employed to decide if a caller has the required admin role to perform
these admin-only operations. The details of this is discussed in a later
section below.


Alternatives
------------

An attempt was made to create an oslo.common library with quota support
for all OpenStack projects.  The first attempt was committed to oslo, however
it has been deprecated and has not been adopted by any projects.  A second
attempt was started, but has been put on hold with no current plans to restart.
There is no other common OpenStack library implementing quotas for Barbican
to adopt.

The quota configuration and logic will be derived by looking at quota
implementations done by other OpenStack projects like nova, cinder
and neutron.  A simplified implementation will developed for Barbican by
using APIs and logic similar to Nova's implementation, while removing unneeded
features, such as pluggable backend drivers and resource reservation.

Another alternative is an initiative by Kevin Mitchell from Rackspace
https://wiki.openstack.org/wiki/Boson. However, the Nova and Cinder design
is more usable for Barbican.


Data model impact
-----------------

The following new data models will be added:

* ProjectQuota

  Represents quotas override for a project.

  If there is no row for a given project id, then the
  default for the deployment is used.  If the quota value for a resource
  is null, then the default for that resource for the deployment is used.
  If the quota value for a resource is 0, creation of that resource is
  disabled.  If the quota value for a resource is -1, creation of that
  resource is not limited by quota enforcement logic.

  Schema: (table name: **project_quotas**)

  * project_id:     String(36) ForeignKey projects.id, nullable=False
  * secrets:        Integer, nullable=True
  * orders:         Integer, nullable=True
  * containers:     Integer, nullable=True
  * transport_keys: Integer, nullable=True
  * consumers:      Integer, nullable=True

  **Constraints**: project_id must be unique
                   project_id must exist as projects.id


* Changes to existing models:

No existing models will be impacted by this addition. However, it needs
to be investigated if new indexes need to be built to speed up resource
consumption lookups.


REST API impact
---------------

The following new REST API will be implemented to manage quotas CRUD
operations. Please note that except for the first GET API, all the
other APIs require the caller to have admin role.

* Get effective quotas (any Barbican user)

  * Returns effective resource quotas for the caller for the specified
    project. If there are no project specific quotas returns the
    deployment default resource limits.

  * GET /v1/quotas

  * Normal http response code(s)
    200 OK

  * Expected error http response code(s)

    * 401 Unauthorized - If the auth token is not present or invalid.
                         Also, if using the unauthenticated context and
                         the X-Project-Id header is not present in the request.

  * Required request headers

    X-Auth-Token, if using keystone auth

    X-Project-Id, if using unauthenticated context

  * Parameters

    None

  * JSON schema definition for the body data if allowed

    None

  * JSON schema definition for the response data if any

    EXAMPLE::

        {
          'type': 'object',
          'properties': {
              'quotas': {
                'type': 'object',
                'properties': {
                  'secrets': {'type':'integer'}
                  'orders': {'type':'integer'},
                  'containers': {'type':'integer'},
                  'transport_keys': {'type':'integer'}
                  'consumers': {'type':'integer'}
                 },
                'additionalProperties': False
              }
          },
          'additionalProperties': False
        }

    * Example 1::

        A non-admin user checking the resource quotas using a token scoped to a
        particular project

        Request:

          GET /v1/quotas

          X-Auth-Token:<token>

        Response:

          200 OK

          Content-Type: application/json

          {
            "quotas": {
              "secrets": 10,
              "orders": 20,
              "containers": 10,
              "transport_keys": 10,
              "consumers": -1
            }
          }

* List all project quotas (admin only)

  * Lists all configured project level resource quotas across all users for all
    projects. If a project does not have project specific quotas configured,
    that project is not included in the returned list.
    If there are only project specific quotas for a subset of resources
    for a project, this call will return null for those resources without a
    configured value in that project. The returned list will be sorted
    by create date, and support standard limit/offset paging.

  * GET /v1/project-quotas?limit=x&offset=y (Admin only)

  * Normal http response code(s)
    200 OK

  * Expected error http response code(s)

    * 401 Unauthorized - If the auth token is not present or invalid.
                         Also, if using the unauthenticated context and
                         the X-Project-Id header is not present in the request.

  * Required request headers

    X-Auth-Token, if using keystone auth

  * Parameters

    limit(optional), integer, maximum number of records retrieved
    offset(optional), integer, number of records to skip

  * JSON schema definition for the body data if allowed

    None

  * JSON schema definition for the response data if any


    EXAMPLE::

        {
          'type': 'object',
          'properties': {
              'project_quotas': {
                'type': 'array'
                'items': {
                  'type': 'object',
                  'properties': {
                     'project_id': {'type':'string'},
                     'project_quotas': {
                          'type':'object',
                          'properties': {
                             'secrets': {'type': 'integer'},
                             'orders': {'type': 'integer'},
                             'containers': {'type': 'integer'},
                             'transport_keys': {'type': 'integer'},
                             'consumers': {'type': 'integer'}
                          }
                     }
                   }
                 }
                }
             },
          'additionalProperties': False
        }

    * Example 1::

        An admin user listing all the project quotas

        Request:

          GET /v1/project-quotas

          X-Auth-Token:<token>

        Response:

          200 OK

          Content-Type: application/json

          {
            "project_quotas": [
              {
                "project_id": "1234",
                "project_quotas": {
                     "secrets": 2000,
                     "orders": 0,
                     "containers": -1,
                     "transport_keys": 100,
                     "consumers": null
                 }
              },
              {
                "project_id": "5678",
                "project_quotas": {
                     "secrets": 200,
                     "orders": 100,
                     "containers": -1,
                     "transport_keys": 0,
                     "consumers": null
                 }
              },
            ]
          }


* Get quotas for a specific project (admin only)

  * Returns a set of configured resource quotas for the specified project.
    If no project specific quota values have been configured (or if the
    project does not exist), the API responds with Not Found.  If there are
    only project specific quotas for a subset of resources for a project, this
    call will return null for those resources without a configured value in
    that project.

  * GET /v1/project-quotas/{project_id}

  * Normal http response code(s)
    200 OK

  * Expected error http response code(s)

    * 401 Unauthorized - If the auth token is not present or invalid.
                         Also, if using the unauthenticated context and
                         the X-Project-Id header is not present in the request.
    * 404 Not Found - If there are no project quota settings to delete
                      for the specified project.

  * Required request headers

    X-Auth-Token, if using keystone auth

    X-Project-Id, if using unauthenticated context

  * JSON schema definition for the body data if allowed
    None

  * JSON schema definition for the response data if any::

        {
          'type': 'object',
          'properties': {
               'project_quotas': {
                  'type':'object',
                  'properties': {
                    'secrets': {'type': 'integer'},
                    'orders': {'type': 'integer'},
                    'containers': {'type': 'integer'},
                    'transport_keys': {'type': 'integer'},
                    'consumers': {'type': 'integer'}
                  }
             }
          },
          'additionalProperties': False
        }

    * Example::

        Request:

          GET /v1/project-quotas/1234

          X-Auth-Token:<token>

        Response:

          200 OK

          Content-Type: application/json

          {
            "project_quotas": {
              "secrets": 10,
              "orders": 20,
              "containers": -1,
              "transport_keys": null,
              "consumers": 10
            }
          }


* Update/Set quotas for a specific project (admin only)

  * Creates or updates the configured resource quotas for the specified
    project. It is not required to specify limits for all Barbican resources.
    If a value for a resource is not specified, the default limits will be
    used for that resource.  If the specified project is not previously
    known to Barbican, a new entry to the projects table will be created.

  * PUT /v1/project-quotas/{project_id}

  * Normal http response code(s)

    204 No Content

  * Expected error http response code(s)

    * 401 Unauthorized - If the auth token is not present or invalid.
                         Also, if using the unauthenticated context and
                         the X-Project-Id header is not present in the request.
    * 400 Bad Request - If the request payload doesn't confirm to schema

  * Required request headers

    X-Auth-Token, if using keystone auth

    X-Project-Id, if using unauthenticated context

    Content-Type, application/json

  * JSON schema definition for the body data if allowed::

        {
          'type': 'object',
          'properties': {
             'project_quotas': {
                  'type':'object',
                  'properties': {
                     'secrets': {'type': 'integer'},
                     'orders': {'type': 'integer'},
                     'containers': {'type': 'integer'},
                     'transport_keys': {'type': 'integer'},
                     'consumers': {'type': 'integer'}
                  }
             }
         },
         'additionalProperties': False
        }


  * JSON schema definition for the response data if any::
    None

    * Example::

        Request:

          PUT /v1/project-quotas/1234

          X-Auth-Token:<token>

          Body::

            {
              "project_quotas": {
                "secrets": 50,
                "orders": 10,
                "containers": 20
              }
            }


        Response:

          204 OK


* Delete quotas for a specific project (admin only)

  * Deletes the configured resource quotas for the specified
    project.  After this call succeeds, the default resource quotas will be
    returned for subsequent calls by the user to list effective quotas.  If
    there are no project specific quota configuration, or the project is
    not previously known in Barbican, Not Found is returned.

  * DELETE v1/project-quotas/{project_id}

  * Parameters
    None

  * Normal http response code(s)
    204 No Content

  * Expected error http response code(s)

    * 401 Unauthorized - If the auth token is not present or invalid.
                         Also, if using the unauthenticated context and
                         the X-Project-Id header is not present in the request.
    * 404 Not Found - If there are no project quota settings to delete
                      for the specified project or the project is unknown
                      to Barbican.

  * Required request headers

    X-Auth-Token, if using keystone auth

    X-Project-Id, if using unauthenticated context

  * Parameters

    None

  * JSON schema definition for the body data if allowed

    None

  * JSON schema definition for the response data if any

    None

* Example 1::

    Request:

      DELETE v1/project-quotas/1234

      X-Auth-Token:<token>


    Response:

      204 No Content



* Policy changes

  For all admin-only APIs, the caller is expected to have a barbican admin
  role. The check for this will be added to the Barbican policy.json


Once implemented and enforced, all Barbican resource creation API could return
a new error message back to the client if the request exceeded the allowed
quota limits.

Example::

  Request::

    POST /v1/secrets

    X-Auth-Token: <token>

    Content-Type: application/json

    {
      # payload to create secret
    }

  Response::

    403 Forbidden

    Retry-After: 0

    Content-Type: application/json

   {
    "error": "Quota exceeded for <project-id>. Only <count> <resource>s
              are allowed"
   }

* Class Quotas

  Class level quotas are not addressed in this spec. Need another spec to cover the
  data model impact and REST API for associated CRUD operations.


Security impact
---------------

None

Notifications & Audit Impact
----------------------------

None

Other end user impact
---------------------

The Barbican client (python-barbicanclient) has to be enhanced to consume
the Quota REST API mentioned. The following scenarios should be supported.

Quota commands that a regular non-admin barbican user can make:

* List all quotas

  barbican quota show


Quota commands that only a barbican admin can make

* List the default quotas applicable to all new projects

  barbican quota show

* List quotas for a specific project

  barbican quota show --project_id <project>

* Update quotas for a specific project

  barbican quota update --project_id <project> --secrets 50 --orders 10

* Delete per-project quotas for a project

  barbican quota delete --project_id <project>



Performance Impact
------------------

TBD

Other deployer impact
---------------------

The new data models introduced will be added by a new Alembic version file.
If automatic migration is turned OFF, the db migration tool has to be run
manually to effect the changes.

Developer impact
----------------

Developers integrating with Barbican API/client now need to handle the case
where the server could return a quota violation error

Implementation
==============

Assignee(s)
-----------

Dave McCowan (dave-mccowan) will be leading the implementation of the code.

Primary assignee:
  <dave-mccowan>

Other assignees:

Work Items
----------

* Quota db provider source code
* Data model additions
* Alembic migration version script
* Updated default config file with quota section
* python-barbicanclient enhancements to support quota operations
* New unit tests to test quota related source changes
* Update existing resource unit tests to handle quota violation errors
* Functional tests


Dependencies
============

TBD

Testing
=======
New unit tests and functional tests need to be added.


Documentation Impact
====================

* A new section about Quotas has to be documented
* Existing resource API documentation needs to be updated with quota violation
  specific errors


References
==========

TBD
