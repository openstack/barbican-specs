..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===============================
Expose CA enrollment templates
===============================

https://blueprints.launchpad.net/barbican/+spec/expose-ca-enrollment-templates

Problem Description
===================

Barbican will allow interaction with multiple Certificate Authorities through
multiple CA plugins.  Each plugin expects the certificate requests to conform
to a specific format and to have specific input parameters.  A common CA
interface will make it easier for clients to construct certificate requests.
Nonetheless, a mechanism is needed to expose the parameters required to make
a certificate request.

Proposed Change
===============

A new resource has been proposed for certificate authorities in [1].  This
interface can be used to expose the parameters needed to create a certificate
request.  The details on the interface changes are shown in the REST API
section below.

The data would be obtained from the plugins by implementing two new API calls:
get_enrollment_templates() and get_enrollment_template(template).  These calls
can have default implementations that return None so that not all plugins need
implement these calls immediately.

For the common API, the methods get_common_enrollment_templates() and
get_common_enrollment_template() will be implemented to return the correct data.

Alternatives
------------

Leave the code-base as it is.

Data model impact
-----------------

None.

REST API impact
---------------

The following REST API operations will be added:

* GET /cas/templates/enrollment  -- returns the request template for the
  common CA API.  This will end up returning the data from
  get_common_enrollment_templates().

* GET /cas/{ca_id}/templates/enrollment -- returns the list of request
  templates for a specific CA (as defined by ca_id).  This will end up returning
  the data from calling get_enrollment_templates(plugin_ca_id) on the
  specific plugin.

The ca_id values are defined in the spec for identifying ca_ids [1], which has
already been implemented in the server.

This approach also leaves open the possibility to exposing other types of
templates in future as needed (like renewal and revocation templates).

The data returned can consist of parameters and/or links for additional data.
All relevant links will be subordinate to the request template URL.

For example, for the Dogtag CA, the call:

* GET /cas/{ca_id}/templates/enrollment could return:

  {"templates":[
      {"id": "caUserCert",
       "name": "Manual User Dual-Use Certificate Enrollment",
       "description": "This certificate profile is for enrolling user certificates.",
       "link": "https://host:port/cas/ca1/templates/enrollment/caUserCert"},

      {"id": "caServerCert",
       "name": "Server Certificate Enrollment",
       "description": "This certificate profile is for enrolling server certificates.",
       "link": "https://host:port/cas/ca1/templates/enrollment/caServerCert"}

  ]}

* GET /cas/{ca_id}/templates/enrollment/caServerCert could return:

  {"parameters":[
      {"id": "profile_id",
       "description": "Profile ID",
       "syntax": "string"},

      {"id": "cert_request_type",
       "description": "Certificate Request Type",
       "syntax": "cert_request_type"},

      {"id": "cert_request",
       "description": "Certificate Request",
       "syntax": "cert_request"},

      {"id": "requestor_name",
       "description": "Requestor Name",
       "syntax": "string"},

      {"id": "requestor_email",
       "description": "Requestor Email",
       "syntax": "string"},

      {"id": "requestor_phone",
       "description": "Requestor Phone",
       "syntax": "string"}

  ]}

The idea would be that a client that filled in all the parameters listed above
would create a certificate request with all the required parameters.

An example request using the above parameters would look like::

  { 'type': 'certificate',
    'meta': { 'profile_id': 'caServerCert',
              'ca_id': '{ca_id}'
              'cert_request_type': 'pkcs10',
              'cert_request': '<put PEM formatted CSR here>',
              'requestor_name': 'John Wood',
              'requestor_email': 'jwood@rackspace.com',
              'requestor_phone': '555-1212'
            }
  }

Note that specification of 'profile_id' requires the specification of the 'ca_id'.
This check is already enforced in the server.

Security impact
---------------

None.  The data that are exposed through this interface are parameters which
the CA chooses to expose to show users how to interact with it. These are the
things that would be exposed by the CA on a web UI , say, or as part of
the CA's client API.  This  data is necessarily public - otherwise no one would
know how to interact with the CA.

All this info is queried by the CA plugin to the CA.  Barbican only reveals
what the CA chooses to expose.

Notifications & Audit Impact
----------------------------

None.

Other end user impact
---------------------

The python-barbicanclient will need to be enhanced to take advantage of this
new feature.  In particular, it may be possible to recurse through the parameters
either interactively or otherwise to generate valid cert requests.

Performance Impact
------------------

None.

Other deployer impact
---------------------

None.

Developer impact
----------------

Plugin developers may choose to implement the new methods to take advantage
of the new functionality.

Implementation
==============

Assignee(s)

Primary assignee:
    alee-3

Work Items
----------

* Add code to implement the new API calls and call get_enrollment_templates() etc.
  Add these calls and their default implementation to the plugin interface.
* Add code for get_common_enrollment_templates() and get_common_enrollment_template(x).
* Implement get_enrollment_templates() and get_enrollment)template() for each plugin,
  as needed.
* Document all, including sphinx documentation.

Dependencies
============

None

Testing
=======

The current unit and functional tests will also be modified to reflect this
change.

Documentation Impact
====================

New feature that will need to be documented.

References
==========

[1] Spec for identifying CAs: https://review.openstack.org/#/c/129048
    This spec was implemented in the server in Kilo.
