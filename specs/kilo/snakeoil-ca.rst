..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================================
Snakeoil CA Certificate Manager Plugin
======================================

https://blueprints.launchpad.net/barbican/+spec/barbican-snakeoil-ca

A certificate management plugin which auto approves all requests can be very
useful for testing and development deployments. This is especially useful for
tools (such as TripleO) which hope to use Barbican as an abstraction layer for
certificate creation in production deployments but still require a
configuration that will work in CI.


Problem Description
===================

In order to use the certificate management interface in a development or
testing environment a user must (currently) configure dogtag or another backend
in a way which auto-approves certificate requests. This is an unnecessary
burden in many environments (such as CI pipelines) which require none of the
advanced features these tools provide.

Proposed Change
===============

Create a plugin which implements the certificate manager interface and will
sign any certificate request sent to it. Allow for configuration of an
on-disk CA certificate and key. If no certificate or key is found at that path,
a new CA certificate and key will be created and stored at that path encoded
in PEM file format. If no certificate or key path is specified then a new CA
key and certificate will be created and stored only in memory.

Serial numbers for certificates will be generated from UUIDs.

Alternatives
------------

By limiting the scope of this plugin to non production uses, and only
certificate signing (no revokation, for example) there ends up being a small
amount of code using the PyOpenSSL library (an ffi for openssl). The amount
of code here is comprable to the amount required to shell out to a tool like
CA.pl and is considerable easier to read, understand, and test. If the scope
of this tool was ever expanded then we may want to reconsider using PyOpenSSL.

Data model impact
-----------------

None

REST API impact
---------------

A certificate order::

    {
        "type": "certificate",
        "meta": {
            "request_data": "PEM encoded X509 Request with optional X509Name",
        }
    }

Security impact
---------------

This feature is explicitly not intended for use in any type of production
environment. This will be explicitly documented as such and is also named
'Snakeoil' to (hopefully) make this as apparent as possible.

Notifications & Audit Impact
----------------------------

None

Other end user impact
---------------------

None

Performance Impact
------------------

None

Other deployer impact
---------------------

None. This feature should never be deployed to production.

Developer impact
----------------

This should ease the setup of development environments for certificate
management. We may want to consider documenting setting up this type of
environment for new users.

Implementation
==============

This is implemented using PyOpenSSL and the motivation for this is explained
in the 'Alternatives' section.

https://review.openstack.org/#/c/140575

Assignee(s)
-----------

Primary assignee:
  greghaynes <greg@greghaynes.net>

Work Items
----------

Implement the plugin. This plugin will implement the general certificate
request API so barbican-client work will be completed in that fashion.

Dependencies
============

None

Testing
=======



Documentation Impact
====================

Documenting this feature for setting up development and testing environments
could be useful but is not requried.

References
==========

