..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==============================
Configuration Language Support
==============================

https://blueprints.launchpad.net/murano/+spec/conf-language-support


Problem description
===================

There is a huge community of applications (opscode, puppet-labs) where
deployment installation instructions are specified in configuration
languages such as Puppet or Chef. In order to reuse these applications,
adaptors like Chef or Puppet are required on the VM side. Both chef and
puppet recipes will not be managed by centralized server (chef-server,
puppet-master), but they will use the standalone version, specifically
the usage of chef-solo and puppet apply.


Proposed change
===============

Inclusion of new executors in the murano-agent project. These executors
will be objects to be used by murano-agent. Specifically, two executors
will be implemented: Puppet and Chef. Both executors will be in charge of:

* Obtaining the required modules or cookbooks in the virtual machine.

This task can be done by passing the information from murano-engine to
murano-agent, obtaining the information from the package itself. It requires
the user to upload the package information plus the cookbooks to be used.
The second option implies the cookbooks are downloaded in the virtual
machine, so that they only need the URL to be accessible.

* Generating the required files for the configuration language.

For instance, manifests and hiera data for puppet, and, node
specifications for chef from the information stored in the
execution plan.

* Executing the chef-solo or puppet-apply process.


Previously, some work has to be done to install Chef or Puppet inside the VM.
This task can be done by using cloud-init from the murano engine. The
following is an example on how these executors work:

::

    ## YAML Template.
    ---
    FormatVersion: 2.0.0
    Version: 1.0.0
    Name: Deploy Tomcat
    Parameters:
        port: $port
    Body: |
        return deployTomcat(port=args.port).stdout
    Scripts:
        deployTomcat:
            Type: Chef
            Version: 1.0.0
            EntryPoint: mycoockbook::myrecipe
            Files:
                tomcat:
                    Name: tomcat
                    URL: git://github.com/opscode-cookbooks/tomcat.git
                    Type: Downloadable
                java:
                    Name: java
                    URL: git://github.com/opscode-cookbooks/java.git
                    Type: Downloadable
                ssl:
                    Name: openssl
                    URL: https://github.com/opscode-cookbooks/ssl.git
                    Type: Downloadable
    Options:
        captureStdout: true
        captureStderr: true

In this case, a new script Type appears (instead of Application). It is
Chef type, which will execute the Chef executor. The same happens with
the Puppet Type. In addition, the EntryPoint contains the information
about the cookbook and the recipe to be installed. The Files section
is used for the cookbooks and its dependence information. The cookbooks
properties are in the Parameter section.


All the required steps to be part of the executor can be summarized as follows.

For Chef,

#. Creating the node.json with the recipes and the configuration parameters::

    {
        orion::ports: 1026
        orion::dbname: oriondb
        "run_list": [
            "recipe[orion::0.13.0_install]"
        ]
    }

#. Executing chef-solo:
    chef-solo -j node.json


For puppet,

#. Generating the manifest (site.pp)::

    node 'default'
    {
        class{
            'orion::install':
        }
    }

#. Creating the hiera data information: hiera.yaml::

    ## YAML Template.
    ---
    orion::port: 1026
    orion::dbname: oriondb

#. Executing:
    puppet apply --hiera_config=hiera.yaml --modulepath=/opt/puppet/modules/orion  site.pp


Alternatives
------------

None

Data model impact
-----------------

None


REST API impact
---------------

None

Versioning impact
-------------------------

None

Other end user impact
---------------------

None

Deployer impact
---------------

The solution proposed is valid for any VM which contains the configuration
language implementation already installed. There are event chef-solo and
puppet agents for Windows.

Developer impact
----------------

None

Murano-dashboard / Horizon impact
---------------------------------

None


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  hmunfru

Other contributors:
  jesuspg

Work Items
----------

#. Generate Chef executor
#. Generate Puppet executor
#. Work on configuration


Dependencies
============

None


Testing
=======

Integration tests will be done


Documentation Impact
====================

Information about how to defines application for Puppet and Chef will have
to be documented, explaining the different fields.


References
==========

* http://es.slideshare.net/hmunfru/fiware-and-murano-support-for-configuration-languages
* https://etherpad.openstack.org/p/conf-language-support-spec


