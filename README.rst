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

    # NOTE: Currently we require that masters be part of the SDN which requires
    # that they also be nodes. However, in order to ensure that your masters are
    # not burdened with running pods you should make them unschedulable by
    # adding openshift_schedulable=False any node that's also a master.
    [nodes]
    origin-master01.example.com openshift_node_labels="{'type': 'infra'}" openshift_schedulable=False
    origin-node01.example.com openshift_node_labels="{'type': 'infra'}"
    origin-node02.example.com openshift_node_labels="{'type': 'infra'}"


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

Licensing
=========

To make licensing easier, license headers in the source files will be
a single line reference to Unique License Identifiers as defined by
the [Linux Foundation's SPDX project](http://spdx.org/).

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

All SPDX Unique License Identifiers available `at spdx.org`_.

.. _Fedora: https://getfedora.org
.. _spdx.org: http://spdx.org/licenses
.. _OpenShift Origin: https://openshift.org
.. _OpenShift Ansible: https://github.com/openshift/openshift-ansible
.. _here:
    https://github.com/openshift/openshift-ansible/blob/master/inventory/byo/hosts.origin.example
.. _Origin Bring Your Own Host:
    https://github.com/openshift/openshift-ansible/blob/1bc6b51585c23670fdc08a1df6a89d35cd0b8149/README_origin.md
.. _OpenShift upstream advanced install guide:
    https://docs.openshift.org/latest/install_config/install/advanced_install.html#install-config-install-advanced-install
