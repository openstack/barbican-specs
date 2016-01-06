..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===============================
Multiple Secret Backend Support
===============================

https://blueprints.launchpad.net/barbican/+spec/multiple-secret-backend

Barbican provides capability to create and store secrets via its plugin
mechanism. Barbican has KMIP, PKCS11 and Dogtag KRA plugins for creating and
managing secrets in HSM (Hardware Security Module) devices and `store_crypto`
plugin for storing secrets in its own database. Enterprises generally have HSM
devices to safely secure their key material (data-at-rest) and to meet
compliance requirements.

Problem Description
===================

Barbican supports secret storage in HSM device as well as in a database. So far,
Barbican has implicit concept of configuring one active plugin for secret store
which means all of the new secrets are going to be stored via same plugin (i.e.
same storage backend). This approach can limit the usage of barbican in a cloud
deployment where not all services/applications have similar data sensitivity
and hence don't have necessity of using same mechanism to safeguard their key
material. Also HSM are expensive devices which have limited storage capacity
and performance characteristics in comparison to database. Current one active
secret store plugin approach is inflexible for following use cases.

* In a deployment, a deployer may be okay storing their dev/test resources
  using a low-security secret store, such as one backend by software-only
  crypto, but may want to use an HSM-backed secret store for production
  resources.
* In a deployment, for certain use cases where a client requires high
  concurrent access of stored keys, HSM might not be a good storage backend.
  Also scaling them horizontally to provide higher scalability is a costly
  approach with respect to database.
* HSM devices generally have limited storage capacity so a deployment will
  have to watch its stored keys size proactively to remain under the limit
  constraint. This is applicable in KMIP backend more than with PKCS11 backend
  because of plugin's different storage approach. This aspect can also result
  from above use case scenario where deployment is storing non-sensitive (from
  dev/test environment) encryption keys in HSM.
* Barbican running as IaaS service or platform component where some class of
  client services have strict compliance requirements (e.g. FIPS) so will use
  HSM backed plugins whereas others may be okay storing keys in software-only
  crypto plugin.

Currently there is no simple solution to support above use-cases. The suggestion
of asking deployment to have a separate barbican endpoint per plugin is not a
good usage experience. See :ref:`alternatives-label` section for further
details. As there is only one secret store backend available, so client (or
class of services) does not have choice to choose backend regardless of its
requirement related to safeguarding those keys.

Proposed Change
===============

Proposal is to allow multiple secret store backends available in a single
barbican deployment. As part of this change, client will have the choice to
select preferred backend at a project level.

For this, a Barbican deployment will have at least 2 secret storage backend
enabled and need to specify one backend to be a deployment level (global)
default option. Project administrators will have choice of pre-selecting one
backend as the preferred choice for secrets created under that project. Any
**new** secret created under that project will use the preferred backend to
store its key material. When there is no project level storage backend
selected, then new secret will use the global secret storage backend.

As each secret already has metadata which captures its backend information,
Barbican can allow clients to specify a specific storage backend when creating
or storing new secret in Barbican. This can be a future enhancement if there is
an actual use case for this requirement. Currently this feature is not in scope
of this effort.

Also to manage this feature, a flag `enable_multiple_secret_stores` will be
added. By default, this flag will be disabled (set as false). So any existing
logic changes need to account for that flag value and modify only when this
flag is enabled. When barbican is started with this flag enabled, then need to
make sure that global default plugin exists in config file.


Change in New Secret Generate and Storage
-----------------------------------------

Currently a secret store plugin expose its generate and store capability via its
`supports` API mechanism. This mechanism checks that input secret specification
(algorithm, length) is supported by plugin backend.

When a project has defined its preferred secret store plugin, then **only** that
plugin is checked for supports criteria. If preferred plugin does not meets the
given input criteria, then 400 error with related message is returned to the
client. For this to happen, `get_active_plugins` logic will return project
level preferred plugin if defined otherwise global default plugin.

