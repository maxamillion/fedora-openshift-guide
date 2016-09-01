.. SPDX-License-Identifier:    CC-BY-SA-4.0

======================
Fedora OpenShift Guide
======================

This is meant to be an evolution of the old `Origin Bring Your Own Host`_.
document that has been retired upstream. The aim is to be a simple step by step
guide for setting up a multi host environment (cluster) of `OpenShift Origin`_
on a set of pre-existing `Fedora`_ hosts using `Ansible`_ and the `OpenShift
Ansible`_ upstream playbooks.

For a more verbose and all-encompassing document, please reference the
`OpenShift upstream advanced install guide`_ as this guide does not cover many
optional or advanced topics about OpenShift's capabilities and the methods to
deploy them.

Also below you will find a section on running your OpenShift Origin cluster as
an OpenShift Build Service (OSBS) instance.

OpenShift v3 Bring Your Own Fedora Hosts
===========================================

In this guide we will be working with a three-node cluster, these three machines
are expected to be pre-existing and have the latest stable version of Fedora
installed.

We will name these machines:

* origin-master01.example.com
* origin-node01.example.com
* origin-node02.example.com

Prep Work
---------

.. FIXME
    Need something about pre-reqs here

The basis (and more verbose version) of the example Ansible inventory we will
be working from can be found upstream `here`_.

The "trimmed down" version of it is as follows, I have three machines (this
could be all one machine or many more at your discretion).

We will save this file as ``~/inventory.txt``.

::

    # This is an example of a bring your own (byo) host inventory

    # Create an OSEv3 group that contains the masters and nodes groups
    [OSEv3:children]
    masters
    nodes
    etcd
    lb

    # Set variables common for all OSEv3 hosts
    [OSEv3:vars]
    # SSH user, this user should allow ssh based auth without requiring a
    # password. If using ssh key based auth, then the key should be managed by
    # an ssh agent.
    ansible_ssh_user=root

    # Debug level for all OpenShift components (Defaults to 2)
    debug_level=2

    # deployment type valid values are origin, online, and openshift-enterprise
    deployment_type=origin

    # Specify the generic release of OpenShift to install. This is used mainly
    # just during installation, after which we rely on the version running on
    # the first master. Works best for containerized installs where we can
    # usually use this to lookup the latest exact version of the container
    # images, which is the tag actually used to configure the cluster. For RPM
    # installations we just verify the version detected in your configured repos
    # matches this release.
    openshift_release=v1.2


    # host group for masters
    [masters]
    origin-master01.example.com openshift_node_labels="{'type': 'infra'}"

    [etcd]
    origin-master01.example.com openshift_node_labels="{'type': 'infra'}"

    [lb]
    origin-master01.example.com openshift_node_labels="{'type': 'infra'}"

    [nodes]
    origin-master01.example.com openshift_node_labels="{'type': 'infra'}" openshift_schedulable=True
    origin-node0[1:2].example.com openshift_node_labels="{'region': 'primary', 'zone': 'default'}"


.. note::
    You will also need to set the ``openshift_ip`` and ``openshift_public_ip``
    variables per host if you are deploying to a pre-existing set of machines
    that do not have the same internal and external IP (such as would be in an
    IaaS environment like OpenStack or AWS). `For reference
    <https://github.com/openshift/openshift-ansible/blob/master/roles/openshift_common/README.md>`_.

Deployment
----------

We now need to clone the `OpenShift Ansible`_ git repository.

::

    $ cd ~/

    $ git clone https://github.com/openshift/openshift-ansible.git

    $ cd ~/openshift-ansible

    $ ansible-playbook playbooks/adhoc/bootstrap-fedora.yml -i ~/inventory.txt

    $ ansible-playbook playbooks/byo/config.yml -i ~/inventory.txt


This is going to take some time, there will be a lot of ansible output.

.. FIXME
    Add missing steps and post-install here

OpenShift Build Service (OSBS)
=============================

Now that OpenShift is successfully deployed, we can deploy `OpenShift Build
Service`_ which is effectively a combination of client tooling, configuration,
and custom build types in OpenShift.

Docker Registry
---------------

If you would like to use a docker registry external to OpenShift, you will first
need to set that up on another machine. In this example we will call ours
``registry.example.com``.

.. note::
    The current generation of docker registry upstream is called
    `docker distribution`_, configuration documentation can be found `here
    <https://github.com/docker/distribution/blob/master/docs/configuration.md>`_.

::

    $ dnf -y install docker-distribution

    # Edit the configuration file if you like:
    #
    #    /etc/docker-distribution/registry/config.yml
    #

    $ systemctl start docker-distribution


