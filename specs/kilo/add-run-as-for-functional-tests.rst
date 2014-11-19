..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================================
Add ability for functional tests to run as different users
==========================================================

https://blueprints.launchpad.net/barbican/+spec/add-run-as-for-functional-tests

Currently all Barbican functional tests run under the same userid.
This blueprint adds the ability for a Barbican functional test to run
under a different userid from that default.  This is needed to
support the ability to execute tests in parallel and ensure that the
underlying system state doesn't change as a result of other concurrently
executing tests.


Problem Description
===================

Today all Barbican functional tests run under the same user.
Since these tests run in parallel then any assumptions about the
underlying system state are invalid - for example, a test for paging
secrets could fail if another tests does secret operations in parallel.


Proposed Change
===============

The proposed change not only solves this problem, but also provides a
generalized way to run tests in an isolated manner, under their own
userid.

The first step involves determining which test(s) need to run as a non-default
user.  This is done at testcase creation time, by the testcase author.

Next the testcase author will create their test within class that extends
a proposed new class
(functionaltests.api.base.DynamicUserTestCase) which extends the existing
functionaltests.api.base.TestCase to indicate that the test(s) should not
be run under the default user, but rather under a new user.

The user name, password, and tenant name for this new user are derived from
settings in barbican's etc/dev_tempest.conf file.  Under a new
[key-manager] section are three new settings::

    dynamic_user_userid_prefix=testuser_
    dynamic_user_password=topsecret
    dynamic_user_tenant_prefix=testtenant_

As part of each invocation of the test, the oslotest fixture for the
DynamicUserTestCase setUp() code will:

1. generate a new, unique user
   - new userid = dynamic_user_userid_prexfix concatenated with a UUID
   - new password = value of dynamic_user_password
   - new tenant name = value of dynamic_user_tenant_name_prefix
   concatenated with the same UUID that was used for the new userid.

2. login that user
3. run the test
4. logout and delete the user we created above

Next the test itself will be run as usual.

When the test completes, the oslotest fixture tearDown() code will
logout and delete the new user.

The components of barbican that will need to be updated to support
this new function are:

1. new class added to functionaltests.api.base.py called
   DynamicUserTestCase that will create and login a new user
   for use with that testcase.

2. DynamicUserTestCase.setup() will create and login the new user

3. DynamicUserTestCase.tearDown() will delete that user

4. Support functions in functionaltests.api.base.TestCase to talk to
   the auth provider for userid creation, login and deletion.

Alternatives
------------

One alternative that was discussed was to create a new decorator
called @run_as for which could be used for any test, not requiring
a new subclass of Testcase.

We rejected that approach for several reasons:

1. Tests requiring this decorator generally have some other focus
   of their testing.  For example, pagination tests for secrets
   have their focus on pagination, not on secrets.  It makes sense
   that these tests aren't grouped in the same class as secrets, but
   rather in their own subclass.

2. Adding decorators where they aren't necessarily the best mechanism
   can lead to confusing and hard to debug code.  Care should be taken
   to ensure that a decorator is really the best way to implement.

Data model impact
-----------------

No data model impact.

REST API impact
---------------

No REST API impact.

Security impact
---------------

This change allows dynamic user creation and deletion on a per-testcase
basis.  It also allows a testcase to specify an existing user to run
under.  There are no changes to any API as all of these updates are
focused under the functionaltests and do not affect the Barbican API.

Notifications & Audit Impact
----------------------------

No notification/audit impacts for this change.

Other end user impact
---------------------

The test logging will show the user id that each test runs under in the
test logs.  They are not part of the API, but could be useful for problem
determination and debugging.

Performance Impact
------------------

The only performance impact of this CR is on the functionaltests path.
It has no effect on the product's performance.

Tests that request a different user will incur additional pathlength to:

* create and login the user (before the test runs)
* delete the user (after the test completes)


Other deployer impact
---------------------

Need to ensure that the fields for dynamic user creation in
etc/dev_tempest.conf are acceptable for the tester.


Developer impact
----------------

Developers adding functional tests can choose to have those tests run
under a different userid, if they desire.  Otherwise, all functional
tests will run under the default user.

Unit tests are NOT impacted by this change.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  sheyman (hockeynut)

Work Items
----------

Upon approval of this blueprint we will code and post for review the
implementation of the new class as well as the supporting code
in the utilities and fixtures.


Dependencies
============

This function depends on keystone for the user and tenant creation and deletion
API.


Testing
=======

This code is part of the functional testing of Barbican and will be exercised
in the devstack gate as well as locally on a developer/tester machine by
running "tox -e functional"


Documentation Impact
====================

Currently there is no documentation on how to write Barbican tests (it is an
open story in the Barbican work backlog).  The prose in this blueprint
can be used as a starting point to describe the usage of the
DynamicUserTestCase.

An important item to cover in the documentation is guidance as to when a test
needs to run as a DynamicUserTestCase.  We should use an example such as:

   * test A that creates a large number of secrets, then fetches the list of all
     secrets and validates the size of that list against the number created.
   
   * test B that simply creates a new secret.

If test B runs in the middle of the execution of test A then the secret count
expected by test A will be off by one, and test A will fail.

However, if test A runs as a different user from test B then there will
be no interference between the two and both tests will pass as expected.


References
==========

* Barbican wiki: https://wiki.openstack.org/wiki/Barbican
* Barbican source code: https://github.com/openstack/barbican
* Barbican functional tests: https://github.com/openstack/barbican/tree/master/functionaltests