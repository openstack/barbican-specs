..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Date Filters
==========================================

https://blueprints.launchpad.net/barbican/+spec/date-filters

Problem Description
===================

In an effort to keep track of the changes being made to secrets or upcoming
expiration dates, users such as IT Auditors or Operations Engineers need to be
able to filter queries sent to the API based on the timestamped properties of a
secret.  These users require new filters to be able to query information about
their secrets such as:

#. Secrets expiring in the near future
#. Secrets created in a particular time range
#. Secrets updated in a particular time range

Currently, Barbican does not provide any filters for querying secrets by
using the properties that hold timestamp values. So a user currently needs
to iterate through their entire secret collection to be able to gather this
data.

Proposed Change
===============

The barbican /secrets resource should be enhanced to allow sorting and
filtering based on a secret's ``created``, ``updated``, and ``expiration``
properties as described in the API Working Group's guideline for `Pagination,
Filtering, and Sorting`_

The filters should allow limiting of returned values to specific times and
time ranges by using the equality (``=``), greater-than (``gt``),
greater-than-or-equal (``gte``), less-than (``lt``), and
less-than-or-equal (``lte``) operators.

The filters should also allow the ordering of return values by using the
``sort`` query string parameter with both ascending (``asc``) and descending
(``desc``) directions for the specified sort key.

Values passed in to these query parameters are assumed to be give in UTC time
using the extended format described in ISO 8601.  The UTC zone designation
represented by appending the "Z" character will be required.  Values that do
not include the zone designation will result in an error response with status
code 400. Values that specify a time offset from UTC will also result in a 400
error response even if the offset is zero to specify UTC.

.. _Pagination, Filtering, and Sorting: https://git.openstack.org/cgit/openstack/api-wg/tree/guidelines/pagination_filter_sort.rst

Alternatives
------------

Requiring the zone designation for UTC ("Z") may be too stringent of a
requirement.  One alternative would be to accept time values without the "Z"
zone designation and just assume that the values are all UTC.

Data model impact
-----------------

This change will not affect the data model, since all values to be used in the
filtering are already part of the data model.

REST API impact
---------------

Additional query parameters will be available for the GET /v1/secrets resource
as described above.

Examples:

List secrets expiring in the next week (assuming current time is June 8, 2016
20:00 UTC) and sort by secrets expiring soonest:

.. code-block:: json

   GET /v1/secrets?expiration=gt:2016-06-08T20:00:00Z,lt:2016-06-15T20:00:00Z&sort=expiration:asc

List secrets created in the previous week assuming same current time as above:

.. code-block:: json

   GET /v1/secrets?created=gt:2016-06-01T20:00:00Z,lt:2016-06-08T20:00:00Z

List secrets updated in the previous week assuming same current time as above:

.. code-block:: json

   GET /v1/secrets?updated=gt:2016-06-01T20:00:00Z,lt:2016-06-08T20:00:00Z

The format of the data in the request and response will not change.

Security impact
---------------

This change should not impact the security of barbican, since it just provides
a way to narrow the results of querying the API.

Notifications & Audit Impact
----------------------------

This change does not impact the notifications or auditing features of barbican.

Python and Command Line Client Impact
-------------------------------------

Both python-barbicanclient and the plugin for the unified CLI will need to be
updated to provide a way for clients to use these new filters.

Other end user impact
---------------------

No additional end user impact should result from this change.

Performance Impact
------------------

The implementation should not use additional database queries, but rather use
the existing queries so that performance is not negatively impacted.

Other deployer impact
---------------------

This change should not affect deployers.

Developer impact
----------------

Developers should not be impacted since these filters are optional.  However,
developers could make use of these filters when applicable to their use
cases.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Douglas Mendizábal

Other contributors:
  Joe Savak

Work Items
----------

#. Update controllers to accept and make use of the new filters
#. Update documentation to include the newly added filters
#. Update python-barbicanclient to make use of the new filters
#. Update the unified CLI plugin to make use of the new filters

Dependencies
============

None.

Testing
=======

New functional and unit tests that exercise the new functionality should be
included in the implementation of this spec.

Documentation Impact
====================

The API change will need to be updated in the API reference as well as the
user guide.

References
==========

API Working Group's Guideline for Pagination, Filtering and Sorting:

https://git.openstack.org/cgit/openstack/api-wg/tree/guidelines/pagination_filter_sort.rst

ISO 8601 in Wikipedia:

https://en.wikipedia.org/wiki/ISO_8601
