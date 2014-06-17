..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Enforce content type on barbican REST API
==========================================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/barbican/+spec/barbican-enforce-content-type

Before barbican moved from falcon to pecan, content types were coerced to
application/json.  This meant that a user could use a tool such as curl,
omit the content type, and the call would succeed.  After moving to pecan,
the default content-type changed to application/x-www-form-urlencoded.
This meant that a curl request without a content type specified would
end up coming into barbican as a urlencoded string and would fail during
JSONifying.

Problem Description
===================

As a result of the move from falcon to pecan, the default content type for
requests has changed from application/json to
application/xxx-www-form-urlencoded.  That means that calls without a
content-type which used to work with the falcon implementation will
now fail with the pecan implementation.

If a caller sets their content-type header to application/json then
everything works as expected.

However, if a caller omits the content-type header then it now defaults to
application/xxx-www-form-urlencoded.  This means that pecan will URLencode
the request body, and that causes barbican to fail when it tries to JSONify
it.


Proposed Change
===============

The proposed change is to check the content-type header on the following
types of Barbican requests, and reject any request (via pecan abort with
HTTP 415) that does not specify the correct value:

=============  =========   ========================
Resource       HTTP Verb   Enforced Content Types
=============  =========   ========================
Secret         PUT         application/octet-stream
                           text/plain
Secrets        POST        application/json
Orders         POST        application/json
Containers     POST        application/json
TransportKeys  POST        application/json
=============  =========   ========================


Alternatives
------------

There are alternatives to the proposed change:

   * Change the content-type header to the correct value.  This would resolve
     the problem described above, but could have unintended consequences as
     the user's input data would be changed from what they provided/expected.

   * Detect the situation where the default was taken for content-type and
     compensate by:

        * URL-decoding the string
        * cleanup any padding characters that may have been added

Data model impact
-----------------

None.

REST API impact
---------------

Users of the following resources will have to provide a valid value in
their content-type header on REST API calls as indicated below:

=============  =========   ======================================
Resource       HTTP Verb   Required Content Types
=============  =========   ======================================
Secret         PUT         application/octet-stream or text/plain
Secrets        POST        application/json
Orders         POST        application/json
Containers     POST        application/json
TransportKeys  POST        application/json
=============  =========   ======================================

Security impact
---------------

None.

Notifications & Audit Impact
----------------------------

None.

Other end user impact
---------------------

None.

Performance Impact
------------------

Each REST call will now incur a check of the content-type.  Calls without the
correct content-type will fail with pecan.abort and HTTP 415.


Other deployer impact
---------------------

None.

Developer impact
----------------

Developers who use a tool such as curl will need to ensure that they pass
content-type:application/json in their HTTP headers, otherwise they will
fail with HTTP 415.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  sheyman

Other contributors:
  john-wood-w
  arunkant-uws

Work Items
----------

There are 3 work items required for this change:

* updating the barbican code to detect incorrect content types and
  pecan abort with HTTP 415
* updating the barbican documentation to describe this change and
  provide recovery action
* updating the barbican tests to validate the behavior of the new code.
  This includes unit and functional tests.


Dependencies
============

None.


Testing
=======

The following positive test scenarios will be needed for this proposed
change:

+----------------+--------+---------------------------+-----------------+
| Resource       | Verb   | Content-Type              | Expected Result |
+================+========+===========================+=================+ 
| Secret         | PUT    | application/octet-stream  | success         |
+----------------+--------+---------------------------+-----------------+
| Secret         | PUT    | text/plain                | success         |
+----------------+--------+---------------------------+-----------------+
| Secrets        | POST   | application/json          | success         |
+----------------+--------+---------------------------+-----------------+
| Orders         | POST   | application/json          | success         |
+----------------+--------+---------------------------+-----------------+
| Containers     | POST   | application/json          | success         |
+----------------+--------+---------------------------+-----------------+
| Transport Keys | POST   | application/json          | success         |
+----------------+--------+---------------------------+-----------------+

In addition, the following negative tests will be used to verify behavior:


+----------------+--------+---------------------------+-----------------+
| Resource       | Verb   | Content-Type              | Expected Result |
+================+========+===========================+=================+ 
| Secret         | PUT    | none (omitted)            | HTTP 415        |
+----------------+--------+---------------------------+-----------------+
| Secret         | PUT    | application/octet-streamx | HTTP 415        |
+----------------+--------+---------------------------+-----------------+
| Secret         | PUT    | text/plainx               | HTTP 415        |
+----------------+--------+---------------------------+-----------------+
| Secret         | PUT    | applicationx/octet-stream | HTTP 415        |
+----------------+--------+---------------------------+-----------------+
| Secret         | PUT    | textx/plain               | HTTP 415        |
+----------------+--------+---------------------------+-----------------+
| Secret         | PUT    | application/json          | HTTP 415        |
+----------------+--------+---------------------------+-----------------+

