..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=======================
Common Certificate API
=======================

https://blueprints.launchpad.net/barbican/+spec/certificate-order-api

Problem Description
===================

We currently have a mechanism for enrolling a certificate request by creating
a new order, and passing parameters in the metadata of the order.  Currently,
the parameters in the metadata are passed through to the CA plugin unchanged.
This means that - as long as the parameters passed in match the parameters
expected by a particular back-end, then a  certificate request can be created
for that back-end.

This diminishes the power of the abstraction though.  A client should not
have to know which CA is served by the Barbican server.  Rather, a client
should be able to request a certificate through a common API, which should be
supported by all known certificate plugins.

The trick is to define an interface/metadata parameters that would be
supported by most CA's.  An attempt was made to do this in [1].

Recently, it was suggested to use RFC 7030 [4], as a standards-based approach
for a common API.  RFC7030 basically proposes an HTTP interface for submitting
CMC (RFC 5272) requests, and receiving the relevant responses.

RFC 7030 is too different from the current Orders interface and functionality
for us to consider adding for Kilo.

On the other hand, CMC requests - both simple and full - as defined in
RFC 5272 [2] - have been standards for some time now, and are supported by
most, if not all, Certificate Authorities.  This makes them ideal candidates
for objects in a common set of parameters supported by multiple back-end
CA's.

Ideally, the CMC request could be passed intact from barbican core
through the plugin to the backend CA.  The following CA's are known to support
CMC requests directly: Dogtag, Microsoft CA.

Even if a CA does not support CMC requests directly, the CMC request provides
a convenient standard-based vechicle for describing all the requested x509
extensions, request data and authentication mechanisms like POP.  These are
things that we do not want to try to re-invent.

And of course, as simple CMC == PKCS10 - which should be supported by all CAs,
writing a plugin method to support simple CMC requests should be trivial.

Proposed Change
===============

We will continue to use the Orders resource to enroll certificate requests,
and the order metadata to contain the request parameters.  Order requests
can look like this:

Option 1: Order using a Simple PKI Request (See Section 3.1 in RFC 5272)::

    {
        "type": "certificate",
        "meta": {
            "ca": "CA identifier (optional, see note in reference [3] below)",
            "request_type": "simple-cmc",
            "request_data": "base64 encoded simple CMC request (PKCS10)"
            "requestor_name": "(optional) string"
            "requestor_email": "(optional) email address - RFC 5322, 5321"
            "requestor_phone" : "(optional) string"
         }
    }

Option 2: Order using a Full PKI Request (PKCS7, see section 3.2 in RFC 5272)::

    {
        "type": "certificate",
        "meta": {
            "ca": "CA identifier (optional, see note in reference [3] below)",
            "request_type": "full-cmc",
            "request_data": "base64 encoded full CMC request (PKCS7)"
            "requestor_name": "(optional) string"
            "requestor_email": "(optional) email address - RFC 5322, 5321"
            "requestor_phone" : "(optional) string"
         }
    }

Option 3: Order using keys already stored in Barbican::

    {
        "type": "certificate",
        "meta": {
            "ca": "CA identifier (optionali, see note in reference [3] below)",
            "request_type": "stored-key",
            "public_key_ref": "barbican UUID for public key",
            "subject_dn": "subject DN (RFC1485 [5])"
            "extensions": "base 64 DER encoded ASN.1 values (RFC 5280)"
            "requestor_name": "(optional) string"
            "requestor_email": "(optional) email address - RFC 5322, 5321"
            "requestor_phone" : "(optional) string"
         }
    }

    Extensions data defined by the following in RFC 5272 (section 3.1), where
    the specific ASN.1 structures for each extension are defined in RFC 5280
    (section 4.2) [6]:

    Extensions  ::=  SEQUENCE SIZE (1..MAX) OF Extension

    Extension  ::=  SEQUENCE  {
         extnID      OBJECT IDENTIFIER,
         critical    BOOLEAN DEFAULT FALSE,
         extnValue   OCTET STRING
                     -- contains the DER encoding of an ASN.1 value
                     -- corresponding to the extension type identified
                     -- by extnID
         }

Order 4: Orders using parameters for a specific CA.

This mechanism is already supported by the existing code.  There is a
blueprint for an API to retrieve the possible parameters from the server [7]::

    {
        "type": "certificate",
        "meta": {
            "ca": "CA identifier (optional, see note in reference [3] below)",
            "request_type": "custom",
            "ca_specific_parameter1": "value",
            "ca_specific_parameter2": "value2",
            ...
         }
    }

In the Barbican server, logic will need to be added to differentiate between
these different request types.  For backward compatibility, if request_type
is not provided, then we will assume that it is "custom".

On the server, we can add the following logic:

* For Orders 1 and 2, we may be able to validate whether the request_data
  is valid PKCS10/ PKCS7. To do this, we could use PyASN.1 to attempt to
  decompose the request into the relevant ASN.1 structure, and fail if the
  decomposition is unsuccessful.

* For Order 3, we should retrieve the relevant public key, create a PKCS10
  request using the public key, subject DN and extensions data.  Submit to
  backend as a simple CMC request.

* For Orders 1,2,3, submit to plugin through new CertPlugin API calls:
  issue_simple_cmc_request(), issue_full_cmc_request().