.. note::
    If your registry is setup without a valid ssl certificate, you will need to
    modify the ``/etc/sysconfig/docker`` file on all OpenShift nodes to contain
    the line ``INSECURE_REGISTRY='--insecure-registry registry.example.com'``.

    If this line already exists in the configuration file then you can simply
    add ``--insecure-registry registry.example.com`` inside the parenthesis.


OSBS Deployment
---------------

We can use the `ansible-osbs-dedicated`_ to deploy OSBS on top of OpenShift,
while the upstream documentation claims that it needs `OpenShift Dedicated`_,
what is actually required is simply a pre-existing OpenShift deployment that is
dedicated to being a build system (i.e. - isn't intermingled with container app
development environments, deployments, or hosting)

We will however want to make some modifications along the way.

First git clone the repo.

::

    $ git clone https://github.com/projectatomic/ansible-osbs-dedicated.git

    $ cd ansible-osbs-dedicated

    $ cp config.yml.example config.yml

We're going to want to edit ``config.yml`` to reflect the following

.. note::
    We're removing koji and pulp content as we're not using either in this
    example.

::

    ---
    # OpenShift Dedicated namespace
    osbs_namespace: default

    # Service Accounts to create
    osbs_service_accounts:
      - koji

    # Permissions
    osbs_readonly_users: []
    osbs_readonly_groups: []
    osbs_readwrite_users:
      - system:serviceaccount:{{ osbs_namespace }}:koji
      - system:serviceaccount:{{ osbs_namespace }}:builder
    osbs_readwrite_groups:
      - system:authenticated
    osbs_admin_users: []
    osbs_admin_groups: []

    # Limit on the number of running pods - undefine or set to -1 to remove limit
    osbs_master_max_pods: -1

    # Set to true if you want to skip importing secrets in case the secret files
    # are not found.
    osbs_secret_can_fail: true

Also going to want to remove the unnecessary from deploy.yml, the result should
be:

::

    ---
    - name: users and permissions
      hosts: masters
      vars_files:
      - hardcoded-vars.yml
      - config.yml
      roles:
      - osbs-master

Now run the scripts provided, but using out inventory:

::

    $ ./update-roles.sh

    $ ./deploy.sh -i ~/inventory.txt


We'll need to configure the ``buildroot`` container for the builds which
requires `atomic-reactor`_.

::

    # Install atomic-reactor on all nodes
    $ ansible nodes -m dnf -a "pkg=atomic-reactor state=installed" -i ~/inventory.txt

    # Create the buildroot container on all nodes
    $ ansible nodes -m shell -a "docker build --no-cache --rm -t buildroot /usr/share/atomic-reactor/images/dockerhost-builder/" -i ~/inventory.txt


OSBS Client
-----------



Licensing
=========

To make licensing easier, license headers in the source files will be
a single line reference to Unique License Identifiers as defined by
the `Linux Foundation's SPDX project`_.

For example, in a source file the full "GPL v2.0 or later" header text will be
replaced by a single line:

::

    SPDX-License-Identifier:    GPL-2.0+

Or alternatively, in a source file the full "CC-BY-SA-4.0" header text will be
replaced by a single line:

::

    SPDX-License-Identifier:    CC-BY-SA-4.0

the license terms of all files in the source tree should be defined by such
License Identifiers; in no case a file can contain more than one such License
Identifier list.

If a ``SPDX-License-Identifier:`` line references more than one Unique License
Identifier, then this means that the respective file can be used under the
terms of either of these licenses, i. e. with

::

    SPDX-License-Identifier:    GPL-2.0+    LGPL-2.1+

All SPDX Unique License Identifiers available at `spdx.org`_.

.. _Fedora: https://getfedora.org
.. _spdx.org: http://spdx.org/licenses
.. _OpenShift Origin: https://openshift.org
.. _Linux Foundation's SPDX project: http://spdx.org
.. _OpenShift Dedicated: https://www.openshift.com/dedicated/
.. _docker distribution: https://github.com/docker/distribution/
.. _OpenShift Ansible: https://github.com/openshift/openshift-ansible
.. _OpenShift Build Service: https://github.com/projectatomic/osbs-client
.. _ansible-osbs-dedicated:
    https://github.com/projectatomic/ansible-osbs-dedicated
.. _here:
    https://github.com/openshift/openshift-ansible/blob/master/inventory/byo/hosts.origin.example
.. _Origin Bring Your Own Host:
    https://github.com/openshift/openshift-ansible/blob/1bc6b51585c23670fdc08a1df6a89d35cd0b8149/README_origin.md
.. _OpenShift upstream advanced install guide:
    https://docs.openshift.org/latest/install_config/install/advanced_install.html#install-config-install-advanced-install
