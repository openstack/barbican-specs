..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===========================
Add barbican-manage command
===========================

Blueprint:
https://blueprints.launchpad.net/barbican/+spec/add-barbican-manage-cmd

Client Blueprint:
None


A new 'barbican-manage' command is introduced as Barbican admin tool. This
command interacts with Barbican service for management operations which usually
cannot be accomplished with REST APIs. This can improve usability and
extensibility in the future.

Other OpenStack services like Keystone [#]_ and Nova [#]_ also provide similar
commands for service admins.


Problem Description
===================

Currently, Barbican uses individual admin commands for management functions.
For example, using barbican-db-manage for database migration, using
pkcs11-key-generation and pkcs11-kek-rewarp for HSM/pkcs11 related management,
etc. More new admin functions will be added in future releases. It's time to
consolidate all these individual commands under a single tool for sake of
simplicity.


Proposed Change
===============

The syntax of new 'barbican-manage' command will be:

  **barbican-manage** [options] *category* *action* [additional args]

*category* and *action* will be a list of subcommands that will be supported.
The initial implementation of **barbican-manage** will be just refactoring
current command code, and unify all functions into one command.

.. Note::
    Existing admin commands will be continue working and will be deprecated
    in future according to OpenStack standard deprecation policy [#]_

Currently we have 2 categories: *db* for database management and *hsm* for
HSM/PKCS11 management.

  Category *db* replaces existing **barbican-db-manage** command:

    ============       ====================================================
    db cleanup         Remove all soft-deleted and expired secrets from DB
    db restore         Restore a soft-deleted secret from DB
    db revision        Create a new DB version file
    db upgrade         Upgrade to a future version DB version
    db history         Show changeset history
    db current         Show current revision for a database
    ============       ====================================================

  Category *db* can take additional argument:

    --dburl       URL to the database
    --from-file   Secret garbage collection configuration file

  Category *hsm* replaces existing **pkcs11-key-generation** and
    **pkcs11-kek-rewrap** commands:

    ==================    =================================================
    hsm gen-mkek          Generate HSM master key encryption key
    hsm gen-mhmk          Generate HSM master HMAC key
    hsm rewrap-pkek       Rewrap Project KEKs after rotating to a new MKEK
    ==================    =================================================

  Category *hsm* can take following additional arguments:

    --library-path    PKCS11 library path
    --slot-id         Slot ID
    --passphrase      PKCS11 login password
    --label           Key label
    --length          Key length
    --dry-run         Displays changes that will be made (Non-destructive)

.. NOTE:: --dry-run requires above 5 arguments be specified

General 'options' includes:

    --help            show help message
    --version         show command version

The command will read standard *barbican.conf* to get setting for *debug*,
*verbose* and *log_file* options.


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

User has to have appropriate privilege to run command barbican-manage.

Notifications & Audit Impact
----------------------------

Event log can be generated for audit support.

Python and Command Line Client Impact
-------------------------------------

No impact to Barbican client and OpenStack client.

A new CLI admin command will be added, so its user guide need to be added.

Other end user impact
---------------------

None

Performance Impact
------------------

None

Other deployer impact
---------------------

None if existing admin commands are not used in deploying script.

There is no immediate impact to deployers even if they use existing admin
commands in script for deployment. The script need to be converted to use new
*barbican-manage* command eventually before old commands are removed according
to procedures in OpenStack standard deprecation policy.

Developer impact
----------------

None


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  <jianhua>

Other contributors:
  <None>

Work Items
----------

Work items or tasks
 - Create a new barbican-manage.py in barbican/cmd and call functions into
   scripts db_manage.py and pkcs11_*.py
 - Add unit testcases
 - Add barbican-manage command script in setup.cfg
 - Add user guide document for barbican-manage command
 - deprecate existing command scripts. Adding deprecation warning message in
   existing commands.

Dependencies
============

None


Testing
=======

Unit tests will be added for all subcommands and various options.


Documentation Impact
====================

A new barbican-manage command user guide will be added, which should include
new user guide for database migration subcommands and user guide of
pkcs11-related subcommands modified from existing
http://docs.openstack.org/developer/barbican/api/userguide/pkcs11keygeneration.html


References
==========

.. [#] http://docs.openstack.org/developer/keystone/man/keystone-manage.html
.. [#] http://docs.openstack.org/developer/nova/man/nova-manage.html
.. [#] https://governance.openstack.org/reference/tags/assert_follows-standard-deprecation.html
