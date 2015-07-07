..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=======================
Allow Admins to add CAs
=======================

https://blueprints.launchpad.net/barbican/+spec/add-cas

Problem Description
===================

In Kilo, code was added to allow clients to select a back-end CA server
by specifying a Barbican defined ca_id.  These ca_ids can be queried by
doing a GET against the /cas interface.

In addition, it is now possible for a project admin to associate a set of CAs
with a project, and to define a preferred (or default) CA in case a CA is not
explicitly requested.  All this amounts to the ability for project admins to
constrain project clients to specific CAs.

It is now time to take the next step.  Dogtag has implemented the ability to
create back-end subordinate CA's on the fly.  We need to expose this
functionality to project admins, so that they will be able to create project
specific CAs.  This would allow all certificates issued to clients in the
project to be scoped to the project only.

This project specific scoping improves security by isolating clients, and gives
project admins greater control over their certificates.  Essentially, it is
PKI-as-a-service.

Proposed Change
===============

Creating a CA
-------------

Most of the infrastructure we need is already present.  We will need to add
a POST /cas call to request the creation of a new CA.  This call will be
restricted to project admins.  This call will require at least three parameters:

* subject DN for the new CA signing certificate.
* name or handle for the new CA
* parent_ca_ref - the reference of the CA to which this CA should be subordinate.
  That is, the CA that will issue the new CA's signing certificate.

Additional parameters may be required to request specific attributes on the
subordinate CA's signing certificate.  These are currently not exposed in
Dogtag, and therefore will not be part of the initial implementation.

Two new methods will be added to the CertificateManager interface.

* supports_create_ca() - which will be called to determine if a given plugin
  supports this operation.  A default implementation returning False will be
  added to CertificateManager,  Plugins that support this option will have to
  override this function.

* create_ca(ca_create_dto) - which would be called to create the subCA,
  A default implementation that throws a NotSupported exception would be added
  to CertManager so that existing plugins would not need to change.
  The relevant parameters would be passed in the ca_create_dto.

Barbican core would select an appropriate CA plugin, and invoke create_ca().
If no plugin is appropriate, a 400 error will be returned.

Otherwise, once the new subCA is created, the data for that subCA will be
returned to Barbican core, which will add the CA to the CA tables.

The newly created CA reference will be returned to the client.

CA Ownership
------------

An additional "owner" field will be added to the CA table.  This field will be
populated with the project_id of the project admin when a CA is created using
the mechanism described above.  Changes will be needed to ensure that this
field is not overwritten when Barbican-core syncs up its CA table with the CA
plugins.

CAs which have been created through the /cas interface (and therefore owned by
a project) will only be list-able and view-able by members of the owning project.
Moreover, only members of that project will be able to submit certificate orders
to that CA.

Deleting a CA
-------------

Only project admins will be able to delete a CA.  The CA must be owned by the
project to be deletable.  To do this, a new REST method DEL /cas/{ca_id} will
be exposed.

This method would look up the necessary CA plugin in the CA table, and would
invoke a new delete_ca(plugin_ca_id) call.  This call would make the relevant
call on the back-end CA, which would, amongst other things, revoke the CA
signing cert.

On success, the CA would be removed from the CA table.

Alternatives
------------

None.

Data model impact
-----------------

A new field "owner" will need to be added to the CA table.

REST API impact
---------------

We will need to add some calls to the /cas/ interface.

* POST /cas - create a CA, as described above.  The attributes name, parent_ca_ref
  and subject_dn are required, as seen in the example below.

  .. code-block:: none

    POST /v1/cas
    Headers:
        X-Project-Id: {project_id}

    Content:
    {
        "name": "Dogtag CA for Project yxz",
        "parent_ca_ref": "https://localhost/v1/cas/{parent_ca_uuid}",
        "subject_dn": "cn=CA Signing Cert for xyz, dc=example.com"
    }

* DEL /cas/{ca_id} - delete a CA, and revoke the CA signing cert, as described
      above.

Also, GET /cas and GET /cas/{ca_id} will need to be modified (most likely
in the DB query) so "owned CAs" are only accessible by project members.

Also, a validation will need to be added to the Orders interface to ensure that
owned CA's that are referenced by the request belong to the requestor's project.

Security impact
---------------

None.

Notifications & Audit Impact
----------------------------

Make sure that all the actions above are audited as per the auditing spec. [1]

Other end user impact
---------------------

python-barbicanclient will need to add methods to perform the new operations.

Performance Impact
------------------

Minimal.  This is a relatively infrequent admin operation.

Other deployer impact
---------------------

Migration scripts will need to be run on already existing deployments to add
the new "owner" field.  The owner field will be nullable, so that the code will
handle cases with/without owners.

Developer impact
----------------

Plugin developers should not need to make any changes, unless they want to
support the new methods.

Implementation
==============

Assignee(s)

Primary assignee:
    alee-3

Work Items
----------

* Update the CAs and Certificates API documentation.
* Write the functional tests.
* Add the new "owner" field and migration scripts.
* Add the new REST API and internal Barbican core logic.
* Add the Dogtag implementation.

Dependencies
============

None

Testing
=======

The current unit and functional tests will also be modified to reflect these
changes.

Documentation Impact
====================

This is a new feature that will need to be documented.

References
==========

1. Auditing spec https://review.openstack.org/#/c/159938/.

