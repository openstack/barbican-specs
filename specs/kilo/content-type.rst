=============================================
Define Content Types for API and Secret Store
=============================================

The current implementation of Barbican is ambiguous to end users and developers
as to the encoding that should be used to input new secrets and the encoding of
secrets that are returned. This spec aims to define the encoding specifications
for each of the secret types and define all of the secret types. The encoding
specifications will allow end users and developers to know how to handle
the secrets.

Problem Description
===================

The biggest problem that currently exists is that data returned from the API
and SecretStores is not clearly defined. An example would be public keys. When
an end user requests a public key there were many options for how to return
that data. One was to return just the public key (not the certificate) as a
Base64 encoded string, but the question still remained of what was the format
of the binary representing the key. However, most interpreted this to mean
return a certificate. Then the question was raised as to how to return the
certificate. Should it be PEM or DER format. Clearly there were lots of
assumptions and unanswered questions.

There are two problems above. One is that we need to clearly define all of the
possible secret types. This will eliminate the public key versus certificate
issue. The other problem is that we need to define the encoding standard
options for each type of secret. Then end users and SecretStore developers will
know that if they have a secret of type X then it has an encoding of type Y.

The scope of this blueprint is limited to unencrypted secrets. In particular
this spec does not cover the encoding standards for keys wrapped with a
transport key nor does it cover the encoding scheme for private keys that are
encrypted with a passphrase.

Proposed Change
===============

Define All Secret Types
-----------------------

The first change will be to modify the SecretType class to include all of the
possible secret types. The SecretType class represents an enum, and the
currently defined values are symmetric, public, and private.

This spec proposes adding the types certificate, passphrase, and opaque data.
The certificate type would represent X.509 certificates. The current spec would
not support PGP certificates, but mostly because I do not know much about them.
If support is needed then we can work through that. More than likely I think
that if certificates other than X509 are to be supported then a subtype enum
for certificates should be created to indicate the type, X.509, PGP, etc. That
would be for another spec, and this one will assume only X.509 certificates.

The opaque data type would represent an unknown blob of data. This is reserved
for types that are not keys or certificates. Barbican will not be able to
understand any of the qualities about the data. It will simply be a blob.
This is the type that would be used for secret data like IVs.

Here is a listing of the proposed secret types

* symmetric key
* public key
* private key
* passphrase
* certificate
* opaque data

Content Encoding for Each Secret Type
-------------------------------------

The second change will be to define the content encoding for each of the secret
types. The types and proposed encodings are below.

* symmetric key - Base64 encoding of byte array in network byte order
* public key - Base64 encoding of DER-encoded SubjectPublicKeyInfo structure as
  defined by X.509 RFC 5280 with PEM header and footer
* private key - Base64 encoding of PKCS#8 structures with PEM header and footer
* passphrase - text with utf-8 encoding
* certificate - Base64 encoding of DER-encoded ASN.1 X.509 structure with PEM
  header and footer
* opaque data - Base64 encoding of byte array in network byte order

The formats for the public and private keys were chosen based on their
popularity. The public key format of SubjectPublicKeyInfo was chosen because
that is the default format for public keys by openssl and Java. Plus it is a
part of the common X.509 standard.

The private key format of PKCS#8 is the default encoding for Java. The default
encoding for generating new private keys in openssl is not PKCS#8. The default
encoding in openssl is referred to as the "traditional" format in openssl
documentation. The term traditional is ambiguous. It is in fact the plain
private key structure, so for RSA it is the private key structure as defined in
PKCS#1. This is a subset of the PKCS#8 structure.

The openssl library does support using PKCS#8 keys, and this does not require
changing any command line options. The library automatically detects the
encoding and uses the key if it is in PKCS#8 encoding or openssl "traditional"
encoding. Because of this and the ambiguity of the term "traditional" I
recommend using the PKCS#8 format. It is clear to developers and users of the
expected format.

Alternatives
------------

One alternative would be to leave everything as is. The downsides of this
approach are identified above.

There may be other ways to represent the keys. We could define our own
structures instead of using ASN.1. I chose to use ASN.1 and other standards
because I think they are widely adopted and accepted. I think exporting keys
from existing applications makes the choices the best option.

Data Model Impact
-----------------

The only expected change is to modify the Secret entity to include a type
column. The type column will be one of the predefined secret types of symmetric
key, public key, private key, passphrase, certificate, or opaque data.

The secret stores currently expect an array of bytes that are Base64 encoded,
so I do not anticipate changing anything with the secret store classes. The
only difference now is that secret stores will have more insight as to the
format of those bytes.

REST API Impact
---------------

The API for storing keys requires setting the payload_content_type and
payload_content_encoding for the secret. For each type of secret there needs to
a set of acceptable payload_content_type and payload_content_encoding values
that are acceptable.

