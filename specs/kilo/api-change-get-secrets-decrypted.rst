..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Change GET decrypted secrets to unique URI
==========================================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/barbican/+spec/api-change-get-secrets-decrypted

Currently retrieving the metadata and actual decrypted data for a secret stored
in Barbican uses the same URI, with only the Accept header used to determine
which response to return. This complicates deployments with RBAC systems in
front of Barbican (such as Repose - http://openrepose.org/) or as middleware
(such as EOM - https://github.com/racker/eom). It also requires adding logic
like this to the Barbican app when it is used to enforce RBAC:
https://github.com/openstack/barbican/blob/master/barbican/api/controllers/__init__.py#L53

This blueprint proposes using a unique URI to access decrypted secrets, such as
this: <host>/v1/secrets/<secret-UUID>/payload


Problem Description
===================

Currently to retrieve a secret's metadata and its decrypted data, the same URI
is used, such as this:

GET <host>/v1/secrets/<secret-UUID>

It would be easier to configure RBAC systems that operate in front of a
Barbican deployment if these two retrieval types were distinguished by URI.


Proposed Change
===============

This blueprint calls for adding a new URI structure to retrieve decrypted
secrets as follows:

GET <host>/v1/secrets/<secret-UUID>/payload

So this would include adding a new Pecan sub-controller to handle the 'payload'
requests, adding unit and functional tests for the new 'payload' resource, and
updating API documentation.

The existing Accept-based decryption approach would *not* be removed in the
current 'v1' API version to avoid breaking the current API contract, but could
be removed in the next version ('v2') of the API (not part of this blueprint).


Alternatives
------------

Another option is to use a URI such as this to retrieve decrypted secrets:

GET <host>/v1/secrets/<secret-UUID>.payload

This approach would require parsing within the Pecan controller logic however.


Data model impact
-----------------

None


REST API impact
---------------

This blueprint would add a new URI to retrieve decrypted secrets:

GET <host>/v1/secrets/<secret-UUID>/payload

The Accept header would still be used as now to specify the desired format to
return decrypted secrets in, but the 'application/json' format would no longer
be needed/used.

The current URI to retrieve secrets would still function as it does now, but
the decryption options could be removed in the next API version ('v2') by a
future blueprint.


Security impact
---------------

None, as the same policy rule ('secret:decrypt') for the new decrypt action
will be used as for the current header-based decryption mechanism.


Notifications & Audit Impact
----------------------------

None, though auditing might actually be easier due to the unique-URI per action
approach taken by this blueprint.


Other end user impact
---------------------

The Python Barbican client will also need to be updated to work with this
change.


Performance Impact
------------------

None


Other deployer impact
---------------------

None


Developer impact
----------------

The proposed change would clean up the Pecan controllers code a bit,
segregating secret metadata and decrypted payload concerns from each other.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
    jaosorior

Other contributors:
    john-wood-w


Work Items
----------

1) Add new 'payload' sub-resource controller off the 'secrets' resource, using the 'secret:decrypt' policy action

2) Add associated unit and functional tests per below

3) Update API documentation per below (currently located here: https://github.com/cloudkeep/barbican/wiki/Application-Programming-Interface)

4) Update the Python Barbican Client to utilize this new resource. If it is not available (due to older Barbican deployments), it should fail over to using the current decrypt mode


Dependencies
============

None


Testing
=======

Add unit and functional tests to test decrypted secret retrievals using the new
'payload' resource off of the 'secrets' resource. These new tests should
function properly in the Devstack gate job.


Documentation Impact
====================

API documentation will need to be updated to reflect these new secret
decryption API call, and to mark as 'deprecated' the current Accept header
based approach.


References
==========

None