This logic of preferred secret generate and store needs to be applied on order
processing side as well. Synchronous or asynchronous order processing logic
needs to be modified so that when generate plugin is looked up for new secret,
it provides project level preferred plugin and then global default plugin. Same
goes for secret store logic needed for symmetric key storage and certificate
storage after certificate are generated successfully during order processing.

Barbican has key wrapping support (in few plugins) to ensure that secret is
pre-encrypted in such a way that only client and backend store can decrypt the
secret. For this client needs to have access to transport key to pre-encrypt
while storing or reading that secret back. With multiple backend support,
reading of transport key will not be impacted. For existing secrets, client
invokes secret metadata API which returns secret transport key id in response
when requested. This id is derived from secret plugin association and plugin
transport key association. For new secrets, if there is project level preferred
backend specified, then returned transport key is specific to that backend. If
project does not have project preferred backend, then it will return transport
key specific to global default backend. This way, there should not be case
where transport key and secret store backends are different.

.. _alternatives-label:

Alternatives
------------

One alternative suggested earlier has been to deploy separate Barbican servers
to have different endpoints for each supported plugin. This may work though it
is not a good usage experience from client perspective. Now onus is on the
client to maintain different barbican endpoints in their configuration and to
remember on their side which secrets is accessed via which endpoint. Also some
of current service integrations generally has support for one barbican endpoint
so this may not be feasible for existing service integrations. Then there is
standard issue of managing these endpoints in keystone service catalog.


Service Configuration impact
----------------------------

To support multiple plugin configuration, following configuration format is
needed as current multiple plugin configuration does not define a clear
relationship between secret store and crypto plugin::

    [secretstore]
    enable_multiple_secret_stores = True
    stores_lookup_suffix = software, kmip, pkcs11, dogtag

    [secretstore:software]
    secret_store_plugin = store_crypto
    crypto_plugin = simple_crypto

    [secretstore:kmip]
    secret_store_plugin = kmip_plugin
    global_default = True

    [secretstore:dogtag]
    secret_store_plugin = dogtag_plugin

    [secretstore:pkcs11]
    secret_store_plugin = store_crypto
    crypto_plugin = p11_crypto

When multiple secret stores support is enabled, then this new list property
`stores_lookup_suffix` is going to be used for looking up relevant supported
plugin names in oslo configuration section constructed using pattern
'secretstore:{suffix}'. One of the plugin must be explicitly identified as
`global_default = True`. There is going be a validation around this
requirement. Ordering of suffix does not matter.

When multiple secret stores are enabled, then property
`enabled_secretstore_plugins` and `enabled_crypto_plugins` are not used for
instantiating supported plugins. Instead above mentioned mechanism is used to
instantiate store and cryto plugins.

Data model impact
-----------------

