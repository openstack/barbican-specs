..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=======================================
Add containers to python-barbicanclient
=======================================

Launchpad blueprint:

https://blueprints.launchpad.net/python-barbicanclient/+spec/client-add-containers

The Secret Containers resource is now available in Barbican, so it should
also be added to the client.

Problem Description
===================

We need to be able to consume and create Container resources in the Python
library using Python objects to represent the containers.

Proposed Change
===============

Add new Classes to the client to represent the different types of Containers
available in Barbican in a new ``containers`` module::

    barbicanclient.containers.Container
    barbicanclient.containers.CertificateContainer
    barbicanclient.containers.RSAContainer

The classes should be usable thusly::

    from barbicanclient.containers import Container, RSAContainer

    # New Generic Container
    my_container = Container()
    my_container.add("Secret Name in Container", Secret(...))
    my_container.save()

    # New RSA Container
    my_rsa_container = RSAContainer()
    my_rsa_container.public_key = Secret(...)
    my_rsa_container.private_key = Secret(...)
    my_rsa_container.private_key_passphrase = Secret(...) #optional
    my_rsa_container.save()

    # RSAContainer should be subclassed from Container
    my_rsa_container.add("public_key", Secret(...)) # should work
    my_rsa_container.add("unsupported_secret_name", Secret(...)) # should raise exception

    # New Certificate Container
    my_cert_container = CertificateContainer()
    my_cert_container.certificate = Secret(...)
    my_cert_container.private_key = Secret(...)
    my_cert_container.private_key_passphrase = Secret(...)
    my_cert_container.intermediates = Secret(...)
    my_cert_container.save()

    # CertificateContainer should be subclassed from Container
    my_cert_container.add('certificate', Secret(...)) # should work
    my_cert_container.add('unsupported_secret_name', Secret(...)) # should raise exception

Objects should be lazy, so requests should not be sent until the ``save()``
method is called.  The ``save()`` method should also recursively store secrets
if they have not yet been stored to the Barbican instance.

Once the ``save()`` method is called for the first time the Container becomes
immutable, subsequent calls to ``add()`` or ``save()`` should raise exceptions.
Basically, any action that would modify the container, except for ``delete()``,
should raise an exception.

Retrieving existing containers should be done via a ``ContainerManager`` accessible
from the client object::

    from barbicanclient import client

    barbican = client.Client(...)
    my_container = barbican.containers.get('https://ref_to_container/UUID')

The ``get()`` method should return an instance of the appropriate Container
class. Listing existing methods should be done via the ``ContainerManager``
as well::

    my_containers = barbican.containers.list(limit=10, offset=0)
    isinstance(my_containers, list) # should return True

Container deletion should be done on either the Container object or using
the ContainerManger::

    my_container = barbican.containers.get('https://ref_to_container/UUID')
    my_container.delete()
    # or
    barbican.containers.delete('https://ref_to_container/UUID')

Alternatives
------------

The proposed functionality above pushes most of the actions to the
Container objects, while keeping the ContainerManager functionality somewhat
light.  As an alternative, the ContanerManager could perform all of the
functionality, while making the Container objects simple.

I think container creation in a single method could get ugly really fast,
so I would prefer to avoid it.

Data model impact
-----------------

None

REST API impact
---------------

None

Security impact
---------------

None

Notifications & Audit Impact
----------------------------

Logging should be done in a manner consistent with the rest of the library.

Other end user impact
---------------------

None as this is new functionality.

Performance Impact
------------------

None

Other deployer impact
---------------------

None

Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Blueprint Draft: dougmendizabal
Implementation: TBD

Work Items
----------

* Implement the new ``containers`` module.

Dependencies
============

None

Testing
=======

Testing should be consistent with existing testing in the library.

Documentation Impact
====================

All new Container functionality needs to be documented.

References
==========

Containers in the Client etherpad: https://etherpad.openstack.org/p/python-barbicanclient-containers
