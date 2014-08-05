..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=========================
Restructure PKCS11 Plugin
=========================

https://blueprints.launchpad.net/barbican/+spec/restructure-pkcs11-plugin

The current PKCS11 plugin assumes that it has sufficient storage to create
a KEK per project/tenant. This assumption is untrue for many HSMs so we
propose to introduce a master KEK stored in the HSM that wraps per project
KEKs to resolve the problem.

Problem Description
===================

The PKCS11 plugin creates a key encrypting key (KEK) per project/tenant.
When connecting to an HSM with limited storage, this system can exhaust
available space on the HSM and fail to create KEKs for new project/tenants.

Proposed Change
===============

A master KEK wrapping a project KEK will resolve the storage problem by
keeping a minimum of keys in the actual HSM while still using a different
encryption key per project/tenant. The hierarchy would look like this:

::

    +--------------+
    |  Master KEK  |
    +-----+--------+
          |
          |
          |
    +-----+-----+       +-----------+
    |Wrapped KEK+-------+Encrypted  |
    +-----------+       |Secret     |
                        +-----------

The sequence diagram of the decryption process will look like this:

::
    barbican                         DB                        HSM
    +        Get master KEK handle   +                        +
    | <-----------------------------------------------------> |
    |  Get wrapped project KEK       |                        |
    | <----------------------------> |                        |
    |       Unwrap project KEK (stays|in HSM)                 |
    +-------------------------------------------------------> |
    |          project KEK handle    |                        |
    | <-------------------------------------------------------+
    |    Get encrypted secret        |                        |
    | <----------------------------> |                        |
    |          Decrypt secret using project KEK handle        |
    | <-----------------------------------------------------> |


Alternatives
------------

None.

Data model impact
-----------------

The data model may need to be expanded to support storing a per project KEK.
We propose that this be accomplished by creating a new table that stores the
wrapped KEK bytes with a foreign key to project/tenant.

REST API impact
---------------

None

Security impact
---------------

The risks of the PKCS11 plugin remain largely the same. While the KEKs are
no longer stored exclusively in the HSM, the KEK is always encrypted when
outside the HSM.

Notifications & Audit Impact
----------------------------

None

Other end user impact
---------------------

None

Performance Impact
------------------

There is an additional decryption round trip to the HSM. Previously
decrypting a secret required finding the project KEK and then sending
the encrypted bytes to the HSM for decryption. With the new system
we must find the master KEK, decrypt the wrapped project KEK, then
decrypt the user secret using that unwrapped KEK.

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

Paul Kehrer (reaperhulk)

Work Items
----------

1. Add new model to barbican

2. Update PKCS11 plugin to do wrap/unwrap and consume the new model.

Dependencies
============

None

Testing
=======

The unit tests for the PKCS11 plugin will change to reflect the new calls
made to the HSM and the data stored in the models.

Documentation Impact
====================

Update the documentation to describe the manner of operation of the plugin.

References
==========

* https://blueprints.launchpad.net/barbican/+spec/restructure-pkcs11-plugin
