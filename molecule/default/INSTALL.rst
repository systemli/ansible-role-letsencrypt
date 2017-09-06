*******
Install
*******

This set of playbooks have specific dependencies on Ansible due to the modules
being used.

Requirements
============

* Ansible 2.2 or higher
* Docker Engine
* docker-py

Install OS dependencies on Debian 9 (Stretch)

.. code-block:: bash

  # apt-get update
  # apt-get install -y python-pip libssl-dev python-docker
  ## If installing Molecule from source.
  # apt-get install -y libffi-dev git

Install OS dependencies on Ubuntu 16.x

.. code-block:: bash

  $ sudo apt-get update
  $ sudo apt-get install -y python-pip libssl-dev docker-engine
  # If installing Molecule from source.
  $ sudo apt-get install -y libffi-dev git

Install using pip:

.. code-block:: bash

  $ sudo pip install ansible
  $ sudo pip install docker-py
  $ sudo pip install molecule --pre