A new table, **secret_stores**, is going to be created for storing secret store
backend information. At barbican application startup, this table is going to
add new entry for plugins supported, if not present, based on
`secret_store_plugin` and `crypto_plugin` combination. One of secret store
plugin entry in db will have `global_default` column value as true.::

   table_exists = ctx.dialect.has_table(con.engine, 'secret_stores')
   if not table_exists:
      op.create_table(
            'secret_stores',
            sa.Column('id', sa.String(length=36), nullable=False),
            sa.Column('created_at', sa.DateTime(), nullable=False),
            sa.Column('updated_at', sa.DateTime(), nullable=False),
            sa.Column('deleted_at', sa.DateTime(), nullable=True),
            sa.Column('deleted', sa.Boolean(), nullable=False),
            sa.Column('status', sa.String(length=20), nullable=False),
            sa.Column('store_plugin', sa.String(length=255), nullable=False),
            sa.Column('crypto_plugin', sa.String(length=255), nullable=True),
            sa.Column('global_default', sa.Boolean(), nullable=False),
            sa.Column('name', sa.String(length=255), nullable=False),
            sa.PrimaryKeyConstraint('id'),
            sa.UniqueConstraint('store_plugin', 'crypto_plugin',
                                name='_secret_stores_plugin_names_uc'),
            sa.UniqueConstraint('name',
                                name='_secret_stores_name_uc'
      )

At service startup, logic is going to be added to populate secret_stores data.
See :ref:`global_default_api_label` for API. This new table is going to have
crypto_plugin field to account for software-only and PKCS11 plugin case. Both
plugins use `store_crypto` as secret store backend and difference is in crypto
plugin name i.e. `simple_crypto` vs. `p11_crypto` plugin.

Each secret store and crypto plugins implementation going to have new config
'plugin_name' added in their plugin specific configuration. This name is
intended to be user friendly name and we will add default for it which can be
updated via config if needed. This plugin name is going to be used when
creating new secret store entry. For PKCS11 and DB backend, crypto plugin names
are going to be used in `name` field as they do not directly extend secret
store base classes.

Another new table, **project_secret_stores**, is going to be created for storing
per project secret store linking data.::

   table_exists = ctx.dialect.has_table(con.engine, 'project_secret_stores')
   if not table_exists:
      op.create_table(
            'project_secret_stores',
            sa.Column('id', sa.String(length=36), nullable=False),
            sa.Column('created_at', sa.DateTime(), nullable=False),
            sa.Column('updated_at', sa.DateTime(), nullable=False),
            sa.Column('deleted_at', sa.DateTime(), nullable=True),
            sa.Column('deleted', sa.Boolean(), nullable=False),
            sa.Column('status', sa.String(length=20), nullable=False),
            sa.Column('secret_store_id', sa.String(length=36), nullable=False),
            sa.Column('project_id', sa.String(length=36), nullable=False),
            sa.PrimaryKeyConstraint('id'),
            sa.ForeignKeyConstraint(['secret_store_id'], ['secret_stores.id'],),
            sa.ForeignKeyConstraint(['project_id'], ['projects.id'],),
            sa.UniqueConstraint('secret_store_id',
                                'project_id',
                                name='_project_secret_stores_id_project_uc')
      )
      op.create_index(op.f('ix_project_secret_stores_store_id'), 'project_secret_stores',
                      ['secret_store_id'], unique=False)
      op.create_index(op.f('ix_project_secret_stores_project_id'), 'project_secret_stores',
                      ['project_id'], unique=False)

This table is going to be populated whenever a project level preferred store is
set. In case of update of existing project preferred value, existing entry will
be removed and new entry is going to be created.

There is no database migration needed as its a new feature.

REST API impact
---------------

There is no impact on *existing* barbican API(s). Following new REST API are
going to be added to support this new feature.

In case multiple secret store backend is not enabled and response is not
applicable ( `/secret-stores/{ID}/preferred` ), secret-stores related API will
result in error 404.


Listing All Secret Store Backend
--------------------------------

A new API resource tree `/secret-stores` is going to be added for managing
available secret storage backends. This API will provide list of available
secret store backends. Only project administrator (users with admin role) are
authorized to invoke this API.

REST API: GET /secret-stores::

   Request:
      GET /secret-stores
      Headers:
         X-Auth-Token: 'f9cf2d480ba3485f85bdb9d07a4959f1'

   Response:
      HTTP/1.1 200 OK

      {
         "secret-stores":[
            {
               "name": "PKCS11 HSM",
               "global_default": True,
               "secret_store_ref": "http://localhost:9311/v1/secret-stores/4d27b7a7-b82f-491d-88c0-746bd67dadc8",
               "store_plugin": "store_crypto"
               "crypto_plugin": "p11_crypto",
               "secret_store_id": "4d27b7a7-b82f-491d-88c0-746bd67dadc8",
               "status": "ACTIVE",
               "created": "2016-08-22T23:46:45.114283",
               "updated": "2016-08-22T23:46:45.114283"
            },
            {
               "name": "KMIP HSM",
               "global_default": False,
               "secret_store_ref": "http://localhost:9311/v1/secret-stores/93869b0f-60eb-4830-adb9-e2f7154a080b",
               "store_plugin": "kmip_plugin",
               "crypto_plugin": None,
               "secret_store_id": "93869b0f-60eb-4830-adb9-e2f7154a080b",
               "status": "ACTIVE",
               "created": "2016-08-22T23:46:45.124554",
               "updated": "2016-08-22T23:46:45.124554"
            },
            {
               "name": "Software Only Crypto",
               "global_default": False,
               "secret_store_ref": "http://localhost:9311/v1/secret-stores/0da45858-9420-42fe-a269-011f5f35deaa",
               "store_plugin": "kmip_plugin",
               "crypto_plugin": "simple_crypto",
               "secret_store_id": "0da45858-9420-42fe-a269-011f5f35deaa",
               "status": "ACTIVE",
               "created": "2016-08-22T23:46:45.127866",
               "updated": "2016-08-22T23:46:45.127866"
            }
      }

Reading A Secret Store Backend
------------------------------

Any barbican user can invoke a given secret store backend info. Returned name is
derived from friendly plugin name defined in service configuration. Each secret
store plugin and crpyto plugins have default name which can be overriden in
`plugin_name` under that plugin section. Only project administrator (users with
admin role) are authorized to invoke this API.

REST API: GET /secret-stores/{secret_store_id}::

   Request:
      GET secret-stores/4d27b7a7-b82f-491d-88c0-746bd67dadc8
      Headers:
         X-Auth-Token: 'f9cf2d480ba3485f85bdb9d07a4959f1'

   Response:
      HTTP/1.1 200 OK

      {
         "name": "PKCS11 HSM",
         "global_default": True,
         "secret_store_ref": "http://localhost:9311/v1/secret-stores/4d27b7a7-b82f-491d-88c0-746bd67dadc8",
         "store_plugin": "store_crypto"
         "crypto_plugin": "p11_crypto",
         "secret_store_id": "4d27b7a7-b82f-491d-88c0-746bd67dadc8",
         "status": "ACTIVE",
         "created": "2016-08-22T23:46:45.114283",
         "updated": "2016-08-22T23:46:45.114283"
      }

Preferred Secret Store Backend
------------------------------

Project administrator (users with admin role) can add, update or remove
preferred secret store backend for their project. Project information is
derived from project scoped token. In a barbican deployment without keystone
setup, project information is obtained from `X-Project-Id` request header.

Getting Per Project Secret Store Backend
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Only project administrator can request a reference to the preferred secret store
if assigned previously. When a preferred secret store is set for a project,
then new project secrets are stored using that store backend. If multiple
secret store support is not enabled, then this resource will return 404 (Not
Found title) error.

REST API:  GET /v1/secret-stores/preferred::

   Request:

   GET /v1/secret-stores/preferred
   Headers:
      X-Auth-Token: "f9cf2d480ba3485f85bdb9d07a4959f1"
      Accept: application/json

   Response:

   HTTP/1.1 200 Ok
   Content-Type: application/json

   {
      "name": "KMIP HSM",
      "global_default": False,
      "secret_store_ref": "http://localhost:9311/v1/secret-stores/93869b0f-60eb-4830-adb9-e2f7154a080b",
      "store_plugin": "kmip_plugin",
      "crypto_plugin": None,
      "secret_store_id": "93869b0f-60eb-4830-adb9-e2f7154a080b",
      "status": "ACTIVE",
      "created": "2016-08-22T23:46:45.124554",
      "updated": "2016-08-22T23:46:45.124554"
   }


Setting Per Project Secret Store Backend
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Only project administrator can set the per project backend. This API will set or
update the per project backend for any new secrets created after that change.

REST API: POST /secret-stores/{secret_store_id}/preferred::

   Request:

   POST /secret-stores/7776adb8-e865-413c-8ccc-4f09c3fe0213/preferred
   Headers:
      X-Auth-Token: '2f30ca66d27d4d0cbff51d44bc5ac66e'

   Response:

   HTTP/1.1 204 No Content

Deleting Per Project Secret Store Backend
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Only project administrator can remove the per project backend.

REST API: DELETE /secret-stores/{secret_store_id}/preferred::

   Request:

   DELETE /secret-stores/7776adb8-e865-413c-8ccc-4f09c3fe0213/preferred
   Headers:
      X-Auth-Token: '2f30ca66d27d4d0cbff51d44bc5ac66e'

   Response:

   HTTP/1.1 204 No Content

Global Default Secret Store Backend
-----------------------------------

Global default secret store is used when a project in barbican does not have a
preferred secret store setting. In that case, new secrets created under that
project will use global secret store to generate and store keys.

Global default secret store value is not expected to change that frequently so
its value is managed via service configuration. At service startup,
global_default value must be explicitly set in one of plugin related
configuration section.

Changing global default secret store plugin in configuration should not impact
project (without preferred plugin) *existing* secrets as those secrets metadata
in db already has information about its associated plugin backend.

.. _global_default_api_label:

Getting Global Default Backend
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Only project administrator can read the current global default backend value.

REST API: GET /secret-stores/global-default::

   Request:

   GET /secret-stores/global-default
   Headers:
      X-Auth-Token: '2f30ca66d27d4d0cbff51d44bc5ac66e'
      Accept: 'application/json'

   Response:

   HTTP/1.1 200 Ok
   Content-Type: application/json

   {
      "name": "PKCS11 HSM",
      "global_default": True,
      "secret_store_ref": "http://localhost:9311/v1/secret-stores/4d27b7a7-b82f-491d-88c0-746bd67dadc8",
      "store_plugin": "store_crypto"
      "crypto_plugin": "p11_crypto",
      "secret_store_id": "4d27b7a7-b82f-491d-88c0-746bd67dadc8",
      "status": "ACTIVE",
      "created": "2016-08-22T23:46:45.114283",
      "updated": "2016-08-22T23:46:45.114283"
   }

Operation `POST` or `DELETE` on `/secret-stores/global-default` is not allowed.
So invoking that resource should generate 405 (Method Not Allowed) error.

Security impact
---------------

No impact as these are new APIs. Some of new APIs will require service
administrator privileges to make changes. As result of this change, deployments
will have choice to use database as secret store backend for some secrets.
Database backend is considered less secure backend with reference to HSM
storage option. But this feature in itself does not introduce a security risk
as database backend is barbican existing backend and deployer can choose not to
use it.


Notifications & Audit Impact
----------------------------

None

Python and Command Line Client Impact
-------------------------------------

These new APIs need to be supported in barbican client API and CLI (command line
interface).

Other end user impact
---------------------

There should not be impact on existing deployment as its a new feature. By
default, this feature will be disabled. Once this feature is enabled, clients
will have choice to define and use specific backend for a project. To use this,
clients will initially have to use new APIs to set their project level
preferred secret store plugins.

Performance Impact
------------------

Minimal. On API usage level, this is expected to be infrequent admin operation.
Impact related to creating new secret and storing secret flow will be also very
minimal.


Other deployer impact
---------------------

This new feature is going to be disabled by default. There is new flag added to
enable i.e. `enable_multiple_secret_stores` in `secretstore` section of
barbican configuration. Also for multiple plugins support, deployer will need
to define supprted plugins configuration as mentioned above (different from way
currently supported).

There is no migration needed and deployment should work with default
configuration as-is.

Developer impact
----------------

There should not be any impact on plugin developers and existing API.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  arunkant

Other contributors:

Work Items
----------

* Add documentation for secret stores API.
* Add model layer changes.
* Logic to populate secret_stores table data.
* Add logic to parse multiple plugin configuration.
* Add controller layer changes for new api.
* Modify secret create and store logic to leverage this feature.The feature
  flag needs to be used in related changes.
* Add functional test for new APIs.
* Update/Add existing functional test around multiple backend support.

Dependencies
============

None

Testing
=======

Current unit and functional test will be modified to reflect related changes.
New unit and functional test will be added for new `secret-stores` APIs.

Documentation Impact
====================

This is a new feature that will be documented for new APIs and its usage.


References
==========

1. Transport Key Wrapping
https://github.com/openstack/barbican-specs/blob/master/specs/juno/add-wrapping-key-to-barbican-server.rst