..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Refactor Client Entity Models
==========================================

https://blueprints.launchpad.net/python-barbicanclient/+spec/client-refactor-models

The current Entity Models in the client are a bit awkward to use.  This
blueprint proposes refactoring some functionality to make the API more
usable and consistent.

Problem Description
===================

The Entity Models in ``barbicanclient`` should be refactored to provide a
more Pythonic api.  This refactor will make the existing Entities consistent
with the recently approved Containers blueprint. [1]_

.. [1] http://specs.openstack.org/openstack/barbican-specs/specs/juno/client-add-containers.html

Proposed Change
===============

Refactor existing Models to provide methods for actions that affect a single
entity inside the entity class.  This will allow for worflows that only affect
a single entity to be completed without the need for a reference to the
corresponding ``EntityManager`` subclass.

The ``Secret`` entity should be refactored to add a ``store()`` method and
a ``payload`` property::

    from barbicanclient import client

    # Set up client connection
    connection = client.Client(tenant_id="1", endpoint=ENDPOINT, insecure=True)

    # Create a new Secret
    my_secret = connection.secrets.Secret(name="My secret name",
                                          payload="the secret sauce")
    my_secret.store()

    # Alternatively set Secret properties instead of passing args
    my_secret = connection.secrets.Secret()
    my_secret.name = "My secret name"
    my_secret.payload = "the secret sauce"
    my_secret.store()

Similarly, ``Orders`` should allow both args to the constructor as well as
setting properties directly.  We should also add a ``submit()`` method to
submit the order to the API::

    from barbicanclient import client

    # Set up client connection
    connection = client.Client(tenant_id="1", endpoint=ENDPOINT, insecure=True)

    # Create and submit a new Order
    my_order = connection.orders.Order(
        name="My Order",
        payload_content_type="application/octet-stream"
        algorithm="AES",
        mode="CBC",
        bit_length=256,
        expiration=None
    )
    my_order.submit()

    # Alternatively set the Order properties instead of passing args
    my_order = connection.orders.Order()
    my_order.name = "My Order"
    my_order.payload_content_type = "application/octet-stream"
    my_order.algorithm = "AES"
    my_order.mode = "CBC"
    my_order.bit_length = 256
    my_order.expiration = None
    my_order.submit()

Listing entities should still be handled via the corresponding
``EntityManager``.  The ability to decrypt a secret, however, should be moved
to the ``Secret`` class, and removed from the ``SecretManager``.

Retrieving entities should be moved from the ``EntityManager`` (replacing
the ``get()`` function) to the ``Entity`` constructor. For example::

    my_secret = connection.secrets.Secret(secret_ref=SECRET_REF)
    my_order = connection.orders.Order(order_ref=ORDER_REF)

Deleting entities can either be done with the existing ``EntityManager``
``delete(entity_ref)`` or with a new ``Entity`` function, ``delete()``.
An example using a Secret::

    # New method
    my_secret = connection.secrets.Secret(secret_ref=SECRET_REF)
    my_secret.delete()

    # Old way still works
    connection.secrets.delete(secret_ref=SECRET_REF)


Alternatives
------------

We could continue to use the objects as they currently exist.

Also note that the Orders functionality will need to be revisited once
the Typed Orders implementation lands. [2]_

.. [2] http://specs.openstack.org/openstack/barbican-specs/specs/juno/api-orders-add-more-types.html

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

This change will require rewriting how Secret objects are consumed, and will
require a new major version for the client library.

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

Blueprint Draft: Douglas Mendizábal (redrobot)
Implementation: Adam Harwell (rm_work)

Work Items
----------

* Refactor Secret entity
* Refactor Order entity

Dependencies
============

None

Testing
=======

Testing should be consistent with existing testing in the library.

Documentation Impact
====================

Common workflows will have to be updated to give examples on how to use
the refactored classes.

References
==========

Containers in the Client etherpad: https://etherpad.openstack.org/p/python-barbicanclient-containers
Containers Blueprint: http://specs.openstack.org/openstack/barbican-specs/specs/juno/client-add-containers.html
