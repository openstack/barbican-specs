..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===============================================
Add Version Responses Consistent with Openstack
===============================================

Launchpad blueprint:
https://blueprints.launchpad.net/barbican/+spec/fix-version-api

Barbican currently returns one-off version information for GETs to the root
resource. This blueprint calls for replacing that response with an OpenStack
consistent response based on json-home
(see http://http://tools.ietf.org/html/draft-nottingham-json-home-03),
that includes which API versions are currently supported.
Such version information could facilitate automatic discovery of services when
appended with the endpoint information retrieved from Keystone service
catalogs for example.


Problem description
===================

When the root resource of Barbican is queried with a GET request, it should
respond with a version response consistent with other OpenStack projects,
based on json-home. This was discussed at a July 2014 mid-cycle meetup for
Keystone and Barbican
(see https://etherpad.openstack.org/p/keystone-juno-hackathon for details).
The following is an example of implementing a json-home type of response from
Keystone
(documented here:
http://docs.openstack.org/api/openstack-identity-service/2.0/content/\
Versions-d1e472.html)::

    {
        "choices": [
            {
                "id": "v1.0",
                "status": "DEPRECATED",
                "links": [
                    {
                        "rel": "self",
                        "href": "http://identity.api.openstack.org/v1.0"
                    }
                ],
                "media-types": {
                    "values": [
                        {
                            "base": "application/xml",
                            "type": "application/vnd.openstack.identity+xml;version=1.0"
                        },
                        {
                            "base": "application/json",
                            "type": "application/vnd.openstack.identity+json;version=1.0"
                        }
                    ]
                }
            },
            {
                "id": "v1.1",
                "status": "CURRENT",
                "links": [
                    {
                        "rel": "self",
                        "href": "http://identity.api.openstack.org/v1.1"
                    }
                ],
                "media-types": {
                    "values": [
                        {
                            "base": "application/xml",
                            "type": "application/vnd.openstack.identity+xml;version=1.1"
                        },
                        {
                            "base": "application/json",
                            "type": "application/vnd.openstack.identity+json;version=1.1"
                        }
                    ]
                }
            },
            {
                "id": "v2.0",
                "status": "BETA",
                "links": [
                    {
                        "rel": "self",
                        "href": "http://identity.api.openstack.org/v2.0"
                    }
                ],
                "media-types": {
                    "values": [
                        {
                            "base": "application/xml",
                            "type": "application/vnd.openstack.identity+xml;version=2.0"
                        },
                        {
                            "base": "application/json",
                            "type": "application/vnd.openstack.identity+json;version=2.0"
                        }
                    ]
                }
            }
        ],
        "choices_links": ""
    }

When a GET is performed for a specific version, version details should be
provided in response such as per this Keystone example::

     {
        "version": {
            "status": "stable",
            "updated": "2014-04-17T00:00:00Z",
            "media-types": [
            {
                    "base": "application/json",
                    "type": "application/vnd.openstack.identity-v2.0+json"
                },
                {
                    "base": "application/xml",
                    "type": "application/vnd.openstack.identity-v2.0+xml"
                }
            ],
            "id": "v2.0",
            "links": [
                {
                    "href": "http://23.253.228.211:5000/v2.0/",
                    "rel": "self"
                },
                {
                    "href": "http://docs.openstack.org/api/openstack-identity-service/2.0/content/",
                    "type": "text/html",
                    "rel": "describedby"
                },
                {
                    "href": "http://docs.openstack.org/api/openstack-identity-service/2.0/identity-dev-guide-2.0.pdf",
                    "type": "application/pdf",
                    "rel": "describedby"
                }
            ]
        }
    }


Proposed change
===============


However, as Chad Lung pointed out in the Launchpad blueprint, several projects
document and implement the version responses shown above, so Barbican can
follow the same approach. Barbican only supports requests and response in the
JSON format, so XML formats will not be supported or implemented via this
blueprint.

Keystone's keystone/controllers.py implements the GET requests and builds the
version responses. Note that the MEDIA_TYPE_JSON should be
'application/vnd.openstack.keymanagement-%s+json'. The 'Version' controller
defined in this file is mapped to the URI path via the keystone/routers.py
module. Finally the keystone/service.py module builds the final WSGI
application, and includes the version routers defined in routers.py.

For Barbican then, the barbican/api/controllers/versions.py VersionController
class could be modified similarly to Keystone's controllers.py, but should
be renamed to VersionsController to handle the root requests, with a new
VersionController added to return specific version detail responses.

The barbican/api/app.py file should be modified to use the new
VersionsController wherever VersionController is used now. Note that while the
root version resource would support unauthenticated requests, specific version
resource requests would require authentication.

Also, the build version information that is responded with the root
resource now would still need to be provided to support automated deployment
and testing processes. This blueprint proposes adding a query parameter to the
root resource such as ?build_version, that would return a JSON response similar
to the current one but without the version information, such as this::

    {
        "build": "2014.1.dev43.g22d1a96"
    }

An alternative to retrieving this build information is detailed in the
Alternatives section below.


Finally, the Barbican Python client should be updated to take advantage of
this new version discovery information to augment the endpoint information it
retrieves from Keystone service catalogs.


Alternatives
------------

Regarding utilizing existing frameworks to generate the version information,
there does not appear to be a Python library that implements json-home.

Regarding the build version information, that is not addressed by the
json-home specification, although in Appendix C of the IETF reference in a
section labeled 'Open Issues', there appears to be a placeholder called
'release info?' that might address this at a future date. So perhaps we could
just add the build information to the root version response.


Data model impact
-----------------

None.


REST API impact
---------------

The root version response will be changed from the one-off version information
returned now to the OpenStack consistent response. A specific version details
response (i.e. when performing a GET on v1/ for example) will need to be added.
The current root response that includes the build version would need to be
modified to only return this information if a query parameter is specified.


Security impact
---------------

None.


Notifications impact
--------------------

None.


Other end user impact
---------------------

Any processes that rely on the current build version response would need to be
modified. Clients using the Barbican Python client will need to update to the
release after the changes in this blueprint are made.


Performance Impact
------------------

None.


Other deployer impact
---------------------

None.


Developer impact
----------------

None.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  jaosorior

Other potential contributors:
  john-wood-w
  chad-lung


Work Items
----------

The following CRs would build out this blueprint:

1) Modify version-related modules per proposal above.

2) Implement the build_version query parameter on the root resource to return
the build version.

3) Update Barbican Python client code base to query the root resource for
version information and then apply that to the endpoint information retrieved
from the Keystone service catalog.


Dependencies
============

None.


Testing
=======

Unit testing of the version resources will be added.


Documentation Impact
====================

Update this page with API changes:
https://github.com/cloudkeep/barbican/wiki/Application-Programming-Interface


References
==========

See code and documentation references embedded in this blueprint above.

Information about json-home is found at
http://http://tools.ietf.org/html/draft-nottingham-json-home-03.

Notes on the json-home discussion at the July 2014 Barbican and Keystone
mid-cycle meetup can be found at
https://etherpad.openstack.org/p/keystone-juno-hackathon.
