..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===========================================================
Allow Castellan to Support different types of Keystone Auth
===========================================================

Blueprint:
https://blueprints.launchpad.net/castellan/+spec/remove-keystone-dependency

Problem Description
===================

Currently in Castellan in order to obtain access to Barbican via the Barbican
Key manager a `context` containing a valid Keystone `auth-token` must be
provided.

The Swift Keymaster under development[1] will access Castellan in order to
obtain Secrets from Barbican, but would like to have a Keystone service user
with access to all keys and in the future be able to have independent user
access. For a service user, a `context` containing a Keystone `username`
and `password` will be used. Castellan must support this.


Proposed Change
===============

The proposed change is to allow Castellan to be able to check the context for
the Barbican Key Manager and be able to determine what type of Keystone
authentication function to use.

There will be a new type of hierarchal context object which will be passed from
the user/service to Castellan. It will be used instead of oslo.context.

The hierarchal context will consist of a `Credential` object as the parent
class and the children will be:

1.) `TokenCredential`, for authenticating with a token.

2.) `PasswordCredential`, for authenticating with a username and password.

3.) `CertificateCredential`, for authenticating with a certificate.

The context is first checked to see what type of object it is, after that we
determine which Keystone auth-type to use.

.. note::

  oslo.context still needs to be supported until it is deprecated in the
  future. `tenant` and `project_id` should be backwards compatible. Patch
  https://review.openstack.org/#/c/235671/ adds check for project_id, It
  allows avoidance of confusion between `tenant` and `project_id` and is
  necessary if passing a keystone auth-middleware object as context.

Example Usage within the swift keymaster:

.. code-block :: python

  from castellan.common.objects import symmetric_key
  from castellan import credential
  from castellan import key_manager

  CONF = <swift conf>

  # If keystone auth-middleware exists then it provides the authentication
  # for the logged in user[2].
  ks_context = env.get('keystone.token_auth').user

  # Depending on how the configuration is setup, then a different 'credential` object
  # will be created and processed by Castellan. More information on this can be
  # found in examples below.
  context = credential.credential_factory(context=ks_context, conf=CONF)

  # create a key
  key = symmetric_key.SymmetricKey('aes', 128, '==8hykeh')
  manager = key_manager.API(configuration=CONF)
  stored_key_id = manager.store(context, key)


The swift configuration must be altered to contain certain variables that
Castellan will use for authentication.

For password-based authentication the user must provide the following in the
Swift Configuration:

.. code-block:: ini

    [castellan]
    auth_type = 'password'
    username = 'swift'
    password = 'xswift'
    project_id = '1a4a0618b306462c9830f876b0bd6af2'
    domain_id = '5abc43'

.. note::

  The above is an example on how it can be used in the Swift Keymaster. It is
  just provided for demonstration purposes.

The `auth_type` variable will be used in order to determine what type of `credential`
Castellan should create. The possible auth_types will be `token`, `password` and
`certificate`. Castellan will contain a context factory in order to process
the different auth_types. Below is an example of the context factory.

.. code-block:: python

    def credential_factory(context=None, conf=None):
        if CONF.castellan.auth_type == 'token':
            if context:
                auth_token = context.auth_token
            else:
                auth_token = CONF.castellan.auth_token

            context = catstellan.KeystoneTokenContext(auth_token,
                                                      CONF.castellan.auth_url,
                                                      CONF.castellan.project_id)
        elif CONF.castellan.auth_type == 'password':
            context = castellan.PasswordContext(CONF.castellan.username,
                                                CONF.castellan.password,
                                                CONF.castellan.auth_url,
                                                CONF.castellan.project_id,
                                                CONF.castellan.domain_id)
        elif CONF.castellan.auth_type == 'certificate':
            context = castellan.CertificateContext(CONF.castellan.public_key,
                                                   CONF.castellan.private_key)

        return context

Alternatives
------------

A Keystone Auth-Token can be derived using Username and Password. An
oslo.context object containing the Auth-Token can be created and passed to a
Castellan call.

Data model impact
-----------------

None

REST API impact
---------------

None

Security impact
---------------

If v3.Password is being used, then the username, password, project_id, and
domain information(id or name) must be stored.

Notifications & Audit Impact
----------------------------

None

Python and Command Line Client Impact
-------------------------------------

None

Other end user impact
---------------------

None

Performance Impact
------------------

None

Other deployer impact
---------------------

None

Developer impact
----------------

Developer has more options of context that can be passed to Castellan.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  diazjf

Other contributors:
  rellerreller


Work Items
----------

1. Create support for multiple Keystone auth types in Castellan for the
Barbican Key manager.
2. Add documentation on how to generate context for the Barbican Key Manager.

Dependencies
============
None


Testing
=======

New unit tests and functional tests need to be added.


Documentation Impact
====================

Castellan documentation must be updated to include these changes.


References
==========
[1] https://github.com/openstack/swift-specs/blob/master/specs/in_progress/at_rest_encryption.rst
[2] https://github.com/openstack/keystonemiddleware/blob/1047cececf68c3c22add66b6a8a1c499a667c036/keystonemiddleware/auth_token/__init__.py#L171-L174