* Plugins will interact with the back-end CA, which presumably will return
  PKCS7 full CMC responses.

  The plugin will be responsible for interpreting the response and returning
  a ResultDTO object to barbican-core.  Processing would then proceed as
  already coded in Barbican-core.

  In order to simplify the plugin code, and to avoid unnecessary duplication
  of code, barbican-core will provide some utility functions to parse a
  CMCResponse and convert the result into a ResultDTO object.

  The details of this work need not be specified in this spec, but essentially
  this involves mapping the CMCStatus field to a Barbican CertificateStatus
  values, and extracting certs, intermediates, Query Pending control/ pendInfo
  data, and/or error messages as appropriate.

* The new and existing certificate request issuance methods
  (issue_*_cmc_request, issue_certificate_request) will be provided default
  implementations (which would raise NotImplemented exceptions) in case any
  plugins have not yet implemented the relevant methods.

  In addition, the supports() on each plugin would need to be modified to
  specify whether the plugin supports each certificate issuance method.

  Moreover, the API that identifies the available CAs will also include
  information on which methods are supported by each plugin, so that clients
  can determine which method to use for a particular CA a priori.  This
  information can be extracted from the supports() method.  See [3] for more
  information.

Note on Certificate Request Updates:

* Typically, a CA will not allow a certificate request to be amended once it
  has been submitted, with the exception of the optional requestor data.

  For the certificate update API, therefore, I propose the following method:

  PUT /orders/{order_id}::

    {
        "type": "certificate",
        "meta": {
            "requestor_name": "(optional) string"
            "requestor_email": "(optional) email address - RFC 5322, 5321"
            "requestor_phone" : "(optional) string"
         }
    }

  This unambiguously specifies the fields that we expect to be able to change
  in a certificate request for all request types above.

Alternatives
------------

One possibility is to implement an API that implements RFC 7030, which is also
an HTTP interface around CMC requests. RFC 7030 expects clients to send CMC
requests to specific URLs, and process CMC responses.

The main problem with adopting a RFC 7030 interface is that it is very
different from what we have already implemented in Barbican, and we would have
to jettison/rewrite much of the code we already have.  That is not doable in
kilo.

Some of the workflows are quite different.

For example, if the cert is pending, there is logic within Barbican to poll
for the certificate until it is approved and issued. Then the certificate
and its intermediates are stored within Barbican for later retrieval.

For RFC7030, the client needs to handle the polling by decoding the pkcs7-mime
response -- which is 202 -- and re-submitting the same request until it is
approved.

Moreover, there is no support for storing the certificate on the server,
and returning things like order_id, secret_id etc. Nor is there support for
amending/ canceling the order.

Instead of supporting CMC, we could try to define a common API with a basic set
of parameters for different profiles of certificates.  This is the approach
started in [1].  If we adopt this blueprint, we will abandon [1].

This is simpler, but seems a little arbitrary.  Using CMC is likely to be
supported by most CA's because it is standards-based.  Also, there are likely
to be existing clients that already do CMC.

The simpler mechanism is also limited, having no support for encryption or POP.
Ultimately, I think we would need to evolve to support something like CMC.

Data model impact
-----------------

We may need to add addtional data in the order_metadata, but this should not
result in database changes.

REST API impact
---------------

See above in proposed changes.

Security impact
---------------

None

Notifications & Audit Impact
----------------------------

None, other than additional audit changes needed.

Other end user impact
---------------------

There will need to be changes in either barbican-client or certmonger
to support the new API.

Performance Impact
------------------

There will be some additional processing that will need to be done to parse
the CMCRequest.  These changes are not covered in the spec.

Other deployer impact
---------------------

None.

Developer impact
----------------

Plugin developers will need to modify the supports() method to specify
which issuance methods they will support.  This information will be used
to appropriately route certificate requests, and to notify the client of
the CA's capabilities when interacting with the interface defined in [3].

Plugin developers will need to implement two new methods:
issue_simple_cmc_request() and issue_full_cmc_request() if they want to
support these requests.  We expect simple CMC to be easy to implement
because simple CMS == PKCS10.

Implementation
==============

Assignee(s)

Primary assignee:
    alee
    chellygel
    dave-mccowan
    woodster

Work Items
----------

* Modify API code to parse the new Order parameters and pass the data
  to lower levels.

* Add code to validate CMC request data.

* Add code to certificate_manger.py to send the requests to the plugins.
  Modify the plugin interface contract to provide default implementations.

* Plugins should modify the supports() method and implement the new methods
  as appropriate.

* Add code to certificate_manager.py to parse a CMC Response and convert
  into a ResultDTO object to return to barbican-core.

* Add client code.  This probably requires a separate blueprint.

Dependencies
============

This work should do done in conjunction with the work on the interface
to get CA identifiers in [3].

Testing
=======

More unit and functional tests will be needed.

Documentation Impact
====================

Docs (including sphinx and plugin docs) will need to be changed to
reflect this API.

References
==========

[1] https://review.openstack.org/#/c/129695 Existing Blueprint to define a
set of common cert API parameters.

[2] http://tools.ietf.org/html/rfc5272 Certificate Management over CMS

[3] API for getting CA identifiers is defined in this blueprint.  In addition,
there is an API for getting the CA signing certificate and intermediates.
https://review.openstack.org/129048

[4] https://tools.ietf.org/html/rfc7030 Enrollment over Secure Transport

[5] http://tools.ietf.org/html/rfc1485
A string representation of distinguished names

[6] http://tools.ietf.org/html/rfc5280 Internet X.509 Public Key
Infrastructure Certificate and Certificate Revocation List (CRL) Profile

[7] https://review.openstack.org/129377 API for getting CA specific
certificate enrollment parameters.

[8] LibEST implementation https://github.com/cisco/libest
