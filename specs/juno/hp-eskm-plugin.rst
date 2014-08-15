..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================================================
Creation of a Babrican Plugin to use HP Atalla ESKM.
====================================================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/barbican/+spec/hp-eskm-plugin

This effort will enhance Barbican by adding a crypto plugin to allow use of
Atalla ESKM from HP. Atalla ESKM is a HSM appliance for generating and managing
cryptographic keys. By adding an optional plugin to Barbican, users have the
option of integrating OpenStack with existing or new Atalla ESKM installations.


Problem Description
===================

Deployers of an OpenStack cloud may have, or wish to have an Atalla ESKM
Appliance for key management. In this case, it is desirable to utilise the
appliance alongside the new OpenStack installation for enhanced security.


Proposed Change
===============

The proposed changes to Barbican are, creation of an optional plugin to talk
to an ESKM appliance. This plugin will be based of off the CryptoPlugin base
and encapsulate all necessary functionality. The plugin will communicate over
a secure network link to a remote appliance using a vendor specific protocol.

No changes to the core of Barbican are required.

Alternatives
------------

Alternatively, Atalla ESKM supports the KMIP industry standard protocol for key
exchange. KMIP is something planned for Barbican in the future but at the time
of writing has not yet been implemented.

Data model impact
-----------------

None

REST API impact
---------------

None

Security impact
---------------

This change touches sensitive data (tokens, keys, and user data) since it will
pass key data through to an ESKM server and receive the results prior to
encryption. User data is protected using a key encryption key and 256 bit AES
CBC encryption. The key encryption key is never stored and is fetched on demand
from the ESKM appliance.

This change utilises encryption algorithms from the 'cryptography' python
module but does not introduce any additional cryptographic dependencies into
Barbican.

This change requires a network connection to be established between Barbican
and an ESKM appliance. This connection is protected using TLSv1 and client
identification certificates. Key material is transmitted over this connection.


Notifications & Audit Impact
----------------------------

None

Other end user impact
---------------------

None

Performance Impact
------------------

A performance impact is introduced by this change, communication over a network
link to a remote ESKM appliance may take time. This is multiplied by the need
to request a key encryption key from ESKM as part of all requests. This
performance impact is accepted in return for the improved security gained by
not caching the key encryption key locally.

Other deployer impact
---------------------

The following new config options are added, they are specific to the plugin and
need not be used otherwise. All options live within the 'eskm_crypto_plugin'
group.

+----------------------------------+-----------------------------------+
| Option                           | Meaning                           |
+==================================+===================================+
| eskm_crypto_plugin.crt_file_path | ESKM client certificate file path.|
+----------------------------------+-----------------------------------+
| eskm_crypto_plugin.key_file_path | ESKM client key file.             |
+----------------------------------+-----------------------------------+
| eskm_crypto_plugin.key_password  | ESKM client key password.         |
+----------------------------------+-----------------------------------+
| eskm_crypto_plugin.user_name     | ESKM user name.                   |
+----------------------------------+-----------------------------------+
| eskm_crypto_plugin.user_pass     | ESKM user password.               |
+----------------------------------+-----------------------------------+
| eskm_crypto_plugin.eskm_host     | ESKM host IPs, comma separated.   |
+----------------------------------+-----------------------------------+
| eskm_crypto_plugin.eskm_port     | ESKM port number.                 |
+----------------------------------+-----------------------------------+

Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  tim-kelsey

Other contributors:
  None

Work Items
----------

- Create spec (this spec).
- Write the plugin and tests.
- Confirm all Barbican tests, new and old, still pass.
- Review the code.

Dependencies
============

No compulsory dependencies are introduced, but an Atalla ESKM appliance must
be available if a deployer wishes to use this plugin.

Testing
=======

A suite of unit tests will be produced to test the new code.

Documentation Impact
====================

Documentation may need updating to reflect the existence of the plugin and its
configuration options.


References
==========

None