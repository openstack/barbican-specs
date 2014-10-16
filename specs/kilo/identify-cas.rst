..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================================================================
Identify available CAs
====================================================================

https://blueprints.launchpad.net/barbican/+spec/identify-cas

Problem Description
===================

It is possible to have multiple CA plugins, each potentially talking to
multiple backend CA servers.  A mechanism is therefore needed to allow
the client to select a backend CA server.

In addition, Dogtag plans to implement the ability to configure lightweight
sub CA's - subordinate CA's that can exist within the same CA instance.  This
opens up the possibility of configuring a separate CA instance for each
project, so that the project could have certificates that are scoped to the
project only.  Thus, a mechanism is also required to associate a project with
a preferred CA, so that if a client does not request a specific CA, the
preferred CA is selected.

Also, a mechanism should be added to allow clients to discover the CA servers
available for a particular Barbican instance.

Proposed Change
===============

We need to add a new resource CertificateAuthorities that will specify the CAs
that are available to Barbican.  This resource will permit listing CAs and
associating projects with a CA.  See the REST API section for details.

Four new tables will be required:

* CertificateAuthority, which will list the CA's available.  This table will
  include the Barbican generated ca_id, the plugin name, plugin_ca_id,
  information about the CA and a cache expiration time.  The data will be
  assumed to be stale and to require a refresh when the cache expiration time
  expires.

* CertificateAuthorityMetadatum, which contains data about the CAs, to
  be provided to the client.

* ProjectCertificateAuthority, which will list the set of CA's defined per
  project.  If a project chooses to do so, it can specify a set of CA's
  (referenced by ca_ids) that a particular project can use.  The first entry
  for a project will automatically be added to the PrefCA table for that
  project to ensure that there will always be a preferred CA for a project.

* PreferredCertificateAuthority (PrefCA), which contains the preferred
  CA entry for each project.  This table requires the project_id to be unique
  so that there is only one preferred CA per project.  A special entry for
  the preferred global CA will have "project_id" = 0.

See the Data model section for details.

The ProjectCertificateAuthority table will be updated through REST API calls
by a project administrator.

The CertificateAuthority table will be updated as follows:

* On startup, Barbican should iterate through all the CA plugins and execute
  certificate_resources.update_ca_list(plugin_name).  This will trigger a call
  to a provides() method for each plugin.

* The provides() call will query the backend-CA to determine which CAs are
  provided.  It will return the relevant data ie. a list/dict of
  (plugin_ca_id, description, ca_ski etc.) as needed to populate/update the
  CertificateAuthority and CertificateAuthorityMetadatum table.
  It will also return an expiration time for the data.

* The CA and CA metadata table would then be updated by a method defined in
  Barbican core.  (eg. certificate_resources.update_ca_list(plugin_name)).
  Barbican core will use the plugin_ca_id to map the CA entry to the correct
  ca_id.  At this point: new CA's will be added; decommisioned CA's will be
  removed; and existing CA's will be updated.  In each case, the expiration
  time will be updated.

* A method is provided below for the user to obtain a list of available CA's.
  If the expiration time for a CA has elapsed, the plugin for that particular
  CA will be queried by calling the provides() method.  The CA table and CA
  metadata tables will be updated accordingly.

* If a certificate request is made and a ca_id is specified, the relevant
  plugin will be looked up in the CA table.  If the entry for this ca_id is
  stale, then the provides() method will be invoked for this CA to update
  the CA table.

Logic when a certificate is requested:

When a certificate is requested,

  * If a ca_id is provided:

    * If the ca_id is not defined in the CA table, a 400 error is returned.
    * If any entries for the project exists in the PCA table, and the ca_id is
      not one of those entries, a 403 error is returned.
    * Otherwise the request is passed through to the plugin specified by the
      entry in the CA table.

  * If a ca_id is not provided:

    * If at least one entry for the project exists in the PrefCA table, then
      barbican core will select the preferred CA, and pass the request
      through to that CA.
    * If no entry exists in the PrefCA table for the project, then the
      global preferred CA specified in the prefCA table will be selected.
      If no preferred CAs are defined, then the first CA plugin will be
      selected.

The plugin_ca_id will be passed to the plugin as part of the order_metadata.

Alternatives
------------

None.

Data model impact
-----------------

Four new tables will be added: CertificateAuthority (CA table),
CertificateAuthorityMetadatum (CAM table), ProjectCertificateAuthority
(PCA table) and PreferredCertificateAuthority (PrefCA table).

