..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================================
Restructure project to better accommodate all plugin types
==========================================================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/barbican/+spec/restructure-for-plugins

The current project structure only naturally accommodates HSM-style plugins in
the 'crypto' package. This blueprint accommodates all the plugin types
required by Barbican by reorganizing the project structure under a 'plugin'
package.


Problem description
===================

The current project structure places the only plugin currently supported by
Barbican (used to interface with HSMs) into a 'crypto' package. Planning for
new Barbican features has revealed the need for new types of plugins, that are
either awkward to implement with the HSM contract (such as Dogtag or KMIP) or
else are entirely different types of plugins. The latter includes SSL
certificate workflow processing plugins, and eventing plugins. These new
plugins do not fit well into the current 'crypto' package, and hence Barbican
has outgrown it.


Proposed change
===============

This blueprint replaces the current 'crypto' package with the following
package structure:

common/
    resources.py  - API and Worker processes call into this module to
                    generate, store or get secrets.

    plugin_managers.py - Contains base-level stevedore plugin lookup logic as
                         needed. Extended by interfaces in plugin package.

plugin/

    interface/   -  Stores Plugin contracts used by other Barbican packages
                     (as abc abstracts). Also has stevedore lookup methods.

        secret_store.py -  Nate's current data storage plugin, that handles
                           securely storing/retrieving secrets (such as Dogtag
                           and KMIP). Would also include a class called
                           SecretStorePluginManager that uses stevedore to look
                           up Nate's secret store plugin implementations.

        certificates.py -  SSL certificate workflow plugins. Would also include
                           a class called CertGenerationPluginManager that uses
                           stevedore to look up certificate generation workflow
                           plugin implementations.

    crypto/             -  Similar to the current 'crypto' package. Stores
                           HSM and security-module related modules. Because
                           this is an 'inner' interface not intended to be
                           called from outside the plugin package, these
                           modules are segregated from the 'interface' package
                           above.

        crypto.py        - Key and secret generation plugin for HSM-style
                           interfaces. Similar to the current
                           crypto/plugin.py::CryptoPluginBase interface. Would
                           also include a class called
                           SecretGenerationPluginManager that uses stevedore to
                           look up the symmetric/asymmetric key generation
                           plugins (the so called 'second level plugin
                           lookup'). This logic is very similar to the current
                           crypto/extension_manager.py.

        p11_crypto.py    - Paul's HSM plugin implementation.

        simple_crypto.py - Simple implementation of security-module plugin.

    store_crypto.py      - A secret_store.py implementation that adapts between
                           the higher-level secret_store.py interface and the
                           lower-level crypto.py interface. Would utilize
                           SecretGenerationPluginManager to locate the 'second
                           level' HSM-style plugin implementation.

    dogtag.py     -  Dogtag implementation of the secret_store.py interface.

    kmip.py       -  A KMIP implementation of the secret_store.py interface.

    symantec.py   -  A Symantec implementation of the certificates.py
                     interface.


This effort first introduces the new structure only. Then existing plugins are
moved over to the new structure, verifying unit tests still pass.

To provide concrete context to this new structure, the following sequence
provides an example flow to store a secret after this refactoring effort:

1. A call is made to POST /secrets (using the one-step method)

2. The API code in barbican.api.controllers.secrets.py makes a call to...

3. Common code in barbican.common.resources.py that has a line such as this:

::

        storer = SecretStorePluginManager.getPlugin(
                      plugin_managers.STORE_SECRET, meta_data)
        storer.store_secret(...)


4. SecretStorePluginManager will look at the config file and see which plugin
is available. Initially it will be one of the implementations defined in
plugin.dogtag, plugin.kmip, or plugin.store_crypto.

5. If the plugin is dogtag or kmip, then store_secret() will implemented by the
plugin code directly.

6. If the plugin is the store_crypto adapter, then store_secret() will in turn
call common.plugin_manager.py to find a crypto/security-module type of plugin
(i.e. a second-stage stevedore lookup). Initially this will be either
plugin/crypto/p11_crypto.py or simple_crypto.py, and the encrypt() method will
be called on it as done now.

7. The plugin returns, updates meta_data, creates the secret record and returns
the url to the user.


Alternatives
------------

More detailed discussion occurred on this etherpad:
https://etherpad.openstack.org/p/extension-manager

An original approach discussed elevating the interfaces to top level Barbican
packages. However, that would place them at the same level as the 'common',
'api' and 'task' packages (for example) which would make the plugin source
code locations harder to locate. Hence it seemed that centralizing all plugins
under a clear 'plugin' package would provide this proper delineation relative
to the rest of the source code. Under the 'plugin' package, the structure is
designed to organize interface, default and external modules.

Data model impact
-----------------

None.


REST API impact
---------------

None.


Security impact
---------------

None.


Notifications impact
--------------------

None.


Other end user impact
---------------------

None.


Performance Impact
------------------

None.


Other deployer impact
---------------------

None.


Developer impact
----------------

This first phase of this change will not be impactful as it sets up the new
folder structure only. Once existing plugin implementations are moved to the
new structure, pending CRs will be impacted as the file structure has changed.
We should be able to sequence this change with our contributors however.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  john-wood-w

Other contributors:
  alee-3
  rellerreller


Work Items
----------

CR #1 - Add new structure in parallel to current structure, should not impact
        current code base.
CR #2 - Move existing plugins over to the new structure, including updating
        api/ and tasks/ modules to refer to the new locations, and then
        verifying unit tests still pass.


Dependencies
============

None.


Testing
=======

Current unit testing will need to be changed to reference the new location of
items in the revised project structure.


Documentation Impact
====================

Update this wiki section, and the sections thereafter it:
https://github.com/cloudkeep/barbican/wiki/Developer-Guide-for-Contributors#detailed-explanation


References
==========

More detailed discussion occurred on this etherpad:
https://etherpad.openstack.org/p/extension-manager