This also means that the secret type should be supplied as a new parameter.
This was something that was missing before. The current implementation can only
infer the secret type for symmetric keys that are uploaded because it can use
the algorithm to determine that. The algorithm is not enough to infer the
secret type for asymmetric keys since it could be a public or private key.
Therefore a secret type should be provided. The current implementation
currently stores them as unknown.

The table below lists the acceptable content types and content encoding values
for each type of secret. Each pair is represented as (payload_content_type,
payload_content_encoding). API calls that do not receive data in the specified
encoding will receive a 406 error code response.

* symmetric key (application/octet-stream, Base64)
* public key (application/octet-stream, Base64)
* private key (application/pkcs8, Base64)
* passphrase (text/plain, utf-8)
* certificate (application/pkix-cert, Base64)
* opaque data (application/octet-stream, Base64), (text/plain, NULL)

Supporting Backwords Compatability
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The new parameter for secret type will be an optional parameter to support
backwards compatability. If it is not supplied then the key type opaque data
will be used. Opaque data will be the only secret type to support multiple
acceptable content types. This was chosen to support backwards compatability.

Secret Store Impact
~~~~~~~~~~~~~~~~~~~

The API content type and content encodings are defined above. However, the
format of keys passed to the secret stores needs to be defined as well. The
keys are passed from Barbican Core to the secret store using the SecretDTO
object.::


 +-------------+       +-------------+
 | SecretType  |       |  KeySpec    |
 +-------------+       +-------------+
 | SYMMETRIC   |       | algorithm   |
 | PUBLIC      |       | bit_length  |
 | PRIVATE     |       | passphrase  |
 | CERTIFICATE |       +-------------+
 | OPAQUE      |        ^
 +-------------+        |
            ^           | has
            | has       |
            |           |
         +---------------+
         | SecretDTO     |
         +---------------+
         | type          |
         | secret        |
         | key_spec      |
         | content_type  |
         | transport_key |
         +---------------+

The current SecretDTO has a content_type property and a secret property. The
secret property is a Python string that is the Base64 encoding of the bytes
that represent the secret. The content_type is the content_type parameter
received from the API call. Since we are defining a single encoding for each
SecretType of a SecretDTO then this parameter can be removed. The encodings
will be the same as that of the API except that opaque data will be as a Base64
encoding of the bytes and not allowed to be text/plain.

Security impact
---------------

None

Notifications & Audit Impact
----------------------------

The ability to audit the types of keys that are managed by Barbican could be
added.

Other End User Impact
---------------------

We can validate secrets before they are stored to validate the structures are
correct. This will help users by preventing them from uploading secrets of a
specific type that are not formatted correctly.

Performance Impact
------------------

None, unless validation is done in which case there will be minor overhead to
validate the secret structures.

Other deployer impact
---------------------

None

Developer impact
----------------

Secret store developers will need to possibly modify their code to return
secrets in the specified format.

The Barbican client will need to be updated to utilize the encodings and add
the secret type parameter.

Implementation
==============

Assignee(s)
-----------

Nathan Reller (rellerreller)

Work Items
----------

1. Add secret type

Modify the API to accept a new secret_type parameter. The secret_type must be
one of the predefined types, so a validator should be added for that. If not
provided then use opaque data unless symmetric algorithm is provided.

Modify barbican.plugin.resources to create a SecretDTO with the appropriate
secret type.

2. Add basic secret type validators

Add some basic validators that ensure the secrets passed to the API are
formatted correctly. This first check would simply be to validate PEM headers.
For example, when passing an RSA key this would check that the proper PEM
header is included.

This step will _not_ do a deep inspection of the secrets passed to Barbican. A
deep inspection would validate each of the structures to make sure the correct
structures are received. For RSA public keys this would check that it contains
a modulus and public exponent.

3. Update API documentation

Update the documentation to include notes on the expected encodings and include
documentation on the new secret type argument for posting secrets.

4. Update barbicanclient

Update barbicanclient to take advantage of the new secret types and comply with
the API changes.

Dependencies
============

None

Testing
=======

Add validator tests to verify the API is correctly parsing the new secret type
and catching the instances where the incorrect encoding is used for keys passed
to the API.

Documentation Impact
====================

Update the API documentation to include information on the new secret type API
parameter and the encodings for each of the secret types.

References
==========

PKCS#8 -https://tools.ietf.org/html/rfc5958
X.509 - https://tools.ietf.org/html/rfc5280
Java Key - http://docs.oracle.com/javase/7/docs/api/java/security/Key.html
openssl to pkcs8 - https://www.openssl.org/docs/apps/pkcs8.html
