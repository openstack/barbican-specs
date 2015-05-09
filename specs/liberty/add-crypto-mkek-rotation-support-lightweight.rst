..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

================================================================
Add Crypto/HSM MKEK Rotation and Migration Support (Lightweight)
================================================================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/barbican/+spec/add-crypto-mkek-rotation-support-lightweight

Currently Barbican has no means to migrate secrets encrypted with a
crypto/HSM-style plugin to a new master key encryption key (MKEK) and its
associated wrapped project KEKs. This blueprint proposes adding a new Barbican
utility that supports completing the rotation process by rewrapping the project
KEKs with the new MKEK. Note that unlike the similarly-named blueprint at [1],
this blueprint does *not* call for re-encrypting secrets and is therefore a
'lightweight' alternative to that blueprint.

Comparing the two approaches, this lightweight approach just rotates
the MEKs and rewraps the project KEKs, leaving each secret's encrypted data
unchanged. The other blueprint rotates the MKEKs, project KEKs and the
encrypted secret information.

Thus this blueprint has a less thorough rotation process for secrets which
could increase the chances of decrypting the secret's encrypted data. The
likelihood of such an attack is expected to be small however. After all, MKEKs
are infrequently rotated due to their low probability of being guessed or
compromised. This blueprint's less thorough approach is expected to be less
process intensive and execute faster when there are many secrets stored in the
database.

Similar to the other blueprint, this utility would be started after deployers,
out of band:

1. generate new MKEK and HMAC signing keys with a binding to new labels, and
   then

2. Replicate these keys to other HSMs that may be in the high availability (HA)
   group, and then

3. Update Barbican's config file to reference these new labels, and finally

4. Restart the Barbican nodes.

The proposed utility would then rewrap the project KEKs with the new MKEKs,
updating the associated project KEK records with the new wrapped project KEKs.
Note that only HSM/crypto-style plugins rotations are proposed for this
blueprint.


Problem Description
===================

When a secret is stored in Barbican using the crypto-style plugin, a
`KEKDatum` entity (from `barbican.model.models`) is retrieved for the secret's
project. If no such entities are found for this project, then the following
steps occur:

1. A new `KEKDatum` entity is created for the project and the specific crypto-
   style plugin with status of `ACTIVE`. The `ACTIVE` status indicates that
   this entity should be used for secret encryptions from then on.

2. The `bind_kek_metadata()` method is invoked on the crypto plugin. The plugin
   then creates a project-level KEK that will be used to encrypt new secrets
   for that project. For the PKCS11 HSM plugin, this is the project KEK that is
   wrapped/encrypted by the MKEK.

3. Information about this project-level KEK is then added to `plugin_meta`
   attribute of the new `KEKDatum` entity.

4. When a secret is finally stored, it has a `Secret` entity created for it to
   hold metadata, and also an associated `EncryptedDatum` entity that holds
   both the encrypted cypher text for the secret, and a reference to the
   project-level `KEKDatum` record.

5. Thereafter, new secrets that need to be encrypted for this same project
   will used this `KEKDatum` entity, with no further attempts made to generate
   a new entity.

Hence the `KEKDatum` entity is associated with project-level KEKs, which in
turn are associated with a single MKEK in the HSM. As rotation involves
creating a new MKEK (but not the project KEKs for this lightweight version),
the `KEKDatum` entity needs to be updated per project to rewrap/encrypt the
project KEK it contains.

Unlike the more stringent key rotation blueprint, no other steps are needed.
The secrets, there UUIDs and their associated encrypted data, do not need to
change at all, which should speed the overall migration process.

A requirement is that this utility be resilient, allowing for the utility to
be re-run if it fails midway.

This blueprint details an approach to rewrap existing project KEKs with a
new MKEK.


Proposed Change
===============

This blueprint proposes the following steps be taken to complete the KEK
rotation and migration effort:

1. Out of band to Barbican (so by deployers), new MKEK and HMAC keys are
   created in the HSM, with new unique labels. For high availability
   configurations such keys must be replicated across all HSMs in the cluster.

2. For each API and worker node in the network, update their
   `/etc/barbican/barbican-api.conf` files' `mkek_label` and `hmac_label`
   attributes with the new unique labels generated above. Note that once
   Barbican is restarted, secrets added to new project-IDs will start to use
   the new MKEK to wrap their new project KEKs. Existing secrets will still
   utilize the old MKEK though hence the need for the new utility.

3. Restart the API and worker nodes. Barbican should be running normally at
   this point, with the following utility-based steps occurring while Barbican
   operates.

