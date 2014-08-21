..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Transport Key Wrapping
==========================================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/barbican/+spec/add-wrapping-key-to-barbican-server

Secrets are currently passed from Barbican clients to the Barbican server
encrypted only by TLS.  For Common Criteria environments, this is 
insufficient.  Secrets need to be encrytped within the TLS stream at the point
of origin, and ideally only decrypted when the secret is stored.  In the case
where a hardware token is used, this would happen in the token, so that even if
an attacker were able to gain access to the Barbican server, and was able to 
introspect the process memory, no secrets could be deciphered.

Problem Description
===================

See above.

Proposed Change
===============

**Mechanism for Storing Keys using Key Wrapping:**

#. Client elects to use key wrapping to store a secret, and therefore will use
   a two step storage mechanism for its secret.
#. Client does a POST to /secrets without a payload.  Client also specifies
   transport_key_needed=true in the request.
#. Server notices that transport_key_needed=true and looks for a plugin
   that supports storage with key wrapping. 

The logic is something like this ::

   for (plugin; list_of_plugins):
       call plugin.get_transport_key()
       if transport_key_found:
           put transport_key URL in transport_key_ref in the response.
           return success
   return failure (No plugin available 400)

4. Note that plugin.get_transport_key() will be defined within the abstract
   base secret_store class to return None.  This means that plugins that wish
   to support key wrapping will need to override this method to return a
   transport_key_id (a reference to the key in the transport key repo).
#. It is expected that the plugin will be responsible for populating the 
   TransportKeyRepo with the correct key.  This could occur, for example, 
   on module startup, or when the first call to get_transport_key() is made.
#. Client gets the transport key from the transport key resource.  To
   improve performance, the client will probably cache this key.
#. Client uses transport key to do key wrapping as expected.
   Specifically - wrap the secret with a session key, then wrap session
   key with transport key, then place in ASN1 structure.
#. Client returns encrypted blob and the transport key reference in the PUT
   /secret/{id} request.  We can use the parameter transport_key_ref.
#. Server notices that transport_key_ref has been set.  It looks up the plugin
   associated with that transport key from the TransportKeyRepo.
   If the plugin is not available or the transport key does not exist, throw
   failure (400)
#. Pass the parameter transport_key_id in the SecretDTO, and call
   store_secret().
#. Plugin detects that transport_key_id is set, and processes accordingly.

**Mechanism for Retrieving Keys using Key Wrapping:**

#. Client elects to retrieve a secret using key wrapping.  It first retrieves
   the metadata using a GET.
#. If the plugin that stored the secret supports key_wrapping and a
   transport key exists, return the URL for the transport key in 
   transport_key_ref.
#. The client fetches the transport key and caches it.
#. Client generates a symmetric key (session key) and wraps it with the
   transport key.  Client sends the trans_wrapped_session_key to the server
   as part of the GET request for the secret.
#. The server writes this key (trans_wrapped_session_key) into the
   secret_metadata and calls get_secret()
#. Plugin detects that this parameter has been written and retrieves the
   secret accordingly.  The storing server will extract the session key
   and will encrypt the response with the session key.
#. On returning the secret, the plugin will remove the
   trans_wrapped_session_key from the secret_metadata.
#. Client will decrypt the response from the server using the session key.

Alternatives
------------

The above solution is the culmination of a series of design discussions
at the Atlanta summit.  An old design can be found here:
http://pki.fedoraproject.org/wiki/Barbican_server_add_wrapping_key

Data model impact
-----------------

This change requires a new resource to be made available through the REST 
interface and stored in the database - transport keys.

The code to implement this new resource/model has already been merged.
https://review.openstack.org/94875 - Add transport key as a resource.

These are new objects.  They will be populated by any plugins that support
key wrapping whenever the get_transport_key() method is invoked.

REST API impact
---------------

The following API methods will be changed:

