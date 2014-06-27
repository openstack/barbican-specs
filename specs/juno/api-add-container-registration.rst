..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==============================================
Barbican Spec - Barbican Consumer Registration
==============================================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/barbican/+spec/api-add-container-registration

Allow consumers to register as "interested parties" on Barbican Containers and
Secrets. This information would then be available when the container is
retrieved, which a client process could use to either warn about these
interested parties prior to performing a delete operation, or else use to
determine what parties to update if a new Container is created that needs to
replace an old one.


Problem description
===================

In some workflows, multiple clients will be accessing the same Container/Secret
information in Barbican. This workflow might involve a client updating or
deleting this 'shared' information. If this client could check to see what
entities are interested in this information, they might be able to perform
follow on operations (such as updating or notifiying the affected entities of
the update), or else warn the user that an operation could impact other entites
(such as before performing a delete).

An example workflow is when a client creates certificate information within
Barbican (which creates a Container with a unique UUID), and then shares that
Container UUID with LBaaS as part of creating a load balancer instance. LBaaS
could both retrieve the Container by UUID to get the certificate to install,
and also register itself as an interested consumer for that Container. Now if a
client wishes to delete that same Container later, they can check to see if one
or more consumers are interested in the Container first. They might choose (for
example) to remove the load balancer first, or else abort the delete process
altogether.


Proposed change
===============

Allow consumers to register their interest for a Barbican Container or Secret.

This will require a new REST Pecan API sub-resource under the 'containers' and
'secrets' resource, so consumers can POST to register and DELETE to deregister
with their consumer data in the JSON body. This is detailed in the API impact
section below.

We will also need two new tables in the data model for storing this consumer
data. This is detailed in the Data model impact section below.


Alternatives
------------

This could also be done without a new resource, and using query parameters like
'register' and 'deregister' on the existing GET call for a Container, but that
does not conform to a true RESTful scheme and would add side effect bahaviors
to the simple GET call.

We could avoid the admin-api and instead add the functionality to the standard
user-api, restricting access using keystone roles.

We could use a single generic table for the Data Model instead of one for each
entity type, but it seems to be consensus that it is worth an extra table in
order to maintain a clear ForeignKey relationship.

Alternatives to the entire service registration concept include the following:
    1) Keep Barbican simple and force clients to handle switching from one
    Container UUID to a new one, in all the dependent consumers such as LBaaS.
    Orchestration systems such as Heat might be able to handle this conversion.
    However the LBaaS community is concerned that since clients would still
    have access to the Containers, they could in turn delete them from under
    such orchestrations. LBaaS attempts to failover to a new load balance
    instance if needed would then fail, as the original Container UUID would no
    longer be available.

    2) Allow Containers to be mutable such that certificates (for example)
    could be updated without changing Container UUIDs. Barbican could then fire
    off events to interest parties (like LBaaS) for them to update certificates
    in load balancer instances. If new certificate is flawed however, LBaaS
    would need to be able to revert back to a previous version of the
    certificate. Adding versioning to Barbican would be a significant effort
    with minimal benefit versus effort.


Data model impact
-----------------

A new model and repository will be created for ContainerConsumerMetadatum
containing the following:

 - container_id (ForeignKey reference to Container.id)
 - name (String, freeform data specified by the consumer, eg: "LBaaS", "VPNaaS")
 - URL (String, resource locator for the object that depends on this entity,
   eg: "https://lbaas.myurl.net/loadbalancer/<lbid>")

A Unique constraint would be put on the combination of "container_id", "name",
and "URL". An Index would be created on "container_id" and "name".

The same will be done for SecretConsumerMetadatum, with column names updated
appropriately.

As this is an optional one to many relationship off of Containers/Secrets, data
migrations should not be required.


REST API impact
---------------

This BP impacts both the admin API and the user API.

**Changes that affect the admin API:**

Need to add a new resource in the admin-api named "consumers", supporting GET,
POST and DELETE. This would be a subresource of "containers" and "secrets", so
we would also need to create a stub resource for "containers" and "secrets" on
the admin-api.
Example: http://admin-api/v1/containers/{container-uuid}/consumers

A simple registration JSON Body would be structured like the following
using the POST method::

    {
        "name": "LBaaS",
        "URL": "https: //lbaas.myurl.net/loadbalancer/4124/"
    }

The response of this registration POST would be the current entity response
as available from a GET call. Hence after a client creates a Container and
hands that UUID to LBaaS (for example) to create a load balancer instance,
LBaaS will make the above POST call to the 'consumers' sub-resource of that
Container UUID. LBaaS would then synchronously receive the Container's
information as if they had performed a GET call, and can then install the
certificate on the load balancer.

Deregistration would use the same resource and JSON Body as the registration,
except using the DELETE method. Hence the delete would be performed by value
rather than by a specific UUID reference to a consumer entity (since the
registration UUID is never exposed to the consumer).

**Changes that affect the user API:**

The JSON Body for the GET response of a Container/Secret would include the
current GET response for that entity, plus a new 'consumers' element, that
will be structured as follows::

    {
        <current GET response data>,
        "consumers": [
            {
                "name": "LBaaS",
                "URL": "https://lbaas.myurl.net/loadbalancer/4124/"
            },
            {
                "name": "LBaaS",
                "URL": "https://lbaas.myurl.net/loadbalancer/4125/"
            },
            {
                "name": "VPNaaS",
                "URL": "https://vpn.myurl.net/vpn/345634/"
            }
        ]
    }

If significant performance degredation is observed in testing, the current
Container/Secret API could change as follows in a future CR:

    1) The GET on the root (list) call can be modified to not return the
    'consumers' attribute to reduce the overall size of the response message.


Security impact
---------------

This blueprint would not require changes to cryptographic resources. We might
need to impose an upper limit on the number of consumers that can be registered
per Container/Secret to prevent denial of service attacks.

There is also some concern that any metadata stored on Barbican entities should
be encrypted for security, but that concern applies to much more than this
functionality, and should probably be handled as its own blueprint.


Notifications impact
--------------------

None.


Other end user impact
---------------------

None.


Performance Impact
------------------

Initially, there could be a performance impact on the GET for /containers/ and
/secrets/ (the list, not individual entities) because of an increase in the
amount of data returned. If this is significant, then consumer registration
information will only be retrieved for GETs on specific Containers/Secrets, and
not on the GET list of all Containers/Secrets.


Other deployer impact
---------------------

None.


Developer impact
----------------

Developers of services that consume Barbican Containers/Secrets would use the
admin-api consumer service instead of using the standard api GET resource when
they wish to register their interest in an entity. This feature is optional
however, and would not impact current workflows.


Implementation
==============

Assignee(s)
-----------

Primary assignees:
 - adam-harwell
 - vivek-jain


Work Items
----------

The initial implementation will consist of two CRs:
 - One CR will be created for the initial implementation with Containers.
 - A subsequent CR will be created to add this functionality to Secrets.

There may be an additional CR for the above mentioned performance changes.

There may be an additional BP/CR for encryption of this (and other) metadata.


Dependencies
============

None.


Testing
=======

Add unit and integration testing for the new feature.


Documentation Impact
====================

Update API guide here:
https://github.com/cloudkeep/barbican/wiki/Application-Programming-Interface


References
==========

https://wiki.openstack.org/wiki/Neutron/LBaaS/SSL#TLS_Certificates_Management
(At the time this was written, this wiki is still assuming Mutable Containers,
which will be corrected soon.)