4. Via a new utility proposed in this blueprint, query for all `KEKDatum`
   entities for a specified crypto-style plugin class
   (e.g. `barbican.plugin.crypto.simple_crypto.SimpleCryptoPlugin`). These
   wrapped-project KEKs need to be rewrapped with the new MKEK.

5. For each `KEKDatum` entity, invoke a new method on the crypto-style
   plugin contract (defined in `barbican.plugin.crypto.crypto.py`) called
   'rewrap_project_kek()', which takes the entity as input and updates it with
   the rewrapped project KEK. The plugin would need to load the wrapped
   project KEK into the HSM, decrypt it with the old MKEK, encrypt it with the
   new MKEK, and then return the new wrapped project KEK (but still containing
   the original project KEK).

6. The `KEKDatum` entity above would then be updated with the new wrapped
   project KEK information. Since this is an in-place update, the secrets
   associated with this entity do not have to be updated at all, unlike [1].

7. Info log the UUID of the migrated `KEKDatum` entity to produce a record of
   the migration.

The proposed utility would be implemented as a Python `barbican.cmd` script
and added as a `pbr` entry point in the `setup.cfg` file, and hence available
as a callable command once Barbican is deployed.

To preserve data integrity steps 5 and 6 should be performed in a database
transaction.

Alternatives
------------

See the more heavy-weight alternative approach to key rotation at [1].

Data model impact
-----------------

No data model or repository changes are needed for the proposed solution. Data
migrations will be required as detailed in the Proposed Change section above.

REST API impact
---------------

None.

Security impact
---------------

The proposed solution completes necessary key rotation processes by rewrapping
project KEKs used to encrypt secrets. There are improbable but possible data
loss risks with the proposed approach however, since project KEKs encrypted
with old MKEKs are replaced with these same project KEKs encrypted with the new
MKEKs. If the database update transaction fails and corrupts this record (very
unlikely with ACID compliant databases, but possible), decryption would fail
for all secrets derived from the failed project KEK. However, since these
wrapped project KEKs do not change often (on the order of the MKEK rotation
schedule) recovery from database backups is very likely, thus mitigating this
risk.

Also, if the MKEK is compromised *and* if the attacker has access to the
database backups for secrets they can then decrypt them by first unwrapping
the project KEKs. Deployers should be mindful of this and securely store
backups.

Notifications & Audit Impact
----------------------------

A log of each `KEKDatum` entity migrated is produced by the proposed utility,
which could be used to prove to auditors that a migration/rotation occurred.

Other end user impact
---------------------

None.

Performance Impact
------------------

The proposed utility could take significant time to process if there are many
project-IDs to migrate, and thus it would represent a load on the HSM to
re-wrap the project KEKs, impacting normal Barbican operations. This utility
would be called quite infrequently however (maybe 1 to 4 times a year). Since
there are fewer project IDs/KEKs then secrets, this utility would be more
performant than the 'all secrets' migration proposed in [1].

Other deployer impact
---------------------

The proposed utility would be executed as a new executable command available
after deployment Barbican. The proposed changes would not require updates to
the configuration file schema, but would require updates to provide new
MKEK and HMAC key labels as detailed above.

Also, if the MKEK is compromised *and* if the attacker has access to the
database backups for secrets they can then decrypt them by first unwrapping
the project KEKs. Deployers should be mindful of this and securely store
backups.

Developer impact
----------------

None.


Implementation
==============

Assignee(s)
-----------

TBD

Work Items
----------

The proposed work items are:

1. Create a Python command script to implement the steps in the Proposed
   Change section.

2. Add unit testing to cover the new code lines.

3. Add integration unit test that, using the default `simple_crypto.py` plugin
   and an in-memory SQLite database, first creates a few known secrets with the
   default MEK in the config file, and then modifies this MEK, and then
   executes the migration script logic. The secrets should be decrypted and
   verified for accuracy. The `KEKDatum` entities should have their
   `updated_at` dates updated. The secrets should again be decrypted and
   verified for accuracy, which proves the migration of all secrets to the
   updated `KEKDatum` record occurred successfully.

4. Document the overall key rotation and migration process, including the usage
   of the new migration utility.


Dependencies
============

None.


Testing
=======

No DevStack functional tests are expected at this time. Once an HSM gate job is
added, a future blueprint add tests will be added.


Documentation Impact
====================

The last work item details the required documentation.


References
==========

[1] https://blueprints.launchpad.net/barbican/+spec/add-crypto-mkek-rotation-support