* POST /secrets

  * A new optional parameter will be added "transport_key_needed" which will
    have the possible values of "true|false".  The parameter defaults to
    "false".
  * This parameter will have an effect when the two step storage mechanism is
    used.  In this case, when there is no payload, if transport_key_needed
    is set to true, then a suitable plugin will be found and a transport key
    URL will be returned in the response in the field "transport_key_ref".
  * Return Error 400 if a suitable plugin/transport key cannot be found when
    transport_key_needed=true.

* POST /secrets (with a payload)
   * A new optional parameter will be added: "transport_key_ref".  This will
     contain the URL for the transport key used to encode the secret.  Use
     of this parameter will imply that the secret has been wrapped.
   * Return error 400 if the suitable plugin/transport key cannot be found.

* PUT /secrets/foo
   * A new optional parameter will be added: "transport_key_ref".  This will
     contain the URL for the transport key used to encode the secret.  Use
     of this parameter will imply that the secret has been wrapped.
   * Return error 400 if the suitable plugin/transport key cannot be found.

* GET /secrets/foo (metadata only)
   * If the secret is stored by a plugin that supports key_wrapping, then
     the a new parameter "transport_key_ref" will be added to the metadata
     response.  This will contain a reference to the transport key for that
     plugin.

* GET /secrets/foo (the actual secret)
   * A new optional query parameter trans_wrapped_session_key
     will be added to this method.  This parameter will contain a session
     key wrapped by a transport key to be used for retrieving the secret.

* GET/DEL/POST /transport-keys
   * This is mentioned here for completeness.  The methods for these resources
     have already been merged into the code.
   * GET will be used by clients to retrieve a transport key.
   * POST/DEL can be used by plugin administrators to add or remove transport
     keys.  These operations will not be available to ordinary users.

Security impact
---------------

This change should improve security in providing end-to-end encryption
for secrets.

Notifications & Audit Impact
----------------------------

None, other than probably some changed notifications if this mechanism is
used instead of the regular mechanism.

Other end user impact
---------------------

None

Performance Impact
------------------

Doing this extra encryption will mean that storing and retrieving secrets will
take two steps, rather than one, so that the correct transport key can be
retrieved.  This could be mitigated though by having the client cache the
transport key - especially given that transport keys change very infrequently.

There will be some addtional work performed by the client, both in encrypting
and decrypting the secrets, but this is likely not to be onerous.

Other deployer impact
---------------------

Client work is required.  We will cover this in a separate spec.

Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  alee-3

Other contributors:
  redrobot (client side)

Work Items
----------

* CR performing the following for storage using transport key:
  *  Add transport_key_needed boolean to POST request flows
  *  Add get_transport_key() method to SecretStore
  *  Return transport key if requested.
  *  Modify secret PUT to add transport_key_ref.
  *  Lookup secret_store plugin associated with that transport_key
  *  Modify the secret_store SecretDTO to accept an optional transport key

* CR performing the following for retrieval using transport key
  * Modify secret metadata GET to add a reference to the transport key
  * Add trans_wrapped_session_key to secret GET.
  * Write this parameter into the secret_metadata and provide to the plugin.
  * Modify the plugin to act accordingly.

* CR to implement fetching the transport key and end to end testing in Dogtag.

Dependencies
============

* https://review.openstack.org/94875 Add transport key as a resource (Done)
* Barbican Client work


Testing
=======

All new functionality and changes in the API/ secret store / Dogtag plugin
will be unit tested.  In addtion, we will perform end to end integration
tests using Dogtag as the backend.  Ideally, these end to end tests will be
made part of the gate testing, with Dogtag being installed using a Chef recipe.

Also eventually, transport key functionality will be added to the dev plugin
and will therefore be tested too.

Documentation Impact
====================

This is a brand new feature and will need to be described in the docs.

References
==========

* https://review.openstack.org/94875 Add transport key as a resource CR.

* An old (since abandoned) implementation: https://review.openstack.org/93165

* Discussion at meetup: https://etherpad.openstack.org/p/barbican-juno-meetup

* Discussion at Atlanta summit:
  https://etherpad.openstack.org/p/barbican-juno-roadmap

