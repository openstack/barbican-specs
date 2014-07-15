
====================================================
Introduce Oslo's cliff as cli framework for Barbican
====================================================

Introduce cliff as a cli framework for python-barbicanclient, and, as a result
promote usage of common OpenStack libraries.

Problem Description
===================

Currently the Barbican cli utilities (such as the migration script and even
the python-barbicanclient itself) rely on the argparse library, yet, their
current implementations do not provide sufficient information for their usage
in production. Usage of "standard" libraries should be encouraged in order to
make Barbican's integration easier. On the other hand, subcommand help is
lacking, and input is not properly validated.

Proposed Change
===============

The proposal is to take into use cliff, which is a framework for building
command line programs introduced by Oslo. This will enable Barbican's scripts
to have more standard outputs (akin to other OpenStack projects) and while
moving the code to the new framework, better documentation will be added to the
commands as part of the work.

Alternatives
------------

The argparse library could still be used, yet, more input validation and
documentation needs to be added.

Data model impact
-----------------

None.

REST API impact
---------------

None.

Security impact
---------------

None for the Barbican server. Barbican client input though should be properly
validated as part of the work.

Notifications impact
--------------------

None.

Other end user impact
---------------------

This will hopefully have a positive impact in the client's usability, as the
main focus of this work is to make it better.

Performance Impact
------------------

None.

Other deployer impact
---------------------

This change takes immediate effect after it's merged, and adds a dependency
(which is cliff) to the python-barbicanclient.

Developer impact
----------------

If other developers are working with the client, it might affect their work,
since the changes are potentially big.

Work Items
----------

* Replicate the current functionality of python-barbicanclient (with help
  messages and subcommands) to the new framework.
* Complete missing sub-command help over the new framework.

Testing
=======

Unit tests are assumed.

Documentation Impact
====================

None.
