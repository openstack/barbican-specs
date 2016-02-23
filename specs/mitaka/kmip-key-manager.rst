..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============================
Add KMIP key manager wrapper
============================

https://blueprints.launchpad.net/castellan/+spec/kmip-key-manager

KMIP[1] devices are key managers and since Castellan is a key manager
interface it seems likely that someone would want Castellan to go directly to
a KMIP device instead of a Barbican with a KMIP device on the backend. This
blueprint offers a solution to directly talk to a KMIP device using Castellan.


Problem Description
===================

In order to currently talk to a KMIP device with Castellan, Castellan must
first talk to a Barbican that is hooked up with a KMIP device on the backend.
By creating a KMIP key manager interface, Castellan can interact with the KMIP
device directly.

This has the added benefit of allowing a client to decide whether or not
they require Barbican sitting in front of their KMIP device. Whether they have
their own device or purchase one from the cloud provider, they might not need
all of the Barbican functionality or resources and simply want a direct line
to the KMIP device.

This will allow for a private or dedicated cloud to use Castellan without
needing Barbican.

An important thing to keep in mind is that multi-tenancy is not guaranteed
with KMIP devices. Some devices offer the ability to create separate users
which does separate keys based on unique user accounts. However, not all
devices support this feature. Moreover, KMIP devices do not currently support
keystone as a means of authentication. The section below details the future
changes and possible support for both multi-tenancy and keystone
authentication.

Proposed Change
===============

A module will be created titled **kmip_key_manager**.
This module will live where the key manager interfaces live now and can be used
to directly interface with a KMIP device.

In order for the KMIP key manager to take in a context object, two new
credential types needs to be created. One is Certificate which will just
take in the location of a certificate and the other is KMIP which will be
an extension of Certificate. The KMIP credential object will take in all
the necessary values in order to establish a connection to the KMIP device
including a certificate.

The values used by the KMIP credential object will be the same variables used
by Barbican to establish a connection to a KMIP device and can be modified
via the castellan.conf file or through castellan itself. This will make it
possible for KMIP devices that support separate user accounts to provide
some multi-tenant capabilities.

When KMIP devices start supporting keystone tokens, the KMIP key manager class
can be updated to take in either the KMIP credential object or keystone token
credential object as a context. This will allow for full multi-tenant
capabilities that we see today in Barbican.

Here is an example of the new values that will be added to the `castellan.conf`
file::

    [kmip]

    #
    # From castellan.config
    #

    # Username for user authorized to access KMIP device  (string value)
    #username = admin

    # Password for KMIP authenticated user (string value)
    #password = password

    # Use this endpoint to connect to the KMIP device (string value)
    #host = localhost

    # Port that KMIP device is listening on (string value)
    #port = 5696

    # Keyfile used by castellan (string value)
    #keyfile = '/path/to/certs/cert.key'

    # Certificate used by castellan to talk to KMIP device (string value)
    #certfile = '/path/to/certs/cert.crt'

    # List of certificates and CA's that castellan willaccept (string value)
    #ca_certs = '/path/to/certs/LocalCA.crt'

These values can reside in the `castellan.conf` file or can be added to an
already existing configuration file. Castellan just needs to be given the
location of the file to read.

The rest of the code will be very similar to the KMIP secret store code
within Barbican.[2]

Alternatives
------------
None


Data model impact
-----------------

None

REST API impact
---------------

None

Security impact
---------------

KMIP devices do not guarantee multi-tenant functionality.
This means users should be aware that when Castellan is the interface to the
KMIP device, any user that has access to the KMIP credentials might be able
to talk to the device. There is no differentiation based to the user's role
or project because keystone cannot currently be used as a form of
authentication. This will need to be heavily documented.

Notifications & Audit Impact
----------------------------

Any CRUD request to the device will need to be logged.
Establishing and ending the connection should also be logged to make sure
the connection is successful and successfully terminated.

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

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  silos (Christopher Solis)


Work Items
----------

1. Create a Certificate and KMIP credential object
2. Create a KMIP key manager
3. Add a KMIP section to the sample Castellan configuration file
4. Unit and functional test the key manager wrapper and credential objects
5. Documentation

Dependencies
============

None

Testing
=======

New unit tests and functional tests need to be added.

Documentation Impact
====================

Castellan documentation should be updated to reflect the addition of the new
backend.

References
==========
[1] https://www.oasis-open.org/committees/tc_home.php?wg_abbrev=kmip

[2] https://github.com/openstack/barbican/blob/master/barbican/plugin/kmip_secret_store.py