The CA table will have the following fields:
    * ca_id - unique identifier for the CA generated by Barbican
    * plugin_name
    * plugin_ca_id - (string) identifier for the CA generated by the plugin
      This identifier must be unique for the plugin.
    * expiration_time
    * ca_metadata (see below)

To store metadata about each CA, data will be stored in a
CertificateAuthorityMetadatum (CAM) table.  This table will consist
of the following fields:

    * ca_id - foreign key to CA table
    * key
    * value

The following key-value data will be stored to begin with:
    * name - human readable name
    * description
    * ca signing certificate
    * intermediates

A note about ids:

    The ca_id that will be presented to the client will be
    the Barbican generated ca_id.  However, because the CAs that are
    available are determined by the back-end CA, the plugin needs to provide
    a plugin_ca_id that is unique to the plugin, so that the plugin_ca_id can
    be mapped to the Barbican generated ca_id.

The PCA table lists the CA's defined for each project.  There can be
multiple entries for each project.  It will have the following fields:

    * project_id
    * ca_id (foreign key to the CA table)

The PrefCA table lists the preferred CA either globally or for any particular
project.  This is to enforce the requirement that only one CA can be defined
to be preferred either globally or pre-project.  This table will have the
following fields:

    * project_id -- required to be unique
    * ca_id -- foreign key to CA table


REST API impact
---------------

We need a new resource to expose the CA's available.  This resource will
allow the following operations:

* GET /cas - List all the available CAs.

* GET /cas/{ca_id} - Get details about CA referenced by ca_id.  This would
  include things like a HATEOAS link to the CA information, name
  and description.

* GET /cas/{ca_id}/cacert - Get the base 64 encoded signing certificate for the
  CA in PKCS7 format

* GET /cas/{ca_id}/intermediates - Get the CA cert and intermediates certs in
  base 64 encoded PKCS7 format.

* GET /cas/{ca_id}/projects - List projects which use the CA with the specified
  ca_id.  Restricted to global admins.  This is an operation that will allow
  global admins to clean up the PCA table, in case a CA is decommisioned.

* POST /cas/{ca_id}/add-to-project -  Add entry in PCA table for ca_id.  The
  project_id is determined from authz.  Restricted to project admins.
  If this is the first CA to be added to a project, it will be set as
  preferred by adding an entry for that project in the PrefCA table.

* POST /cas/{ca_id}/remove-from-project -  Remove entry in PCA table for
  ca_id.  The project_id is determined from authz.  Restricted to project
  admins.  If the CA is the preferred CA for the project and no other CAs
  exist in the PCA table, then the PrefCA entry will be removed too.  If
  other entries exist in the PCA table, then a 400 error is returned.
  ("Cannot remove a preferred CA. Select another project CA to be preferred
  first.")

* POST /cas/{ca_id}/set-preferred -  Set ca_id to be a preferred CA for the
  project.  A 400 error should be returned if a PCA entry does not already
  exist for this project_id/ca_id. Restricted to project admins.

* POST /cas/{ca_id}/set-global-preferred -  Set ca_id as a global preferred
  CA.  This CA is invoked if no ca_id is specified in a request, and no PCA
  entries exist for the project.  Restricted to global admins.

* POST /cas/{ca_id}/unset-global-preferred -  Unset ca_id as a global
  preferred CA.  Restricted to global admins.

Also, an optional metadata parameter (ca_id) will be provided in the Order
for a Certificate, and will be processed as described above.

Security impact
---------------

None.

Notifications & Audit Impact
----------------------------

None.

Other end user impact
---------------------

python-barbicanclient will need new methods to check the list of CAs and
to invoke a specific ca_id if requested.

Performance Impact
------------------

This feature requires database access to get the CA/project/plugin information
on each certificate order request, which will have an impact on both the API
and worker nodes.

Other deployer impact
---------------------

Migration scripts will need to be run on already existing deployments to add
the new tables.

Developer impact
----------------

Plugin developers should write a provides() method that returns the correct
information.  We can provide a default implementation that creates a CA table
entry based on the plugin_name.

Implementation
==============

Assignee(s)

Primary assignee:
    alee-3
    dave-mccowan

Work Items
----------

* Add new data models (PCA, PrefCA, CA, CAM tables)
* Add default provides() call to the back-end plugin contract.  Add the logic
  that populates the CA table on startup.
* Add /cas and the relevant REST API calls to retrieve/set these resources.
* Add CA selection logic to the certificate request flow.
* Add each plugin specific provides() method.

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

None