+----------------+--------+---------------------------+-----------------+
| Resource       | Verb   | Content-Type              | Expected Result |
+================+========+===========================+=================+ 
| Secrets        | POST   | none (omitted)            | HTTP 415        |
+----------------+--------+---------------------------+-----------------+
| Secrets        | POST   | text/plain                | HTTP 415        |
+----------------+--------+---------------------------+-----------------+
| Secrets        | POST   | application/octet-stream  | HTTP 415        |
+----------------+--------+---------------------------+-----------------+
| Secrets        | POST   | application/jsonx         | HTTP 415        |
+----------------+--------+---------------------------+-----------------+
| Secrets        | POST   | applicationx/json         | HTTP 415        |
+----------------+--------+---------------------------+-----------------+
| Secrets        | POST   | application/jsonx         | HTTP 415        |
+----------------+--------+---------------------------+-----------------+

+----------------+--------+---------------------------+-----------------+
| Resource       | Verb   | Content-Type              | Expected Result |
+================+========+===========================+=================+ 
| Orders         | POST   | none (omitted)            | HTTP 415        |
+----------------+--------+---------------------------+-----------------+
| Orders         | POST   | text/plain                | HTTP 415        |
+----------------+--------+---------------------------+-----------------+
| Orders         | POST   | application/octet-stream  | HTTP 415        |
+----------------+--------+---------------------------+-----------------+
| Orders         | POST   | application/jsonx         | HTTP 415        |
+----------------+--------+---------------------------+-----------------+
| Orders         | POST   | applicationx/json         | HTTP 415        |
+----------------+--------+---------------------------+-----------------+
| Orders         | POST   | application/jsonx         | HTTP 415        |
+----------------+--------+---------------------------+-----------------+


+----------------+--------+---------------------------+-----------------+
| Resource       | Verb   | Content-Type              | Expected Result |
+================+========+===========================+=================+ 
| Containers     | POST   | none (omitted)            | HTTP 415        |
+----------------+--------+---------------------------+-----------------+
| Containers     | POST   | text/plain                | HTTP 415        |
+----------------+--------+---------------------------+-----------------+
| Containers     | POST   | application/octet-stream  | HTTP 415        |
+----------------+--------+---------------------------+-----------------+
| Containers     | POST   | application/jsonx         | HTTP 415        |
+----------------+--------+---------------------------+-----------------+
| Containers     | POST   | applicationx/json         | HTTP 415        |
+----------------+--------+---------------------------+-----------------+
| Containers     | POST   | application/jsonx         | HTTP 415        |
+----------------+--------+---------------------------+-----------------+


+----------------+--------+---------------------------+-----------------+
| Resource       | Verb   | Content-Type              | Expected Result |
+================+========+===========================+=================+ 
| Transport Keys | POST   | none (omitted)            | HTTP 415        |
+----------------+--------+---------------------------+-----------------+
| Transport Keys | POST   | text/plain                | HTTP 415        |
+----------------+--------+---------------------------+-----------------+
| Transport Keys | POST   | application/octet-stream  | HTTP 415        |
+----------------+--------+---------------------------+-----------------+
| Transport Keys | POST   | application/jsonx         | HTTP 415        |
+----------------+--------+---------------------------+-----------------+
| Transport Keys | POST   | applicationx/json         | HTTP 415        |
+----------------+--------+---------------------------+-----------------+
| Transport Keys | POST   | application/jsonx         | HTTP 415        |
+----------------+--------+---------------------------+-----------------+





Documentation Impact
====================

Documentation will need to specify that the content-type header is now
required for the API/verbs shown above under "Proposed Change".  Omitted 
content-type, or content-type other than the allowed values will result
in an HTTP 415.

NOTE: current documentation says that a secret PUT "should" include
the appropriate content-type.  That will have to be updated to "must".  See
https://github.com/cloudkeep/barbican/wiki/Application-Programming-Interface#put


References
==========

The following bugs and proposed fixes led to the creation of this blueprint:

*  https://review.openstack.org/#/c/97554
*  https://review.openstack.org/#/c/99423
*  https://bugs.launchpad.net/barbican/+bug/1321555
*  https://bugs.launchpad.net/barbican/+bug/1320276
