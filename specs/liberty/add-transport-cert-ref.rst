..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============================
Add Transport Cert Reference
============================

Launchpad blueprint:
https://blueprints.launchpad.net/barbican/+spec/add-transport-cert-ref

Problem description
===================

Transport keys are used to ensure that the secret is pre-encrypted in such
a way that only the client and the back-end store can decrypt the secret.

This is for users which do not trust Barbican, but do trust the back-end
secret store. Alternatively, clients that are required by FIPS or Common Criteria
(CC) requirements only to use CC certified components may be able to argue for
being able to use Barbican if the secrets remain opaque in transit through
Barbican to their final storage back-ends.

Currently, the client gets the transport key from Barbican. But if the client
does not trust Barbican, this is a potential vulnerability. We need to add the
ability for the client to retrieve the transport key from the back-end store
directly.

Proposed change
===============

The change here is straightforward.  Instead of only returning the transport
certificate as a result of the GET /transportkeys/{key_id} call, we will also
return a reference to the transport key in the header (TRANSPORT_KEY_URL)
This reference will be provided by the plugin, and should be a link to a trusted
location where the transport cert could be retrieved.  In practice, this would
most likely be an HTTPS connection to the backend secret store.

Internally, what this means is that the get_transport_key() method in the
secret_store interface would be modified to return a dict containing both the
value of the transport key and an external URL.  We would use this dict to fill
in the contents and header of the response.

As this method already has a None default configuration, this change should not
require any changes in existing plugins.


Alternatives
------------

This is a trivial change that makes the transport key story more complete.

Data model impact
-----------------

None.


REST API impact
---------------

A new field will be added to the header in the response to GET /transportkeys/{key_id}
as described above.


Security impact
---------------

Makes transport keys more secure.


Notifications impact
--------------------

None.


Other end user impact
---------------------

The client would need to decide whether to accept the value of the transport
cert as returned by Barbican, or to retrieve the value from the provided URL.


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

Primary assignee:
  alee


Work Items
----------

1.  The server side changes can likely be done in a single CR.
2.  Client side changes.


Dependencies
============

None.


Testing
=======

Unit and functional tests will need to be written.


Documentation Impact
====================

Docs on transport key use will need to be written and updated.

References
==========

None
