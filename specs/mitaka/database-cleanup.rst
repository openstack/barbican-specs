..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================================================
Add cron job garbage collector for barbican database
====================================================

Blueprint:

https://blueprints.launchpad.net/barbican/+spec/clean-db-soft-deletes

Problem Description
===================

For all tables in the barbican database that use soft deletion,
the entries in the table are not removed but flagged for deletion. We
want to create a configurable command that will be called by a cron job to
go through the database and delete entries where
the deleted flag equals 1 or if the entry is not needed anymore (like zombie
projects).


Proposed Change
===============

Create a barbican command called 'barbican-manage db cleanup' that will be run
by a cron job to clean up soft deletions in the barbican database. The command
will utilize python sqlalchemy libary calls to go through the database and
remove entries where deleted equals 1. The only tables that will be checked are
tables which have a corresponding class in 'barbican/model/models.py' and the
class has 'SoftDeleteMixIn' as a parent class. Any other table will not be
checked due to the assumption that soft deletes are not used. The
script will dynamically load in the table classes defined in models.py to a
list. Once the list is created, the script will then check the configuration
in barbican.conf to add or remove specific models.

An example cron job file will be provided so that the administrator can modify
the contents to call the python script in intervals to their liking. It is
intended that the barbican deployer calls the barbican-mange db cleanup command
in a cron job.

The command is configurable for these options:
minimum number of days to keep since soft deletion (default is 90 days)
delete zombie projects (no associated resources and default is true)
models to force check (no inheritence of SoftDeleteMixIn and default is None)
models to ignore (have inheritence of SoftDeleteMixIn and default is None)
log levels for log (default is INFO).
location of cleanup log (default is None).


.. note ::
  note on zombie projects:
  Barbican will create a project entry anytime a user
  tries to make a request. If there are no resources
  associated with a project, then that project
  database entry can be removed.


.. note ::
  note on audit logging:
  DELETEs via API are already logged by the CADF middleware. So there
  should be no reason to add functionality to make an audit log
  of who performed deletions.


.. note ::
  For this iteration, orders will not be touched due to different
  available SLAs that barbican offerings could have and should be
  discussed further. Certificate orders should not be
  deleted because certificate renewals require the old orders.


Configuration -
Option parameters can be passed with the barbican manage command
to customize cleanup.
For example:
'barbican db cleanup --min_num_days_to_keep_softdeletes 30'

The command can also pass in a configuration file for a command.
'barbican db cleanup -f "/etc/barbican/barbican.conf"

The option parameters will have higher precedence than the variables
in the conf file if both the conf file and other options are given
as parameters. So the barbican admin can give only option parameters,
only a conf file, or both. The default value for the configuration file
will be 'None'.

Below is an example of how a configuration file can be configured:

[db_cleanup]
#how long to keep the entry in the database after a soft deletion
minimum_num_days_to_keep_soft_deletions = 90

#remove projects that have no associated resources
cleanup_unassociated_projects = True

#table classes to force check  e.g., CertificateAuthority,SecretACL,..
table_classes_to_force_check = None

# If a secret is expired, then it should be soft deleted
soft_delete_expired_secrets = True

#table classes to ignore, their parent class is SoftDeleteMixIn
#e.g, Project,Secret...
table_classes_to_ignore = None

# Show more verbose log output (sets INFO log level output)
verbose = True

# Show debugging output in logs (sets DEBUG log level output)
#debug = True

#log file to store information on deletions
db_cleanup_log_file = "/var/log/barbican/barbican-cleanup.log"


Alternatives
------------

1. Add a REST API parameter for delete operations where users can specify hard
   deletion.
2. Leave it to the database admins to clean up their database manually.
3. Leave it to the database admin to create database triggers for cleanup.
4. Create a daemon instead of a cron job which is similar to the
   glance scrubber.


Data model impact
-----------------

The data models should not change.

The tables that are checked are tables which have a class that has
SoftDeleteMixIn as a parent class. Other tables will be ignored unless
configured. Entries where 'deleted == 1' will be removed.


REST API impact
---------------

None


Security impact
---------------

The barbican manage command and cron job file should have valid
barbican admin user permissions. No global user should
be able to modify/run the script.


Notifications & Audit Impact
----------------------------

A log should be kept with total entries deleted for each table. The
log location will be configurable.

If the level of logging is debug, then the table name,
id of entry, soft deletion date, and hard deletion date will be recorded.

Since user deletions are already logged by CADF middleware,
it will not be necessary to log who made the soft deletions.

Python and Command Line Client Impact
-------------------------------------

None


Other end user impact
---------------------

None


Performance Impact
------------------

Since this is a recurring find and delete operation on a database,
this may take up a great ammount of compute cycles. It will be up to
the admin/operator of Barbican to find the best time to run,
what interval to run the job, and configure how much to delete.
The admin will have to customize the cron job entry to their liking.


Other deployer impact
---------------------

The deployer will have to configure the cron job file.


Developer impact
----------------

None


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  edtubill

Other contributors:
  None


Work Items
----------

Each phase listed below is intended to be a CR. (Phase 2 will be split
into different CRs)

Phase 1: Create simple barbican-manage command to go through
the database and delete everything with 'deleted == 1'. Create unit and
functional tests for this first phase.
Phase 2: Add logic to read configuration file and go through database based on
the configuration. Create additional unit and functional tests.
Phase 4: Create example cron job configuration files
Phase 5: Document the cron job garbage collector on Barbican Wiki


Dependencies
============

None


Testing
=======

Unit tests must be written for internal component testing.
Functional tests must be created based on different possible
configurations. The functional tests will check the entries of
the database directly.


Documentation Impact
====================

Wiki documentation will be created for usage of the 'barbican-manage db cleanup'
command, how to configure, and examples on how to setup a cron job
will be provided.

If a user does barbican-manage db cleanup --help, then usage documentation
should be shown.


References
==========

https://blueprints.launchpad.net/barbican/+spec/clean-db-soft-deletes
