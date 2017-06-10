..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===============
Rolling Upgrade
===============

Barbican users would like to upgrade their existing environments with minimal downtime.
This spec outlines the changes required to be able to assert the various upgrade tags.
The goal for Pike is to assert ``assert:supports-rolling-upgrade`` tag.

Problem description:
====================

There are currently six upgrade tags:
  * assert:follows-standard-deprecation
  * assert:supports-upgrade
  * assert:supports-accessible-upgrade
  * assert:supports-rolling-upgrade
  * assert:supports-zero-downtime-upgrade
  * assert:supports-zero-impact-upgrade

Status tags in Barbican project:

  * assert:follows-standard-deprecation: Done [1]
  * assert:supports-upgrade: Done [2]
  * assert:supports-accessible-upgrade: Not yet [3]
  * assert:supports-rolling-upgrade: Not yet [4]
  * assert:supports-zero-downtime-upgrade: Not yet [5]
  * assert:supports-zero-impact-upgrade: Not yet [6]

Proposed change:
================

To support rolling upgrade for Barbican, we need to compare feature lists
support for upgrade and rolling upgrade as follows:

1. Maintenance Mode
2. Live Migration
3. Upgrade Orchestration
4. Multi-version Interoperability
5. Online Schema Migration
6. Graceful Shutdown
7. Upgrade Orchestration - Remove
8. Upgrade Orchestration - Tooling
9. Upgrade Gating
10. Project Tagging

For Barbican, upgrading from release N-1 to release N then to N+1,
we are following supporting status and performing detail items as follows:
https://github.com/openstack/development-proposals/blob/master/development-proposals/proposed/rolling-upgrades.rst

1. Maintenance Mode: No need to implemented

  * No need to implement it because of plugin mechanisms and no
    scheduler/placement services

2. Live Migration: No need to implemented

  * Because Barbican does not have data plane. All information will be stored
    in its database and secret store plugins.

3. Upgrade Orchestration: Not yet implemented

4. Multi-version interoperability

  * API microversion: Not yet - Barbican does not need microversions for
    rolling upgrade.

  * RPC versioning: Not yet implemented.

  * RPC object version (Oslo.VersionedObjects): Not yet implemented.

5. Online Schema Migration: Not yet implemented

6. Graceful Shutdown: Implemented

  * Barbican-api is using Pecan to expose API and Pecan has supported graceful
    shutdown so it means barbican-api has supported this feature as well

  * Other services in Barbican are using oslo.service to launch the services
    and oslo.service has implemented graceful shutdown so they have supported
    this feature also.

7. Upgrade Orchestration: Not yet implemented

8. Upgrade Orchestration: Tooling implemented

9. Upgrade Gating: Implemented

10. Project Tagging: Not yet implemented

  * Assert the tag and notify the OpenStack Technical Committee

Data model impact
-----------------

* Impacted because the O.VO codebase will change the way Barbican services
  communicate to database as well as the data flow between Barbican internal
  services.

REST API impact
---------------
None

Security impact
---------------
None

Notifications impact
--------------------
None

Other end user impact
---------------------

End users will have minimal downtime when upgrading barbican
services.

Performance Impact
------------------
None

Other deployer impact
---------------------

It is anticipated that a rolling upgrade will require the operators
intervention.

Developer impact
----------------

Developers will need to be aware of Barbican features that enable rolling
upgrades and make sure they are not removed (Developers will also need to work
within the constraints of the database strategy for rolling upgrades, but that
developer impact is covered by another spec).

Implementation
--------------

Assignee(s)
-----------

Primary assignee:

  namnh

Other contributors:

  daidv

  hieulq

Work Items:
-----------

* Apply O.VO in Barbican before rolling upgrade:

    * Add oslo.versionedobjects to requirements docs.

    * Implement barbican O.VO base objects.

    * Migrate current database object to OVO objects.

    * Implement extra fields in objects/fields.py.

    * Implement RPC object registry and register all objects.

    * Implement and attach the object serializer.

    * Implement the indirection API(VersionedObjectIndirectionAPI)
      in the objects/base.py

    * Implement Unit Test for Barbican O.VO module.

* Perform rolling upgrade for O.VO.

    * Providing serialized versions.

    * Implement a new method to check version.

    * Implement RPC pinning version.

    * Implement methods to communicating with DB, such as query, save and create.

* Perform rolling upgrade for OSM along with O.VO which completed before.

    * Implementing DB upgrade mechanism for Barbican to perform
      ``barbican-db-manage`` command via CLI including three branches:

      * '--expand' branch: To add columns, tables or triggers in barbican
        database.

      * '--contract' branch: to delete columns, tables or trigger after all
        services in branch are upgraded to new release.

    * Perform data gradually migrate to new schema.
      by multi-version interoperability (included O.VO and RPC Pinning).

    * Ensure all data have migrated to new schema via online_data_migrations
      commandline.

* Write documentation for rolling upgrade (operator docs).

* Assert the tag and notify the OpenStack Technical Committee.

References
----------

[1] https://governance.openstack.org/tc/reference/tags/assert_follows-standard-deprecation.html#tag-assert-follows-standard-deprecation

[2] https://governance.openstack.org/tc/reference/tags/assert_supports-upgrade.html#application-to-current-projects

[3] https://governance.openstack.org/tc/reference/tags/assert_supports-accessible-upgrade.html

[4] https://governance.openstack.org/tc/reference/tags/assert_supports-rolling-upgrade.html

[5] https://governance.openstack.org/tc/reference/tags/assert_supports-zero-downtime-upgrade.html

[6] https://governance.openstack.org/tc/reference/tags/assert_supports-zero-impact-upgrade.html
